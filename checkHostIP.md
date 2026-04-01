# CheckHostIP

## The Problem it will Solve

When our SSH client connects to `example.com`, it resolves the hostname to an IP address, opens a TCP connection to that IP, and checks the server's host key against `~/.ssh/known_hosts`. The check uses the hostname `example.com` as the lookup key.

This is where the gap is.

If an attacker poisons our DNS cache and makes `example.com` resolve to their machine at `5.6.7.8` instead of the real `1.2.3.4`, our client connects to the wrong machine. The hostname lookup in known_hosts finds the entry for `example.com`, but that entry was created when `example.com` pointed somewhere else. The keys do not match and SSH warns us. Until now it is fine.

Let us say there is a subtler attack. What if the attacker has obtained the real private key for `example.com` (or is performing a BGP hijack at the IP level)? Then the hostname check alone cannot catch IP-level routing attacks. `CheckHostIP yes` tells SSH to also check the resolved IP address against known_hosts, separately from the hostname. If the IP has changed since we last connected, SSH will warn us even if the hostname entry looks clean.

`CheckHostIP yes` is the default in OpenSSH because its cost is near zero and it adds a meaningful second line of defense against attackers.

## What Checking the IP Actually Does

When `CheckHostIP yes`, the SSH client does two independent lookups in known_hosts when connecting to `hostname`:

1. It looks up the server key by `hostname` -> does this key match what we stored for this name?
2. Than looks up the server key by the resolved IP address -> does this key match what we stored for this IP?

If either check produces a mismatch,than SSH warns about a possible man-in-the-middle attack.

When a new host is added to known_hosts, both entries are written: one for the hostname and one for the IP. Future connections then benefit from both checks.

The IP check catches scenarios that hostname-only checking misses:

- DNS cache poisoning where the hostname now resolves to a different IP with a different key
- BGP route hijacking where traffic to the correct IP is intercepted
- Situations where the same IP is shared by multiple hostnames and one gets compromised

When `CheckHostIP no`, only the hostname is checked. This is sometimes necessary for hosts behind load balancers or NAT where the IP address changes legitimately, and a second check against the IP would produce(wasteful) false alarms.

## How OpenSSH Implements It

OpenSSH reads `CheckHostIP` from `~/.ssh/config` and stores it as a session-level flag. In `sshconnect.c`, after resolving the hostname to an IP address, the connection flow does the following:

1. Perform the standard host key check using the hostname.
2. If `CheckHostIP yes` and the connection is not to localhost or a loopback address: perform a second host key check using the resolved IP address string.
3. If the IP address is not yet in known_hosts but the hostname was known and verified, add the IP address entry automatically so future connections benefit from the IP check without prompting the user again.

The two checks are independent. A clean hostname check does not suppress a failing IP check and vice versa.

## Current State in libssh

In `src/config.c`:

```c
{"checkhostip", SOC_UNSUPPORTED, true},
```

`SOC_UNSUPPORTED` means this keyword is recognized from `~/.ssh/config`,but no warning is printed, but the value is silently discarded. Any libssh application whose config file contains `CheckHostIP no` (for example, to suppress false alarms behind a NAT) will not get that behavior. The option is read and ignored.

There is no `SSH_OPTIONS_CHECK_HOST_IP` constant in the codebase and no `check_host_ip` field in `session->opts`.

## What Needs to be Changed

### Step 1 -> Add the opcode

In `include/libssh/config.h`, in `enum ssh_config_opcode_e`, add before `SOC_MAX`:

```c
SOC_CHECKHOSTIP,
SOC_MAX /* Keep this one last in the list */
```

`SOC_MAX` sizes the `options_seen[SOC_MAX]` array that enforces first-write-wins semantics during config parsing. The new opcode will be placed before it.

### Step 2 -> Add the option constant

In `include/libssh/libssh.h`, in `enum ssh_options_e`, add before the closing brace:

```c
SSH_OPTIONS_NEXT_IDENTITY,
SSH_OPTIONS_CHECK_HOST_IP,
};
```

### Step 3 -> Add the field to the session options struct

In `include/libssh/session.h`, inside the `opts` anonymous struct:

```c
int address_family;
bool check_host_ip;
char *originalhost;
```

### Step 4 -> Initialize the default

In `src/session.c`, inside `ssh_new()` in the OPTIONS block alongside the other explicit defaults:

```c
session->opts.identities_only = false;
session->opts.check_host_ip = true;
session->opts.control_master = SSH_CONTROL_MASTER_NO;
```

**The default is `true`** as OpenSSH defaults to `CheckHostIP yes`. Relying on `calloc` zeroing would give us the wrong default. The initialization will be explicit and must be `true`.

### Step 5 -> Handle it in ssh_options_set()

In `src/options.c`:

```c
case SSH_OPTIONS_CHECK_HOST_IP:
    if (value == NULL) {
        ssh_set_error_invalid(session);
        return -1;
    } else {
        bool *x = (bool *)value;
        session->opts.check_host_ip = *x;
    }
    break;
```

### Step 6 -> Promote in config.c

Change the keyword table entry from `SOC_UNSUPPORTED` to `SOC_CHECKHOSTIP`:

```c
{"checkhostip", SOC_CHECKHOSTIP, true},
```

Add the parsing case. We use `ssh_config_get_yesno()` which reads the next token and returns `1` for yes, `0` for no, and `-1` if the argument is missing or unrecognized. We return `SSH_ERROR` directly on failure rather than using `CHECK_COND_OR_FAIL`, because that macro is gated by the `fail_on_unknown` flag which is `false` during file and string parsing, meaning it would silently ignore invalid arguments instead of failing:

```c
case SOC_CHECKHOSTIP:
    i = ssh_config_get_yesno(&s, -1);
    if (i < 0) {
        SSH_LOG(SSH_LOG_WARNING,
                "line %d: invalid argument for keyword \"checkhostip\"",
                count);
        SAFE_FREE(x);
        return SSH_ERROR;
    }
    if (*parsing) {
        bool b = i;
        ssh_options_set(session, SSH_OPTIONS_CHECK_HOST_IP, &b);
    }
    break;
```

`SAFE_FREE(x)` is required before the early `return SSH_ERROR` because `x` holds a heap-allocated copy of the current config line that was allocated earlier in the parsing function. Returning without freeing it would be causing a memory leak.

## How the CLI will use check_host_ip

In the `tools/ssh/` binary, after connecting and resolving the server's address, the host key verification step reads `session->opts.check_host_ip` to decide whether to perform the second IP-based check.

The standard host key check will use `ssh_session_is_known_server()` which internally checks by hostname. For the IP check it will be doing something like this :

```c
if (session->opts.check_host_ip) {
    char *ip = get_connected_ip(session); /* resolved IP as a string */
    enum ssh_known_hosts_e ip_state =
        ssh_session_is_known_server_by_ip(session, ip);

    if (ip_state == SSH_KNOWN_HOSTS_CHANGED) {
        fprintf(stderr,
                "WARNING: Remote host identification has changed "
                "for IP %s.\n"
                "This may indicate a DNS spoofing attack.\n", ip);
        return -1;
    }
}
```

If the IP address is unknown (first connection to this IP), the CLI adds it to known_hosts alongside the hostname entry. If the IP key is changed, the CLI warns and refuses to connect. If `check_host_ip` is false, the block will be skipped entirely.

## What We will be Solving at the Project Level

Currently any libssh application that reads SSH config ignores `CheckHostIP no` entirely. For applications deployed behind load balancers, Kubernetes ingress controllers, or NAT gateways where the outward IP changes regularly, this means they cannot suppress the false-alarm IP mismatch warnings that `CheckHostIP no` is designed to prevent. Promoting `CheckHostIP` to a first-class library option fixes this for all libssh applications, not just for our CLI.

The default of `true` also means that once the library layer is complete, the CLI gets correct OpenSSH-compatible behavior without any additional wiring for the common case.

## Changes Summary

| File | Changes |
|------|---------|
| `include/libssh/config.h` | Add `SOC_CHECKHOSTIP` to `enum ssh_config_opcode_e` before `SOC_MAX` |
| `include/libssh/libssh.h` | Add `SSH_OPTIONS_CHECK_HOST_IP` to `enum ssh_options_e` |
| `include/libssh/session.h` | Add `bool check_host_ip` to `opts` struct |
| `src/session.c` | Explicit default `session->opts.check_host_ip = true` in `ssh_new()` |
| `src/options.c` | Add doxygen entry and `case SSH_OPTIONS_CHECK_HOST_IP:` in `ssh_options_set()` |
| `src/config.c` | `SOC_UNSUPPORTED` -> `SOC_CHECKHOSTIP`, add `SOC_CHECKHOSTIP` parse case |
| `tests/unittests/torture_options.c` | Unit test: default true, NULL rejected, false and true stored correctly |
| `tests/unittests/torture_config.c` | Config test: parse `CheckHostIP yes/no`, missing arg, invalid arg |
