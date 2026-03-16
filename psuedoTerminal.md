# Psuedoterminals

When we run program in terminal like vim or htop it not only print lines. It takes over whole screen, moves cursor around and responds to arrow keys and also knows window size.
This is done by talking to terminal. Terminal is not just output pipe. It has special capabilities such as :
- It has size (rows * columns)
- It handles signals : ctrl + c (sends SIGINT),ctrl + z (sends SIGSTP)
- It has modes : Raw mode -> (every keypress will be sent immediately)
                 cooked mode -> (wait for enter, Handle backspace)
- It has escape sequences such as : \033[2J (clears the screen)
                                    \033[1;32m (sets text green)


When we SSH into remote machine and run vim, vim will be running on the server.
We will be at terminal on our client, server needs to give vim property such that it behaves as a real terminal, even though actual display is on another machine, this functionality is handled by psuedoterminal(PTY).

## PTY at OS level

PTY is pair of file descriptors created by kernel :

master fd  <------------- bytes flow both ways ------------>  slave fd
(your SSH client / server side that controls it)           (given to the remote program as its stdin/stdout/stderr)

- Slave side looks like real terminal to any program which opens it.
Program will check isatty(fd) ? -> this will return true.
vim running on slave side thinks it has real terminal.

- master side is what us or SSH server read from and write to. What we write to master will appear as input on slave. What program writes to slave comes out on master.

## Creating a PTY

```c
#include <pty.h>  /* or <util.h> on some systems */

int master_fd, slave_fd;
char slave_name[64];  /* example :  "/dev/pts/7" */

openpty(&master_fd, &slave_fd, slave_name, NULL, NULL);
```

openpty() does total of three things:
1. It create a master/slave pair.
2. It returns both file descriptor.
3. It tells the name of slave.

After this :

- Give slave_fd to child process (as stdin/stdout/stderr)
- keep master_fd for ourselves to read/write.

on SSH server side, server calls openpty() forks a child process gives that child slave_fd and bridge master_fd to SSH channel. Data from client travels through channel,arrives at master and passes through PTY kernel logic and appears on slave as if typed on real keyboard.

### termios -> controlling terminal behaviour.

PTY has configurable behaviour, configuration will be stored in struct termios and changed with two system calls:

```c
#include <termios.h>

struct termios t;

tcgetattr(fd, &t);  /* read current settings from the terminal fd */
tcsetattr(fd, TCSANOW, &t);  /* write new settings, apply immediately */
```

TCSANOW -> apply right now.
TCSADRAIN -> apply after all pending output is written.

### Cooked Mode :

This is default and terminal will be in cooked mode(also called canonical mode).

In this mode :
- kernel buffers input we press enter.
- backspace erases previous character in buffer.
- ctrl + c sends SIGINT to foreground process.
- ctrl + z sends SIGSTP (suspend)
- terminal echoes keypresses back to screen.

Kernel handles editing so our program does not need to handle it.

### Raw mode :

SSH will need raw mode, because in cooked mode : when user presses arrow key, it sends 3 byte escape sequence \033[A. In cooked mode kernel will buffer those butes, remote vim will never see it until enter is pressed. This is broken.

In raw mode :
- Every keypress will be sent immediately, one byte at a time.
- No line buffering.
- backspace will not be handled by kernel -> it will be sent as raw byte 0x7f to program.
- ctrl + c is also not converted to SIGINT -> it is sent as raw byte 0x03 to channel.

Remote program(running through SSH) gets every byte exactly as we type it and handles it itself. cfmakeraw() is helper which applies all necessary flag at once :

```c
struct termios t;
tcgetattr(STDIN_FILENO, &t);  /* read current settings */

/* save a copy BEFORE modifying, so you can restore later */
struct termios saved = t;

cfmakeraw(&t);  /* modify: turn off all buffering, echo, signal processing */

tcsetattr(STDIN_FILENO, TCSANOW, &t);  /* apply raw mode */

/* ... run SSH session ... */

/* restore when done */
tcsetattr(STDIN_FILENO, TCSANOW, &saved);
```

If we forget to restore and the program crashes, terminal will stay in raw mode. Shell prompt works but looks completely broken, no echo, backspace will not wor. We need to type reset blindly and press enter to recover.

ssh_client.c is doing same : as we can see here

```c
static void shell(ssh_session session)
{
    ssh_channel channel = NULL;
    struct termios terminal_local;
    int interactive = isatty(0);

    channel = ssh_channel_new(session);
    if (channel == NULL) {
        return;
    }

    if (interactive) {
        tcgetattr(0, &terminal_local);
        memcpy(&terminal, &terminal_local, sizeof(struct termios));
    }

    if (ssh_channel_open_session(channel)) {
        printf("Error opening channel : %s\n", ssh_get_error(session));
        ssh_channel_free(channel);
        return;
    }
    if (interactive) {
        ssh_channel_request_pty(channel);
        sizechanged(channel);
    }

    if (ssh_channel_request_shell(channel)) {
        printf("Requesting shell : %s\n", ssh_get_error(session));
        ssh_channel_free(channel);
        return;
    }

    if (interactive) {
        cfmakeraw(&terminal_local);
        tcsetattr(0, TCSANOW, &terminal_local);
        setsignal();
    }
    signal(SIGTERM, do_cleanup);
    select_loop(session, channel);
    if (interactive) {
        do_cleanup(0);
    }
    ssh_channel_free(channel);
}
```


### SIGWINCH -> window resize signal

When we resize terminal window, OS needs SIGWINCH (window change) to foreground process.
Our SSH client recieves it. When client recieves SIGWINCH, it needs to tell server new window size, server will pass that to PTY on server side, and running program (vim,htop etc) will get notified and redraws itself to fit to new size.

```c
static volatile int window_changed = 0;

static void sigwinch_handler(int sig) {
    (void)sig;
    window_changed = 1;  /* can't do much in a signal handler safely */
}

signal(SIGWINCH, sigwinch_handler);
```

In the event loop :

```c
if (window_changed) {
    window_changed = 0;
    struct winsize ws;
    ioctl(STDOUT_FILENO, TIOCGWINSZ, &ws);  /* gets new terminal size */
    ssh_channel_change_pty_size(channel, ws.ws_col, ws.ws_row);
}
```

ioctl(STDOUT_FILENO, TIOCGWINSZ, &ws) -> is a system call which doesn't fit read/write model, so it uses ioctl ("I/O control").
- TIOCGWINSZ -> get window size. It fills ws.ws_col and ws.ws_row.

ssh_channel_change_pty_size() sends SSH_MSG_CHANNEL_REQUEST with type of "window change" and new dimensions. Server will update PTY and will send SIGWINCH to running process.

We cannot call ssh_channel_change_pty_size() directly inside signal handler. Signal handlers run asynchronously and calling non-async-signal-safe function inside can corrupt the state. So the pattern is : set flag in handler, check flag in event loop. It is done in ssh_client.c for signal_delayed.

Example :

```c
#ifdef SIGWINCH
static void sigwindowchanged(int i)
{
    (void)i;
    signal_delayed = 1;
}
#endif

static void setsignal(void)
{
#ifdef SIGWINCH
    signal(SIGWINCH, sigwindowchanged);
#endif
    signal_delayed = 0;
}

static void sizechanged(ssh_channel chan)
{
    struct winsize win = {
        .ws_row = 0,
    };

    ioctl(1, TIOCGWINSZ, &win);
    ssh_channel_change_pty_size(chan,win.ws_col, win.ws_row);
    setsignal();
}
```


### isatty() -> when not to request PTY

```c
int interactive = isatty(STDIN_FILENO);
```

isatty() returns 1 if given fd is connected to real terminal 0 for pipe or file.

If we run

```sh
echo "ls" | ssh server
```

stdin is pipe and not a terminal. isatty(0) returns 0. Here we should not request PTY, as there is no user sitting there, it is just a script. Requesting PTY in non - interactive mode will waste the resource and can break the output formatting (PTYs add carriage returns, converts newlines).

openSSH's -t flag forces PTY allocation while -T prevents it. RequestTTY has four values:
- no -> never allocate a PTY
- yes -> always try to allocate (It fails if the server refuses)
- force -> allocates even if it is not connected to terminal locally.
- auto -> allocates only if local stdin is terminal (it is default,it is the isatty())

We have to implement this in our CLI which is SOC_NA config, check will look like this :

```c
int should_request_pty(struct cli_opts *opts) {
    switch (opts->request_tty) {
        case TTY_NO:    return 0;
        case TTY_YES:   return 1;
        case TTY_FORCE: return 1;
        case TTY_AUTO:  return isatty(STDIN_FILENO);
    }
}
```


### How client side PTY works

On client side -> there is no PTY creation, Client will be attached to actual terminal, real /dev/pts/N terminal emulator gave the shell.
Client puts that terminal in to raw mode and connects event loop via ssh_connector.

PTY is created on server side, by SSH server, for remote process.

Client only :

1. Sends pty-req channel request telling server the terminal type ($term),dimensions and terminal modes.
2. Puts the local stdin to raw mode.
3. Pipes stdin-> channel,channel->stdout.
4. Handles SIGWINCH by notifying server or size changes.
5. Restores terminal on exit.


## What Changes for CLI we need here

Current ssh_client.c already handles basic PTY setup,we will add :

1. RequestTTY option -> isatty() check is hardcoded in ssh_client.c, we will replace it with should_request_pty() logic above, reading from parsed config/CLI flags.

2. Escape Sequences -> currently ssh_client.c uses ssh_connector to pipe stdin directly to channel, connectors are simple pass-through, they do not inspect bytes, To handle escape charecters like ~.(disconnect) or ~^Z(suspend), we replace stdin connector with custom fd callback which runs an escape sequence state machine on each byte before forwarding it.

State machine will have three states:

STATE_NEWLINE  — we just saw a newline (or it's the start)
               if next byte is '~': → STATE_ESCAPE
               otherwise: forward byte, → STATE_MIDLINE

STATE_MIDLINE  — middle of a line
               if byte is '\n' or '\r': forward byte, → STATE_NEWLINE
               otherwise: forward byte, stay in STATE_MIDLINE

STATE_ESCAPE   —> we saw newline + ~, waiting for command

               '.' → disconnect

               '~' → send literal '~', → STATE_MIDLINE

               '^Z' → suspend (SIGTSTP to self)

               '?' → print help
               
               otherwise → send '~' + byte, → STATE_MIDLINE


3. SendEnv -> after PTY is requested but before shell is started we need to send SSH_MSG_CHANNEL_REQUEST with type "env" for each environment variable which matches SendEnv patterns, It lets us propogate variables like LANG or COLORTERM to remote session. API call will be ssh_channel_request_env(channel,"LANG,getenv("LANG)).

