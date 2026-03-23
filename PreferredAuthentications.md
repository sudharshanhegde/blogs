# PreferredAuthentications

## The Problem it will solve : 

SSH supports multiple ways to prove who we are like public key, password, keyboard-interactive, GSSAPI. When we connect to a server, the server tells the client which of these it supports. The client then tries them. But there is no order for this in libssh.

The default order OpenSSH uses is: `gssapi-with-mic`, `hostbased`, `publickey`, `keyboard-interactive`, `password`. This means if GSSAPI is advertised by the server, OpenSSH tries it first even if we never configured a Kerberos ticket and it will fail immediately. Only then  it will try public key.

On some networks this causes a noticeable delay and each failed method adds a round trip. On corporate networks with GSSAPI-enabled servers, users will see a pause before their key is even tried.

`PreferredAuthentications` lets us say: "try these methods, in this order, and try nothing else." For example: `PreferredAuthentications publickey,password` means try public key first, then password, ignore everything else.

This also matters for security. If we only want public key auth to ever be used and no password fallback than we set `PreferredAuthentications publickey`. Even if the server offers password auth, the client will not attempt it.

## How the Auth Negotiation Works

Before the client can try any authentication method, it needs to know which methods the server accepts. This happens through a mechanism called "none" authentication.

When the client sends a `SSH2_MSG_USERAUTH_REQUEST` with method `"none"`, the server almost always rejects it and in the rejection message it tells the client what methods it does support. This rejection message is `SSH2_MSG_USERAUTH_FAILURE` and its payload is:

```
string  auth_methods   (comma-separated like this : "publickey,password,keyboard-interactive")
boolean partial        (whether a partial success has happened)
```

libssh parses this in `ssh_packet_userauth_failure()` at src/auth.c :

```c
if (strstr(auth_methods, "password") != NULL) {
    session->auth.supported_methods |= SSH_AUTH_METHOD_PASSWORD;
}
if (strstr(auth_methods, "keyboard-interactive") != NULL) {
    session->auth.supported_methods |= SSH_AUTH_METHOD_INTERACTIVE;
}
if (strstr(auth_methods, "publickey") != NULL) {
    session->auth.supported_methods |= SSH_AUTH_METHOD_PUBLICKEY;
}
/* ... like this it goes on */
```

`strstr(haystack, needle)` is a standard C function that checks if `needle` appears anywhere in `haystack`. After this parsing, `session->auth.supported_methods` is a bitmask of what the server supports.

`ssh_userauth_list(session, NULL)` in src/auth.c returns this bitmask so the application can query it.

## The Auth Method Constants

Defined in include/libssh/libssh.h:

```c
#define SSH_AUTH_METHOD_UNKNOWN      0x0000u
#define SSH_AUTH_METHOD_NONE         0x0001u
#define SSH_AUTH_METHOD_PASSWORD     0x0002u
#define SSH_AUTH_METHOD_PUBLICKEY    0x0004u
#define SSH_AUTH_METHOD_HOSTBASED    0x0008u
#define SSH_AUTH_METHOD_INTERACTIVE  0x0010u
#define SSH_AUTH_METHOD_GSSAPI_MIC   0x0020u
```

These are bit flags. we can OR them together. `session->auth.supported_methods` is a variable that holds one or more of these OR'd together. To check if the server supports password auth we do:

```c
if (session->auth.supported_methods & SSH_AUTH_METHOD_PASSWORD) { ... }
```

Separate from server-supported methods, libssh has `SSH_OPT_FLAG_*` flags in `session->opts.flags` at include/libssh/session.h :

```c
#define SSH_OPT_FLAG_PASSWORD_AUTH 0x1
#define SSH_OPT_FLAG_PUBKEY_AUTH   0x2
#define SSH_OPT_FLAG_KBDINT_AUTH   0x4
#define SSH_OPT_FLAG_GSSAPI_AUTH   0x8
```

These are the client's permission flags -> whether the user has allowed each method. All four are enabled by default at session init at src/session.c. The existing `SSH_OPTIONS_PASSWORD_AUTH`, `SSH_OPTIONS_PUBKEY_AUTH` etc. toggle these individual flags. But there is no ordering here.

## Current State in libssh

At src/config.c:
```c
{"preferredauthentications", SOC_UNSUPPORTED, true},
```

There is no `SSH_OPTIONS_PREFERRED_AUTHENTICATIONS` constant. There is no ordered preference list anywhere in `session->opts`. The current auth code has a hard-coded order: try agent -> try key files -> (application decides the rest).

There is no way to tell libssh "try keyboard-interactive before password" or "only try publickey, nothing else."

## What Needs to be changed

### Step 1 -> Represent the preference list

`PreferredAuthentications` is a comma-separated ordered list of method names: `"publickey,keyboard-interactive,password"`. We need to store this as an array of method IDs in the session options.

In include/libssh/session.h inside `struct ssh_options_struct`:

```c
uint32_t preferred_auth_methods[8]; /* ordered list of SSH_AUTH_METHOD_* values */
int      preferred_auth_count;       /* how many entries in the list */
```

An array of up to 8 entries is enough as there are only 5-6 meaningful auth methods. Each entry holds one of the `SSH_AUTH_METHOD_*` constants.

### Step 2 -> Add the option constant

In include/libssh/libssh.h at `enum ssh_options_e`:

```c
SSH_OPTIONS_PREFERRED_AUTHENTICATIONS,  /* char*: comma-separated method names */
```

### Step 3 -> Parse the string and than store it in ssh_options_set()

In src/options.c:

```c
case SSH_OPTIONS_PREFERRED_AUTHENTICATIONS: {
    const char *list = (const char *)value;
    char *copy, *token, *saveptr;
    int count = 0;

    if (list == NULL) {
        ssh_set_error_invalid(session);
        return -1;
    }

    /* reset existing preference */
    session->opts.preferred_auth_count = 0;

    copy = strdup(list);  /* strtok_r modifies the string, work on a copy */
    token = strtok_r(copy, ",", &saveptr);
    while (token != NULL && count < 8) {
        /* trim leading/trailing whitespace */
        while (*token == ' ') token++;

        uint32_t method = 0;
        if      (strcmp(token, "publickey")            == 0) method = SSH_AUTH_METHOD_PUBLICKEY;
        else if (strcmp(token, "password")             == 0) method = SSH_AUTH_METHOD_PASSWORD;
        else if (strcmp(token, "keyboard-interactive") == 0) method = SSH_AUTH_METHOD_INTERACTIVE;
        else if (strcmp(token, "gssapi-with-mic")      == 0) method = SSH_AUTH_METHOD_GSSAPI_MIC;
        else if (strcmp(token, "hostbased")            == 0) method = SSH_AUTH_METHOD_HOSTBASED;

        if (method != 0) {
            session->opts.preferred_auth_methods[count++] = method;
        }
        token = strtok_r(NULL, ",", &saveptr);
    }
    session->opts.preferred_auth_count = count;
    free(copy);
    break;
}
```

`strtok_r` is the thread-safe version of `strtok`. It splits a string by a delimiter (here `,`). The first call takes the string to split; subsequent calls pass `NULL` to continue where it left off. `saveptr` is an internal pointer that `strtok_r` uses to track position and it must be provided because `strtok_r` is usable from multiple threads simultaneously without interfering. `strtok` uses a global pointer and is not thread-safe.

### Step 4 -> Promote in config.c

In src/config.c, change `SOC_UNSUPPORTED` to `SOC_PREFERREDAUTHENTICATIONS`, add parsing case:

```c
case SOC_PREFERREDAUTHENTICATIONS:
    p = ssh_config_get_str_tok(&s, NULL);
    CHECK_COND_OR_FAIL(p == NULL, "Missing argument");
    if (*parsing) {
        ssh_options_set(session, SSH_OPTIONS_PREFERRED_AUTHENTICATIONS, p);
    }
    break;
```

## How the CLI will use it:

The preference list drives the entire authentication loop in our `tools/ssh/` binary. Instead of a fixed sequence of `if (try publickey) ... if (try password) ...`, we iterate the preference list:

```c
static int authenticate(ssh_session session, struct cli_opts *opts)
{
    int rc;

    /* Step 1: send "none" to get server's supported methods list */
    rc = ssh_userauth_none(session, NULL);
    if (rc == SSH_AUTH_SUCCESS) return rc;  /* some servers allow none auth */

    int server_methods = ssh_userauth_list(session, NULL);

    /* Step 2: determine which methods to try and in what order */
    uint32_t *order    = session->opts.preferred_auth_methods;
    int       n        = session->opts.preferred_auth_count;

    /* if no preference list set, use default order */
    static uint32_t default_order[] = {
        SSH_AUTH_METHOD_PUBLICKEY,
        SSH_AUTH_METHOD_INTERACTIVE,
        SSH_AUTH_METHOD_PASSWORD,
    };
    if (n == 0) {
        order = default_order;
        n = 3;
    }

    /* Step 3: try each method in preference order */
    for (int i = 0; i < n; i++) {
        uint32_t method = order[i];

        /* skip if server doesn't support it */
        if (!(server_methods & method)) continue;

        switch (method) {
        case SSH_AUTH_METHOD_PUBLICKEY:
            rc = ssh_userauth_publickey_auto(session, NULL, NULL);
            break;

        case SSH_AUTH_METHOD_INTERACTIVE:
            rc = try_kbdint(session);   /* calls ssh_userauth_kbdint() in a loop */
            break;

        case SSH_AUTH_METHOD_PASSWORD:
            rc = try_password(session); /* prompts user, calls ssh_userauth_password() */
            break;

        case SSH_AUTH_METHOD_GSSAPI_MIC:
            rc = ssh_userauth_gssapi(session);
            break;
        }

        if (rc == SSH_AUTH_SUCCESS) return SSH_AUTH_SUCCESS;
        if (rc == SSH_AUTH_PARTIAL) continue; /* partial success: than keep going */
        /* SSH_AUTH_DENIED: try next method */
    }

    return SSH_AUTH_DENIED;
}
```

The key line is that `if (!(server_methods & method)) continue` -> we only try methods the server has advertised. There is no point trying password auth if the server does not offer it.

### The Intersection Principle

What actually gets tried is the **intersection** of three sets:

1. What the server supports (`session->auth.supported_methods` -> from the "none" auth probe)
2. What the user has allowed (`session->opts.flags` -> the `SSH_OPT_FLAG_*` bitmask, affected by `PasswordAuthentication no` etc.)
3. What the user prefers (`session->opts.preferred_auth_methods` -> from `PreferredAuthentications`)

A method is tried only if it appears in all three, and in the order given by the preference list.

### Example Scenarios

`PreferredAuthentications publickey` -> only public key is ever attempted. If the server does not support it or all keys are rejected: "Permission denied (publickey)." No password fallback, no keyboard-interactive. Useful for scripts where we want a hard failure if the key stops working.

`PreferredAuthentications keyboard-interactive,password` -> skip public key entirely. Useful when connecting to a server where we intentionally want password-style auth (OTP systems, fresh VMs where we have not copied our key yet).

`PreferredAuthentications publickey,keyboard-interactive` -> try key first, fall back to keyboard-interactive (OTP). Never try simple password. Useful on servers that enforce two-factor auth.

## What We Are Solving at the Project Level

Currently any libssh application ignores `PreferredAuthentications` from `~/.ssh/config`. There is also no `SSH_OPTIONS_PREFERRED_AUTHENTICATIONS` for applications that want to control auth order programmatically. The existing per-method flags (`SSH_OPTIONS_PASSWORD_AUTH` etc.) let us disable methods but not control their order or restrict to a specific subset efficiently. Promoting `PreferredAuthentications` fills this gap for both config-file users and API users.

## Changes Summary

| File | Changes needed |
|------|--------|
| include/libssh/libssh.h | Adding `SSH_OPTIONS_PREFERRED_AUTHENTICATIONS` to `enum ssh_options_e` |
| include/libssh/session.h | Adding `preferred_auth_methods[8]` and `preferred_auth_count` to `struct ssh_options_struct` |
| src/options.c | Parse comma-separated string, map method names to `SSH_AUTH_METHOD_*` constants |
| src/config.c | `SOC_UNSUPPORTED` to `SOC_PREFERREDAUTHENTICATIONS`, add parsing case |
| tests/unittests/torture_options.c | Unit test: set option with various strings, verify stored order |
| tests/unittests/torture_config.c | Config test: parse `PreferredAuthentications publickey,password`, verify stored |
