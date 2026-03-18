# Port Forwarding

SSH port forwarding is also called as tunnelling which lets us route arbitrary TCP connections through an encrypted SSH channel. There are three kinds, each of them solving a different connectivity problem.

---

## The Three Types — Mental Model First

```
LOCAL  (-L)  :  our machine listens  →  traffic goes through SSH  →  server connects to target
REMOTE (-R)  :  server listens        →  traffic comes through SSH  →  our machine connects to target
DYNAMIC (-D) :  our machine listens  →  SOCKS5 proxy over SSH      →  server connects to wherever SOCKS5 says
```

---

## 1. Local Port Forwarding (-L)

### The Problem It Solves

We are on our laptop. There is a database on `db.internal:5432` which is only reachable from the server `ssh.company.com`. We cannot reach `db.internal` directly.

```
Laptop  -> SSH ->  ssh.company.com  -> TCP ->  db.internal:5432
```

`-L 5432:db.internal:5432` makes our laptop's port 5432 an entrance to that tunnel.

```sh
ssh -L 5432:db.internal:5432 user@ssh.company.com
```

Now `psql -h localhost -p 5432` on our laptop connects through the server to the database.

### What Happens at the OS Level

```
Laptop                             SSH Server
──────                             ──────────
1. ->ssh opens listening socket
   ->bind(localhost:5432)
   ->listen()

2. ->psql connects to localhost:5432
   → accept() on the listener end
   → new connection fd

3. ssh calls ssh_channel_new()
   ssh_channel_open_forward(
     "db.internal", 5432,
     "localhost", 5432)
   → sends SSH2_MSG_CHANNEL_OPEN
     type="direct-tcpip"           ->  server receives open request
                                        server does connect("db.internal",5432)
                                        sends SSH2_MSG_CHANNEL_OPEN_CONFIRMATION

4. Bidirectional bridge:
   connection_fd <-> ssh_channel   <- SSH channel data ->  db socket
```

### SSH Protocol (direct-tcpip)

The SSH message for local forwarding is `SSH2_MSG_CHANNEL_OPEN` with type string `"direct-tcpip"`.

Payload format (src/channels.c):
```
string   remotehost       ("db.internal")
uint32   remoteport       (5432)
string   sourcehost       ("localhost")   -- for logging purpose
uint32   sourceport       (5432)          -- for logging purpose
```

The server sees this,it does `connect(remotehost, remoteport)`, and if successful replies with `SSH2_MSG_CHANNEL_OPEN_CONFIRMATION`. The channel is now open and data flows.

### libssh API for it : 

```c
/* Step 1: open the channel */
ssh_channel ch = ssh_channel_new(session);
rc = ssh_channel_open_forward(ch,
        "db.internal", 5432,   /* where the server should be connecting */
        "localhost",    5432); /* our address, for logging purpose*/
if (rc != SSH_OK) {
    fprintf(stderr, "forward failed: %s\n", ssh_get_error(session));
}

/* Step 2: bridge accepted connection fd <-> channel */
ssh_connector fwd = ssh_connector_new(session);
ssh_connector_set_in_fd(fwd, connection_fd);       /* reads from local socket */
ssh_connector_set_out_channel(fwd, ch, SSH_CONNECTOR_STDOUT);
ssh_connector fwd2 = ssh_connector_new(session);
ssh_connector_set_in_channel(fwd2, ch, SSH_CONNECTOR_STDOUT);
ssh_connector_set_out_fd(fwd2, connection_fd);     /* writes to local socket */

ssh_event ev = ssh_event_new();
ssh_event_add_session(ev, session);
ssh_event_add_connector(ev, fwd);
ssh_event_add_connector(ev, fwd2);

while (!done) {
    ssh_event_dopoll(ev, -1);
}
```

`ssh_connector` (src/connector.c) is the libssh's data bridge: it registers poll callbacks so that when data appears on one side it writes it to the other side automatically. We do not need to loop calling `ssh_channel_read` / `ssh_channel_write` ourself. It uses `ssh_poll_handle` internally and integrates with `ssh_event`.

### The Listener Loop for -L

The listening socket itself is our responsibility in the CLI. Pattern is :

```c
/* This is one-time setup */
int listen_fd = socket(AF_INET, SOCK_STREAM, 0);
setsockopt(listen_fd, SOL_SOCKET, SO_REUSEADDR, &(int){1}, sizeof(int));
bind(listen_fd, ...);
listen(listen_fd, 128);

/* add listen_fd to event loop with accept callback */
ssh_event_add_fd(ev, listen_fd, POLLIN, accept_callback, userdata);

static int accept_callback(socket_t fd, int revents, void *userdata) {
    int conn_fd = accept(fd, NULL, NULL);
    /* opens a direct-tcpip channel for this connection */
    /* bridges conn_fd to new channel via ssh_connector */
    return 0; /* keeps on listening */
}
```

`ssh_event_add_fd` (src/poll.c) registers any file descriptor with the event loop. Its callback fires when `POLLIN`/`POLLOUT` events arrive.

Multiple connections can exist simultaneously with each accept creates its own channel and its own pair of connectors.

---

## 2. Remote Port Forwarding (-R)

### The Problem It Solves

We are running a web server on our laptop at port 8080. The laptop is behind NAT so nobody on the internet can reach it directly. The server `ssh.company.com` has a public IP.

```sh
ssh -R 9090:localhost:8080 user@ssh.company.com
```

Now anyone on the internet hitting `ssh.company.com:9090` gets connected to our laptop's port 8080.

```
Internet user  ->TCP ->  ssh.company.com:9090  -> SSH ->  Laptop -> TCP ->  localhost:8080
```

### SSH Protocol — Two-Step

Remote forwarding is fundamentally different because the *server* must open channels back *to the client*. This requires a two-step protocol:

**Step 1: Global Request, ask server to start listening**

```
client (sends) →  SSH2_MSG_GLOBAL_REQUEST
           request-name = "tcpip-forward"
           want-reply   = 1 (true)
           string  address-to-bind  ("" means all interfaces, "localhost" means loopback)
           uint32  port-to-bind     (9090, or 0 to let server pick a free port)
```

Server replies:
```
SSH2_MSG_REQUEST_SUCCESS
uint32  bound_port   (only present if we requested port 0)
```

**Step 2: Server pushes forwarded-tcpip channels**

When a connection arrives at `ssh.company.com:9090`, the server sends:

```
server (sends) →  SSH2_MSG_CHANNEL_OPEN
           channel-type = "forwarded-tcpip"
           string  destination address  ("ssh.company.com")
           uint32  destination port     (9090)
           string  originator address   (IP of internet user)
           uint32  originator port      (ephemeral port)
```

The client receives this, opens a TCP connection to `localhost:8080`, and bridges the channel to it.

### libssh API to do so is :

```c
/* Step 1: ask the server to listen */
int bound_port;
rc = ssh_channel_listen_forward(session,
        "",     /* bind address: empty = all interfaces */
        9090,   /* port: 0 = dynamic */
        &bound_port);
/* bound_port is populated only when We passed port=0 */

/* Step 2: event loop to accept incoming forwarded-tcpip channels */
/* Non-blocking poll: ssh_channel_open_forward_port() */
int dst_port;
char *originator = NULL;
int orig_port;
ssh_channel ch = ssh_channel_open_forward_port(session, 100/*ms*/,
        &dst_port, &originator, &orig_port);
if (ch != NULL) {
    /* connect to local target */
    int local_fd = connect_to("localhost", 8080);
    /* bridge ch to local_fd using ssh_connector */
}

/* To stop: */
ssh_channel_cancel_forward(session, "", 9090);
```

`ssh_channel_listen_forward` (src/channels.c) sends the global request.
`ssh_channel_open_forward_port` (src/channels.c) accepts one pending forwarded-tcpip channel; returns NULL if none ready yet.
`ssh_channel_cancel_forward` (src/channels.c) sends `cancel-tcpip-forward`.

### GatewayPorts

By default OpenSSH binds remote forwarding only to loopback (`127.0.0.1`), even if We specify `""`. To bind to all interfaces and let external hosts connect, the server must have `GatewayPorts yes` in its `sshd_config`. This is a *server-side* setting, not a client option. The client can hint at the bind address in the global request string, but the server enforces GatewayPorts policy.

### ExitOnForwardFailure

If `-R` setup fails (server refuses the global request), by default OpenSSH continues the session anyway. With `ExitOnForwardFailure yes`, a forward failure is fatal. We will check `rc` from `ssh_channel_listen_forward()` and call `exit(1)` if that option is set.

---

## 3. Dynamic Port Forwarding (-D) — SOCKS5

### The Problem It Solves

We want to route our *entire browser* through the server, not just one port. `-L` would need one rule per destination. `-D` makes ssh a SOCKS5 proxy.

```sh
ssh -D 1080 user@ssh.company.com
```

Set browser's SOCKS5 proxy to `localhost:1080`. Every browser connection goes:

```
Browser  -> SOCKS5 ->  localhost:1080  -> SSH direct-tcpip ->  server  -> TCP ->  any destination
```

`-D` is just `-L` with dynamic destinations. Instead of `ssh_channel_open_forward` with a hardcoded destination, We read the destination from the SOCKS5 handshake that the connecting application sends.

### SOCKS5 Protocol

**Important finding**: libssh has **zero SOCKS5 code**. We have to implement the SOCKS5 state machine entirely in our CLI.

SOCKS5 is defined in [RFC 1928](https://tools.ietf.org/html/rfc1928). Every connection goes through:

```
Client → Proxy:  Version + Method Selection
ver -> 5
NMETHODS -> 1
METHODS -> 1 to 255

VER = 5 ( means SOCKS5)
METHODS: 0x00 = no auth, 0x02 = username/password

Proxy → Client:  Chosen method
ver = 5, method = 0 (no auth accepted)



Client → Proxy:  Connection Request
version = 5
CMD:  0x01 = CONNECT (only type we will be handling for SSH -D)
RSV = 0x00
ATYP: 0x01 = IPv4 (4 bytes), 0x03 = domain (1 byte len + N bytes), 0x04 = IPv6 (16 bytes)
DST.PORT: big-endian uint16
DST.PORT : 2


Proxy → Client:  Reply
version = 5
REP: 0x00 = succeeded, 0x01 = general failure, 0x04 = host unreachable, 0x05 = connection refused
RSV = 0x00
ATYP = 1
BND.ADDR = 0.0.0.0
BND.PORT = 0
```

### SOCKS5 State Machine We Must Implement

```c
enum socks5_state {
    S5_WAIT_GREETING,      /* waiting for client version+methods */
    S5_WAIT_REQUEST,       /* sent method choice, waiting for CONNECT request */
    S5_CONNECTING,         /* sent CONNECT to SSH, waiting for channel open */
    S5_FORWARDING,         /* channel open, bridging data */
};
```

Per-connection state:

```c
struct socks5_conn {
    int             fd;           /* accepted client socket */
    enum socks5_state state;
    char            dst_host[256];
    uint16_t        dst_port;
    ssh_channel     channel;
    ssh_connector   c_in, c_out;
};
```

State machine transitions in our accept/read callback:

```
S5_WAIT_GREETING:
    read: VER(must be 5), NMETHODS, METHODS[]
    if VER != 5 → close
    send: \x05\x00  (version 5, no auth)
    → S5_WAIT_REQUEST

S5_WAIT_REQUEST:
    read: VER, CMD, RSV, ATYP, DST.ADDR, DST.PORT
    if CMD != 0x01 → send failure reply, close

    parse DST.ADDR based on ATYP:
        0x01: read 4 bytes → inet_ntop → dst_host
        0x03: read 1 byte length, then N bytes → dst_host (domain name)
        0x04: read 16 bytes → inet_ntop(AF_INET6) → dst_host

    read 2 bytes big-endian → dst_port
    open ssh_channel_new() + ssh_channel_open_forward(dst_host, dst_port, ...)

    if channel open fails:
        send \x05\x05\x00\x01\x00\x00\x00\x00\x00\x00  (connection refused)
        close

    send success reply:
        \x05\x00\x00\x01\x00\x00\x00\x00\x00\x00
    → S5_FORWARDING

S5_FORWARDING:
    set up ssh_connector pair: fd to channel
    data flows automatically through event loop
```

### Full -D Implementation Sketch/ Rough plan 

```c
/* startup: bind the SOCKS5 listener */
int socks_fd = socket(AF_INET, SOCK_STREAM, 0);
setsockopt(socks_fd, SOL_SOCKET, SO_REUSEADDR, &(int){1}, sizeof(int));
/* bind to 127.0.0.1:1080 (or opts->dynamic_port) */
listen(socks_fd, 128);
ssh_event_add_fd(ev, socks_fd, POLLIN, socks_accept_cb, &socks_state);

/* accept callback it fires when browser connects */
static int socks_accept_cb(socket_t fd, int revents, void *userdata) {
    int conn = accept(fd, NULL, NULL);
    struct socks5_conn *c = calloc(1, sizeof(*c));
    c->fd = conn;
    c->state = S5_WAIT_GREETING;
    /* register connection with event loop for readable events */
    ssh_event_add_fd(ev, conn, POLLIN, socks_data_cb, c);
    return 0;
}

/* data callback — runs state machine */
static int socks_data_cb(socket_t fd, int revents, void *userdata) {
    struct socks5_conn *c = userdata;
    /* run state machine transitions as mentioned above */
    /* when reaching S5_FORWARDING: */
    /*   remove the fd from event loop raw handler */
    /*   set up ssh_connector pair instead */
}
```

---

## 4. ssh_connector — How Data Bridging Works

Every forwarding type eventually needs to bridge a file descriptor (local socket) with an SSH channel. `ssh_connector` will handles this.

### Connector Internals (src/connector.c)

```c
struct ssh_connector_struct {
    ssh_session session;

    ssh_channel in_channel;   /* read data from this channel */
    ssh_channel out_channel;  /* write data to this channel */

    socket_t in_fd;           /* read data from this fd */
    socket_t out_fd;          /* write data to this fd */

    bool fd_is_socket;        /* affects polling for writes */

    ssh_poll_handle in_poll;  /* poll handle for in_fd */
    ssh_poll_handle out_poll; /* poll handle for out_fd */

    ssh_event event;          /* the event loop we're attached to */

    int in_available;         /* data ready to read */
    int out_wontblock;        /* can write without blocking */

    struct ssh_channel_callbacks_struct in_channel_cb;
    struct ssh_channel_callbacks_struct out_channel_cb;
};
```

### Two Connectors Per Forwarded Connection

One connector handles one direction. For a full-duplex bridge We need two:

```
local_fd -> in_fd -> connector_A -> out_channel-> ssh_channel
ssh_channel -> in_channel -> connector_B -> out_fd -> local_fd
```

```c
ssh_connector c_in  = ssh_connector_new(session);
ssh_connector c_out = ssh_connector_new(session);

ssh_connector_set_in_fd(c_in, local_fd);
ssh_connector_set_out_channel(c_in, ch, SSH_CONNECTOR_STDOUT);

ssh_connector_set_in_channel(c_out, ch, SSH_CONNECTOR_STDOUT);
ssh_connector_set_out_fd(c_out, local_fd);

ssh_event_add_connector(ev, c_in);
ssh_event_add_connector(ev, c_out);
```

When the channel or fd closes:
- The connector detects EOF via `channel_eof_function` or fd read returning 0
- It propagates EOF to the other side
- We should free connectors and close the fd in response

### SSH_CONNECTOR_STDOUT vs SSH_CONNECTOR_BOTH

For forwarded TCP connections we use `SSH_CONNECTOR_STDOUT`. The `SSH_CONNECTOR_STDERR` flag is for when We also want to pipe the SSH extended data (stderr) stream. For port forwarding there is no stderr channel, only the main data stream.

---

## 5. ssh_event — The Event Loop

All three forwarding modes share one event loop. The event loop multiplexes:

- The SSH session socket (keepalive, control messages, new channel requests)
- All local listening sockets (accept new connections)
- All per-connection sockets (SOCKS5 state machine, or bridged via connector)

```c
ssh_event ev = ssh_event_new();
ssh_event_add_session(ev, session);  /* session's socket joins the loop */

/* -L: add listener socket */
ssh_event_add_fd(ev, listen_fd, POLLIN, accept_cb, userdata);

/* -D: same as above but different accept_cb */

/* -R: ssh_channel_open_forward_port() is called inside the poll loop */
/* after ssh_event_dopoll returns, check for pending forwarded-tcpip channels */

while (!session_done) {
    rc = ssh_event_dopoll(ev, timeout_ms);
    if (rc == SSH_ERROR) break;

    /* check for -R incoming channels here if needed */
    /* check keepalive timer here */
    /* check SIGWINCH flag here */
}
```

`ssh_event_dopoll` (src/poll.c) blocks up to `timeout_ms` milliseconds, dispatches all the ready events, and returns. Set `timeout_ms = -1` to block indefinitely, or a value in ms to allow periodic tasks (keepalive checks, SIGWINCH, etc.).

`ssh_event_add_session` (src/poll.c) moves the session's internal poll handles from the default context into our event. Without this, the session socket would be polled separately and We would need two poll calls.

---

## 6. Forwarding Spec Parsing ([bind_addr:]port:host:hostport)

OpenSSH's `-L` and `-R` accept these formats:

```
-L [bind_address:]port:host:hostport
-L [bind_address:]port:remote_socket
-R [bind_address:]port:host:hostport
-D [bind_address:]port
```

We have write a parser for this. The general form `[bind_addr:]port:host:hostport` needs careful handling because IPv6 addresses contain colons:

```c
struct forward_spec {
    char bind_addr[256];  /* default: "127.0.0.1" */
    int  bind_port;
    char host[256];       /* remote host (empty for -D) */
    int  host_port;       /* remote port (0 for -D) */
};

/* example: "8080:db.internal:5432"           → bind=127.0.0.1, port=8080, host=db.internal:5432 */
/* example: "0.0.0.0:8080:db.internal:5432"   → bind=0.0.0.0,   port=8080, host=db.internal:5432 */
/* example: "[::1]:8080:db.internal:5432"     → bind=::1,        port=8080, host=db.internal:5432 */
```

The parser counts colons and handles IPv6 brackets. This is the same parser needed when we promote `LocalForward`/`RemoteForward` directives in `src/config.c` from `SOC_NA` to real options.

---

## 7. Config Directives — Current State

In `src/config.c` all forwarding directives are currently `SOC_NA` (silently discarded):

| Directive           | Current State | What It Controls                             |
|---------------------|---------------|----------------------------------------------|
| `LocalForward`      | SOC_NA        | -L equivalent in config file                 |
| `RemoteForward`     | SOC_NA        | -R equivalent in config file                 |
| `DynamicForward`    | SOC_NA        | -D equivalent in config file                 |
| `GatewayPorts`      | SOC_NA        | Allow external hosts to use -R listeners     |
| `ExitOnForwardFailure` | SOC_NA     | Fatal if any forward fails                   |
| `ClearAllForwardings` | SOC_NA      | Cancel all forwarding specs on the connection|

Promoting these to `SSH_OPTIONS_*` constants will require changes to:
- `include/libssh/libssh.h` — adding enum values to `enum ssh_options_e`
- `include/libssh/session.h` — adding fields to `struct ssh_session_struct`
- `src/config.c` — change `SOC_NA` → `SOC_SESSION`, add case to `ssh_config_parse_global()`
- `src/options.c` — add case to `ssh_options_set()` and `ssh_options_get()`

For forwarding specs (`LocalForward`/`RemoteForward`/`DynamicForward`) we also need a linked list of `struct forward_spec` attached to the session, since a config file can have multiple `LocalForward` lines.

---

## 8. What We Need to Implement in our CLI

### Argument Parsing

```
-L [bind_addr:]port:host:hostport   → call parse_forward_spec(), store in list
-R [bind_addr:]port:host:hostport   → same, different list
-D [bind_addr:]port                 → store bind_addr + port
```

### Local Forwarding (-L) Setup (Rough Idea)

For each `-L` spec:
1. Create listener socket, bind, listen
2. Register with `ssh_event_add_fd(ev, listen_fd, POLLIN, l_accept_cb, spec)`
3. In `l_accept_cb`: accept → `ssh_channel_new` → `ssh_channel_open_forward` → create two connectors → `ssh_event_add_connector`
4. Handle `ExitOnForwardFailure`: if `ssh_channel_open_forward` fails, exit(1) if the option is set

### Remote Forwarding (-R) Setup (Rough Idea)

For each `-R` spec:
1. Call `ssh_channel_listen_forward(session, bind_addr, port, &bound_port)`
2. If it fails and `ExitOnForwardFailure` is set, exit(1)
3. In the event loop, after `ssh_event_dopoll`, call `ssh_channel_open_forward_port` with timeout=0 to drain pending incoming channels
4. For each accepted channel: `connect(localhost, local_port)` → two connectors

### Dynamic Forwarding (-D) Setup (Rough Idea)

For each `-D` spec:
1. Create SOCKS5 listener socket
2. `ssh_event_add_fd(ev, socks_fd, POLLIN, socks_accept_cb, ...)`
3. In `socks_accept_cb`: accept → allocate `socks5_conn` → `ssh_event_add_fd(ev, conn_fd, POLLIN, socks_data_cb, conn)`
4. In `socks_data_cb`: run SOCKS5 state machine; when `S5_FORWARDING`: `ssh_channel_open_forward(dst_host, dst_port, ...)` → connectors; remove raw fd poll handler

### ClearAllForwardings 

If `ClearAllForwardings yes` is seen in config: ignore all LocalForward/RemoteForward/DynamicForward specs parsed before it in the same config block. This matches OpenSSH behaviour where `-o ClearAllForwardings=yes` in a Match block cancels earlier directives.

---

## Key Files Reference (with functions they have)

| File | Relevance |
|------|-----------|
| src/channels.c| `ssh_channel_open_forward`, `ssh_channel_listen_forward`, `ssh_channel_open_forward_port`, `ssh_channel_cancel_forward` |
| src/connector.c| `ssh_connector_new/set_in_fd/set_out_fd/set_in_channel/set_out_channel` |
| src/poll.c| `ssh_event_new`, `ssh_event_add_fd`, `ssh_event_add_session`, `ssh_event_add_connector`, `ssh_event_dopoll` |
| src/config.c| Forwarding directives currently at SOC_NA |
| src/options.c | `ssh_options_set/get` — where we add new SSH_OPTIONS_* cases |
| include/libssh/libssh.h | `enum ssh_options_e`, `enum ssh_channel_type_e`, `enum ssh_connector_flags_e` |
| examples/sshnetcat.c | Minimal direct-tcpip client example |
| examples/sshd_direct-tcpip.c| Server-side direct-tcpip with event loop pattern |
| doc/forwarding.dox| Existing forwarding API docs |
| tests/client/torture_forward.c | Integration tests for -R forwarding |
