# BatchMode

## The Problem it will Solve

Not every SSH connection has a human sitting at a keyboard. Scripts, cron jobs, deployment pipelines, and monitoring systems all use SSH to run commands on remote servers. They cannot respond to prompts hence there is nobody there to type a password.

The danger is: if we write a script that uses SSH, and for some reason public key authentication fails, SSH falls back to asking for a password. The script hangs. The job hangs. The deployment pipeline hangs. Nobody notices for hours.

`BatchMode yes` tells SSH: "we are running non-interactively, never ask the user anything, and if we cannot authenticate without prompting than fail immediately with an error."

It turns a hang into a fast, clear failure. Which is always better in automated systems.

## What "Interactive" Means in SSH Auth

SSH has several authentication methods and each has a different relationship with the user:

**Public key authentication** -> the client signs a challenge with the private key from `~/.ssh/id_rsa` (or similar). If the key has no passphrase, this is fully non-interactive and no user input is needed. If the key has a passphrase, SSH must prompt for it. In batch mode, a passphrase-protected key cannot be used (as there is no one to enter the passphrase) so SSH should skip it or fail.

**Agent authentication** -> the client asks the ssh-agent to sign. The agent holds decrypted keys in memory, so no prompt is needed. This is safe in batch mode.

**Password authentication** -> SSH must ask the user for their password. Completely interactive.This Must be disabled in batch mode.

**Keyboard-interactive authentication** -> the server sends arbitrary text prompts (questions like "Enter OTP:", "Security question:", etc.) and the client answers them. Completely interactive.This also must be disabled in batch mode.

So when `BatchMode yes` is set, the rule is: if an authentication method requires user input to proceed, skip it. If all non-interactive methods fail and the only remaining options require prompting than exit with an error instead of asking.

## How OpenSSH Implements It

OpenSSH's `BatchMode yes` does three things:

1. **Disables password authentication** -> `PasswordAuthentication` is forced to no. Even if the server offers it and even if `PasswordAuthentication yes` is set in config, the client refuses to use it.

2. **Disables keyboard-interactive authentication** -> `KbdInteractiveAuthentication` is also forced to no for the same reason.

3. **Suppresses passphrase prompts** -> if the private key file is encrypted with a passphrase and no agent has the key loaded, than OpenSSH skips that key instead of prompting. The `auth_function` callback is not called; the key is simply not tried.

Host key checking prompts are also suppressed so that if an unknown host key is encountered and `StrictHostKeyChecking` is set to ask, it behaves as `StrictHostKeyChecking yes` (refuse instead of asking).

## Current State in libssh

In src/config.c :
```c
{"batchmode", SOC_UNSUPPORTED, true},
```

`SOC_UNSUPPORTED` means: the keyword is recognized from `~/.ssh/config`, no warning is printed, but the value is silently thrown away. A developer running a script that has `BatchMode yes` in their SSH config gets no batch behavior at all and libssh will still try password auth, still call passphrase callbacks, and will potentially hang.

There is no `SSH_OPTIONS_BATCH_MODE` constant anywhere in the codebase. There is no `batch_mode` field in `session->opts`.

## What Needs to be changed

### Step 1 -> Add the option constant

In include/libssh/libssh.h in `enum ssh_options_e`, add at the end before the closing brace:

```c
SSH_OPTIONS_BATCH_MODE,  /* bool: if true, never prompt user interactively */
```

### Step 2 -> Add the field to the session options struct

In include/libssh/session.h inside `struct ssh_options_struct`:

```c
bool batch_mode;
```

The existing flags bitmask (`SSH_OPT_FLAG_PASSWORD_AUTH`, `SSH_OPT_FLAG_KBDINT_AUTH` etc.) controls which methods are allowed. `batch_mode` is a separate concept: it does not disable the methods at the library level, it is a signal to the application and CLI to skip any step that would require a user prompt.

### Step 3 -> Handle it in ssh_options_set()

In src/options.c, following the existing pattern for boolean options we will be:

```c
case SSH_OPTIONS_BATCH_MODE:
    if (value == NULL) {
        ssh_set_error_invalid(session);
        return -1;
    } else {
        bool *x = (bool *)value;
        session->opts.batch_mode = *x;
    }
    break;
```

`ssh_set_error_invalid(session)` is the libssh's way of reporting that a required argument was NULL. Every option case starts with this null-check pattern and we do not skip it.

### Step 4 -> Promote in config.c

In src/config.c, we need to change `SOC_UNSUPPORTED` to `SOC_BATCHMODE` (a new constant, or reuse a pattern like `SOC_SESSION` with a dedicated sub-handler). Adding the parsing case to `ssh_config_parse_global()`:

```c
case SOC_BATCHMODE:
    p = ssh_config_get_str_tok(&s, NULL);
    CHECK_COND_OR_FAIL(p == NULL, "Missing argument");
    if (*parsing) {
        bool val = (strcasecmp(p, "yes") == 0);
        ssh_options_set(session, SSH_OPTIONS_BATCH_MODE, &val);
    }
    break;
```

`ssh_config_get_str_tok()` reads the next whitespace-separated token from the config line and the value after the keyword. `strcasecmp` compares case-insensitively so `yes`, `YES`, `Yes` all will work. `CHECK_COND_OR_FAIL` is a macro that logs an error and skips the line if the condition is true and is used for malformed config entries.

## How the CLI will use batch_mode

In our `tools/ssh/` binary, after connecting and before starting the auth loop, we read `session->opts.batch_mode`. It drives two things:

### Disabling interactive auth methods

When building the authentication chain:

```c
if (opts->batch_mode) {
    /* force-disable methods that require user input */
    int disabled = 0;
    ssh_options_set(session, SSH_OPTIONS_PASSWORD_AUTH, &disabled);
    ssh_options_set(session, SSH_OPTIONS_KBDINT_AUTH,   &disabled);
}
```

`ssh_options_set(session, SSH_OPTIONS_PASSWORD_AUTH, &(int){0})` clears the `SSH_OPT_FLAG_PASSWORD_AUTH` bit from `session->opts.flags`. When libssh's internal auth negotiation runs, it checks this flag and if it is cleared, password auth is not offered as a method.

### Suppressing passphrase prompts

`ssh_userauth_publickey_auto()` calls the `auth_function` callback when it needs a passphrase for an encrypted private key. The callback is set via `session->common.callbacks->auth_function`. In our CLI's callback:

```c
static int auth_callback(const char *prompt, char *buf, size_t len,
                         int echo, int verify, void *userdata)
{
    struct cli_state *st = userdata;

    if (st->session_opts->batch_mode) {
        /* cannot prompt in batch mode so fail this key */
        return -1;  /* returning -1 tells libssh to skip this key */
    }

    /* interactive: prompt the user */
    return ssh_getpass(prompt, buf, len, echo, verify);
}
```

Returning `-1` from the callback tells `ssh_userauth_publickey_auto()` it cannot decrypt this key and should move to the next one. If all keys fail, `ssh_userauth_publickey_auto()` returns `SSH_AUTH_DENIED`.

`ssh_getpass()` (src/getpass.c) is libssh's cross-platform function for reading a password from the terminal. It disables echo if `echo=0`, re-prompts for verification if `verify=1`, and reads from `/dev/tty` directly (so it works even if stdin is a pipe). In batch mode we will never reach this call.

### Failing fast on unknown host keys

In batch mode, if a host key is unknown or changed, we should not prompt "Do we want to continue connecting?" inspite of that we should reject immediately. In the host key verification callback:

```c
if (st->batch_mode && verification_result == SSH_KNOWN_HOSTS_UNKNOWN) {
    fprintf(stderr, "Host key verification failed.\n");
    return -1;
}
```

### The Full Auth Loop in Batch Mode

The client arrives at the server's door and knocks with a none request, not because it expects to be let in, just checking if the door is already open. It isn't.

So it reaches into its keychain. The ssh-agent goes first: it holds decrypted keys in memory, no questions asked, fully silent. It tries each one. The server shakes its head.

Next come the key files on disk. The client picks one up, but this key is locked, wrapped in a passphrase. Normally it would call out: "Hey, what's the passphrase for this?" But not today. The auth callback looks at batch_mode, says -1, and the key goes back on the shelf, untried. Same for the next encrypted key. And the next.

Now the server offers password authentication. The client doesn't even respond and that door is bolted shut. SSH_OPT_FLAG_PASSWORD_AUTH was cleared before the conversation even started.

The server tries keyboard-interactive. "Security question? OTP?" The client has already walked away. That flag is cleared too.

There is nothing left to try. No more methods. No more keys. No human to ask.

The client prints to stderr: Permission denied (publickey) — and exits with code 1.

No hang. No waiting. A fast, clean failure, exactly the way a pipeline needs it.

## What We Are Solving at the Project Level

Currently any libssh application that reads SSH config (scripts, deployment tools, anything calling `ssh_options_parse_config()`) ignores `BatchMode yes` entirely. Those applications will hang indefinitely when password auth is offered, even though the user explicitly told them not to ask for passwords. Promoting `BatchMode` to a first-class library option fixes this for all libssh applications, not just our CLI.

## Changes Summary

| File | Changes needed |
|------|--------|
| include/libssh/libssh.h | Add `SSH_OPTIONS_BATCH_MODE` to `enum ssh_options_e` |
| include/libssh/session.h | Add `bool batch_mode` to `struct ssh_options_struct` |
| src/options.c | Add `case SSH_OPTIONS_BATCH_MODE:` in `ssh_options_set()` |
| src/config.c | `SOC_UNSUPPORTED` → `SOC_BATCHMODE`, add parsing case |
| tests/unittests/torture_options.c | Unit test: set option, verify stored |
| tests/unittests/torture_config.c | Config test: parse `BatchMode yes/no`, verify stored |
