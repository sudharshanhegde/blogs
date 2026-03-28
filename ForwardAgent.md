# ForwardAgent

## The Problem it will solve

Agent forwarding lets us use our local SSH private keys to authenticate to a third machine even when we are already inside an SSH session on a second machine -> without ever copying our private key to any remote server.

For a full explanation of how the SSH agent works, the Unix domain socket at `$SSH_AUTH_SOCK`, the wire protocol between the client and the agent, and the full flow of agent-assisted authentication, read the (Agent Forwarding) blog. This section is about promoting `ForwardAgent` from `SOC_UNSUPPORTED` to a first-class option and understanding exactly what the CLI must do when the option is enabled.

## Current State in libssh

In src/config.c:
```c
{"forwardagent", SOC_UNSUPPORTED, true},   /* line 107 */
```

`SOC_UNSUPPORTED` again, not `SOC_NA`. libssh considers agent forwarding applicable to the library -> it just has not been wired to a config option yet.

The two functions that do the actual work already exist and are fully implemented.

**`ssh_channel_request_auth_agent()`** in src/channels.c at line 2427:

```c
int ssh_channel_request_auth_agent(ssh_channel channel) {
    if (channel == NULL) {
        return SSH_ERROR;
    }
    return channel_request(channel, "auth-agent-req@openssh.com", NULL, 0);
}
```

This sends a channel request of type `"auth-agent-req@openssh.com"` to the server. `channel_request()` is the internal libssh function that serialises a channel request into an `SSH2_MSG_CHANNEL_REQUEST` packet and sends it. The third argument `NULL` means there is no extra payload beyond the request name. The fourth argument `0` means `want_reply = false` -> we do not ask the server to confirm that it enabled forwarding, we just send the request and trust it worked.

When the server receives this request, it knows that if the client later opens a channel of type `"auth-agent@openssh.com"`, it should accept it. The server will then use this reverse channel to talk to the SSH agent running on the original client machine.

**`ssh_channel_open_auth_agent()`** in src/channels.c at line 1100:

```c
int ssh_channel_open_auth_agent(ssh_channel channel)
{
    if (channel == NULL) {
        return SSH_ERROR;
    }
    return channel_open(channel,
                        "auth-agent@openssh.com",
                        WINDOW_DEFAULT,
                        CHANNEL_MAX_PACKET,
                        NULL);
}
```

This is called from the server side. When the server needs to talk to the agent (because a program on the server is trying to authenticate via SSH), it opens a new channel of type `"auth-agent@openssh.com"` back to the client. `WINDOW_DEFAULT` and `CHANNEL_MAX_PACKET` are the standard SSH flow-control parameters that control how much data can be in flight before the sender has to pause and wait for a window update from the receiver.

The client (our CLI) receives this incoming channel open request and must respond by connecting to the local agent socket and bridging the two.

There is no `SSH_OPTIONS_FORWARD_AGENT` constant. There is no `forward_agent` field in `struct ssh_options_struct`. The `opts` struct has `char *agent_socket` (the path to the local agent socket) but nothing that controls whether to enable forwarding.

## What Needs to Change

### Step 1 -> Add the option constant

In `enum ssh_options_e` in include/libssh/libssh.h:

```c
SSH_OPTIONS_FORWARD_AGENT,   /* int: 1 = enable agent forwarding, 0 = disable */
```

### Step 2 -> Add the field to session options

In include/libssh/session.h inside `struct ssh_options_struct`:

```c
int forward_agent;   /* 0 = no (default), 1 = yes */
```

The existing boolean options in libssh use two patterns. Simple yes/no options like `StrictHostKeyChecking` use a plain `int` field. Auth method options like `PASSWORD_AUTH` use a shared `int flags` field with bitmasks. `ForwardAgent` is a standalone yes/no, so we follow the `StrictHostKeyChecking` pattern: a plain `int` with 0 meaning false and 1 meaning true.

### Step 3 -> Handle in ssh_options_set()

In src/options.c, following the `SSH_OPTIONS_STRICTHOSTKEYCHECK` pattern:

```c
case SSH_OPTIONS_FORWARD_AGENT:
    if (value == NULL) {
        ssh_set_error_invalid(session);
        return -1;
    } else {
        int *x = (int *)value;
        session->opts.forward_agent = (*x & 0xff) > 0 ? 1 : 0;
    }
    break;
```

`(*x & 0xff) > 0` -> masks to one byte and checks non-zero. This defensively normalises any non-zero value to exactly 1. It is the same pattern used for `StrictHostKeyChecking` in the existing options.c at line 1213.

### Step 4 -> Promote in config.c

`ForwardAgent` accepts `yes` or `no`. In src/config.c, change `SOC_UNSUPPORTED` to `SOC_FORWARDAGENT`:

```c
case SOC_FORWARDAGENT:
    i = ssh_config_get_yesno(&s, -1);
    CHECK_COND_OR_FAIL(i < 0, "Invalid argument");
    if (*parsing) {
        ssh_options_set(session, SSH_OPTIONS_FORWARD_AGENT, &i);
    }
    break;
```

`ssh_config_get_yesno()` reads the next token from the config line and returns 1 for `"yes"`, 0 for `"no"`, -1 for anything else. `CHECK_COND_OR_FAIL(i < 0, ...)` catches parse errors. This is identical to the `StrictHostKeyChecking` config case.

We also need to add `SOC_FORWARDAGENT` to the SOC enum in include/libssh/config.h.

### Step 5 -> Default in session init

In src/session.c, in `ssh_new()`:

```c
session->opts.forward_agent = 0;   /* disabled by default, same as OpenSSH */
```

OpenSSH defaults to `ForwardAgent no` for security reasons -> blindly forwarding the agent to every server means a compromised server could use our agent to impersonate us on other machines.

## How the CLI Uses It -> Two Separate Actions

Enabling agent forwarding in the CLI requires two separate things that happen at different moments.

### Action 1: Tell the server we want forwarding

This happens once, right after the session channel is opened and before the shell is requested:

```c
if (session->opts.forward_agent) {
    rc = ssh_channel_request_auth_agent(channel);
    if (rc != SSH_OK) {
        /* forwarding request failed -> warn but do not exit */
        fprintf(stderr, "Warning: agent forwarding request failed: %s\n",
                ssh_get_error(session));
    }
}
```

`ssh_channel_request_auth_agent(channel)` sends `"auth-agent-req@openssh.com"` on the session channel. After this, the server knows we are ready to accept incoming agent channels. The function returns `SSH_OK` even if the server's sshd does not have `AllowAgentForwarding yes` in its config -> the failure will surface later when the server tries and fails to open the reverse channel.

### Action 2: Accept and bridge incoming agent channels

This is the more complex part. When a program on the server (say, another `ssh` process) wants to authenticate using our agent, the server opens a new channel of type `"auth-agent@openssh.com"` back to our CLI. Our CLI must:

1. Detect this incoming channel open request
2. Connect to the local agent socket (`$SSH_AUTH_SOCK`)
3. Bridge the two -> incoming agent channel ↔ local agent socket

In the event loop, we register a channel open callback. libssh calls this whenever the server opens a new channel:

```c
static ssh_channel auth_agent_channel_open_cb(ssh_session session,
                                               const char *type,
                                               int window,
                                               int maxpacket,
                                               void *userdata)
{
    if (strcmp(type, "auth-agent@openssh.com") != 0) {
        return NULL;  /* not our channel type, reject */
    }

    /* open a new channel object to represent this incoming channel */
    ssh_channel agent_ch = ssh_channel_new(session);
    if (agent_ch == NULL) return NULL;

    /* connect to the local agent socket */
    const char *auth_sock = session->opts.agent_socket
                            ? session->opts.agent_socket
                            : getenv("SSH_AUTH_SOCK");
    if (auth_sock == NULL) {
        ssh_channel_free(agent_ch);
        return NULL;
    }

    int agent_fd = socket(AF_UNIX, SOCK_STREAM, 0);
    struct sockaddr_un addr = {0};
    addr.sun_family = AF_UNIX;
    strncpy(addr.sun_path, auth_sock, sizeof(addr.sun_path) - 1);

    if (connect(agent_fd, (struct sockaddr *)&addr, sizeof(addr)) < 0) {
        close(agent_fd);
        ssh_channel_free(agent_ch);
        return NULL;
    }

    /* bridge: agent_fd <-> agent_ch */
    ssh_connector c_in  = ssh_connector_new(session);
    ssh_connector c_out = ssh_connector_new(session);

    ssh_connector_set_in_fd(c_in, agent_fd);
    ssh_connector_set_out_channel(c_in, agent_ch, SSH_CONNECTOR_STDOUT);

    ssh_connector_set_in_channel(c_out, agent_ch, SSH_CONNECTOR_STDOUT);
    ssh_connector_set_out_fd(c_out, agent_fd);

    ssh_event_add_connector(event, c_in);
    ssh_event_add_connector(event, c_out);

    return agent_ch;  /* returning the channel accepts the open request */
}
```

Let us go through every piece of this function.

`strcmp(type, "auth-agent@openssh.com")` -> when the server opens a channel, it sends the channel type string. We check it is exactly `"auth-agent@openssh.com"`. If we return `NULL`, libssh sends `SSH2_MSG_CHANNEL_OPEN_FAILURE` back to the server, rejecting the open request.

`ssh_channel_new(session)` -> allocates a new local channel object. This is different from opening a channel -> it just creates the client-side struct. The channel is not "open" until we return it from this callback, at which point libssh sends `SSH2_MSG_CHANNEL_OPEN_CONFIRMATION` to the server.

`getenv("SSH_AUTH_SOCK")` -> reads the environment variable that the SSH agent daemon sets when it starts. It contains the path to the Unix domain socket, something like `/run/user/1000/ssh-agent.sock` or `/tmp/ssh-XXXXXX/agent.1234`. The `session->opts.agent_socket` field lets the user override this with `IdentityAgent` in the config, but for agent forwarding we fall back to the environment variable.

`socket(AF_UNIX, SOCK_STREAM, 0)` -> creates a Unix domain socket. `AF_UNIX` means it uses the filesystem namespace (a path) rather than IP addresses. `SOCK_STREAM` means it is a stream socket (like TCP), giving us ordered, reliable byte delivery. The `0` is the protocol, which for Unix sockets is always 0.

`struct sockaddr_un` -> the address structure for Unix domain sockets. It has two fields: `sun_family` (always `AF_UNIX`) and `sun_path` (the filesystem path to the socket file). We copy the agent socket path into `sun_path`.

`connect(agent_fd, ...)` -> connects our new socket to the agent's listening socket. This is a regular POSIX `connect()` call, exactly the same as connecting to a TCP server but using a Unix socket address instead of an IP:port address. If the agent is not running or the socket file does not exist, this returns -1 and we reject the channel.

`ssh_connector_set_in_fd(c_in, agent_fd)` -> tells connector `c_in` to read data from the local agent socket fd. When the agent writes a response (e.g., a list of keys or a signature), `c_in` reads those bytes.

`ssh_connector_set_out_channel(c_in, agent_ch, SSH_CONNECTOR_STDOUT)` -> tells `c_in` to write what it reads onto the SSH channel. `SSH_CONNECTOR_STDOUT` means use the main data stream (not the extended stderr stream). So data flows: agent socket → connector → SSH channel → server → remote program.

`c_out` does the reverse direction: SSH channel → connector → agent socket. Together they create a full-duplex transparent bridge. The server-side program thinks it is talking directly to an SSH agent. Our local agent processes every request and sends replies back through the bridge.

`return agent_ch` -> returning a non-NULL channel from the callback accepts the open request. libssh will send `SSH2_MSG_CHANNEL_OPEN_CONFIRMATION` with the channel parameters. Returning `NULL` would reject it.

## What We will be Solving in this Project

Today `forwardagent` is `SOC_UNSUPPORTED` -> writing `ForwardAgent yes` in `~/.ssh/config` does nothing for any libssh application. `ssh_channel_request_auth_agent()` and `ssh_channel_open_auth_agent()` both exist and work perfectly, but nothing reads the config and decides to call them. By promoting this option and wiring the callback in our CLI, the user's config file controls agent forwarding behaviour automatically. The `-A` flag on the command line sets `session->opts.forward_agent = 1` before config processing, and `-a` sets it to 0.

## Changes Summary

| File | Changes needed |
|------|--------|
| include/libssh/libssh.h | Add `SSH_OPTIONS_FORWARD_AGENT` to `enum ssh_options_e` |
| include/libssh/session.h | Add `int forward_agent` to `struct ssh_options_struct` |
| include/libssh/config.h | Add `SOC_FORWARDAGENT` to SOC enum |
| src/session.c | Default to `0` in `ssh_new()` |
| src/options.c | Add `case SSH_OPTIONS_FORWARD_AGENT:` following StrictHostKeyChecking pattern |
| src/config.c | `SOC_UNSUPPORTED` to `SOC_FORWARDAGENT`, parse with `ssh_config_get_yesno` |
| tests/unittests/torture_options.c | Unit test: set 1, verify stored; set 0, verify stored |
| tests/unittests/torture_config.c | Config test: parse `ForwardAgent yes`, verify 1 stored |
