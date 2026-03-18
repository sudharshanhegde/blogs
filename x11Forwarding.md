# X11 Forwarding

## What is X11?

When we run a graphical application on Linux like a browser, a text editor, anything with a window, that application does not draw pixels directly to the screen. It will send drawing commands to a separate program called the **X server** which owns the screen and the keyboard and the mouse.

This separation of "the program that wants to draw" from "the program that controls the screen" is what the foundation of X11. The drawing commands travel over a protocol called the **X protocol** (version 11, hence X11).

The **X server** is the program that runs on our machine and owns our monitor. It is the "server" because it serves the access to the display hardware.

The **X client** is the application, like `xterm`, `gedit`, or `xclock`. It is the "client" because it connects to the X server and requests drawing services.

So "client" and "server" are the opposite of what we expect when we are thinking of them in a web context. The application is the client. The display is the server.

## How does an application know which display to use?

Every X client needs to know where the X server is. This is communicated through an environment variable called `DISPLAY`.

```sh
echo $DISPLAY
# :0
```

The format is `hostname:display_number.screen_number`.

`:0` means, connect to the X server on the local machine, display number 0, screen 0 (the screen part is optional which defaults to 0).

`localhost:10.0` means, connect to the X server running at `localhost` on display 10. This exact format `:10` or `localhost:10` is what gets set during X11 forwarding over SSH.

The X server listens in two ways depending on the hostname part of DISPLAY:
- If hostname is empty or `unix` Than it uses a Unix domain socket at `/tmp/.X11-unix/X0` (for display 0).It is Fast and local only.
- If hostname is a real hostname than it uses a TCP connection to port `6000 + display_number`. So display 10 = port 6010.

## What is xauth?

If any program could connect to our X server just by knowing the socket path, that would be a serious security problem. So a program could intercept our keystrokes or capture our screen.

The X server requires authentication before accepting connections. The most common scheme is called as **MIT-MAGIC-COOKIE-1**.

It works exactly like a secret password, but it is called a "cookie". The X server generates a random 128-bit number (16 bytes, displayed as 32 hex characters). Only clients that present this cookie in their connection handshake are allowed in.

This cookie is stored in a file, typically `~/.Xauthority`. The tool `xauth` manages this file.

```sh
xauth list
# laptop/unix:0  MIT-MAGIC-COOKIE-1  a3f2b1c4d5e6f7890a1b2c3d4e5f60718
```

This means: for the X display `laptop/unix:0`, the required cookie will be `a3f2b1c4d5e6f7090a1b2c3d4e5f60718`.

When a graphical application starts, it reads `DISPLAY`, connects to the corresponding X server socket or TCP port, and presents the cookie from `~/.Xauthority`. If the cookie matches, the X server lets it draw.

## The Problem X11 Forwarding Solves

we SSH into a remote server. we run `xclock` there — a simple graphical clock. `xclock` tries to connect to an X server. But on the remote server there is no X server. So there is no screen. The `DISPLAY` variable is not set. `xclock` will fail with an error.

X11 forwarding tunnels the X protocol over SSH so that graphical applications running on the remote server can display their windows on our local screen.

When X11 forwarding is active:

- The remote server has `DISPLAY=localhost:10` set in the SSH session
- When `xclock` tries to connect to `localhost:10` on the server, SSH intercepts that connection
- SSH tunnels the X protocol bytes back through the SSH connection to our machine
- our local X server receives the X drawing commands and displays the window

The application will run on the server. The window appears on our screen.

## Why Authentication gets complicated

Here is the problem. Our local X server requires the magic cookie to let connections in. When we do X11 forwarding, the application on the remote server will eventually connect to our local X server. That means the remote machine needs our cookie.

But if we simply sent our real cookie over SSH to the remote machine, any process on that server, including malicious ones can/could extract that cookie and use it to connect to our X server directly, bypassing SSH entirely. They could then capture keystrokes, take screenshots, inject fake input events.

This is a real attack. It was done historically. The fix is **fake cookie injection**.
+
### Trusted Forwarding (-Y)

With `-Y` (ForwardX11Trusted), we will send our real cookie to the server. The remote application gets full access to our X server, it can do anything a local application can do. This is convenient but dangerous. Only use it if we fully trust the remote machine and everyone with access to it can be trusted.

### Untrusted Forwarding (-X)

With `-X` (ForwardX11 / untrusted), SSH generates a **fake cookie** for the remote session. SSH tells the remote machine "the cookie for display localhost:10 is this fake value". When the remote application presents this fake cookie to what it thinks is an X server on `localhost:10`, SSH intercepts it. SSH verifies the fake cookie itself, then makes the actual connection to our real X server using our real cookie. The remote application never sees the real cookie.

Additionally, the X11 security extension is used to mark the forwarded connection as "untrusted", restricting what the remote application can do like for example, it cannot read the contents of windows it did not create (no keylogging from other apps), cannot grab the keyboard globally.

## How the Channel Works

When we SSH with `-X` or `-Y`, this is what happens step by step.

**Step 1 -> Client sends x11-req channel request.**

After opening a session channel for the shell, the client sends an `x11-req` channel request containing:
- `single_connection`: false (usually (allow multiple X applications))
- `auth_protocol`: "MIT-MAGIC-COOKIE-1"
- `auth_cookie`: the **fake** cookie that SSH generated randomly
- `screen_number`: 0 (usually)

This is done with `ssh_channel_request_x11()` in libssh.

**Step 2 -> Server sets DISPLAY in the shell.**

The server's sshd receives the `x11-req`. It will set up a fake X server listener on the remote machine which is typically listening on TCP `localhost:6010` (display :10, the 10 is to avoid conflict with a real display). It sets `DISPLAY=localhost:10` in the environment before starting the shell.

**Step 3 -> Application runs and tries to connect to DISPLAY.**

The user runs `xclock`. `xclock` reads `DISPLAY=localhost:10`, tries to connect to `localhost:6010` via TCP. sshd intercepts this connection on the server side.

**Step 4 -> Server opens x11 channel to client.**

sshd opens an `x11` channel back to the SSH client. The channel open message contains the originator address and port (the address that `xclock` connected from on the server).

The client's callback fires and this is `ssh_channel_open_request_x11_callback` in libssh. This callback must create and return a new channel to accept the request.

**Step 5 -> Client connects to the real local X server.**

After accepting the `x11` channel, the client needs to connect to the real local X server. It reads the real `DISPLAY` on the local machine (`:0`), connects to the Unix socket `/tmp/.X11-unix/X0`, and presents the real cookie.

Then it bridges: `x11 channel` <-> `local X server socket`, using `ssh_connector`, exactly like port forwarding.

**Step 6 — Fake cookie gets swapped for real cookie.**

This is the step that makes untrusted forwarding work. The remote application sent the fake cookie in its X handshake. As those bytes travel through the channel to the client, the client must find the fake cookie bytes at the start of the stream and replace them with the real cookie before forwarding to the real X server.

This swapping only needs to happen once, at the very start of each X connection (the first few bytes of the X protocol are the client's initial connection request which contains the cookie). After that the channel is pure transparent data.

## libssh API

### Requesting X11 forwarding (client side)

```c
int ssh_channel_request_x11(ssh_channel channel,
                             int single_connection,
                             const char *protocol,
                             const char *cookie,
                             int screen_number);
```

`channel` — the session channel (the shell channel, not a new channel)
`single_connection` — if 1, only one X application will be forwarded, then forwarding stops
`protocol` — pass NULL and libssh defaults to "MIT-MAGIC-COOKIE-1"
`cookie` — pass NULL and libssh generates a random fake cookie for we using `generate_cookie()` internally. If we pass a cookie, that value is used instead (useful for trusted -Y, where we pass our real cookie)
`screen_number` — pass 0 for the default screen

Internally `generate_cookie()` (src/channels.c) generates 16 random bytes with `ssh_get_random()` and converts them to a 32-character hex string. `ssh_get_random()` is libssh's cryptographically secure random function, it uses the underlying crypto library (OpenSSL or libgcrypt). we must not use `rand()` for this as a predictable cookie will defeat the entire security model.

The function then calls `channel_request()` which sends `SSH2_MSG_CHANNEL_REQUEST` with type `"x11-req"` and payload format `"bssd"`:
- `b` = 1 byte boolean (single_connection)
- `s` = string (protocol name)
- `s` = string (cookie)
- `d` = uint32 (screen number)

### Accepting incoming X11 channels (client side)

```c
ssh_channel ssh_channel_accept_x11(ssh_channel channel, int timeout_ms);
```

`channel` -> the original session channel (not a new one). The accepted X11 channels are associated with this session channel.
`timeout_ms` -> how long we have to wait for an incoming X11 channel open from the server.

This is polling-based. In our event loop we call it to drain any pending incoming X11 channel open requests from the server.

### Opening an X11 channel (server side — sshd implementation)

```c
int ssh_channel_open_x11(ssh_channel channel,
                          const char *orig_addr,
                          int orig_port);
```

This is for if we are writing an SSH server with libssh. The server calls this to push an X11 channel open to the client, after it has accepted a connection from the remote X application. We do not call this in our CLI as it is the server's job.

### Callback for incoming X11 channels (client side)

```c
typedef ssh_channel (*ssh_channel_open_request_x11_callback)(
    ssh_session session,
    const char *originator_address,
    int originator_port,
    void *userdata);
```

`originator_address` -> the IP address the remote application connected from on the server (usually "127.0.0.1")
`originator_port` -> the TCP port the remote app connected from

our callback creates a new channel, connects to the local X server, sets up the cookie swap (for untrusted forwarding), bridges with `ssh_connector`, and returns the channel to accept the request.

## What We Need to Implement in the CLI (Rough Idea)

### For -Y (trusted forwarding)

Step 1. Read the real cookie for our local DISPLAY using `xauth`:

```c
static int get_real_cookie(const char *display, char *proto_out, char *cookie_out) {
    char cmd[256];
    FILE *f;
    char line[512];

    snprintf(cmd, sizeof(cmd), "xauth list %s 2>/dev/null", display);
    f = popen(cmd, "r");
    if (f && fgets(line, sizeof(line), f)
         && sscanf(line, "%*s %255s %255s", proto_out, cookie_out) == 2) {
        pclose(f);
        return 0;
    }
    if (f) pclose(f);
    return -1;
}
```

`popen()` will run a shell command and returns a FILE* to read its stdout. 
`xauth list <display>` will print the authentication entries for that display. 
We parse the output to extract the protocol name and cookie value. 
`sscanf` with `%*s` skips the first word (display name) since we only want the protocol and cookie.

Step 2. Pass the real cookie to `ssh_channel_request_x11(channel, 0, proto, cookie, 0)`.

Step 3. Register a callback.
In the callback,we connect to local X server Unix socket (no cookie swap needed as the remote app has the real cookie and the local X server accepts it directly).

### For -X (untrusted forwarding) This requires cookie swap

Step 1. Generate fake cookie.
Call `ssh_channel_request_x11(channel, 0, NULL, NULL, 0)` passing NULL for both protocol and cookie lets libssh generate the fake cookie internally.

The problem is: we have to keep a copy of the fake cookie libssh generated so that we can swap it at the channel level. Unfortunately, when we pass NULL, libssh generates the cookie internally and does not give it back to us. So we need to generate our own fake cookie, pass it to `ssh_channel_request_x11()`, and keep a copy of it.

Step 2. In the callback, when an x11 channel arrives, we will read the first bytes of the X protocol handshake. 

The structure of the X11 client initial connection message is:

```
byte    order    (0x6C for little-endian, 0x42 for big-endian)
byte    pad
uint16  major-version    (always 11)
uint16  minor-version    (always 0)
uint16  len-auth-proto-name
uint16  len-auth-data
uint16  pad
string  auth-proto-name  (example : "MIT-MAGIC-COOKIE-1")
padding to 4-byte boundary
string  auth-data        (the cookie bytes -> 16 bytes for MIT-MAGIC-COOKIE-1)
```

We read the first message, find the auth-data field, check that it matches our fake cookie, replace it with the real cookie, then send the modified handshake to the real X server. After this first message, all subsequent bytes will be forwarded unchanged by the connector.

Step 3. We will bridge the channel to the real local X server using `ssh_connector`, but with a one-time read-modify-write step at the start for the cookie swap.

### Registering the callback

```c
struct ssh_callbacks_struct sess_cb = {
    .channel_open_request_x11_function = x11_open_cb,
    .userdata = &x11_state,
};
ssh_callbacks_init(&sess_cb);
ssh_set_callbacks(session, &sess_cb);

/* then request X11 forwarding on the shell channel */
ssh_channel_request_x11(shell_channel, 0, NULL, fake_cookie, 0);
```

`ssh_callbacks_init(&cb)` is a macro that sets `cb.size = sizeof(cb)`. libssh checks the size field to know which version of the callbacks struct it is dealing with we should never skip this call or callbacks will not fire.

### Connecting to the local X server

The local `DISPLAY` variable tells us how to connect:

```c
const char *display = getenv("DISPLAY");
/* display = ":0" */
```

If `DISPLAY` is `:N` or `unix:N` — connect via Unix domain socket:
```c
char path[108];
int display_num = atoi(strchr(display, ':') + 1);
snprintf(path, sizeof(path), "/tmp/.X11-unix/X%d", display_num);
/* connect AF_UNIX SOCK_STREAM to our path */
```

`/tmp/.X11-unix/X0` is the standard location for display 0. The display number after `:` maps directly to the socket file. `108` is the `sun_path` limit for Unix domain sockets it is same constraint which we saw with the SSH agent socket.

If `DISPLAY` is `hostname:N` we have to connect via TCP to port `6000 + N`:
```c
int port = 6000 + display_num;
/* connect TCP to hostname:port */
```

The `6000` offset is defined by the X protocol specification. Display 0 = port 6000, display 1 = port 6001, and so on.

### ForwardX11Timeout

OpenSSH has a `ForwardX11Timeout` option which stops accepting new X11 channels after a certain time (default 20 minutes). This is a security measure necause if an application on the server tries to connect to X after a long time, it might be a delayed attack. We need a timestamp from when the session started and stop accepting x11 channels after `ForwardX11Timeout` seconds.

## Config Directives — Current State in src/config.c

All X11-related directives are currently `SOC_NA` (silently discarded):

| Directive          | Status | What It Controls                                          |
|--------------------|--------|-----------------------------------------------------------|
| `ForwardX11`       | SOC_NA | Enable untrusted X11 forwarding (-X equivalent)           |
| `ForwardX11Trusted`| SOC_NA | Enable trusted X11 forwarding (-Y equivalent)             |
| `ForwardX11Timeout`| SOC_NA | Stop accepting new X11 channels after N seconds           |
| `XAuthLocation`    | SOC_NA | Path to the xauth binary (default will be /usr/bin/xauth)         |

To promote these we have to :
- Add `SSH_OPTIONS_FORWARD_X11`, `SSH_OPTIONS_FORWARD_X11_TRUSTED`, `SSH_OPTIONS_FORWARD_X11_TIMEOUT`,
 `SSH_OPTIONS_XAUTH_LOCATION` to `enum ssh_options_e` in `include/libssh/libssh.h`
- Add corresponding fields to `struct ssh_options_struct` in `include/libssh/session.h`
- Change `SOC_NA` -> `SOC_SESSION`, and add parsing cases in `src/config.c`
- Add cases in `src/options.c` for `ssh_options_set()`

## What We will be Solving at the Project Level

Both `ForwardX11` and `ForwardX11Trusted` are currently `SOC_NA` as they are read from `~/.ssh/config` and silently thrown away. A developer using libssh who writes `ForwardX11 yes` in their config gets no error, no warning, and no forwarding. This is the gap we will be closing.

At the CLI level: the `examples/ssh_X11_client.c` file exists as an API demonstration but it is not a production tool, it has no proper argument parsing, no integration with the rest of the session options, and is not built as part of any installable binary. Our `tools/ssh/` binary will be the first production-quality binary that exercises these X11 library paths end-to-end.

## Key Files

| File | What is Relevant |
|------|----------------|
| src/channels.c | `ssh_channel_request_x11`, `ssh_channel_accept_x11`, `ssh_channel_open_x11`, `generate_cookie` |
| include/libssh/callbacks.h | `ssh_channel_open_request_x11_callback`, `ssh_channel_x11_req_callback`|
| src/messages.c | x11-req parsing, x11 channel open parsing |
| src/config.c | All X11 directives at SOC_NA |
| examples/ssh_X11_client.c | Full X11 client example is there `x11_get_proto()`, display parsing, DISPLAY connection |
| tests/unittests/torture_server_x11.c| Unit test for x11-req, shows callback wiring |
