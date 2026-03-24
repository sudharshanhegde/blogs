# SendEnv

## The Problem It Solves

When we open a terminal on our laptop and type `echo $EDITOR`, We (might) get `vim`. Our laptop knows this because our shell startup files set `EDITOR=vim` when we logged in. This variable lives in our local shell's environment that is a table of name-value pairs that every process inherits from its parent.

Now we run `ssh server`. A new shell starts on the remote machine. That shell has its own environment, built from the remote server's `/etc/environment`, the remote user's `.bashrc`, and so on. our local `EDITOR=vim` is nowhere to be found on the remote side. The SSH connection is a clean boundary as the environment variables do not cross it automatically.

`SendEnv` is the config directive that tells the SSH client: "when connecting, also send these local environment variables to the remote shell." we list patterns like `SendEnv LANG LC_*` and the client will send the value of `LANG` and every variable whose name starts with `LC_` from our local environment to the server. The server's `sshd` then sets them in the remote shell's environment before starting it.

This is why our locale settings (`LANG`, `LC_ALL`, `LC_TIME`) often just work when we SSH and most systems ship with `SendEnv LANG LC_*` in their default `/etc/ssh/ssh_config`.

## What Happens at The Protocol Level

SSH has a dedicated channel request type for this: `"env"`. After we open a session channel but before we start the shell (or run a command), we can send as many `"env"` requests as we want. Each one carries exactly two strings: the variable name and its value.

In libssh, `src/channels.c` implements this:

```c
int ssh_channel_request_env(ssh_channel channel,
                             const char *name, const char *value)
{
    ssh_buffer buffer = ssh_buffer_new();
    ssh_buffer_pack(buffer, "ss", name, value);   /* pack two strings */
    rc = channel_request(channel, "env", buffer, 1);
    SSH_BUFFER_FREE(buffer);
    return rc;
}
```

`ssh_buffer_pack(buffer, "ss", name, value)` here the format string `"ss"` means "two SSH strings". An SSH string is a 4-byte length followed by that many bytes of data. So if `name` is `"LANG"` and `value` is `"en_US.UTF-8"`, the buffer contains `[0x00 0x00 0x00 0x04] LANG [0x00 0x00 0x00 0x0B] en_US.UTF-8`. This packed form is sent as the payload of the `"env"` channel request.

The server receives this request, looks up `name` in the list of `AcceptEnv` patterns configured in `sshd_config`, and if the name is accepted, sets it in the environment that the remote shell will inherit.

One another detail is : the server only accepts variables whose names match its `AcceptEnv` configuration. The client saying `SendEnv FOO` does not guarantee the server will honour it. The server is the gatekeeper. The client just offers; the server decides whether to accept or not.

## The Pattern Syntax

`SendEnv` takes patterns, not literal names. The patterns use shell-style syntax:

- `LANG` -> match exactly the variable named `LANG`
- `LC_*` -> match any variable whose name starts with `LC_`
- `*` -> match all the variables
- `!LC_ALL` -> explicitly exclude `LC_ALL` even if another pattern would have matched it

Multiple patterns can appear on one line separated by spaces, and multiple `SendEnv` lines accumulate (they do not override each other). So:

```
SendEnv LANG LC_*
SendEnv !LC_ALL
```

...means: send `LANG` and all `LC_*` variables, but not `LC_ALL` specifically.

libssh already has a function for exactly this type of matching in `src/match.c`, `match_pattern_list()` handles a comma-separated list of patterns with optional `!` negation. It is declared in include/libssh/priv.h:

```c
int match_pattern_list(const char *string, const char *pattern,
                       size_t len, int dolower);
```

Returns 1 if the string matches a positive pattern, -1 if it matches a negation pattern, 0 if no match. This is exactly what we need to filter which local variables to send.

## Current State in libssh

At `src/config.c`:
```c
{"sendenv", SOC_NA, true},
```

`SOC_NA` is the current state of sendenv. libssh knows the keyword exists in SSH config files, but since libssh is a library, it declared it "not applicable" and ignored it. The `ssh_channel_request_env()` API is fully implemented and works perfectly, but there is no option that tells the session "here is a list of patterns to send automatically."

There is no `SSH_OPTIONS_SEND_ENV` constant. There is no `send_env` field in `struct ssh_options_struct`. An application using libssh today has to manually call `ssh_channel_request_env()` for each variable it wants to send, and it has to implement the pattern matching logic itself.

## What Needs to Change

### Step 1 -> Store the list of patterns inside the session.h

`SendEnv` can accumulate multiple patterns across multiple config lines. We need a list, not a single string. In `include/libssh/session.h` inside `struct ssh_options_struct`:

```c
struct ssh_list *send_env;  /* list of char* patterns, e.g. "LANG", "LC_*", "!LC_ALL" */
```

`struct ssh_list` is libssh's singly-linked list type. We append with `ssh_list_append()` and iterate with `ssh_list_get_iterator()` / `ssh_iterator_value()`.

### Step 2 -> Add the option constant in libssh.h

In `enum ssh_options_e` in `include/libssh/libssh.h` :

```c
SSH_OPTIONS_SEND_ENV,   /* append a pattern string to the send_env list */
```

We will "append" not "set." Each call to `ssh_options_set(session, SSH_OPTIONS_SEND_ENV, "LC_*")` adds one pattern to the list. This matches how libssh handles `SSH_OPTIONS_IDENTITY` such that each call adds one identity file and they accumulate.

### Step 3 -> Handle in ssh_options_set()

In `src/options.c` :

```c
case SSH_OPTIONS_SEND_ENV: {
    const char *pattern = (const char *)value;
    if (pattern == NULL) {
        ssh_set_error_invalid(session);
        return -1;
    }
    if (session->opts.send_env == NULL) {
        session->opts.send_env = ssh_list_new();
        if (session->opts.send_env == NULL) {
            ssh_set_error_oom(session);
            return -1;
        }
    }
    char *dup = strdup(pattern);
    if (dup == NULL) {
        ssh_set_error_oom(session);
        return -1;
    }
    if (ssh_list_append(session->opts.send_env, dup) < 0) {
        free(dup);
        return -1;
    }
    break;
}
```

`ssh_list_new()` allocates a fresh linked list. `strdup()` duplicates the pattern string so the session owns its own copy. `ssh_list_append()` adds it to the tail.

### Step 4 -> Promote in config.c from SOC_NA

In src/config.c, change `SOC_NA` to `SOC_SENDENV` and add a parsing case. `SendEnv` can have multiple space-separated patterns on one line, and each config line accumulates into the same list. We loop through tokens:

```c
case SOC_SENDENV:
    if (*parsing) {
        /* tokenize the rest of the line that is multiple patterns are allowed */
        while ((p = ssh_config_get_str_tok(&s, NULL)) != NULL) {
            if (p[0] == '\0') break;
            ssh_options_set(session, SSH_OPTIONS_SEND_ENV, p);
        }
    }
    break;
```

`ssh_config_get_str_tok()` returns successive whitespace-delimited tokens from the config line. We call it in a loop until it returns NULL or an empty string, adding each token as a pattern.

### Step 5 -> Free in session cleanup

In the session destructor (wherever `opts` is freed), add:

```c
if (session->opts.send_env != NULL) {
    struct ssh_iterator *it = ssh_list_get_iterator(session->opts.send_env);
    while (it != NULL) {
        free((char *)ssh_iterator_value(it));
        it = it->next;
    }
    ssh_list_free(session->opts.send_env);
}
```

Each string was `strdup()`'d, so each one needs a `free()`.

## How the CLI will use it

After the session channel is open and before requesting the shell, the CLI iterates through the environment. For each variable in the current process's environment, it checks whether the name matches any pattern in `session->opts.send_env`. If it matches a positive pattern (and does not match a negation), it calls `ssh_channel_request_env()`.

`environ` is a POSIX global, it is a null-terminated array of strings in the form `"NAME=VALUE"`. We walk it and split each entry at the `=`:

```c
extern char **environ;

void send_env_vars(ssh_session session, ssh_channel channel)
{
    struct ssh_iterator *it;

    if (session->opts.send_env == NULL) return;

    /* build a single comma-separated pattern string for match_pattern_list */
    char pattern_buf[4096] = {0};
    it = ssh_list_get_iterator(session->opts.send_env);
    while (it != NULL) {
        if (pattern_buf[0]) strncat(pattern_buf, ",", sizeof(pattern_buf)-1);
        strncat(pattern_buf, (char *)ssh_iterator_value(it), sizeof(pattern_buf)-1);
        it = it->next;
    }

    /* walk the local environment */
    for (char **ep = environ; *ep != NULL; ep++) {
        char *eq = strchr(*ep, '=');  /* find the = separator */
        if (eq == NULL) continue;

        /* extract just the name part */
        size_t namelen = eq - *ep;
        char name[256];
        if (namelen >= sizeof(name)) continue;
        memcpy(name, *ep, namelen);
        name[namelen] = '\0';

        /* does this name match any of our patterns? */
        int m = match_pattern_list(name, pattern_buf, strlen(pattern_buf), 0);
        if (m == 1) {
            /* positive match -> send it */
            ssh_channel_request_env(channel, name, eq + 1);
        }
        /* m == -1: explicitly negated -> skip */
        /* m == 0: no match -> skip */
    }
}
```

`strchr(*ep, '=')` -> `strchr` searches a string for the first occurrence of a character. `*ep` is the full `"NAME=VALUE"` string; `strchr(*ep, '=')` returns a pointer to the `=` character. Everything before it is the name, everything after it (`eq + 1`) is the value.

`match_pattern_list(name, pattern_buf, strlen(pattern_buf), 0)` -> the `0` at the end means "do not convert to lowercase".

This function is called after `ssh_channel_open_session()` succeeds and before `ssh_channel_request_pty()` or `ssh_channel_request_shell()`.

## What We will be Solving at the Project Level

Today, any libssh application that wants `SendEnv` semantics has to implement pattern matching itself it should walk `environ`, split at `=`, do glob matching against whatever patterns the user has configured, then call `ssh_channel_request_env()` per variable. That is a non-trivial amount of logic that every SSH client reimplements.

By promoting `sendenv` from `SOC_NA` to a real option, any libssh application can call `ssh_options_parse_config()` and get `SendEnv` for free. The user's `~/.ssh/config` lines are respected automatically. Applications do not need to know about pattern matching or environ walking, they just call the send function at the right moment.

## Changes Summary

| File | Changes needed |
|------|--------|
| include/libssh/libssh.h| Add `SSH_OPTIONS_SEND_ENV` to `enum ssh_options_e` |
| include/libssh/session.h | Add `struct ssh_list *send_env` to `struct ssh_options_struct` |
| src/options.c| Add `case SSH_OPTIONS_SEND_ENV:` -> allocate list on first call, append pattern |
| src/config.c | `SOC_NA` to `SOC_SENDENV`, add tokenizing parse loop |
| src/session.c | Free the list and its strings in session cleanup |
| tests/unittests/torture_options.c | Unit test: call `SSH_OPTIONS_SEND_ENV` twice, verify both patterns in list |
| tests/unittests/torture_config.c| Config test: parse `SendEnv LANG LC_*`, verify two patterns stored |
