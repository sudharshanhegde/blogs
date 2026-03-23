# Keepalive

## Why do connections go Silent and die

Imagine we SSH into a server and then walk away from our computer for 30 minutes. When we come back and try to type something, nothing will happen, because the connection is dead.

Usually one of this three things are the reason for the kill :

**NAT timeout.** Most home routers and corporate firewalls track active TCP connections in a table. When our SSH connection has been idle for a while and no bytes are flowing in either direction than the router silently removes that entry from its table. When we try to type, the TCP packet goes through the router, the router has no record of this connection, and it drops the packet. our TCP stack keeps retrying and eventually gives up. From our perspective it just looks frozen.

**Firewall idle timeout.** Same idea but at the office or cloud firewall level. AWS security groups, corporate proxies, and load balancers will do this. Even 5 minutes of silence can be enough on some networks.

**The server went away.** The server rebooted, crashed, or the network link dropped. Our client's TCP stack does not have any idea yet, TCP does not have a built-in "tell me if the other side is gone" mechanism beyond the 3-way handshake. So our client is sitting there waiting to hear something, and it could wait for 2 hours before the OS TCP timeout fires.

Keepalive solves all three: it sends small probe packets during silence to keep the connection alive and to detect failures early.

## Two Levels of Keepalive

There are actually two separate keepalive systems that can both be active at the same time, working at different levels:

**TCP keepalive (SO_KEEPALIVE socket option)** — This is done by the OS kernel, not by our application. When this is enabled on a socket, the kernel automatically sends TCP-level probes if the connection has been completely silent for a while (typically 2 hours by default on Linux,which is configurable via `/proc/sys/net/ipv4`). The problem with this is that 2-hour default is far too long for interactive sessions, and the interval is set system-wide, not per-connection.

**SSH-level keepalive** — This is done by the SSH application itself, sending real SSH protocol messages during silence. We control the interval and failure count precisely. This is what `ServerAliveInterval` and `ServerAliveCountMax` control.

SSH-level keepalive is more useful for interactive use because we can set it to 15 or 30 seconds and detect a dead connection quickly, rather than waiting hours.

## What will ServerAliveInterval and ServerAliveCountMax Do

`ServerAliveInterval 30` -> if no data has come from the server for 30 seconds, send a keepalive probe.

`ServerAliveCountMax 3` -> if 3 consecutive keepalive probes get no response, close the connection.

So with the above settings: after 30 seconds of silence the first probe is sent. If no response, another at 60 seconds. Another at 90 seconds. If still no response than the connection is declared dead and the session exits. Total wait before giving up: 90 seconds.

This is the "I'd rather fail fast and reconnect than sit frozen" tradeoff.

## The keepalive@openssh.com Protocol

At the SSH protocol level, keepalive is implemented as a **global request** which is a special SSH message that applies to the whole session, not just a specific channel.

The client sends:

```
SSH2_MSG_GLOBAL_REQUEST (type 80)
  request-name = "keepalive@openssh.com"
  want-reply   = 1
  (no additional payload)
```

`want-reply = 1` means: the server must send back either `SSH2_MSG_REQUEST_SUCCESS` or `SSH2_MSG_REQUEST_FAILURE`. Most SSH servers (including OpenSSH) respond with `SSH2_MSG_REQUEST_FAILURE` because they do not implement `keepalive@openssh.com` as a meaningful operation, they only acknowledge it, which is fine. The client is not asking the server to *do* anything; it is just asking the server to *respond* to prove it is still alive.

If the response never arrives within the timeout window than the server is gone, the NAT table dropped the connection, whatever and than the client knows and can take action.

This is defined in (include/libssh/libssh.h) as `SSH_GLOBAL_REQUEST_KEEPALIVE` in `enum ssh_global_requests_e`.

### Client-side receiving (responding to server's keepalive)

When it is the *server* that sends a `keepalive@openssh.com` request to the client, libssh automatically responds. The code at (src/channels.c) :

```c
if (strcmp(request, "keepalive@openssh.com") == 0) {
    SSH_LOG(SSH_LOG_DEBUG, "Responding to Openssh's keepalive");

    rc = ssh_buffer_pack(session->out_buffer,
                         "bd",
                         SSH2_MSG_CHANNEL_FAILURE,
                         channel->remote_channel);
    ssh_packet_send(session);
    return SSH_PACKET_USED;
}
```

This responds automatically such that libssh client handles it without any code from us.

### Who sends what

In a typical interactive SSH session, it is the **client** that sends keepalive probes to the server (which is controlled by `ServerAliveInterval` and `ServerAliveCountMax`). This is because the client is the one waiting at the keyboard and cares about detecting a dead connection interactively.

The server side is handled by `ssh_send_keepalive()` in (src/server.c):

```c
int ssh_send_keepalive(ssh_session session)
{
    /* Client denies the request, so the error code is not meaningful */
    (void)ssh_global_request(session, "keepalive@openssh.com", NULL, 1);
    return SSH_OK;
}
```

This is exposed as a server-side API, but it just calls `ssh_global_request()` — which we can call directly from the client side too. There is no reason a client cannot use the same mechanism.

## What ssh_global_request Does

`ssh_global_request()` in (src/channels.c) builds and sends `SSH2_MSG_GLOBAL_REQUEST`, then waits synchronously for the response. The format it packs is `"bsb"`:
- `b` = message type byte (SSH2_MSG_GLOBAL_REQUEST = 80)
- `s` = request name string ("keepalive@openssh.com")
- `b` = want_reply byte (1 = yes, we want a response)

After sending, it calls `ssh_handle_packets_termination()` which blocks processing incoming packets until either `SSH2_MSG_REQUEST_SUCCESS` or `SSH2_MSG_REQUEST_FAILURE` arrives. When it arrives, `session->global_req_state` is updated to `SSH_CHANNEL_REQ_STATE_ACCEPTED` or `SSH_CHANNEL_REQ_STATE_DENIED`, and the function returns.

The "denied" case is not an error for keepalive because the server is saying "no I don't support this" still proves it is alive. That is why `ssh_send_keepalive()` ignores the return value.

## The Problem With Blocking in Our Event Loop

`ssh_global_request()` with `reply=1` blocks until a response arrives. In a simple linear program this is fine. But our CLI has a single-threaded event loop running `ssh_event_dopoll()` which we cannot block inside the event loop.

If we call `ssh_global_request(session, "keepalive@openssh.com", NULL, 1)` from inside the event loop, it will block the entire loop until the server responds. During that time no data from the shell, no forwarding, no escape sequence processing and the terminal will freeze for the duration of the round trip.

The solution is to call `ssh_global_request()` with `reply=0` (fire and forget) and use a different mechanism to detect failure. We can do this because the keepalive goal has two parts:

1. Keep the NAT alive -> just sending any bytes achieves this, we do not need a response
2. Detect a dead server -> the TCP stack itself will tell us when the connection is dead, it will just take longer

For most use cases, `reply=0` is sufficient. The NAT tables reset from our probe. If the server is truly dead, the TCP stack will eventually time out and `ssh_event_dopoll()` will return `SSH_ERROR`.

If we want fast failure detection like OpenSSH provides, we need to track unanswered probes ourselves:

```c
struct keepalive_state {
    time_t   last_activity;     /* last time any packet arrived from server */
    int      unanswered;        /* consecutive keepalives with no response */
    int      interval;          /* ServerAliveInterval seconds */
    int      max_count;         /* ServerAliveCountMax */
};
```

Whenever any packet arrives from the server (data, shell output, anything), reset `last_activity = time(NULL)` and `unanswered = 0`.

When we send a keepalive and no response arrives before the next interval, increment `unanswered`. When `unanswered >= max_count`, we will close the session.

## The Timer Pattern in the Event Loop

libssh has no built-in automatic keepalive. The application is responsible for the timing. The pattern is:

```c
struct keepalive_state ka = {
    .last_activity = time(NULL),
    .unanswered = 0,
    .interval = 30,      /* from ServerAliveInterval */
    .max_count = 3,      /* from ServerAliveCountMax */
};

while (!done) {
    time_t now = time(NULL);
    time_t next_due = ka.last_activity + ka.interval;
    int timeout_ms;

    if (next_due <= now) {
        /* keepalive is due right now */
        timeout_ms = 0;
    } else {
        /* wait until keepalive is due, but no longer */
        timeout_ms = (int)((next_due - now) * 1000);
    }

    rc = ssh_event_dopoll(ev, timeout_ms);

    if (rc == SSH_ERROR) {
        /* connection lost */
        break;
    }

    now = time(NULL);
    if (now >= ka.last_activity + ka.interval) {
        /* interval elapsed, no activity from server — send probe */
        ka.unanswered++;
        if (ka.unanswered > ka.max_count) {
            fprintf(stderr, "Timeout, server not responding.\n");
            break;
        }
        ssh_global_request(session, "keepalive@openssh.com", NULL, 0);
    }
}
```

`time(NULL)` returns the current Unix timestamp (seconds since 1970). `next_due - now` gives how many seconds until the next keepalive is due, which we convert to milliseconds for `ssh_event_dopoll`. This way `ssh_event_dopoll` wakes up at exactly the right time and it does not spin and burn CPU, it sleeps precisely until either an event or the keepalive deadline.

`ssh_event_dopoll` returning `SSH_ERROR` means the socket itself errored and TCP reported the connection as broken. This is the fast path: if the server reboots and properly sends a TCP FIN or RST, we detect it immediately regardless of keepalive settings.

The keepalive mechanism catches the slow path: NAT drop, server kernel panic (no TCP FIN sent), network cable pulled.

## TCPKeepAlive vs ServerAlive

`TCPKeepAlive yes` in SSH config enables `SO_KEEPALIVE` on the socket. This asks the OS kernel to send TCP-level probes. The problem is that : the kernel's keepalive parameters (`tcp_keepalive_time`, `tcp_keepalive_intvl`, `tcp_keepalive_probes`) are system-wide. we cannot set them per-connection from userspace. Default `tcp_keepalive_time` is 7200 seconds (2 hours) on Linux. This is useless for interactive sessions.

`ServerAliveInterval` is SSH-level keepalive, fully controlled by the application, per-session. This is what actually helps users.

For our CLI: We will implement `ServerAliveInterval` and `ServerAliveCountMax` properly with the timer-based pattern above. For `TCPKeepAlive`, we set `SO_KEEPALIVE` on the session socket using `setsockopt()`:

```c
int optval = 1;
setsockopt(ssh_get_fd(session), SOL_SOCKET, SO_KEEPALIVE, &optval, sizeof(optval));
```

`ssh_get_fd(session)` returns the underlying file descriptor of the SSH socket. `SOL_SOCKET` means we are setting a socket-level option. `SO_KEEPALIVE` enables the OS TCP keepalive. This is a single call,so that no timer is needed from our side.

`setsockopt()` is a system call for configuring socket behavior. The `SOL_SOCKET` level means the option applies to the socket generically (not to TCP specifically). `SO_KEEPALIVE` at this level just switches on the kernel's built-in TCP keepalive prober.

## Config Directives — Current State

All three keepalive directives are `SOC_UNSUPPORTED` in (src/config.c) and they are parsed from `~/.ssh/config` but thrown away:

| Directive              | Status          | Controls                                               |
|------------------------|-----------------|--------------------------------------------------------|
| `ServerAliveInterval`  | SOC_UNSUPPORTED | Seconds between SSH keepalive probes                   |
| `ServerAliveCountMax`  | SOC_UNSUPPORTED | Max unanswered probes before closing                   |
| `TCPKeepAlive`         | SOC_UNSUPPORTED | Enable OS-level SO_KEEPALIVE on the socket             |

`SOC_UNSUPPORTED` means this keyword is recognized (no "unknown option" warning) but the value is not acted on. Promoting these will require :
- Adding `SSH_OPTIONS_SERVER_ALIVE_INTERVAL`, `SSH_OPTIONS_SERVER_ALIVE_COUNT_MAX`, `SSH_OPTIONS_TCP_KEEPALIVE` to `enum ssh_options_e` in (include/libssh/libssh.h)
- Adding `int server_alive_interval`, `int server_alive_count_max`, `int tcp_keepalive` to `struct ssh_options_struct` in (include/libssh/session.h)
- Changing `SOC_UNSUPPORTED` to `SOC_SESSION`, adding parsing cases in (src/config.c)
- Adding cases in (src/options.c) for `ssh_options_set()`

## What We are Solving at the Project Level is : 

A developer using libssh who writes `ServerAliveInterval 30` in `~/.ssh/config` currently gets no effect and no warning. Their application will hang forever when a NAT drops the connection, just like having no config at all. This is a real usability gap.

Promoting these options means any libssh application amd not just our CLI also gets keepalive for free once the library options are set. Our `tools/ssh/` binary is where we will exercise this end-to-end and prove the implementation works, but the benefit will be library-wide.

## Key Files

| File | What's Relevant |
|------|----------------|
| (src/server.c) | `ssh_send_keepalive()`  wrapper around `ssh_global_request` |
| (src/channels.c) | Client-side keepalive response handler, `ssh_global_request` implementation |
| (src/messages.c) | Server-side keepalive@openssh.com parsing |
| (src/poll.c) | `ssh_event_dopoll`  the timer-based event loop we will be using |
| (src/config.c) | `serveraliveinterval`, `serveralivecountmax`, `tcpkeepalive` all at SOC_UNSUPPORTED |
| (include/libssh/libssh.h) | `SSH_GLOBAL_REQUEST_KEEPALIVE` in `enum ssh_global_requests_e` |
