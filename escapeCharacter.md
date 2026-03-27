# EscapeChar

## The Problem it will solve

When we are inside an interactive SSH session, our terminal is in raw mode. Every key we press goes directly to the remote machine. There is no reserved key to talk to the SSH client itself.

Let us say we need to disconnect because the remote machine has hung and the connection is frozen. We cannot type `exit` because the remote shell will not be responding. We cannot close the terminal because we need to keep it open. How do we kill just the SSH connection?

OpenSSH solves this with an **escape character** -> a special character that, when typed at the start of a new line, tells the SSH client "the next character is a command for me, not for the remote machine." The default escape character is the tilde `~`.

So we type `~.` (tilde then dot) on a fresh line, and the SSH client recognises this as the "disconnect" command and closes the connection. The remote machine never sees those characters at all -> the SSH client intercepts them before they are sent.

These are the standard escape sequences that gets used:

- `~.` -> disconnect immediately
- `~^Z` (tilde + Ctrl-Z) -> suspend the SSH process, drop us back to the local shell temporarily
- `~&` -> background SSH and return to local shell
- `~C` -> open a command line for adding port forwards on the fly without reconnecting
- `~#` -> list all currently active port-forwarded connections
- `~?` -> print the escape sequence help menu
- `~~` -> send a literal tilde to the remote machine (the escape of the escape)

`EscapeChar` lets us change the `~` to a different character. If the remote program we are running uses `~` heavily, we can set `EscapeChar $` and use `$.` to disconnect instead. Setting it to `none` disables escape sequences entirely.

## Why This will be purely a CLI Feature

Most important thing to understand about EscapeChar: **it has nothing to do with the SSH protocol**. The remote machine never knows about escape sequences. The escape character is entirely client-side logic -> the SSH client watches the raw bytes flowing from the keyboard before forwarding them to the network.

This is exactly why `escapechar` is `SOC_NA` in libssh's config.c. libssh is a library -> it gives us the raw channel I/O but does not sit in the middle of the input stream watching for special characters. That is the application's job. There is no "send escape sequence" API because escape sequences are a feature of the interactive terminal loop, not the SSH transport.

The escape character does not belong in `ssh_options_set()` the way a port number or username does. It belongs in the I/O loop of the CLI we are building.

However it still makes sense to promote it to a library option -> not because libssh will do anything with it automatically, but so that the config file value is stored and the CLI can read it from `session->opts.escape_char` instead of having to parse `~/.ssh/config` separately.

## How The State Machine will work :

The escape character detection is a small state machine. At any moment, the input stream is in one of two states:

- **After a newline** -> the next character might be an escape sequence
- **Mid-line** -> we are in the middle of a line, so tilde is just a tilde(I am just a tilde)

When the client is in "after a newline" state and sees the escape character (`~` by default), it enters "seen escape" state and waits for the next character. If the next character is `.`, it disconnects. If it is `~`, it sends a single `~` to the remote and goes back to mid-line state. If it is anything else that is not a recognised command, it sends both characters (the tilde and the unrecognised character) to the remote as if the escape never happened.

The "after a newline" check is important. If we type `hello~world`, the tilde in the middle is not an escape -> only the tilde at position zero of a fresh line (right after pressing Enter, or at the very beginning of the session) is treated as an escape.

In C, the state machine lives in the CLI's I/O loop:

```c
static int escape_state = 1;   /* start as if after a newline */
static char escape_char = '~'; /* configurable */

/*
 * Returns 1 if the byte should be forwarded to the remote, 0 if it was
 * consumed as part of an escape sequence.
 */
int process_escape(char c, ssh_channel channel)
{
    if (escape_char == 0) {
        /* escape sequences disabled (EscapeChar none) */
        return 1;
    }

    if (escape_state == 1 && c == escape_char) {
        /* we are at start of line and saw the escape character */
        escape_state = 2;  /* waiting for the second character */
        return 0;          /* do not send the tilde yet */
    }

    if (escape_state == 2) {
        escape_state = 0;  /* back to mid-line */
        if (c == '.') {
            /* ~. : disconnect */
            ssh_channel_close(channel);
            return 0;
        } else if (c == escape_char) {
            /* ~~ : send a literal tilde */
            return 1;
        } else if (c == ('z' & 0x1f)) {
            /* ~^Z : suspend */
            kill(getpid(), SIGTSTP);
            return 0;
        } else {
            /* unrecognised: send escape char + this char */
            ssh_channel_write(channel, &escape_char, 1);
            return 1;
        }
    }

    /* track newlines to know when we are at start-of-line */
    escape_state = (c == '\n' || c == '\r') ? 1 : 0;
    return 1;  /* forward normally */
}
```

`kill(getpid(), SIGTSTP)` -> `kill()` is the system call that sends a signal to a process. `getpid()` returns the PID of the current process. `SIGTSTP` is the "terminal stop" signal -> the same signal the kernel sends when we press Ctrl-Z normally. Sending it to ourselves suspends the SSH process and drops us back to the local shell. When we run `fg` to resume, the SSH session continues.

`c == ('z' & 0x1f)` -> this is how we check for a control character. When we hold Ctrl and press a letter, the terminal sends that letter's ASCII code with the top 3 bits cleared, which is the same as `letter & 0x1f`. `'z' & 0x1f` equals `0x1a`, which is Ctrl-Z.

The `escape_state` variable: `1` means "we are at the start of a line", `2` means "we just saw the escape character", `0` means "we are mid-line." The session starts at `1` because the very beginning of the connection counts as being at the start of a line.

## Current State in libssh

In src/config.c:
```c
{"escapechar", SOC_NA, true},
```

`SOC_NA` -> same as RequestTTY. There is no `SSH_OPTIONS_ESCAPE_CHAR` constant. There is no `escape_char` field in `struct ssh_options_struct`. The entire escape logic needs to be built in the CLI, but we still want the session to store the configured value so it flows naturally from `~/.ssh/config`.

## What Needs to Change

### Step 1 ->We need to add the option constant

In `enum ssh_options_e` in include/libssh/libssh.h:

```c
SSH_OPTIONS_ESCAPE_CHAR,  /* char or 0 for 'none' */
```

### Step 2 -> Add the field to session options

In include/libssh/session.h inside `struct ssh_options_struct`:

```c
char escape_char;  /* default '~', 0 means disabled */
```

A single `char` is all we need. The special value `0` represents `EscapeChar none`.

### Step 3 -> Handle in ssh_options_set()

In src/options.c:

```c
case SSH_OPTIONS_ESCAPE_CHAR:
    if (value == NULL) {
        ssh_set_error_invalid(session);
        return -1;
    }
    session->opts.escape_char = *(const char *)value;
    break;
```

The caller passes a pointer to a single `char`.

### Step 4 -> Promote Escape Char in config.c

Config files use either a single printable character (`~`, `$`) or the word `none` or `^X` notation for control characters. In src/config.c, change `SOC_NA` to `SOC_ESCAPECHAR`:

```c
case SOC_ESCAPECHAR: {
    p = ssh_config_get_str_tok(&s, NULL);
    CHECK_COND_OR_FAIL(p == NULL, "Missing argument");
    if (*parsing) {
        char v;
        if (strcasecmp(p, "none") == 0) {
            v = 0;  /* disable escape sequences */
        } else if (p[0] == '^' && p[1] != '\0') {
            /* ^X notation -> control character */
            v = toupper((unsigned char)p[1]) & 0x1f;
        } else {
            v = p[0];  /* literal character */
        }
        ssh_options_set(session, SSH_OPTIONS_ESCAPE_CHAR, &v);
    }
    break;
}
```

`p[0] == '^'` handles the `^X` notation which OpenSSH supports -> `EscapeChar ^B` means Ctrl-B (`0x02`). `toupper(p[1]) & 0x1f` converts the letter to its control code.

### Step 5 -> Default in session init

In src/session.c, in `ssh_new()`:

```c
session->opts.escape_char = '~';  /* OpenSSH default */
```

## How the CLI Uses It

After connecting and entering the interactive I/O loop, the CLI reads `session->opts.escape_char` to initialise its escape state machine:

```c
char escape_char = session->opts.escape_char;
/* escape_char == 0 means EscapeChar none -> all input forwarded raw */
```

The `process_escape()` function above is called on every byte read from local stdin before forwarding it to the channel. This is the only place the option matters as it is entirely CLI-side logic.

The `-e` command-line flag overrides the config file value:

```
ssh -e none server   # disable escape sequences for this session
ssh -e $ server      # use $ as escape character
```

## What We will be Solving in this Project

Today `escapechar` is `SOC_NA` so an application reading `~/.ssh/config` gets nothing. By promoting it to a real option, the value flows from config file -> `session->opts.escape_char` -> CLI escape state machine without the CLI having to do its own config parsing. Any future libssh application that wants escape sequence support can read the stored value rather than parsing config themselves. The CLI then implements `~.`, `~^Z`, `~C`, `~#`, and `~?` driven by whatever character the user has configured.

## Changes Summary

| File | Changes needed |
|------|--------|
| include/libssh/libssh.h | Add `SSH_OPTIONS_ESCAPE_CHAR` to `enum ssh_options_e` |
| include/libssh/session.h | Add `char escape_char` to `struct ssh_options_struct` |
| src/session.c | Default to `'~'` in `ssh_new()` |
| src/options.c | Add `case SSH_OPTIONS_ESCAPE_CHAR:` |
| src/config.c | `SOC_NA` to `SOC_ESCAPECHAR`, parse single char / `^X` / `none` |
| tests/unittests/torture_options.c | Unit test: set `'$'`, verify stored; set `0`, verify stored |
| tests/unittests/torture_config.c | Config test: parse `EscapeChar none`, verify `0`; parse `EscapeChar ~`, verify `'~'` |
