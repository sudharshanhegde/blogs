# ExitOnForwardFailure

## The Problem it will solve

When we set up port forwarding with `-L`, `-R`, or `-D`, there is a question about what should happen if the forwarding fails to start. By default OpenSSH is lenient towards it, if the server refuses a remote forward request, or if the local bind fails, SSH continues the session anyway. The forwarding just silently does not work.

`ExitOnForwardFailure yes` changes this behaviour. If any forwarding spec fails to set up, the entire SSH session exits with an error. This is useful in scripts or automation where the forwarding is the whole point of the connection and if the tunnel is not up, there is nothing useful to do.

Lets say we write a script that runs `ssh -L 5432:db.internal:5432 server sleep 60` to keep a tunnel open for 60 seconds while our program connects to localhost:5432. If the `-L` setup fails and the script does not know, the program silently gets a connection refused. With `ExitOnForwardFailure yes`, the whole ssh command exits immediately with a non-zero code and our script can detect and handle the failure.

## What Happens At The Protocol Level

For RemoteForward (`-R`), failure is detectable at the protocol level. The CLI sends a `"tcpip-forward"` global request via `ssh_channel_listen_forward()`. The server can reply with `SSH2_MSG_REQUEST_FAILURE`, which means it refused to open the listener (wrong permissions, port already in use, GatewayPorts policy, etc.). libssh's `ssh_channel_listen_forward()` returns `SSH_ERROR` in this case.

For LocalForward and DynamicForward, failure happens at the OS level before SSH is even involved that is the `bind()` system call fails because the port is already in use, or we do not have permission to bind to a port below 1024.

Both failure modes need to be checked and, if `ExitOnForwardFailure` is set, it should be treated as fatal.

## Current State in libssh

In src/config.c:
```c
{"exitonforwardfailure", SOC_NA, true},
```

No `SSH_OPTIONS_EXIT_ON_FORWARD_FAILURE` constant. No field in `struct ssh_options_struct`. No existing handling anywhere.

## What Needs to Change

### Step 1 -> Add the option constant

In `enum ssh_options_e` in include/libssh/libssh.h:

```c
SSH_OPTIONS_EXIT_ON_FORWARD_FAILURE,  /* int: 1 = exit if any forward fails */
```

### Step 2 -> Add the field to session options

In include/libssh/session.h inside `struct ssh_options_struct`:

```c
int exit_on_forward_failure;  /* 0 = no (default), 1 = yes */
```

We will follow the same pattern as `StrictHostKeyChecking` which is already in the struct as `int StrictHostKeyChecking` -> boolean options in libssh are stored as `int` with 0 meaning false and 1 meaning true.

### Step 3 -> Handle in ssh_options_set()

In src/options.c, following the exact same pattern as `SSH_OPTIONS_STRICTHOSTKEYCHECK`:

```c
case SSH_OPTIONS_EXIT_ON_FORWARD_FAILURE:
    if (value == NULL) {
        ssh_set_error_invalid(session);
        return -1;
    } else {
        int *x = (int *)value;
        session->opts.exit_on_forward_failure = (*x & 0xff) > 0 ? 1 : 0;
    }
    break;
```

`(*x & 0xff) > 0` -> this masks to one byte and checks non-zero. It is defensive so that, if the caller passes a non-zero int like 2 or -1, it still correctly maps to 1. The pattern comes directly from how `StrictHostKeyChecking` is handled in options.c.

### Step 4 -> Promote in config.c

Config files use `yes` or `no`. In src/config.c, change `SOC_NA` to `SOC_EXITONFORWARDFAILURE`:

```c
case SOC_EXITONFORWARDFAILURE:
    i = ssh_config_get_yesno(&s, -1);
    CHECK_COND_OR_FAIL(i < 0, "Invalid argument");
    if (*parsing) {
        ssh_options_set(session, SSH_OPTIONS_EXIT_ON_FORWARD_FAILURE, &i);
    }
    break;
```

`ssh_config_get_yesno()` is a libssh helper that reads the next token from the config line and returns 1 for `"yes"`, 0 for `"no"`, and -1 if the value is neither. The `CHECK_COND_OR_FAIL` macro handles the error case. This is the same pattern used for `StrictHostKeyChecking` at src/config.c.

### Step 5 -> Default in session init

In src/session.c, in `ssh_new()`:

```c
session->opts.exit_on_forward_failure = 0;  /* default: no */
```

## How the CLI Uses It

After the session is authenticated and we begin setting up forwards, every setup call will be checked:

**For RemoteForward:**
```c
int rc = ssh_channel_listen_forward(session, bind_addr, port, &bound_port);
if (rc != SSH_OK && session->opts.exit_on_forward_failure) {
    fprintf(stderr, "Remote forward to %d failed: %s\n",
            port, ssh_get_error(session));
    exit(1);
}
```

`ssh_channel_listen_forward()` returns `SSH_OK` on success, `SSH_ERROR` on failure. If the server sent `SSH2_MSG_REQUEST_FAILURE`, libssh translates that to `SSH_ERROR`. Checking `!= SSH_OK` catches it.

**For LocalForward and DynamicForward:**
```c
int listen_fd = socket(AF_INET, SOCK_STREAM, 0);
setsockopt(listen_fd, SOL_SOCKET, SO_REUSEADDR, &(int){1}, sizeof(int));
if (bind(listen_fd, (struct sockaddr *)&addr, sizeof(addr)) < 0) {
    if (session->opts.exit_on_forward_failure) {
        fprintf(stderr, "Local bind on port %d failed: %s\n",
                port, strerror(errno));
        exit(1);
    }
    /* else: warn and continue */
    close(listen_fd);
}
```

`strerror(errno)` here `errno` is the global integer that the OS sets when a system call fails. `strerror()` converts it to a human-readable string like `"Address already in use"` or `"Permission denied"`.

## What We will be Solving in this Project

Today `exitonforwardfailure` is `SOC_NA` so a config line `ExitOnForwardFailure yes` is silently ignored by any libssh application. By promoting it, scripts and programs that depend on forwarding can reliably detect setup failures without having to write their own check logic.

## Changes Summary

| File | Changes needed |
|------|--------|
| include/libssh/libssh.h | Add `SSH_OPTIONS_EXIT_ON_FORWARD_FAILURE` to `enum ssh_options_e` |
| include/libssh/session.h | Add `int exit_on_forward_failure` to `struct ssh_options_struct` |
| src/session.c | Default to `0` in `ssh_new()` |
| src/options.c | Add `case SSH_OPTIONS_EXIT_ON_FORWARD_FAILURE:` |
| src/config.c | `SOC_NA` to `SOC_EXITONFORWARDFAILURE`, parse with `ssh_config_get_yesno` |
| tests/unittests/torture_options.c | Unit test: set 1, verify stored; set 0, verify stored |
| tests/unittests/torture_config.c | Config test: parse `ExitOnForwardFailure yes`, verify 1 stored |

---

# GatewayPorts

## The Problem it will solve

When we use RemoteForward (`-R 9090:localhost:8080`), we ask the server to listen on port 9090 and forward connections back to us. But by default the server only listens on the loopback interface (`127.0.0.1:9090`). That means only processes running on the server itself can reach that port -> not other machines on the internet.

`GatewayPorts yes` (set in the server's `sshd_config`) allows the server to bind remote forwards on all interfaces, making them reachable from the outside world. Without it, even if we request a remote forward with bind address `""` or `"0.0.0.0"`, the server silently binds only on loopback.

From the client side, `GatewayPorts` in `~/.ssh/config` is a hint we can send in the bind address field of the `"tcpip-forward"` global request. When we specify `clientspecified` as the value, OpenSSH sends the exact bind address we asked for in the `-R` spec, and leaves it to the server to enforce its own `GatewayPorts` policy. Without `GatewayPorts clientspecified`, OpenSSH always sends `""` (meaning "loopback") as the bind address.

In summary:
- `GatewayPorts no` (default) -> client always asks for loopback bind
- `GatewayPorts yes` -> client asks for all-interfaces bind (`0.0.0.0`)
- `GatewayPorts clientspecified` -> client sends whatever bind address was in the `-R` spec

## Current State in libssh

In src/config.c:
```c
{"gatewayports", SOC_NA, true},
```

No `SSH_OPTIONS_GATEWAY_PORTS` constant. No field in `struct ssh_options_struct`. `GatewayPorts` is different from most options because it controls the bind address string that is sent to the server inside `ssh_channel_listen_forward()` so the actual enforcement is on the server, but the client decides what to ask for.

## What Needs to Change

### Step 1 -> Represent the three states

Unlike a simple yes/no, GatewayPorts has three values. We will define a small enum in include/libssh/libssh.h:

```c
enum ssh_gateway_ports_e {
    SSH_GATEWAY_PORTS_NO              = 0,  /* default: bind on loopback */
    SSH_GATEWAY_PORTS_YES             = 1,  /* bind on all interfaces */
    SSH_GATEWAY_PORTS_CLIENTSPECIFIED = 2,  /* use whatever bind addr was in -R spec */
};
```

### Step 2 -> Add the option constant

In `enum ssh_options_e` in include/libssh/libssh.h:

```c
SSH_OPTIONS_GATEWAY_PORTS,  /* enum ssh_gateway_ports_e */
```

### Step 3 -> Add the field to session options

In include/libssh/session.h inside `struct ssh_options_struct`:

```c
enum ssh_gateway_ports_e gateway_ports;  /* default SSH_GATEWAY_PORTS_NO */
```

### Step 4 -> Handle in ssh_options_set()

In src/options.c:

```c
case SSH_OPTIONS_GATEWAY_PORTS:
    if (value == NULL) {
        ssh_set_error_invalid(session);
        return -1;
    } else {
        int *x = (int *)value;
        session->opts.gateway_ports = (enum ssh_gateway_ports_e)*x;
    }
    break;
```

### Step 5 -> Promote in config.c

`GatewayPorts` will accept `"no"`, `"yes"`, or `"clientspecified"`. In src/config.c,we need to change `SOC_NA` to `SOC_GATEWAYPORTS`:

```c
case SOC_GATEWAYPORTS: {
    p = ssh_config_get_str_tok(&s, NULL);
    CHECK_COND_OR_FAIL(p == NULL, "Missing argument");
    if (*parsing) {
        int v;
        if      (strcasecmp(p, "no")              == 0) v = SSH_GATEWAY_PORTS_NO;
        else if (strcasecmp(p, "yes")             == 0) v = SSH_GATEWAY_PORTS_YES;
        else if (strcasecmp(p, "clientspecified") == 0) v = SSH_GATEWAY_PORTS_CLIENTSPECIFIED;
        else break;
        ssh_options_set(session, SSH_OPTIONS_GATEWAY_PORTS, &v);
    }
    break;
}
```

We cannot use `ssh_config_get_yesno()` here because it only handles yes/no -> we use `ssh_config_get_str_tok()` and `strcasecmp()` to handle the three-way choice, the same pattern as `RequestTTY`.

### Step 6 -> Default in session init

In src/session.c:

```c
session->opts.gateway_ports = SSH_GATEWAY_PORTS_NO;  /* OpenSSH default is No*/
```

## How the CLI will use It

When setting up a RemoteForward, the CLI decides what bind address to pass to `ssh_channel_listen_forward()` based on this option:

```c
const char *resolve_gateway_bind(struct forward_spec *spec,
                                  ssh_session session)
{
    switch (session->opts.gateway_ports) {
        case SSH_GATEWAY_PORTS_YES:
            return "";          /* empty string = all interfaces in SSH protocol */
        case SSH_GATEWAY_PORTS_CLIENTSPECIFIED:
            return spec->bind_address ? spec->bind_address : "";
        case SSH_GATEWAY_PORTS_NO:
        default:
            return "localhost"; /* loopback only */
    }
}

/* usage */
const char *bind_addr = resolve_gateway_bind(spec, session);
int rc = ssh_channel_listen_forward(session, bind_addr, spec->bind_port, &bound_port);
```

This is purely about what we ask the server. What the server actually does depends on its own `GatewayPorts` setting in `sshd_config`. As the server can and will override our request if its policy does not allow external binding.

## What We will be Solving in this Project

Today `gatewayports` is `SOC_NA` so a libssh application cannot read this from `~/.ssh/config` and its remote forwards always default to loopback binding. By promoting it, users can control remote forward bind behaviour through their config file, and any libssh application that calls `ssh_channel_listen_forward()` can use the stored value to pass the correct bind address.

## Changes Summary

| File | Changes needed |
|------|--------|
| include/libssh/libssh.h | Add `enum ssh_gateway_ports_e`, add `SSH_OPTIONS_GATEWAY_PORTS` to `enum ssh_options_e` |
| include/libssh/session.h | Add `enum ssh_gateway_ports_e gateway_ports` to `struct ssh_options_struct` |
| src/session.c | Default to `SSH_GATEWAY_PORTS_NO` in `ssh_new()` |
| src/options.c | Add `case SSH_OPTIONS_GATEWAY_PORTS:` |
| src/config.c | `SOC_NA` to `SOC_GATEWAYPORTS`, parse no/yes/clientspecified with `strcasecmp` |
| tests/unittests/torture_options.c | Unit test: set each enum value, verify stored |
| tests/unittests/torture_config.c | Config test: parse `GatewayPorts clientspecified`, verify enum value stored |
