# Agent Forwarding

## What Is an SSH Agent?

When we use SSH with public key authentication, our private key must sign a challenge sent by the server. Normally we would have to decrypt the private key from disk every time (entering our passphrase). The SSH agent solves this: it is a background process that holds our decrypted private keys in memory. When SSH needs a signature it asks the agent, never touching the key file itself.

When SSH needs to authenticate, the client sends a sign request to the agent. The agent signs the data using the private key it holds in memory and hands the signature back to the client. The client then presents that signature to the server, which verifies it against the stored public key and grants access.

The agent is accessed through a Unix domain socket whose path is exported as `$SSH_AUTH_SOCK`:

```sh
echo $SSH_AUTH_SOCK
# /run/user/1000/ssh-agent.socket
```

---

## What Is Agent Forwarding?

We are on our laptop, SSH into `server-A`. From `server-A` I want to SSH into `server-B`. `server-B` needs my private key to authenticate me. But my private key is on my laptop, not on `server-A`. Copying the key to `server-A` is a security risk.

Agent forwarding solves this by tunnelling the agent protocol through the SSH connection.

We SSH from our laptop into `server-A` with `-A` (agent forwarding enabled). Our laptop's `ssh-agent` holds the private key. The first SSH tunnel is now open.

From the shell on `server-A`, we run `ssh server-B`. A second tunnel is established. `server-B` sends a challenge that requires a signature from our private key -> but `server-A` does not have that key.

Because agent forwarding is active, `server-A`'s SSH client does not give up. Instead, it opens a special channel called `auth-agent@openssh.com` back through the first tunnel to our laptop. Our laptop's SSH client receives this request and connects the channel to our local `ssh-agent` via `$SSH_AUTH_SOCK`.

Now the sign request travels all the way back: from `server-B` through the second tunnel to `server-A`, then through the first tunnel to our laptop, and finally into the agent over its Unix socket. The agent signs with the private key and sends the signature back along the same path in reverse. `server-B` receives a valid signature and lets us in, and our private key never left the laptop.

---

## 1. The Agent Wire Protocol

The agent communicates over a Unix socket using a simple length-prefixed binary protocol. Every message is:

```
[uint32 length][uint8 type][... payload ...]
```

Key message types defined in [include/libssh/agent.h]:

| Constant                          | Value | Direction       | Meaning                              |
|-----------------------------------|-------|-----------------|--------------------------------------|
| `SSH2_AGENTC_REQUEST_IDENTITIES`  | 11    | client → agent  | Give me all our public keys          |
| `SSH2_AGENT_IDENTITIES_ANSWER`    | 12    | agent → client  | Here are N keys                      |
| `SSH2_AGENTC_SIGN_REQUEST`        | 13    | client → agent  | Sign this data with that key         |
| `SSH2_AGENT_SIGN_RESPONSE`        | 14    | agent → client  | Here is the signature                |
| `SSH_AGENT_FAILURE`               | 5     | agent → client  | Request failed                       |
| `SSH_AGENT_SUCCESS`               | 6     | agent → client  | Request succeeded                    |

### How the Request Identities Flows

```
client →  [0x00 0x00 0x00 0x01]  [0x0B]
           ^--- length = 1         ^--- SSH2_AGENTC_REQUEST_IDENTITIES

agent  →  [length]  [0x0C]  [count:uint32]
                     ^--- SSH2_AGENT_IDENTITIES_ANSWER

           then for each key:
             [pubkey_blob:string]  [comment:string]
```

### How the Sign Request Flows

```
client →  [length]  [0x0D]  [pubkey_blob:string]  [data:string]  [flags:uint32]
                     ^--- SSH2_AGENTC_SIGN_REQUEST

           flags: SSH_AGENT_RSA_SHA2_256 (0x02) or SSH_AGENT_RSA_SHA2_512 (0x04)
           for RSA SHA2 compliance (RFC 8332)

agent  →  [length]  [0x0E]  [signature_blob:string]
                     ^--- SSH2_AGENT_SIGN_RESPONSE
```

libssh implements this in `agent_talk()` (src/agent.c):
1. Send 4-byte big-endian length
2. Send request payload
3. Read 4-byte big-endian length
4. Read response payload

---

## 2. How libssh Connects to the Agent

`agent_connect()` (src/agent.c) does is as follows :

```c
static int agent_connect(ssh_session session)
{
    /* if using a forwarded channel, no socket needed */
    if (session->agent->channel != NULL) {
        return 0;
    }

    /* agent_socket option takes priority over $SSH_AUTH_SOCK */
    const char *auth_sock = session->opts.agent_socket
                          ? session->opts.agent_socket
                          : getenv("SSH_AUTH_SOCK");

    if (auth_sock && *auth_sock) {
        return ssh_socket_unix(session->agent->sock, auth_sock);
    }
    return -1;
}
```

`ssh_socket_unix()` (src/socket.c) does:

```c
int ssh_socket_unix(ssh_socket s, const char *path) {
    struct sockaddr_un sunaddr;
    sunaddr.sun_family = AF_UNIX;
    snprintf(sunaddr.sun_path, sizeof(sunaddr.sun_path), "%s", path);

    fd = socket(AF_UNIX, SOCK_STREAM, 0);
    fcntl(fd, F_SETFD, FD_CLOEXEC);
    connect(fd, (struct sockaddr *)&sunaddr, sizeof(sunaddr));
    ssh_socket_set_fd(s, fd);
}
```

**Unix domain socket path constraint**: `sun_path` is 108 bytes max (including the null terminator). This is  hard OS limit. If `$SSH_AUTH_SOCK` is longer, the connection will fail silently. This is why some systems put agent sockets in `/tmp/ssh-XXX/agent.NNN` -> short paths.

### Agent Struct

```c
struct ssh_agent_struct {
    struct ssh_socket_struct *sock;   /* Unix socket to local agent */
    ssh_buffer ident;                 /* buffered identities from last REQUEST_IDENTITIES */
    unsigned int count;               /* number of remaining identities to iterate */
    ssh_channel channel;              /* if non-NULL: use this channel instead of sock */
};
```

The `channel` field is key to forwarding -> it makes libssh's agent code work identically over a Unix socket or an SSH channel.

### atomicio -> The Dual-Mode Transport

`atomicio()` (src/agent.c) abstracts the transport:

```c
static uint32_t atomicio(struct ssh_agent_struct *agent,
                         void *buf, uint32_t n, int do_read)
{
    if (agent->channel == NULL) {
        /* direct socket mode */
        fd = ssh_socket_get_fd(agent->sock);
        while (pos < n) {
            res = do_read ? recv(fd, buf+pos, n-pos, 0)
                          : send(fd, buf+pos, n-pos, 0);
            /* handle EINTR, EAGAIN */
        }
    } else {
        /* forwarded channel mode */
        while (pos < n) {
            res = do_read ? ssh_channel_read(agent->channel, buf+pos, n-pos, 0)
                          : ssh_channel_write(agent->channel, buf+pos, n-pos);
        }
    }
}
```

Everything else in `src/agent.c`, `ssh_agent_get_ident_count`, `ssh_agent_get_next_ident`, `ssh_agent_sign_data` -> calls `atomicio`. So the same code path works whether the agent is local or forwarded through an SSH channel.

---

## 3. Using the Agent for Authentication

`ssh_userauth_agent()` (src/auth.c) implements the full loop:

```
1. ssh_agent_get_first_ident()   → get first public key from agent
2. ssh_userauth_try_publickey()  → ask server: "would you accept this key?"
   (sends pubkey with no signature -> a probe)
3. If server says yes:
   ssh_userauth_agent_publickey()
     → build auth request
     → ssh_agent_sign_data()    → agent signs the session token with private key
     → send signed request
     → wait for SSH_MSG_USERAUTH_SUCCESS
4. If server says no: ssh_agent_get_next_ident() → repeats with next key
5. If all keys exhausted: returns SSH_AUTH_DENIED
```

After each `ssh_agent_get_next_ident()` call, it also tries the corresponding certificate (if the key has one) -> this is the `SSH_AGENT_STATE_CERT` state.

libssh sets `SSH_AGENT_RSA_SHA2_256` or `SSH_AGENT_RSA_SHA2_512` flags in the sign request based on what server extensions were negotiated during key exchange. This ensures RSA keys use SHA2 as required by RFC 8332, not the deprecated SHA1.

---

## 4. Agent Forwarding at the SSH Channel Level

This is the mechanism that lets a remote server use our local agent.

### Step 1 -> Client Requests Forwarding

After opening a session channel for an interactive shell, the client sends a channel request:

```c
ssh_channel_request_auth_agent(channel);
/* sends SSH_MSG_CHANNEL_REQUEST with request type "auth-agent-req@openssh.com" */
```

This is a *channel* request (not the global request). It tells the server: "when something on our side needs to use an SSH agent, open a channel back to me."

Implementation is in (src/channels.c):
```c
int ssh_channel_request_auth_agent(ssh_channel channel) {
    return channel_request(channel, "auth-agent-req@openssh.com", NULL, 0);
}
```

### Step 2 -> Server Sets Up Fake $SSH_AUTH_SOCK

When the server's sshd receives `auth-agent-req@openssh.com`, it:
1. Creates a Unix domain socket at a temp path like `/tmp/ssh-XXX/agent.NNN`
2. Exports `SSH_AUTH_SOCK=/tmp/ssh-XXX/agent.NNN` to the shell it will start
3. Starts listening on that socket

### Step 3 -> Remote Program Contacts Fake Agent

When the user runs `ssh server-B` from the remote shell, OpenSSH on the server:
1. Reads `$SSH_AUTH_SOCK`
2. Connects to the fake Unix socket
3. Sends `SSH2_AGENTC_REQUEST_IDENTITIES` over it

### Step 4 -> Server Opens auth-agent Channel to Client

The server's sshd forwards each agent protocol message by opening an `auth-agent@openssh.com` channel back to the original client:

```
SSH2_MSG_CHANNEL_OPEN
channel-type = "auth-agent@openssh.com"
(no payload)
```

The client receives this open request. The client's callback is invoked:

```c
/* this code is in include/libssh/callbacks.h */
typedef ssh_channel (*ssh_channel_open_request_auth_agent_callback)(
    ssh_session session,
    void *userdata);
```

This callback must create a new channel and return it. That channel becomes the pipe through which agent protocol messages flow.

### Step 5 -> Bridging Channel to Local Agent

In the callback, the client:
1. Opens a new channel for the agent request
2. Bridges that channel to the local Unix agent socket

```c
static ssh_channel agent_channel_cb(ssh_session session, void *userdata) {
    ssh_channel ch = ssh_channel_new(session);
    /* accepts the open request */
    return ch;
}
```

Then after accepting, the client uses `ssh_set_agent_channel()` on a nested session:
```c
ssh_set_agent_channel(nested_session, ch);
/* now nested_session->agent->channel = ch */
/* ssh_agent_sign_data() will talk through ch instead of a Unix socket */
```

Or for a simpler approach: bridge the channel directly to the local `$SSH_AUTH_SOCK` unix socket using `ssh_connector`, the same way local port forwarding works.

### Step 6 -> Messages Flow Back

The remote `ssh` process sends an agent protocol message into the fake `$SSH_AUTH_SOCK`. The server's `sshd` picks it up and forwards it over to `auth-agent@openssh.com` channel back to our client. Our client reads it off the channel and writes it into the local agent's Unix socket. The agent signs and replies, and the response travels the same path in reverse back to `server-B`.

The entire agent protocol travels through the SSH channel transparently.

---

## 5. Opening auth-agent Channel (Client Side)

If we are the one wanting to *use* a forwarded agent (not provide one), we open an `auth-agent@openssh.com` channel ourself:

```c
int ssh_channel_open_auth_agent(ssh_channel channel) {
    return channel_open(channel, "auth-agent@openssh.com",
                        WINDOW_DEFAULT, CHANNEL_MAX_PACKET, NULL);
}
```

Then pass that channel to `ssh_set_agent_channel()`:

```c
ssh_channel agent_ch = ssh_channel_new(session_B);
ssh_channel_open_auth_agent(agent_ch);
ssh_set_agent_channel(session_B, agent_ch);
/* session_B's auth can now use forwarded agent */
```

---

## 6. Full Picture -> Our CLI (-A flag)

With `-A` (ForwardAgent), here is what our `tools/ssh/` binary must do:

```
1. We need to Connect + authenticate to server (using local agent for auth)

2. Open session channel
   ssh_channel_open_session(ch)

3. Send agent-forwarding request
   ssh_channel_request_auth_agent(ch)

4. Register callback on session for incoming auth-agent channels:
   struct ssh_callbacks_struct cb = {
       .channel_open_request_auth_agent_function = our_agent_callback,
   };
   ssh_callbacks_init(&cb);
   ssh_set_callbacks(session, &cb);

5. In our_agent_callback(session, userdata):
   - Create a new channel: agent_ch = ssh_channel_new(session)
   - Return agent_ch to accept the open request

6. After returning agent_ch, set up bridge:
   - Connect to local $SSH_AUTH_SOCK: sock_fd = connect_unix(getenv("SSH_AUTH_SOCK"))
   - Bridge sock_fd ↔ agent_ch using ssh_connector pair
   - Add connectors to event loop

7. Event loop handles everything:
   - Remote program sends agent protocol → comes through agent_ch → forwarded to sock_fd → local agent
   - Local agent replies → comes through sock_fd → forwarded to agent_ch → back to remote
```

### The Callback and Bridge Code Sketch should look something like this

```c
struct agent_fwd_state {
    ssh_session session;
    ssh_event   ev;
};

static ssh_channel agent_open_cb(ssh_session session, void *userdata) {
    struct agent_fwd_state *st = userdata;

    ssh_channel ch = ssh_channel_new(session);
    if (ch == NULL) return NULL;

    /* connect to local agent */
    const char *sock_path = getenv("SSH_AUTH_SOCK");
    if (sock_path == NULL) {
        ssh_channel_free(ch);
        return NULL;
    }

    int agent_fd = socket(AF_UNIX, SOCK_STREAM, 0);
    struct sockaddr_un addr = { .sun_family = AF_UNIX };
    strncpy(addr.sun_path, sock_path, sizeof(addr.sun_path) - 1);
    if (connect(agent_fd, (struct sockaddr *)&addr, sizeof(addr)) < 0) {
        close(agent_fd);
        ssh_channel_free(ch);
        return NULL;
    }

    /* bridge agent_fd ↔ ch */
    ssh_connector c_in  = ssh_connector_new(session);
    ssh_connector c_out = ssh_connector_new(session);
    ssh_connector_set_in_fd(c_in, agent_fd);
    ssh_connector_set_out_channel(c_in, ch, SSH_CONNECTOR_STDOUT);
    ssh_connector_set_in_channel(c_out, ch, SSH_CONNECTOR_STDOUT);
    ssh_connector_set_out_fd(c_out, agent_fd);
    ssh_event_add_connector(st->ev, c_in);
    ssh_event_add_connector(st->ev, c_out);

    return ch;  /* accept the open request */
}
```

---

## 7. Config and Options -> Current State

| Directive       | config.c status  | Notes                                              |
|-----------------|------------------|----------------------------------------------------|
| `ForwardAgent`  | `SOC_UNSUPPORTED`| Silently discarded -> needs promotion               |
| `IdentityAgent` | `SOC_IDENTITYAGENT` | **Already implemented** -> maps to `SSH_OPTIONS_IDENTITY_AGENT` |

`IdentityAgent` already works (src/config.c):
```c
case SOC_IDENTITYAGENT:
    p = ssh_config_get_str_tok(&s, NULL);
    if (*parsing) {
        ssh_options_set(session, SSH_OPTIONS_IDENTITY_AGENT, p);
    }
```

`SSH_OPTIONS_IDENTITY_AGENT` (include/libssh/libssh.h) stores the path in `session->opts.agent_socket`, overriding `$SSH_AUTH_SOCK`.

`ForwardAgent` needs to be promoted as following :
- Add `SSH_OPTIONS_FORWARD_AGENT` (boolean) to `enum ssh_options_e`
- Add `bool forward_agent` to `struct ssh_options_struct` in include/libssh/session.h
- Change `SOC_UNSUPPORTED` → `SOC_SESSION` in src/config.c, add a parsing case
- Add case in src/options.c `ssh_options_set()`

---

## 8. Points to remember while implementing Agent Forwarding

**ForwardAgent is dangerous.** While connected with `-A`, if someone has root access on the server they can use our forwarded agent to authenticate as us to other servers. They cannot extract the private key -> they can only sign through the agent socket -> but that is enough to impersonate us.

OpenSSH mitigations:
- `sshd` sets `SSH_AUTH_SOCK` to a temp path owned by the user and gets removed on logout
- Some servers disallow forwarding entirely (`AllowAgentForwarding no` in sshd_config)

Our CLI should:
1. Not enable `-A` by default  as OpenSSH does not either
2. Honour `ForwardAgent no` in config once we promote it as the default
3. Only enable when explicitly `-A` or `ForwardAgent yes`

---

## Key Files

| File | What is Relevant |
|------|----------------|
| src/agent.c| Full agent protocol: `agent_connect`, `agent_talk`, `ssh_agent_get_ident_count`, `ssh_agent_get_next_ident`, `ssh_agent_sign_data`, `ssh_set_agent_channel` |
| include/libssh/agent.h | Agent protocol constants, `ssh_agent_struct`and public API |
| src/auth.c | `ssh_userauth_agent`, `ssh_userauth_agent_publickey`, authentication loop using the agent |
| src/channels.c | `ssh_channel_request_auth_agent` , `ssh_channel_open_auth_agent`  |
| src/socket.c | `ssh_socket_unix` -> Unix domain socket connection |
| include/libssh/callbacks.h| `ssh_channel_open_request_auth_agent_callback`, `ssh_channel_auth_agent_req_callback` |
| src/config.c| `forwardagent` = SOC_UNSUPPORTED, `identityagent` = SOC_IDENTITYAGENT (already works) |
| tests/client/torture_auth_agent_forwarding.c| Integration tests showing full forwarding flow |
