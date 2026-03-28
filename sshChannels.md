# SSH Channels

You now have one TCP socket connected to the SSH server. One socket, one stream of bytes.

But you want to do multiple things simultaneously over that one connection:

- Run a shell
- Forward local port 8080 to a remote server
- Forward your SSH agent so the server can use your keys
- Forward your X11 display

These are four independent data streams, but TCP only gives one pipe, we can't mix all of this. SSH solves this through **channels**.

A channel is a logical, independent data stream multiplexed inside one SSH connection. You can have dozens of channels open at the same time, all flowing over the same single TCP socket.

---

## How SSH Packets Work

Every TCP packet has this structure:

```
Packet length  -> 4 bytes
Pad_length     -> 1 byte
Payload        -> variable length
Padding
```

The first byte of the payload is the **message type** -> a number that tells you what the packet means:

```
20   SSH_MSG_KEXINIT                   -> start key exchange
21   SSH_MSG_NEWKEYS                   -> switch to encrypted keys
50   SSH_MSG_USERAUTH_REQUEST          -> authentication attempt
90   SSH_MSG_CHANNEL_OPEN              -> "I want to open a channel"
91   SSH_MSG_CHANNEL_OPEN_CONFIRMATION -> "yes, channel is open"
92   SSH_MSG_CHANNEL_OPEN_FAILURE      -> "no, refused"
94   SSH_MSG_CHANNEL_DATA              -> "here is data for channel N"
96   SSH_MSG_CHANNEL_EOF               -> "I'm done sending on channel N"
97   SSH_MSG_CHANNEL_CLOSE             -> "channel N is closed"
```

Every packet that carries channel data includes a **channel number** -> a 4-byte integer that says which channel this data belongs to. That's how the receiver separates the streams.

libssh reads raw bytes off the TCP socket, decrypts them, then looks at the message type and dispatches to the right handler. We never see any of this -> but it explains why `ssh_event_dopoll()` can feed data to the right channel callback.

---

## Channel Numbers Get Tracked on Both Sides

Each side keeps a table of open channels. The client picks a number starting from 0 and sends it in `SSH_MSG_CHANNEL_OPEN`. The server picks its own number for the same channel and sends it back in `SSH_MSG_CHANNEL_OPEN_CONFIRMATION` with both numbers.

- When the client sends data, it uses the **server's** channel number as the recipient
- When the server sends data, it uses the **client's** channel number as the recipient

This is like two people using different names for the same conversation -> both sides know which channel is which because they agreed during the open handshake.

libssh handles all of this internally. We never deal with channel numbers directly. When you call `ssh_channel_new()`, libssh assigns a number. When data arrives for that channel, libssh routes it to the right `ssh_channel` object.

---

## Channel Types

```
"session"                -> a shell, command execution, or subsystem (sftp)
"direct-tcpip"           -> local port forward: "connect to host:port for me"
"forwarded-tcpip"        -> remote port forward: server initiated, going to client
"auth-agent@openssh.com" -> agent forwarding
"x11"                    -> X11 forwarding
```

---

## libssh Channel API

### Creating and Opening a Session Channel

```c
ssh_channel channel = ssh_channel_new(session);
```

This allocates an `ssh_channel` object. The channel is not open yet -> nothing has been sent to the server.

```c
int rc = ssh_channel_open_session(channel);
```

This sends `SSH_MSG_CHANNEL_OPEN` with type `"session"` and waits for confirmation. After this returns `SSH_OK`, there is an active channel between you and the server.

### Requesting a Shell or Command

After a session channel is open, you request what you want to happen on the server side:

```c
/* request a PTY (terminal) */
ssh_channel_request_pty(channel);

/* then request a shell */
ssh_channel_request_shell(channel);

/* execute a specific command instead of a shell */
ssh_channel_request_exec(channel, "ls -la /etc");
```

`request_pty` tells the server to allocate a pseudoterminal.
`request_shell` tells the server to start `/bin/bash` (or whatever the user's default shell is) attached to that PTY.

### Reading and Writing Channel Data

```c
/* write to the channel (stdin of the remote process) */
ssh_channel_write(channel, "hello\n", 6);

/* read from the channel (stdout of the remote process) */
char buf[256];
int nbytes = ssh_channel_read(channel, buf, sizeof(buf), 0);
/* last arg: 0 = stdout, 1 = stderr */
```

### Callbacks -> How Data Arrives in the Event Loop

```c
struct ssh_channel_callbacks_struct cb = {
    .channel_data_function  = on_channel_data,
    .channel_eof_function   = on_channel_eof,
    .channel_close_function = on_channel_close,
};

ssh_callbacks_init(&cb);
ssh_set_channel_callbacks(channel, &cb);
```

`on_channel_data` is the function we write:

```c
int on_channel_data(ssh_session session, ssh_channel channel,
                    void *data, uint32_t len, int is_stderr,
                    void *userdata) {
    write(STDOUT_FILENO, data, len);
    return len;  /* tell libssh how many bytes you consumed */
}
```

Every time `ssh_event_dopoll()` reads an `SSH_MSG_CHANNEL_DATA` packet for this channel, it calls `on_channel_data` with the payload. We write it to stdout. No blocking, no manual reads.

---

## Opening a `direct-tcpip` Channel (Local Port Forwarding)

When you run `ssh -L 8080:google.com:80`, what happens is: a local client connects to your port 8080, you accept them and get a `client_fd`, and now you need the server to connect to `google.com:80` on your behalf and bridge between the two.

```c
ssh_channel channel = ssh_channel_new(session);

int rc = ssh_channel_open_forward(
    channel,
    "google.com", 80,    /* destination the server should connect to */
    "127.0.0.1", 8080    /* where the originating connection came from
                            (for logging on the server side, not routing) */
);
```

Internally this sends:

```
SSH_MSG_CHANNEL_OPEN
  channel_type: "direct-tcpip"
  recipient:    google.com port 80
  originator:   127.0.0.1 port 8080
```

The server connects to `google.com:80` and sends back `SSH_MSG_CHANNEL_OPEN_CONFIRMATION`. Now the channel is an open pipe to `google.com:80`.

Bytes from `client_fd` go into the channel; bytes from the channel go back to `client_fd`. The client thinks it's talking to `google.com:80` directly; the server thinks it's proxying on behalf of `127.0.0.1:8080`.

---

## Remote Port Forwarding

When you run `ssh -R 9090:localhost:22`, you want the server to listen on port 9090 and, when someone connects, forward that connection back to your machine at `localhost:22`. This is the reverse direction -> the server initiates the channel, not us.

```c
int actual_port = 0;
ssh_channel_listen_forward(session, NULL, 9090, &actual_port);
```

This sends a global request asking the server to bind port 9090. `actual_port` gets filled with the port the server actually bound.

### Handling the Server-Initiated Channel Open

When someone connects to the server's port 9090, the server sends `SSH_MSG_CHANNEL_OPEN` with type `"forwarded-tcpip"`. We receive this as a callback:

```c
struct ssh_callbacks_struct session_cb = {
    .channel_open_request_forwarded_tcpip_function = on_remote_forward,
};

ssh_channel on_remote_forward(ssh_session session,
                               const char *dest_addr, int dest_port,
                               const char *orig_addr, int orig_port,
                               void *userdata) {
    /* connect to the actual destination */
    int local_fd = connect_to("localhost", 22);

    ssh_channel channel = ssh_channel_new(session);
    /* accept the channel */
    ssh_channel_accept_forward(session, 0, &dest_port);

    /* bridge channel ↔ local_fd */
    /* ... add local_fd and channel to event loop ... */

    return channel;
}
```

---

## How Multiple Channels Share One Event Loop

`ssh_event_dopoll()` calls `poll()` on the session socket:

```
Packet arrives: SSH_MSG_CHANNEL_DATA  channel_number=0  payload="hello"
  -> libssh looks up channel 0 -> calls on_channel_data for channel 0

Packet arrives: SSH_MSG_CHANNEL_DATA  channel_number=1  payload="GET /"
  -> libssh looks up channel 1 -> calls on_channel_data for channel 1

Packet arrives: SSH_MSG_CHANNEL_OPEN  type="auth-agent@openssh.com"
  -> libssh calls on_agent_request callback
```

We set different callbacks on each channel. libssh routes incoming packets to the right callback using the channel number. The event loop doesn't know or care how many channels are open.

---

## SSH Flag -> Channel Type Reference

| SSH Flag | Channel Type | Direction | libssh API Call |
|----------|-------------|-----------|-----------------|
| *(shell)* | `session` | client -> server | `ssh_channel_open_session()` |
| `-L` | `direct-tcpip` | client -> server | `ssh_channel_open_forward()` |
| `-R` | `forwarded-tcpip` | server -> client | `ssh_channel_listen_forward()` + callback |
| `-D` | `direct-tcpip` | client -> server | `ssh_channel_open_forward()` (dest from SOCKS5) |
| `-A` | `auth-agent` | server -> client | `ssh_channel_open_auth_agent()` via callback |
| `-X` / `-Y` | `x11` | server -> client | `ssh_channel_request_x11()` + callback |
| `-W` | `direct-tcpip` | client -> server | `ssh_channel_open_forward()` (stdio connected directly) |

