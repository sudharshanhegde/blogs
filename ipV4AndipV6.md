# IPv4 and IPv6 in SSH: The -4 and -6 Flags

## The Problem It Will Solve

Modern servers are dual-stack. A hostname like `server.example.com` often resolves to both an A record (IPv4) and an AAAA record (IPv6). By default when we run `ssh server.example.com`, the resolver returns both and SSH tries them in the order the OS returns them, usually IPv6 first.

This causes three real problems:

**Broken IPv6 tunnels**: we are on a network where IPv6 is advertised but broken. SSH tries the AAAA address, waits for the TCP timeout (often 30 seconds), then falls back to IPv4. The connection eventually works but every session starts with a 30-second hang. The fix is `ssh -4 server.example.com` it tells SSH to not even look at AAAA records.

**Firewall and routing requirements**: Some corporate firewalls pass IPv4 traffic through an inspection proxy but route IPv6 directly. A script that must go through the proxy needs to force `-4`. A script that must bypass it needs `-6`. The address family is a routing decision, not just a preference.

**Testing and debugging**: When we are debugging a dual-stack server, we need to test the IPv4 path and the IPv6 path independently. `-4` and `-6` give we that control without editing `/etc/hosts` or the SSH config.

The config file equivalent is `AddressFamily inet` or `AddressFamily inet6`, which lets we set this permanently for specific hosts in `~/.ssh/config`:

```
Host legacy-server
    AddressFamily inet
```

## What -4 and -6 Actually Do

Both flags control one thing: what `getaddrinfo()` is told to look for when resolving the hostname.

`getaddrinfo()` is the POSIX function that turns a hostname into a list of socket addresses. Its second argument, `hints.ai_family`, tells it which address families to return:

- `PF_UNSPEC` (0): it returns everything -> both A and AAAA records, in OS-determined order
- `PF_INET` (2): return only IPv4 addresses -> only A records
- `PF_INET6` (10 on Linux): return only IPv6 addresses -> only AAAA records

The `-4` flag maps to `PF_INET`. The `-6` flag maps to `PF_INET6`. Default (no flag) maps to `PF_UNSPEC`.

If `-6` is specified and the hostname has no AAAA record, `getaddrinfo()` returns an empty list and the connection fails immediately with "No address associated with hostname." There is no fallback. That is the correct behavior as we asked for IPv6 only.

## How OpenSSH Implements It

OpenSSH's `ssh` binary parses `-4` and `-6` in its `main()` `getopt` loop:

```c
case '4':
    options.address_family = AF_INET;
    break;
case '6':
    options.address_family = AF_INET6;
    break;
```

The value is stored in `options.address_family`. When `ssh_connect()` is called, this value is passed directly as `hints.ai_family` to `getaddrinfo()`.

The config file keyword `AddressFamily any|inet|inet6` sets the same field. The CLI flag takes precedence over the config file because flags are parsed after config and overwrite whatever the config set.

The two flags are mutually exclusive by convention so if we specify both, the last one wins. No error will be raised.

## Current State in libssh

**The library layer is already fully implemented.** There is nothing to promote from `SOC_UNSUPPORTED` or `SOC_NA` because `AddressFamily` is not in either of those lists. It was implemented as a first-class option at some point in libssh's history. Every piece of the pipeline exists and works.

Here is the complete picture of what is already there:

### The public enum in include/libssh/libssh.h

```c
enum ssh_address_family_options_e {
    SSH_ADDRESS_FAMILY_ANY,
    SSH_ADDRESS_FAMILY_INET,
    SSH_ADDRESS_FAMILY_INET6
};
```

This is the type-safe wrapper around the raw `PF_*` constants. Callers use these symbolic names rather than platform socket constants directly. `SSH_ADDRESS_FAMILY_ANY` maps to `PF_UNSPEC`, `SSH_ADDRESS_FAMILY_INET` maps to `PF_INET`, `SSH_ADDRESS_FAMILY_INET6` maps to `PF_INET6`.

`SSH_OPTIONS_ADDRESS_FAMILY` is the corresponding constant in `enum ssh_options_e` that we pass to `ssh_options_set()`.

### The storage field in include/libssh/session.h

```c
struct ssh_session_struct {
    struct {
        /* ... other options ... */
        int address_family;
    } opts;
};
```

The value lives in `session->opts.address_family`. It is an `int` rather than the enum type because `ssh_options_set()` takes a `void *` and the internal storage predates C's stricter enum handling.

### The setter in src/options.c

```c
case SSH_OPTIONS_ADDRESS_FAMILY:
    if (value == NULL) {
        ssh_set_error_invalid(session);
        return -1;
    } else {
        int *x = (int *)value;
        if (*x < SSH_ADDRESS_FAMILY_ANY ||
            *x > SSH_ADDRESS_FAMILY_INET6) {
            ssh_set_error_invalid(session);
            return -1;
        }
        session->opts.address_family = *x;
    }
    break;
```

The handler validates that the value is within the known enum range before storing it. Passing an out-of-range integer returns `-1` with `SSH_EINVAL`. This is the same null-and-range-check pattern used by other integer options in the file.

### The config parser in src/config.c

```c
{"addressfamily", SOC_ADDRESSFAMILY, true},   /* This is in the keyword table */

case SOC_ADDRESSFAMILY:
    p = ssh_config_get_str_tok(&s, NULL);
    if (p == NULL) {
        SSH_LOG(SSH_LOG_WARNING,
                "line %d: no argument after keyword \"addressfamily\"",
                count);
        SAFE_FREE(x);
        return SSH_ERROR;
    }
    if (*parsing) {
        int value = -1;

        if (strcasecmp(p, "any") == 0) {
            value = SSH_ADDRESS_FAMILY_ANY;
        } else if (strcasecmp(p, "inet") == 0) {
            value = SSH_ADDRESS_FAMILY_INET;
        } else if (strcasecmp(p, "inet6") == 0) {
            value = SSH_ADDRESS_FAMILY_INET6;
        } else {
            SSH_LOG(SSH_LOG_WARNING,
                    "line %d: invalid argument \"%s\"",
                    count, p);
            SAFE_FREE(x);
            return SSH_ERROR;
        }
        ssh_options_set(session, SSH_OPTIONS_ADDRESS_FAMILY, &value);
    }
    break;
```

`ssh_config_get_str_tok()` reads the next token from the config line that is the value after the keyword. The `strcasecmp` comparisons make `inet`, `INET`, and `Inet` all equivalent. A missing or unrecognized argument returns `SSH_ERROR` with a log message rather than silently continuing. The `*parsing` guard means the value is only stored when the current `Host` block actually matches the session's target host.

### The socket creation in src/connect.c

This is where the stored value is actually used:

```c
switch (session->opts.address_family) {
case SSH_ADDRESS_FAMILY_INET:
    ai_family = PF_INET;
    ai_family_str = "inet";
    break;
case SSH_ADDRESS_FAMILY_INET6:
    ai_family = PF_INET6;
    ai_family_str = "inet6";
    break;
case SSH_ADDRESS_FAMILY_ANY:
default:
    ai_family = PF_UNSPEC;
    ai_family_str = "any";
}
SSH_LOG(SSH_LOG_PACKET,
        "Resolve target hostname %s port %d (%s)",
        host, port, ai_family_str);
rc = getai(host, port, ai_family, &ai);
```

`getai()` is libssh's thin wrapper around `getaddrinfo()`. The `ai_family` value is passed as `hints.ai_family`. The log line at `SSH_LOG_PACKET` level lets us verify which address family is being used by running with `-vvv`. The `for` loop that follows iterates over the returned addresses and creates sockets so that if `ai_family` was `PF_INET`, only IPv4 `sockaddr_in` structures are in the list, so only IPv4 sockets are ever created.

### What is already tested

`tests/client/torture_connect.c` contains `torture_connect_addrfamily()` which sets up a test server reachable via both IPv4 and IPv6 and verifies that:
- `SSH_ADDRESS_FAMILY_ANY` connects to both `afinet` and `afinet6` hosts
- `SSH_ADDRESS_FAMILY_INET` connects to `afinet` but fails on `afinet6`
- `SSH_ADDRESS_FAMILY_INET6` connects to `afinet6` but fails on `afinet`

`tests/unittests/torture_config.c` verifies that `AddressFamily any/inet/inet6` parses correctly into the right enum values, including error cases for missing and invalid arguments.

## What the CLI will need to Do

Since the library is complete, the CLI's job is purely flag wiring. In `tools/ssh/main.c` (This is my plan on implementation we will be creating tools/ssh folder for CLI):

```c
/* default: no preference */
int af = SSH_ADDRESS_FAMILY_ANY;

/* in getopt loop */
case '4':
    af = SSH_ADDRESS_FAMILY_INET;
    break;
case '6':
    af = SSH_ADDRESS_FAMILY_INET6;
    break;

/* after ssh_new() and before ssh_connect() */
if (af != SSH_ADDRESS_FAMILY_ANY) {
    ssh_options_set(session, SSH_OPTIONS_ADDRESS_FAMILY, &af);
}
```

The `if` guard is not strictly necessary because setting `SSH_ADDRESS_FAMILY_ANY` explicitly has the same effect as the default and also it avoids a redundant call when neither flag was given.

The CLI does not need to handle the `AddressFamily` config directive separately. `ssh_options_parse_config()` already does that. If both the config file sets `AddressFamily inet` and the user passes `-6`, we want `-6` to win. That is achieved naturally by calling `ssh_options_set()` after `ssh_options_parse_config()` -> later calls overwrite earlier ones.

## Changes Summary

| File | Change |
|------|--------|
| `include/libssh/libssh.h` | No change needed -> `SSH_OPTIONS_ADDRESS_FAMILY` and `ssh_address_family_options_e` already exist |
| `include/libssh/session.h` | No change needed -> `session->opts.address_family` already exists |
| `src/options.c` | No change needed -> case handler already exists |
| `src/config.c` | No change needed -> `SOC_ADDRESSFAMILY` parser already exists |
| `src/connect.c` | No change needed -> `PF_INET`/`PF_INET6`/`PF_UNSPEC` switch already exists |
| `tools/ssh/main.c` | **New file** -> parse `-4`/`-6` flags, call `ssh_options_set(session, SSH_OPTIONS_ADDRESS_FAMILY, &af)` |
| `tools/ssh/CMakeLists.txt` | **New file** -> build the `ssh` binary |
| `tools/CMakeLists.txt` | **New file** -> add `tools/ssh` subdirectory |
| `CMakeLists.txt` | **Modify** -> add `add_subdirectory(tools)` gated on `WITH_SSH_CLI` option |

The library does all the heavy lifting. The CLI adds four lines of `getopt` handling and one `ssh_options_set()` call.
