# NumberOfPasswordPrompts

## The Problem it will solve : 

When we type the wrong password over SSH, the server sends back a failure response. Our SSH client then prompts us again. We can type the wrong password again. And again. Without any limit, the client will keep asking forever this goes on until the server disconnects us.

`NumberOfPasswordPrompts` will put a limit on this. The default in OpenSSH is 3 such that if we fail three times, the client gives up and exits rather than prompting a fourth time.

This will matter for two reasons:

**User experience**: if our key is not working and we have also forgotten our password, we do not want to sit there failing indefinitely. Three tries and a clear error message is better.

**Security in scripts**: if a script is running SSH and something is wrong with credential setup, a prompt limit ensures it does not hang asking for input forever.

## Two Different Prompt Mechanisms

### Direct password authentication

The client calls `ssh_userauth_password()` with the password. If the server rejects it, the client returns `SSH_AUTH_DENIED`. To try again, the application prompts the user for a new password and calls `ssh_userauth_password()` again. The client controls the retry so the server just responds with success or failure each time.

`ssh_userauth_password()` (src/auth.c) has no loop inside it. It sends one attempt and returns. The application's outer loop drives retries.

### Keyboard-interactive authentication

This is more complex. The server takes control of the prompting. When we call `ssh_userauth_kbdint()`, the server may respond with `SSH_AUTH_INFO` which means "here are my questions, answer them." After the client answers and calls `ssh_userauth_kbdint()` again, the server may respond with another `SSH_AUTH_INFO` with a new set of questions. This can repeat.

For a typical "Password:" prompt via keyboard-interactive: we type the wrong password, the server responds with `SSH_AUTH_INFO` again showing a new "Password:" prompt. The client calls `ssh_userauth_kbdint()` in a loop:

```c
int err = ssh_userauth_kbdint(session, NULL, NULL);
while (err == SSH_AUTH_INFO) {
    int n = ssh_userauth_kbdint_getnprompts(session);
    for (int i = 0; i < n; i++) {
        char echo;
        const char *prompt = ssh_userauth_kbdint_getprompt(session, i, &echo);
        /* echo=0 means this is a password/secret field, hide the input */
        char answer[256] = {0};
        ssh_getpass(prompt, answer, sizeof(answer), echo, 0);
        ssh_userauth_kbdint_setanswer(session, i, answer);
    }
    err = ssh_userauth_kbdint(session, NULL, NULL);
}
```

`ssh_userauth_kbdint_getprompt()` returns the prompt text and sets `echo` to 0 or 1. If `echo` is 0, the answer should not be shown on screen (because it is a secret). `ssh_getpass()` (src/getpass.c) uses `tcsetattr()` to disable terminal echo when `echo=0`, this is the same mechanism the PTY(pseudoTerminal:https://github.com/sudharshanhegde/blogs/blob/main/psuedoTerminal.md) section covered.

`ssh_userauth_kbdint_setanswer()` stores the user's answer. It uses `ssh_burn()` to securely wipe the previous answer before overwriting and `ssh_burn()` fills memory with zeros so passwords are not left floating in RAM after they are no longer needed.

Without a prompt limit, this `while (err == SSH_AUTH_INFO)` loop could spin forever on a misbehaving server.

## Current State in libssh

In src/config.c :
```c
{"numberofpasswordprompts", SOC_UNSUPPORTED, true},
```

No `SSH_OPTIONS_NUMBER_OF_PASSWORD_PROMPTS` constant exists. No counter field in the session struct.

## What Needs to be Changed

This is mostly a CLI-level concern as the counter lives in our authentication loop, not deep inside libssh's auth state machine. But we do need the library to store and expose the configured value.

### Step 1 -> Add the option constant

In include/libssh/libssh.h in `enum ssh_options_e`:

```c
SSH_OPTIONS_NUMBER_OF_PASSWORD_PROMPTS,  /* int: max times to prompt for password */
```

### Step 2 -> Add the field to session options

In include/libssh/session.h inside `struct ssh_options_struct`:

```c
int number_of_password_prompts;  /* default 3, 0 = unlimited */
```

### Step 3 -> Handle in ssh_options_set()

In src/options.c:

```c
case SSH_OPTIONS_NUMBER_OF_PASSWORD_PROMPTS:
    if (value == NULL) {
        ssh_set_error_invalid(session);
        return -1;
    } else {
        int *x = (int *)value;
        session->opts.number_of_password_prompts = *x;
    }
    break;
```

### Step 4 -> Set the default at session init

In src/session.c in `ssh_new()`, where other defaults are set:

```c
session->opts.number_of_password_prompts = 3;  /* This is OpenSSH default */
```

### Step 5 -> Promote in config.c

In src/config.c, change `SOC_UNSUPPORTED` → `SOC_NUMBEROFPASSWORDPROMPTS`, add a parsing case:

```c
case SOC_NUMBEROFPASSWORDPROMPTS:
    i = ssh_config_get_int(&s, -1);
    CHECK_COND_OR_FAIL(i < 0, "Invalid number");
    if (*parsing) {
        ssh_options_set(session, SSH_OPTIONS_NUMBER_OF_PASSWORD_PROMPTS, &i);
    }
    break;
```

`ssh_config_get_int()` reads the next token from the config line and converts it to an integer. It returns -1 on failure, which we check with `CHECK_COND_OR_FAIL`.

## How the CLI will use It

The counter lives in the authentication loop in `tools/ssh/`. We count how many times we have asked the user for a password across both auth mechanisms:

```c
static int authenticate(ssh_session session)
{
    int password_attempts = 0;
    int max_prompts = session->opts.number_of_password_prompts;
    int rc;

    ssh_userauth_none(session, NULL);
    int methods = ssh_userauth_list(session, NULL);

    /* try public key first (no password is involved) */
    if (methods & SSH_AUTH_METHOD_PUBLICKEY) {
        rc = ssh_userauth_publickey_auto(session, NULL, NULL);
        if (rc == SSH_AUTH_SUCCESS) return rc;
    }

    /* keyboard-interactive: server sends prompts, we answer */
    if (methods & SSH_AUTH_METHOD_INTERACTIVE) {
        rc = ssh_userauth_kbdint(session, NULL, NULL);
        while (rc == SSH_AUTH_INFO) {

            /* check prompt limit before asking */
            if (max_prompts > 0 && password_attempts >= max_prompts) {
                fprintf(stderr, "Too many authentication failures\n");
                return SSH_AUTH_DENIED;
            }

            int n = ssh_userauth_kbdint_getnprompts(session);
            for (int i = 0; i < n; i++) {
                char echo;
                const char *prompt = ssh_userauth_kbdint_getprompt(session, i, &echo);
                char answer[256] = {0};
                ssh_getpass(prompt, answer, sizeof(answer), echo, 0);
                ssh_userauth_kbdint_setanswer(session, i, answer);
                ssh_burn(answer, sizeof(answer)); /* zero the stack copy too */
            }
            password_attempts++;
            rc = ssh_userauth_kbdint(session, NULL, NULL);
        }
        if (rc == SSH_AUTH_SUCCESS) return rc;
    }

    /* direct password authentication */
    while (methods & SSH_AUTH_METHOD_PASSWORD) {

        if (max_prompts > 0 && password_attempts >= max_prompts) {
            fprintf(stderr, "Too many authentication failures\n");
            return SSH_AUTH_DENIED;
        }

        char password[256] = {0};
        ssh_getpass("Password: ", password, sizeof(password), 0, 0);
        password_attempts++;

        rc = ssh_userauth_password(session, NULL, password);
        ssh_burn(password, sizeof(password));

        if (rc == SSH_AUTH_SUCCESS) return rc;
        if (rc == SSH_AUTH_DENIED) continue;  /* wrong password, try again */
        break;  /* error */
    }

    return SSH_AUTH_DENIED;
}
```

`ssh_burn(answer, sizeof(answer))` -> after every use of a password buffer, we explicitly zero it. This is a security practice: because passwords in stack-allocated buffers would otherwise stay in memory until something else overwrites that stack space. The function fills the buffer with zeros. We do this even for the local `answer` array on the stack, because other code could potentially read those memory regions.

### Counting Across Both Methods

We use a single `password_attempts` counter across both keyboard-interactive and direct password auth. This matches OpenSSH's behavior: `NumberOfPasswordPrompts 3` means three total password prompts, regardless of whether they come from keyboard-interactive or password auth.

`max_prompts = 0` means unlimited, this is a convenience for scripts that want to drive retries themselves.

## What We will be Solving at the Project Level

Without this option, libssh-based tools either loop forever on wrong passwords or have arbitrary hardcoded limits. An application that reads `~/.ssh/config` with `NumberOfPasswordPrompts 1` currently gets no effect. Promoting this option means the limit is respected by any libssh application that will read config, not just our CLI.

## Changes Summary

| File | Changes needed |
|------|--------|
| include/libssh/libssh.h | Adding `SSH_OPTIONS_NUMBER_OF_PASSWORD_PROMPTS` to `enum ssh_options_e` |
| include/libssh/session.h | Adding `int number_of_password_prompts` to `struct ssh_options_struct` |
| src/session.c | Set default value of 3 in `ssh_new()` |
| src/options.c | Adding `case SSH_OPTIONS_NUMBER_OF_PASSWORD_PROMPTS:` in `ssh_options_set()` |
| src/config.c | `SOC_UNSUPPORTED` to `SOC_NUMBEROFPASSWORDPROMPTS`, add parsing case |
| tests/unittests/torture_options.c | Unit test: set option, verify stored value |
| tests/unittests/torture_config.c | Config test: parse `NumberOfPasswordPrompts 1`, verify stored |
