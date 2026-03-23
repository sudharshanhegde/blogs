# RequestTTY

## The Problem it will solve : 

When we run `ssh server vim file.txt`, should vim get a proper terminal or not? That depends on whether a PTY was allocated. Without a PTY, vim does not know the screen size, arrow keys send raw escape bytes that never get interpreted properly, and Ctrl-C does not send SIGINT, it will just send the literal byte `0x03` which vim may or may not handle.

`RequestTTY` controls when the client asks the server to allocate a PTY. The four values are:

- **no** -> never request a PTY. Useful for running non-interactive commands where PTY overhead and the extra carriage-return behavior would corrupt output.
- **yes** -> always try to request a PTY. If the server refuses, fail.
- **auto** -> the default. Request a PTY only if the client's own stdin is a terminal. If we are piping input to ssh (`echo "cmd" | ssh server`), stdin is a pipe, so no PTY is requested.
- **force** -> request a PTY even if the client's stdin is not a terminal. Useful when we redirect stdin but still want the remote side to behave interactively.

The current `ssh_client.c` example in libssh hardcodes the `auto` logic:

```c
int interactive = isatty(0);   /* 0 is STDIN_FILENO */
/* ... */
if (interactive) {
    ssh_channel_request_pty(channel);
}
```

`isatty(fd)` is a POSIX function that returns 1 if the file descriptor is connected to a real terminal, 0 if it is a pipe, file, or anything else. Checking `isatty(STDIN_FILENO)` is how `auto` mode is implemented and it is the most common behaviour so it was baked in directly. Our job will be to replace that hardcoded check with a proper option.

## What Happens When we Request a PTY

The `RequestTTY` implementation in our CLI calls these functions.

### Step 1: Open a session channel

Everything starts with `ssh_channel_open_session()` at src/channels.c. This sends `SSH2_MSG_CHANNEL_OPEN` with type `"session"` to the server. Only after this succeeds can we make further requests on that channel.

### Step 2: Send "pty-req"

`ssh_channel_request_pty_size_modes()` at src/channels.c sends a `"pty-req"` channel request. The request carries:

```
string  terminal_type    (example: "xterm-256color")
uint32  columns          (example: 80)
uint32  rows             (example: 24)
uint32  pixel_width      (0 if unknown)
uint32  pixel_height     (0 if unknown)
string  terminal_modes   (encoded key-value pairs)
```

The server allocates a PTY, forks the shell with the slave side of the PTY as its stdin/stdout/stderr, and will forward data between the channel and the PTY master.

### The Terminal Modes -> what is being sent

The `terminal_modes` field is a list of key-value pairs. Each pair is one byte for the mode code, followed by four bytes for the value. The list ends with a single `0x00` byte (TTY_OP_END).

These modes tell the server PTY how to behave such that whether to echo input, what byte Ctrl-C maps to, whether to translate newlines, what baud rate to pretend to use, and so on.

libssh encodes these in src/ttyopts.c. When stdin is a real terminal, it reads the current terminal settings with `tcgetattr(STDIN_FILENO, &attr)` and encodes them:

```c
if (isatty(STDIN_FILENO)) {
    tcgetattr(STDIN_FILENO, &attr);
    return encode_termios_opts(&attr, buf, buflen);
}
/* stdin is not a TTY -> use sane defaults */
return encode_default_opts(buf, buflen);
```

`tcgetattr()` fills a `struct termios` with the local terminal's settings. `encode_termios_opts()` then walks through every flag in that struct and encodes the ones that have SSH equivalents. The idea is: the remote PTY should behave exactly like the client's local terminal. If our local terminal has UTF-8 mode enabled (`IUTF8`), the remote PTY should too. If our local backspace key sends `0x7f`, the remote PTY should treat `0x7f` as backspace.

For the `force` RequestTTY mode, stdin is not a terminal so `isatty()` returns 0 so the default modes are used instead of the actual terminal settings. That is fine for `force` because the goal is just to make the remote side interactive, not to mirror a local terminal that does not exist.

### Step 3: Request the shell (or execute the command)

After PTY allocation, `ssh_channel_request_shell(channel)` starts an interactive shell. Or `ssh_channel_request_exec(channel, "command")` runs a specific command. Both use the PTY that was just set up.

## Current State in libssh

In src/config.c:
```c
{"requesttty", SOC_NA, true},
```

`SOC_NA` means it is not just unimplemented -> it is explicitly marked as "not applicable to libssh." The reason is that libssh is a library, not a client. Whether to request a PTY is the application's decision. But for our CLI, we might need to honour this option like OpenSSH does.

There is no `SSH_OPTIONS_REQUEST_TTY` constant. The `ssh_channel_request_pty()` and `ssh_channel_request_pty_size()` API is fully implemented but we just have no option to control *when* to call them.

## What Needs to Change

### Step 1 -> Representing the four states

We define an enum for the four values. In include/libssh/libssh.h :

```c
enum ssh_request_tty_e {
    SSH_REQUEST_TTY_AUTO  = 0,  /* default: allocate PTY if local stdin is a terminal */
    SSH_REQUEST_TTY_NO    = 1,  /* never allocate a PTY */
    SSH_REQUEST_TTY_YES   = 2,  /* always try to allocate a PTY */
    SSH_REQUEST_TTY_FORCE = 3,  /* allocate PTY even if local stdin is not a terminal */
};
```

### Step 2 -> Add the option constant

In `enum ssh_options_e` in include/libssh/libssh.h:

```c
SSH_OPTIONS_REQUEST_TTY,  /* enum ssh_request_tty_e */
```

### Step 3 -> Adding the field to session options

In include/libssh/session.h inside `struct ssh_options_struct`:

```c
enum ssh_request_tty_e request_tty;  /* default SSH_REQUEST_TTY_AUTO */
```

### Step 4 -> Handle in ssh_options_set()

In src/options.c:

```c
case SSH_OPTIONS_REQUEST_TTY:
    if (value == NULL) {
        ssh_set_error_invalid(session);
        return -1;
    } else {
        int *x = (int *)value;
        session->opts.request_tty = (enum ssh_request_tty_e)*x;
    }
    break;
```

### Step 5 -> Promote in config.c

Config files use the string values `"auto"`, `"yes"`, `"no"`, `"force"`. We need to parse them. In src/config.c, change `SOC_NA` → `SOC_REQUESTTTY`, add:

```c
case SOC_REQUESTTTY: {
    p = ssh_config_get_str_tok(&s, NULL);
    CHECK_COND_OR_FAIL(p == NULL, "Missing argument");
    if (*parsing) {
        int v;
        if      (strcasecmp(p, "auto")  == 0) v = SSH_REQUEST_TTY_AUTO;
        else if (strcasecmp(p, "no")    == 0) v = SSH_REQUEST_TTY_NO;
        else if (strcasecmp(p, "yes")   == 0) v = SSH_REQUEST_TTY_YES;
        else if (strcasecmp(p, "force") == 0) v = SSH_REQUEST_TTY_FORCE;
        else { /* unrecognised */ break; }
        ssh_options_set(session, SSH_OPTIONS_REQUEST_TTY, &v);
    }
    break;
}
```

`strcasecmp()` compares strings ignoring case -> `"Yes"`, `"YES"`, `"yes"` all will match. This is important because SSH config files are case-insensitive for option values.

## How the CLI Uses It

This is the function our `tools/ssh/` binary will use to decide whether to request a PTY:

```c
static int should_request_pty(ssh_session session)
{
    switch (session->opts.request_tty) {
        case SSH_REQUEST_TTY_NO:
            return 0;
        case SSH_REQUEST_TTY_YES:
        case SSH_REQUEST_TTY_FORCE:
            return 1;
        case SSH_REQUEST_TTY_AUTO:
        default:
            return isatty(STDIN_FILENO);
    }
}
```

And it feeds into the session setup:

```c
int pty_requested = should_request_pty(session);

ssh_channel_open_session(channel);

if (pty_requested) {
    /* read actual window size */
    struct winsize ws = {0};
    ioctl(STDOUT_FILENO, TIOCGWINSZ, &ws);  /* get terminal dimensions */
    int cols = ws.ws_col > 0 ? ws.ws_col : 80;
    int rows = ws.ws_row > 0 ? ws.ws_row : 24;

    int rc = ssh_channel_request_pty_size(channel, getenv("TERM") ?: "xterm", cols, rows);

    if (rc != SSH_OK && session->opts.request_tty == SSH_REQUEST_TTY_YES) {
        /* "yes" means we require a PTY -> fail if server refuses */
        fprintf(stderr, "PTY allocation failed\n");
        return SSH_ERROR;
    }

    /* put local terminal in raw mode so every keypress is sent immediately */
    struct termios raw;
    tcgetattr(STDIN_FILENO, &raw);
    saved_termios = raw;          /* save for restoration on exit */
    cfmakeraw(&raw);
    tcsetattr(STDIN_FILENO, TCSANOW, &raw);
}
```

`ioctl(STDOUT_FILENO, TIOCGWINSZ, &ws)` -> `ioctl` is a system call for I/O operations that does not fit the normal read/write model. `TIOCGWINSZ` is the "get window size" control code. It fills the `winsize` struct with `ws_col` (columns) and `ws_row` (rows). We read this to tell the server the exact size of our terminal so programs like vim and htop know how much space they have.

`getenv("TERM")` reads the `TERM` environment variable, which tells the remote side what escape sequences our terminal understands -> `"xterm-256color"` for 256-color support, `"xterm"` for basic xterm, etc. The server passes this to the shell which sets it as the `TERM` variable for remote programs. We use `?: "xterm"` as a fallback if `TERM` is not set.

`cfmakeraw()` applies all the raw mode settings in one call -> it is a shorthand for clearing all the input buffering, echo, and signal-conversion flags in the `termios` struct. We covered this in the PTY notes.

### The `-t` and `-T` Flags

The CLI also honours the `-t` flag (force PTY, overrides `RequestTTY yes` in behaviour) and `-T` (disable PTY). These set the same `session->opts.request_tty` field before the option file is even read also then the config file value only applies if neither flag was given on the command line:

```c
/* command-line flags take precedence over config file */
if (cli_args.force_tty) {
    v = SSH_REQUEST_TTY_FORCE;
    ssh_options_set(session, SSH_OPTIONS_REQUEST_TTY, &v);
} else if (cli_args.no_tty) {
    v = SSH_REQUEST_TTY_NO;
    ssh_options_set(session, SSH_OPTIONS_REQUEST_TTY, &v);
}
/* then parse config file -> but only set if not already overridden */
```

## What We will be Solving in this Project

Today `requesttty` is `SOC_NA` so a libssh application reading `~/.ssh/config` gets no effect from it. Any application that wants this behaviour has to implement its own `isatty()` check and hardcode the logic. By promoting it to a first-class option, any libssh application can read and respect the user's `~/.ssh/config` setting without writing that logic themselves.

## Changes Summary

| File | Changes needed |
|------|--------|
| include/libssh/libssh.h| Add `enum ssh_request_tty_e`, add `SSH_OPTIONS_REQUEST_TTY` to `enum ssh_options_e` |
| include/libssh/session.h| Add `enum ssh_request_tty_e request_tty` to `struct ssh_options_struct` |
| src/session.c | Default to `SSH_REQUEST_TTY_AUTO` in `ssh_new()` |
| src/options.c | Add `case SSH_OPTIONS_REQUEST_TTY:` in `ssh_options_set()` |
| src/config.c | `SOC_NA` to `SOC_REQUESTTTY`, add parsing case with `strcasecmp` |
| tests/unittests/torture_options.c | Unit test: set each enum value, verify stored valuee|
| tests/unittests/torture_config.c | Config test: parse `RequestTTY force`, verify stored value|
