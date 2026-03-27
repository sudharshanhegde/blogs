# ServerAliveInterval and ServerAliveCountMax

## The Problem it will solve

When an SSH session sits idle, the TCP connection can silently die if a NAT router drops the flow table entry, a firewall resets the connection, or the remote machine crashes. Neither side notices until one of them tries to send data. We sit at a prompt that looks fine, type a command, and nothing happens. The connection died minutes ago.

For a full explanation of what keepalives are, the two mechanisms that exist (TCP-level and SSH application-level), and the wire format of the `keepalive@openssh.com` message, read the ([KeepAlive](https://github.com/sudharshanhegde/blogs/blob/main/Keepalive.md)) blog. This section is about promoting `ServerAliveInterval` and `ServerAliveCountMax` from `SOC_UNSUPPORTED` to first-class options and wiring the timer logic in the CLI.

## Current State in libssh

In src/config.c:
```c
{"serveralivecountmax", SOC_UNSUPPORTED, true},   /* line 126 */
{"serveraliveinterval", SOC_UNSUPPORTED, true},   /* line 127 */
```
libssh recognises these belong in the library but they have just never been wired up.

The function that does the actual sending already exists. In src/server.c:

```c
int ssh_send_keepalive(ssh_session session)
{
    (void)ssh_global_request(session, "keepalive@openssh.com", NULL, 1);
    return SSH_OK;
}
```

`ssh_global_request()` sends an `SSH2_MSG_GLOBAL_REQUEST` packet. The third argument `NULL` means no extra payload data. The fourth argument `1` means `want_reply = true` that is we are asking the server to send back an acknowledgement.

The server (OpenSSH sshd) does not recognise `"keepalive@openssh.com"` as a valid request type, so it replies with `SSH2_MSG_REQUEST_FAILURE`. This sounds like an error but it is not. The important thing is that the server replied at all. Any reply either success or failure only means the TCP connection is alive and the server is responsive.

The `(void)` cast discards the return value of `ssh_global_request()`. This is intentional: `SSH_ERROR` from `ssh_global_request()` would mean no reply came at all (timeout or connection drop), while `SSH_OK` just means we got the expected `REQUEST_FAILURE` reply. For the purposes of the keepalive mechanism, what matters is not the return value here but whether subsequent data arrives from the server in a reasonable time window. We handle that at the timer loop level.

There are no `SSH_OPTIONS_SERVER_ALIVE_INTERVAL` or `SSH_OPTIONS_SERVER_ALIVE_COUNT_MAX` constants. There are no corresponding fields in `struct ssh_options_struct`.

## What Needs to Change

### Step 1 -> Add the option constants

In `enum ssh_options_e` in include/libssh/libssh.h:

```c
SSH_OPTIONS_SERVER_ALIVE_INTERVAL,   /* long: seconds between keepalive probes, 0 = disabled */
SSH_OPTIONS_SERVER_ALIVE_COUNT_MAX,  /* long: max consecutive unanswered probes before disconnect */
```

### Step 2 -> Add fields to session options

In include/libssh/session.h inside `struct ssh_options_struct`:

```c
long server_alive_interval;    /* 0 = disabled (default) */
long server_alive_count_max;   /* default 3, matches OpenSSH */
```

We use `long` to match the type convention already used for `timeout` in the same struct (`unsigned long timeout`). The caller passes a `long *` to `ssh_options_set()`, same as `SSH_OPTIONS_TIMEOUT`.

### Step 3 -> Handle in ssh_options_set()

In src/options.c. The closest existing pattern is `SSH_OPTIONS_TIMEOUT` :

```c
case SSH_OPTIONS_TIMEOUT:
    if (value == NULL) {
        ssh_set_error_invalid(session);
        return -1;
    } else {
        long *x = (long *) value;
        if (*x < 0) {
            ssh_set_error_invalid(session);
            return -1;
        }
        session->opts.timeout = *x & 0xffffffffU;
    }
    break;
```

We follow the same shape for our two new options:

```c
case SSH_OPTIONS_SERVER_ALIVE_INTERVAL:
    if (value == NULL) {
        ssh_set_error_invalid(session);
        return -1;
    } else {
        long *x = (long *)value;
        if (*x < 0) {
            ssh_set_error_invalid(session);
            return -1;
        }
        session->opts.server_alive_interval = *x;
    }
    break;

case SSH_OPTIONS_SERVER_ALIVE_COUNT_MAX:
    if (value == NULL) {
        ssh_set_error_invalid(session);
        return -1;
    } else {
        long *x = (long *)value;
        if (*x < 0) {
            ssh_set_error_invalid(session);
            return -1;
        }
        session->opts.server_alive_count_max = *x;
    }
    break;
```

The negative check matters: a negative interval makes no sense and a negative count would cause the keepalive loop to never disconnect. We reject them with `ssh_set_error_invalid()`.

### Step 4 -> Promote in config.c

Both directives take a plain integer. In src/config.c, change `SOC_UNSUPPORTED` to `SOC_SERVERALIVEINTERVAL` and `SOC_SERVERALIVECOUNTMAX`:

```c
case SOC_SERVERALIVEINTERVAL: {
    long i = ssh_config_get_long(&s, -1);
    CHECK_COND_OR_FAIL(i < 0, "Invalid argument");
    if (*parsing) {
        ssh_options_set(session, SSH_OPTIONS_SERVER_ALIVE_INTERVAL, &i);
    }
    break;
}

case SOC_SERVERALIVECOUNTMAX: {
    long i = ssh_config_get_long(&s, -1);
    CHECK_COND_OR_FAIL(i < 0, "Invalid argument");
    if (*parsing) {
        ssh_options_set(session, SSH_OPTIONS_SERVER_ALIVE_COUNT_MAX, &i);
    }
    break;
}
```

`ssh_config_get_long()` is the libssh config helper that reads the next whitespace-delimited token from the current config line and converts it to a `long`. If the token is not a valid integer, it returns the default value we passed (here it will be `-1`). The `CHECK_COND_OR_FAIL(i < 0, ...)` macro then catches both parse failures (returned `-1`) and genuinely negative values and logs an error.

We also need to add `SOC_SERVERALIVEINTERVAL` and `SOC_SERVERALIVECOUNTMAX` to the SOC enum in include/libssh/config.h.

### Step 5 -> Defaults in session init

In src/session.c, in `ssh_new()`:

```c
session->opts.server_alive_interval  = 0;  /* 0 = disabled by default */
session->opts.server_alive_count_max = 3;  /* OpenSSH default is 3 */
```

## How the CLI Uses It -> The Timer Loop

Storing the two numbers in the session is the easy part. The real work will be in the CLI's main event loop, where we have to check the time, decide when to probe, count missed responses, and know when to give up.

### Why We Cannot Use a Background Thread

The obvious approach would be a background thread that sleeps for `server_alive_interval` seconds, calls `ssh_send_keepalive()`, and exits the program if the count exceeds `server_alive_count_max`. This does not work because libssh's session is not thread-safe. The session struct has internal buffers, state machines, and counters that are not protected by locks. If a background thread calls `ssh_send_keepalive()` — which calls `ssh_global_request()` which writes to the session's output buffer all this while the main thread is inside `ssh_event_dopoll()` reading and writing the same socket, we get data corruption and crashes.

The solution is to use `ssh_event_dopoll()`'s `timeout` parameter to wake the main thread up at the right moment.

### How ssh_event_dopoll() Timeout Works

```c
LIBSSH_API int ssh_event_dopoll(ssh_event event, int timeout);
```

The `timeout` parameter is in milliseconds:
- `-1` means "block indefinitely until there is network activity"
- `0` means "check what is ready right now and return immediately (non-blocking)"
- `N > 0` means "block for at most N milliseconds, then return even if nothing happened"

Internally dopoll calls the OS `poll()` system call with this exact timeout. `poll()` monitors the file descriptors registered with the event loop (the SSH socket, any local forwarding sockets, etc.) and returns when any of them become ready for reading or writing, or when the timeout expires returns whichever comes first.

When dopoll returns due to the timeout (not because of network activity), `rc` is still `SSH_OK`. It just means nothing happened in that time window. This is our signal: if nothing happened for as long as `server_alive_interval`, we need to probe.

### The Full Timer Loop

```c
time_t last_activity = time(NULL);  /* timestamp of last received data */
int    alive_misses  = 0;           /* consecutive unanswered probes */

while (!session_done) {

    /* --- calculate how long to sleep before the next probe --- */
    int timeout_ms = -1;   /* default: sleep indefinitely */

    long interval = session->opts.server_alive_interval;
    if (interval > 0) {
        time_t now     = time(NULL);
        time_t elapsed = now - last_activity;   /* seconds since we last heard anything */
        long   wait    = interval - (long)elapsed;

        if (wait <= 0) {
            timeout_ms = 0;    /* already overdue so probe immediately */
        } else {
            timeout_ms = (int)(wait * 1000);  /* convert seconds to milliseconds */
        }
    }

    /*  block until network activity or our timeout fires  */
    int rc = ssh_event_dopoll(event, timeout_ms);

    if (rc == SSH_ERROR) {
        /* a real connection error: socket closed, RST received, etc. */
        break;
    }

    /*  did we receive any data? */
    time_t now = time(NULL);
    int got_data = ssh_channel_is_open(channel) &&
                   !ssh_channel_is_eof(channel);
    /* more precisely: track whether dopoll dispatched any read callbacks */
    /* for simplicity: if dopoll returned due to activity, now > last_activity */

    if (interval > 0 && (now - last_activity) >= interval) {
        /*
         * No data has arrived for 'interval' seconds.
         * Send a keepalive probe.
         */
        ssh_send_keepalive(session);
        alive_misses++;

        if (alive_misses >= session->opts.server_alive_count_max) {
            fprintf(stderr, "Timeout, server %s not responding.\n",
                    ssh_get_host(session));
            session_done = 1;
            break;
        }
    } else if (got_data) {
        /*
         * Data arrived from the server — connection is alive.
         * Reset both counters.
         */
        last_activity = now;
        alive_misses  = 0;
    }
}
```

`time(NULL)` -> returns the current time as a `time_t`, which is the number of seconds since 1 January 1970 (the Unix epoch). Subtracting two `time_t` values gives us elapsed seconds. This is POSIX standard and does not require any extra libraries.

`elapsed = now - last_activity` -> how many seconds have passed since we last received anything from the server. When this reaches `interval`, it is time to probe.

`wait * 1000` -> we convert seconds to milliseconds because `ssh_event_dopoll()` takes milliseconds. If `interval` is 15 seconds and 3 seconds have already passed, `wait` is 12 seconds, so we pass `12000` ms to dopoll. It will wake us up in exactly 12 seconds even if nothing arrives.

`alive_misses` -> we increment this every time we send a probe and the idle clock is still running when we wake up. If real data arrives between probes, we reset `alive_misses` to 0. When `alive_misses` reaches `server_alive_count_max`, we have sent that many probes with no reply, so the connection is considered dead.

`ssh_get_host(session)` -> returns the hostname we connected to, used in the error message so the user knows which server timed out.

### The alive_misses Reset Problem

There is a subtle detail: we need to know whether the probe itself was answered. When we call `ssh_send_keepalive()`, the server will eventually send `SSH2_MSG_REQUEST_FAILURE` in response. When dopoll processes that reply, it counts as "activity" and `last_activity` would be updated. This means `alive_misses` would be reset even though we only got a keepalive reply, not real session data.

That is actually the correct behaviour. The point of `alive_misses` is to count probes that got no reply at all. If the server replied to our probe (even with REQUEST_FAILURE), the connection is alive, so we reset the counter. We only increment `alive_misses` when we send a probe and the next wakeup still shows no activity meaning the server neither replied to our probe nor sent anything else.

## What We will be Solving in this Project

Today `serveraliveinterval` and `serveralivecountmax` are `SOC_UNSUPPORTED` -> writing them in `~/.ssh/config` does nothing. `ssh_send_keepalive()` exists in libssh but nothing ever calls it automatically. By promoting these options and wiring the timer loop, the user's config directly controls connection persistence. A user who writes `ServerAliveInterval 15` will now get automatic connection health monitoring without any application needing to implement this logic itself.

## Changes Summary

| File | Changes needed |
|------|--------|
| include/libssh/libssh.h | Add `SSH_OPTIONS_SERVER_ALIVE_INTERVAL` and `SSH_OPTIONS_SERVER_ALIVE_COUNT_MAX` to `enum ssh_options_e` |
| include/libssh/session.h | Add `long server_alive_interval` and `long server_alive_count_max` to `struct ssh_options_struct` |
| include/libssh/config.h | Add `SOC_SERVERALIVEINTERVAL` and `SOC_SERVERALIVECOUNTMAX` to SOC enum |
| src/session.c | Default interval=0 and count_max=3 in `ssh_new()` |
| src/options.c | Add cases for both options with negative-value guard, following `SSH_OPTIONS_TIMEOUT` pattern |
| src/config.c | `SOC_UNSUPPORTED` to `SOC_SERVERALIVEINTERVAL` / `SOC_SERVERALIVECOUNTMAX`, parse with `ssh_config_get_long` |
| tests/unittests/torture_options.c | Unit test: set interval=30, verify stored; set count_max=5, verify stored |
| tests/unittests/torture_config.c | Config test: parse `ServerAliveInterval 30`, verify 30 stored |
