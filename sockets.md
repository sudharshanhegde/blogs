# How sockets work

## How the data gets moved between computers :

Computers communicate by sending electric signals in a wire which carries voltage 0 for low 1 for high gets mapped, by mapping this low and high voltages and switching it between high and low we send streams of bits from one computer to another.

As we now 8 bits together form a byte, every peice of data in computers are just sequences of bytes. Speed at which the wire can switch voltage determines how fast the data can be travelled, modern ethernet cables switch hundreds of millions of times per second, while fiber optics cable  uses light they switch billions of times per second (engineering physics :) ).

Wi-Fi also works same way in principle but they use radio waves instead of wire, transmitter modules vary those radio signals to encode the bits, reciever detects this variations and recovers the bits.

At this point when data is being streamed there is a immediate problem of how the reciever will know when the current msg is ending and next msg is starting, to solve this we use framing where frame is chunk of bits with defined structure. ethernet protocol defines frames as follows :

Preamble : Fixed pattern of bits which tells reciever for frames are about to be started. This is 7 bytes of alternating pattern 10101010, followed by 1 byte of 10101011, recievers hardware keeps on watching for this exact pattern to synchronize.

Destination mac address : 6 bytes which is used to identify which machine should be recieving the frame.` MAC(message access control)`. Every network interface card in the world has unique 6 byte address which gets burned into it by manufacturer. It is used for hardware identification.

Source MAC address : 6 bytes which identifies who sent the frame.

Ethertype field : 2 bytes which indicates what kind of data is present in payload. Like 0x0800 means payload is ip packet.

Payload : Actual data which gets sent (up to 1500 bytes)

Checksum : 4 byte mathematical summary of frame contents, Reciever recalculates this for frame and compares it with the transmitted one, if it differs than the frame is corrupted and is discarded.

Ok so now we have our frames ready which can be recognised by computer but we need routing to get it from one place to the other.

 `Router` is the machine which connects two or more networks. When the packets arrive at router, router reads its destination address, looks up at its own routing table, and forwards those packets towards the destination. It is called hop by hop routing in which each router only knows where to send packet next and it also does not know full path.

 IP (internet protocol) is protocol which makes things work it has a header containing source IP address, destination IP address, time to live (TTL) value, TTL gets started by number like 64 and decreases by 1 at each router and when this reaches zero packets gets discarded and error is sent back to sender. It prevents packets from getting looped forever if the forwarding table has errors.

 IP address in IPv4 is 32 bits in which four decimal numbers are seperated by dots, first part identifying which network it is and last part detecting the specific machine, Routers use this address which makes it really efficient.

 ## What TCP adds on top of IP :

 TCP is protocol which runs on IP, while IP handles getting individual packets from one machine to another, TCP provides reliability such that every byte sent is guranteed to arrive, TCP achiever this by requiring acknowledgement for the data. Sender keeps the copy of every byte it sends, if acknowledgment is not arrived within the timeout period than the sender will re transmit it. It continues until acknowledgement is arrived or the connection is been given up.
 
TCP assigns sequence number to every byte, such that if packet arrives out of order reciever holds onto them in the buffer and reassembles in correct order before delivering it to application.

TCP prevents fast sender overwhelming the slow reciever. Reciever advertises window size, how many bytes it can accept, before its buffer fills up. Sender may not have more than many unacknowledged bytes in the flight, When reciever processes data frees up buffer space, it sends window advertisement allowing sender to send more.

TCP establishes connection between two endpoints before the data can flow, this connections has state, such that both sides maintain sequence numbers, window sizes and other bookkeeping.

## TCP Handshake :

Before data flow through TCP two sides must perform handshake. It is called three way handshake because three messages are exchanged.

SYN : Client sends TCP segment with SYN (synchronize) flag set, It also contains client's initial sequence number(ISN we will use C for client and S for server), randomly chosen 32 bit value.

SYN - ACK : Server recieves the SYN, it responds with segment that has both SYN and ack (acknowledge) flags set. ACK field contains ISN_C + 1 which means that I recieved the ISN_C and is ready for byte ISN_C + 1. Server's SYN portion contains server's own initial sequence number, ISN_S.

ACK : Client acknowledges server's SYN. It sends segment with ACK flag set and acknowledgment number ISN_S + 1. It says I recieved ISN_S and I am ready for byte ISN_S + 1. Connection is established.

We use this random initial sequence number because if sequence numbers started at 0 than the attacker who knew a connection was happening can craft a packet with right sequence number and inject data into the connection.

`TCP segments and the header` :

TCP segment consists of header and payload. 

Source Port(2 bytes) : Sender will be using this

Destination port (2 bytes) : port the reciever should be routing it to.

Sequence number (4 bytes): the sequence number of the first byte of payload in this segment.

Acknowledgement number (4 bytes): the next sequence number the sender is expecting from the other side. This is useful when ACK flag is set.

Data offset (4 bits): how long the header is in 32-bit words. The minimum header is 20 bytes (5 words of 4 bytes each). Options can always extend this.

Flags (6 bits in the classic definition,this is be expanded in newer RFCs): SYN (connection setup), ACK (acknowledgement field valid), FIN (no more data from sender), RST (reset, abort connection), PSH (push data to application immediately), URG (urgent data present).

Window size (2 bytes): the flow control window. This defines how many bytes of unacknowledged data the receiver can accept.

Checksum (2 bytes): error detection for the header and payload.

Urgent pointer (2 bytes): used with URG flag, this is very rarely used.

Options: variable length, used for things like Maximum Segment Size (MSS) negotiation and selective acknowledgements (SACK).


### TCP States
 
A TCP connection goes through a series of named states. Understanding these matters because they directly affect socket behavior.
 
CLOSED: initial state, no connection.
 
LISTEN: server is waiting for incoming connections (after `bind()` and `listen()` functionality is done).
 
SYN_SENT: client has sent SYN, waiting for SYN-ACK.
 
SYN_RECEIVED: server has received SYN and sent SYN-ACK, waiting for ACK.
 
ESTABLISHED: three-way handshake complete, data can flow.
 
FIN_WAIT_1: one side has called `close()`, FIN has been sent, waiting for ACK.
 
FIN_WAIT_2: FIN was acknowledged, waiting for the other side's FIN.
 
CLOSE_WAIT: the other side has sent FIN, waiting for our application to call `close()`.
 
LAST_ACK: our FIN has been sent, waiting for the final ACK.
 
TIME_WAIT: both FINs have been exchanged and acknowledged. Waiting for twice the maximum segment lifetime (2*MSL) to expire before truly closing. This ensures any delayed packets from the connection cannot be mistaken for packets from a new connection on the same port. The 2*MSL is typically 60 seconds to 120 seconds, which is why ports can be temporarily unavailable after a connection closes.
 
CLOSED: back to initial state after TIME_WAIT expires.
 
The reason TIME_WAIT exists is because lets say a connection is there between A:1234 and B:5678. After they close, a new connection between A:1234 and B:5678 could be established. But if there is a delayed packet from the old connection still floating in the network (TCP packets can be delayed by routers some time), it might arrive and can be mistaken for a packet belonging to the new connection. Since TCP uses sequence numbers, this is very unlikely but not impossible, especially if the new connection happens to start with similar sequence numbers so, TIME_WAIT prevents reuse of the same {source IP, source port, destination IP, destination port} tuple until the lifetime of any delayed packets has expired.


Ok Now we now what is there in first 3 layers and what is driving most of the things, we will proceed to the main topic.

## Part Three: What are sockets?
 
### The Abstraction
 
A socket is the programming interface between a process (a running program) and a network. It is an abstraction like a simplified representation of the complex machinery of network communication that hides all the details and provides a uniform interface.
 
Network socket is a standardized interface to the network, regardless of whether it is ethernet, Wi-Fi, fiber optic, or any other physical medium underneath.
 
From our program's perspective, a socket is a file descriptor it is a small non-negative integer that the operating system gives us as a handle to the resource. We have already used file descriptors without necessarily thinking about them: file descriptor 0 is stdin (standard input), 1 is stdout (standard output), 2 is stderr (standard error). When we open a file with `open()`, the OS returns the next available file descriptor number (3, then 4, then 5, etc. assuming 0, 1, 2 are already in use).
 
A socket is just another file descriptor.We can call `read()` and `write()` on it, just like how we do it in a file. The OS handles the rest -- it knows that this particular file descriptor represents a network connection and routes our reads and writes to the network stack.

### The Kernel's Role in sockets
 
The operating system kernel  owns the network stack.
 The kernel:

- Manages the network interface cards (sends and receives raw frames)
 
- Implements the IP protocol (routing decisions, packet fragmentation and reassembly)
 
- Implements TCP (connection management, reliability, ordering, flow control)
 
- Provides the socket interface to programs
 
When the program calls `write(socket_fd, data, len)`, it is not directly writing to the wire. It is making a system call: a request to the kernel. The kernel receives this request, takes the data, wraps it in a TCP segment (adding sequence numbers, checksum, etc.), wraps that in an IP packet (adding source/destination IP addresses), wraps it in an ethernet frame (adding MAC addresses), and sends it to the network card. The network card than transmits it.
 
When data arrives from the network, the reverse happens: the network card receives a frame, the kernel unwraps it layer by layer, checks the destination IP and port, finds the socket that matches, and puts the data in that socket's receive buffer. When the program later calls `read(socket_fd, buf, len)`, the kernel copies data from the socket's receive buffer into program's buffer.
 
This separation between the program and the kernel is important as our program does not need to know anything about ethernet frames or IP routing. The kernel handles all of that. Our program just reads and writes bytes on a file descriptor.
 
### The Socket API's

The socket API is a set of system calls that programs use to work with sockets.

#### `socket(domain, type, protocol)`

Creates a new socket.

`domain` specifies the address family: which type of address will be used. The values you need to know:

`AF_INET`: IPv4 addressing (using 32-bit IP addresses like 192.168.1.100).

`AF_INET6`: IPv6 addressing (using 128-bit addresses like 2001:db8::1). The newer version of IP, designed to address the exhaustion of IPv4 addresses.

`AF_UNIX` (it is also called `AF_LOCAL`): Unix domain sockets. Not a network socket at all -- this is for communication between two processes on the same machine, using a filesystem path as the address instead of an IP address. The SSH agent uses this.

`type` specifies the socket
 type:

`SOCK_STREAM`: a stream socket. It provides a reliable, ordered, bidirectional byte stream. It corresponds to TCP when combined with `AF_INET`, this is what SSH uses for its main connection and for port forwarding.

`SOCK_DGRAM`: a datagram socket. It provides an unreliable, unordered message service. This corresponds to UDP when combined with `AF_INET`. Not used by SSH (as SSH requires reliability).

`SOCK_RAW`: raw socket access. Lets us construct packets yourself, bypassing the OS's TCP/IP implementation. Requires root privileges. Not used by SSH.

`protocol`: usually 0, which tells the kernel to pick the default protocol for the given domain and type combination. For `AF_INET` + `SOCK_STREAM`, the default protocol is TCP. For `AF_INET` + `SOCK_DGRAM`, the default protocol is UDP.

Return value: a file descriptor (a non-negative integer) on success, or -1 on error. On error, the global variable `errno` is set to an error code indicating what went wrong.

```c
int fd = socket(AF_INET, SOCK_STREAM, 0);
if (fd == -1) {
    perror("socket");
    exit(1);
}
```
`fd` is a socket that exists in the kernel but is not connected to anything and has no address.

#### `bind(sockfd, addr, addrlen)`

Associates a socket with a local address and port.

`sockfd`: the socket file descriptor returned by `socket()`.

`addr`: a pointer to an address structure. For IPv4, this is a `struct sockaddr_in`. This struct must be cast to `struct sockaddr*` because `bind()` is designed to work with multiple address families and it uses the generic `sockaddr` type.

`addrlen`: the size of the address structure in bytes.

The `struct sockaddr_in` for IPv4:

```c
struct sockaddr_in {
    sa_family_t    sin_family;  /* address family: AF_INET */
    in_port_t      sin_port;    /* port in network byte order */
    struct in_addr sin_addr;    /* IPv4 address */
    char           sin_zero[8]; /* padding to match the size of sockaddr */
};

struct in_addr {
    uint32_t s_addr; /* address in network byte order */
};
```

`sa_family_t` is typically `uint16_t` (an unsigned 16-bit integer). `in_port_t` is `uint16_t`. `uint32_t` is an unsigned 32-bit integer. These typedefs (type aliases defined in headers) ensure the fields have the right sizes regardless of platform.

Filling in the struct:

```c
struct sockaddr_in addr;
memset(&addr, 0, sizeof(addr)); /* zero all bytes first */
addr.sin_family = AF_INET;
addr.sin_port   = htons(9090);  /* port 9090 in network byte order */
addr.sin_addr.s_addr = htonl(INADDR_LOOPBACK);
/* INADDR_LOOPBACK is the constant for 127.0.0.1 */
/* htonl converts a 32-bit integer from host byte order to network byte order */
```

In C, local variables have undefined initial values. So they contain whatever bytes were in memory at that address previously. If we forget to set a field, we get garbage values. `memset(&addr, 0, sizeof(addr))` sets all bytes to 0, giving you a known starting point.

Computers store multi-byte numbers either big-endian (most significant byte first) or little-endian (least significant byte first). Most modern processors (x86, ARM) are little-endian: the number 0x1234 is stored as 0x34 followed by 0x12. Network protocols use big-endian (most significant byte first): 0x1234 is stored as 0x12 followed by 0x34. When we put port numbers or IP addresses into socket address structures, we must use network byte order.

`htons()`: "host to network short" -- converts a 16-bit value from host byte order to network byte order. On little-endian machines, this reverses the bytes. On big-endian machines, this is a no-op.

`htons(9090)`: 9090 in decimal is 0x2382. On a little-endian machine, htons converts this from the host representation (0x82 0x23 in memory) to network byte order (0x23 0x82 in memory).

`htonl()`: "host to network long" -- same idea but for 32-bit values (IP addresses).

`ntohs()` and `ntohl()` are the reverse: network to host, for reading values received from the network.

`INADDR_LOOPBACK` is 0x7F000001 in network byte order, which is 127.0.0.1. `INADDR_ANY` is 0x00000000, which means "bind to all interfaces" -- any network interface on the machine.

After `bind()`, the socket has a local address. Now the kernel knows: if an incoming packet is addressed to port 9090 on this machine, deliver it to this socket.

kernel does not yet allow incoming connections. That requires `listen()`. And before `listen()`, for client sockets (outgoing connections), we usually do not call `bind()` at all: the kernel automatically picks an ephemeral (temporary, high-numbered) port when you `connect()`. Only server sockets -- sockets that will accept connections -- need to bind to a specific port.

#### `listen(sockfd, backlog)`

Marks a socket as passive -- ready to accept incoming connections.

`sockfd`: the bound socket.

`backlog`: the maximum number of pending connections to queue. When a client sends a SYN, the kernel performs the three-way handshake on behalf of our program. The fully established connection waits in a queue until our program calls `accept()`. The backlog is the maximum size of this queue. If the queue is full when a new SYN arrives, the kernel drops the SYN (the client has to retransmit).

The backlog value of 128 or 256 is conventional for servers that call `accept()` quickly. For our SSH port forwarding listener (which will be called very infrequently -- only when a new forwarding connection arrives), even a backlog of 1 would be fine.

After `listen()`, the socket is a listening socket. It is in the LISTEN state in the TCP state machine. It cannot be used for reading or writing data directly; it is only used for accepting connections.

---
#### `accept(sockfd, addr, addrlen)`

Takes a completed connection from the listening queue and returns a new socket for that specific connection.

`sockfd`: the listening socket.

`addr`: pointer to a `sockaddr_in` struct that `accept()` will fill with the client's address. Can be NULL if you do not care who connected.

`addrlen`: pointer to a `socklen_t` integer. Before calling `accept()`, set it to `sizeof(struct sockaddr_in)`. After `accept()` returns, it is set to the actual size of the address written. The reason it is a pointer (not just a value) is so `accept()` can tell you how much it actually wrote.

Return value: a new file descriptor for the connected socket. This new socket is fully connected to the client; it is in the ESTABLISHED state. You use this new socket to read and write data. The original listening socket is unchanged and will accept the next connection.

This is an important design: the listening socket and the connected socket are separate objects. You can have many connected sockets (one per active client) all derived from the same listening socket.

```c
struct sockaddr_in client_addr;
socklen_t client_addr_len = sizeof(client_addr);

int client_fd = accept(listening_fd,
                       (struct sockaddr*)&client_addr,
                       &client_addr_len);
if (client_fd == -1) {
    perror("accept");
    return;
}

/* Now you can read from and write to client_fd */
/* client_addr.sin_addr contains the client's IP address */
/* client_addr.sin_port contains the client's port (in network byte order) */
/* ntohs(client_addr.sin_port) gives the port as a regular integer */
```

By default, `accept()` blocks if there are no pending connections. Since we are using an event loop, we only call `accept()` when `poll()` tells us the listening socket is ready (POLLIN), which means there is at least one pending connection, so `accept()` will not block.

---
 
#### `connect(sockfd, addr, addrlen)`

Initiates a connection to a remote address. Used by clients.

`sockfd`: a socket that has not been connected yet. Does not need to be bound (the kernel will assign an ephemeral source port automatically).

`addr`: the address of the server to connect to.

`addrlen`: size of the address struct.

`connect()` initiates the three-way handshake. By default it blocks until the handshake completes (the connection is established) or fails.

```c
struct sockaddr_in server_addr;
memset(&server_addr, 0, sizeof(server_addr));
server_addr.sin_family = AF_INET;
server_addr.sin_port   = htons(22); /* SSH port */
inet_pton(AF_INET, "1.2.3.4", &server_addr.sin_addr);

int rc = connect(sockfd, (struct sockaddr*)&server_addr, sizeof(server_addr));
if (rc == -1) {
    perror("connect");
    close(sockfd);
    return;
}
```

`inet_pton()`: "presentation to network" -- converts a human-readable IP address string to binary. The opposite of `inet_ntop()` (network to presentation), which converts binary to a string for display.

After a successful `connect()`, the socket is in the ESTABLISHED state and ready for data transfer.

Common errors from `connect()`:

`ECONNREFUSED`: the remote machine is up but nothing is listening on that port. The server sent RST.

`ETIMEDOUT`: no response received within the timeout. The machine may be down, or a firewall is silently dropping packets.

`ENETUNREACH`: no route to the destination. The routing table has no entry for the destination IP prefix.

`ECONNRESET`: the connection was reset by the other side.

---

#### `read(fd, buf, count)` and `write(fd, buf, count)`

These are the basic I/O calls, identical for files and sockets.

`read(fd, buf, count)` reads up to `count` bytes from `fd` into the buffer `buf`. Returns the number of bytes actually read. This might be less than `count` even if more data is available and also TCP makes no promises about how much data arrives in each read. Returns 0 when the other side has closed the connection (sent FIN). Returns -1 on error.

`write(fd, buf, count)` writes `count` bytes from `buf` to `fd`. Returns the number of bytes written, which can be less than `count` if the send buffer is full. Returns -1 on error. A common issue we must check is the return value and write the remaining bytes in a loop if the return value is less than `count`.

```c
/* Reliable write: keeps writing until all bytes are sent */
ssize_t write_all(int fd, const void *buf, size_t count) {
    const uint8_t *p = buf;
    size_t remaining = count;

    while (remaining > 0) {
        ssize_t written = write(fd, p, remaining);
        if (written == -1) {
            if (errno == EINTR) continue; /* interrupted by signal, retry */
            return -1; /* real error */
        }
        p += written;
        remaining -= written;
    }
    return count;
}
```

`ssize_t` is "signed size type" -- a signed integer type used for sizes and return values that can be negative (to indicate errors). (`size_t` is the unsigned version).

`EINTR` means the system call was interrupted by a signal. When a signal is delivered to a process, any blocking system call in progress returns -1 with `errno == EINTR`. The correct behavior for most system calls is to retry when you see `EINTR`.

`uint8_t` is an unsigned 8-bit integer which is exactly one byte. Using `uint8_t*` for byte buffers is more explicit than `char*` and avoids signedness confusion.


#### `close(fd)`

It Closes a file descriptor. For sockets, this initiates the TCP connection teardown: the kernel sends a FIN to the other side, indicating that we have no more data to send.

After `close()`, the file descriptor number is returned to the pool and may be reused for the next `open()`, `socket()`, or `accept()` call.

An important subtlety: `close()` decrements the file descriptor's reference count. If we have duplicated a file descriptor with `dup()` or `dup2()`, closing one copy does not close the underlying socket it is only when the last copy is closed does the FIN get sent. This matters in the SSH CLI because all the shell process created by `fork()` inherits the parent's file descriptors. We must `close()` inherited sockets in the child process to avoid keeping them open unexpectedly.

#### `setsockopt(sockfd, level, optname, optval, optlen)`

Sets an option on a socket.

`level`: which protocol layer the option applies to. `SOL_SOCKET` for socket-level options, `IPPROTO_TCP` (also written as `SOL_TCP`) for TCP-level options, `IPPROTO_IP` for IP-level options.

`optname`: the specific option to set.

`optval`: pointer to the option value.

`optlen`: size of the option value.

The options relevant to the SSH CLI:

`SO_REUSEADDR` at level `SOL_SOCKET`: This allows us binding to a port that is in TIME_WAIT. Set this on every server socket. The value is an `int` set to 1.

`SO_KEEPALIVE` at level `SOL_SOCKET`: enables TCP keepalive probes. The value is an `int` set to 1. With this set, the kernel's TCP stack will send keepalive probes when the connection has been idle for a while. If the probes go unanswered, the kernel closes the connection.

`TCP_KEEPIDLE` at level `IPPROTO_TCP` (Linux-specific): the number of seconds the connection must be idle before the first keepalive probe is sent. The value is an `int`.

`TCP_KEEPINTVL` at level `IPPROTO_TCP`: the interval in seconds between subsequent keepalive probes (after the first one). The value is an `int`.

`TCP_KEEPCNT` at level `IPPROTO_TCP`: how many probes to send before giving up and closing the connection. The value is an `int`.

`TCP_NODELAY` at level `IPPROTO_TCP`: disables Nagle's algorithm. Nagle's algorithm buffers small writes and combines them into larger TCP segments to reduce overhead. For SSH, We want keystrokes to be sent immediately, not buffered. Disabling Nagle ensures that each `write()` of a small amount of data (like a single keypress) is sent immediately as its own segment. The value is an `int` set to 1.

```c
int flag = 1;
setsockopt(sockfd, IPPROTO_TCP, TCP_NODELAY, &flag, sizeof(flag));
```

#### `getsockopt(sockfd, level, optname, optval, optlen)`

Reads a socket option. Like `setsockopt` but for reading. The `optlen` parameter is a pointer and also before calling, set it to the size of your buffer; after calling, it contains the actual size written.

Useful for checking options that were set, or for options that report state (like `SO_ERROR` which gives the pending error code for a socket).

#### `shutdown(sockfd, how)`

A more controlled way to close one or both directions of a connection.

`how` can be:

`SHUT_RD` (0): close the read direction. Further reads return 0 (EOF). The other side's writes will be discarded. Does not send FIN.

`SHUT_WR` (1): close the write direction. Sends a FIN to the other side. Further writes fail. But the socket can still receive data. This is used to signal end-of-data in one direction while still being able to read responses.

`SHUT_RDWR` (2): close both directions. Equivalent to `close()` but without releasing the file descriptor.

In port forwarding, when we want to signal to the SSH channel that the local socket has no more data to write (but might still receive data), we call `shutdown(client_fd, SHUT_WR)` and then send EOF on the SSH channel. The other direction remains open until the remote side also sends EOF.

## Part Four: `poll()` -- The Heart of the Event Loop

### Why Blocking I/O is Not Enough

Let us Consider a SSH port forwarding scenario: our SSH client is simultaneously managing a local forwarding listener socket, a connected forwarding socket to a database tool, the SSH session socket, and stdin (keyboard input).

If we call `read(ssh_socket_fd, buf, sizeof(buf))`, our program blocks it. Nothing else happens until data arrives on the SSH socket. During that time, if the user presses a key, we cannot process it. If the database tool sends data to the forwarding port, we cannot process it. The program is frozen and is waiting on one thing.

This is unacceptable. So we need a way to wait on all these things simultaneously and handle whichever one becomes ready first. That is what `poll()` does.

### The `poll()` System Call

```c
int poll(struct pollfd *fds, nfds_t nfds, int timeout);
```

`fds`: an array of `pollfd` structs, one per file descriptor we want to watch.

`nfds`: the number of elements in the `fds` array.

`timeout`: how long to wait in milliseconds. 0 means return immediately (check but do not wait). -1 means wait forever. Positive values set a maximum wait time.

Return value: the number of file descriptors that are ready (have events), 0 if the timeout expired with nothing ready, or -1 on error.

The `pollfd` struct:

```c
struct pollfd {
    int   fd;      /* file descriptor to watch */
    short events;  /* events we are interested in (input to poll) */
    short revents; /* events that actually occurred (output from poll) */
};
```

`events` and `revents` use bitmask flags:

`POLLIN`: there is data to read, or for a listening socket, a connection is ready to accept.

`POLLOUT`: the socket's send buffer has space, meaning `write()` will not block.

`POLLERR`: an error condition occurred. This is always monitored even if not specified in `events`.

`POLLHUP`: the other side has closed the connection (hang-up). For a socket, this typically means the other side sent a FIN.

`POLLNVAL`: the file descriptor is not valid (not open). This indicates a programming error.

You fill in `fd` and `events` before calling `poll()`. After `poll()` returns, you check `revents` for each struct to see what actually happened.

Minimal event loop example 

### A Minimal Event Loop

```c
/* Set up the array of file descriptors to watch */
struct pollfd watched[4];

/* The SSH session's underlying TCP socket */
watched[0].fd     = ssh_get_fd(session); /* libssh gives us the socket fd */
watched[0].events = POLLIN;

/* stdin (keyboard input from the user) */
watched[1].fd     = STDIN_FILENO; /* STDIN_FILENO is the constant 0 */
watched[1].events = POLLIN;

/* The listening socket for the forwarded port */
watched[2].fd     = forward_listen_fd;
watched[2].events = POLLIN;

/* The self-pipe for SIGWINCH signal notifications */
watched[3].fd     = sigwinch_pipe_read_end;
watched[3].events = POLLIN;

int running = 1;
while (running) {
    int n_ready = poll(watched, 4, timeout_ms);

    if (n_ready == -1) {
        if (errno == EINTR) continue; /* signal interrupted, retry */
        perror("poll");
        break;
    }

    if (n_ready == 0) {
        /* timeout: check keepalive, etc. */
        handle_timeout();
        continue;
    }

    /* Check which file descriptors are ready */

    if (watched[0].revents & POLLIN) {
        /* Data from SSH server: let libssh process it */
        ssh_event_dopoll(event, 0); /* 0 timeout: don't block */
    }

    if (watched[1].revents & POLLIN) {
        /* Keyboard input from user */
        uint8_t buf[256];
        ssize_t n = read(STDIN_FILENO, buf, sizeof(buf));
        if (n > 0) {
            process_keyboard_input(main_channel, buf, n, &escape_state, '~');
        }
    }

    if (watched[2].revents & POLLIN) {
        /* New connection on forwarded port */
        handle_new_forward_connection(forward_listen_fd);
    }

    if (watched[3].revents & POLLIN) {
        /* SIGWINCH notification: terminal was resized */
        char byte;
        read(sigwinch_pipe_read_end, &byte, 1);
        update_terminal_size(main_channel);
    }
}
```

This loop is the entire runtime of the interactive SSH client. While it is running, the program responds to any of these events as they occur, with no blocking and no wasted CPU time (when nothing is happening, `poll()` puts the process to sleep until something happens).

### Adding and Removing File Descriptors Dynamically

In a real SSH CLI, the number of file descriptors being watched changes at runtime. When a new forwarding connection arrives, we add its socket to the watched array. When it closes, we remove it.

Managing a fixed-size array becomes cumbersome. In practice, we use a dynamically allocated array that grows as needed:

```c
struct pollfd *watched = NULL;
int n_watched = 0;
int capacity  = 0;

void add_fd(int fd, short events) {
    if (n_watched == capacity) {
        capacity = capacity ? capacity * 2 : 8;
        watched  = realloc(watched, capacity * sizeof(struct pollfd));
        /* realloc resizes a dynamically allocated buffer */
    }
    watched[n_watched].fd      = fd;
    watched[n_watched].events  = events;
    watched[n_watched].revents = 0;
    n_watched++;
}

void remove_fd(int fd) {
    for (int i = 0; i < n_watched; i++) {
        if (watched[i].fd == fd) {
            /* Replace this entry with the last entry */
            watched[i] = watched[n_watched - 1];
            n_watched--;
            return;
        }
    }
}
```

`realloc()` resizes a dynamically allocated memory block. It takes the old pointer and the new size, and returns a new pointer to a block of the new size. If the new size is larger, the old contents are preserved and the extra space is uninitialized.

## Unix Domain Sockets -- How the SSH Agent Communicates

### What Unix Domain Sockets Are

Unix domain sockets are a type of socket that, instead of using IP addresses and port numbers, uses filesystem paths as addresses. They exist only for communication between processes on the same machine. They are faster than TCP sockets (no network stack processing, no IP routing, just data copied from one process's buffer to another's) and have additional features like credential passing (we can find out the PID, UID, and GID of the process on the other end, which is used for access control).

The `AF_UNIX` address family (also called `AF_LOCAL`) is used instead of `AF_INET`. The address structure is `struct sockaddr_un`:

```c
#include <sys/un.h>

struct sockaddr_un {
    sa_family_t sun_family; /* address family: AF_UNIX */
    char        sun_path[108]; /* filesystem path of the socket */
};
```

`sun_path` is a null-terminated string of up to 107 characters (plus the null byte) giving the socket's path in the filesystem. A path like `/tmp/ssh-XXXXXX/agent.1234` is typical for the SSH agent.

### Creating a Unix Domain Socket Server (The Agent's Side)

The SSH agent creates a Unix domain socket and waits for SSH clients to connect:

```c
#include <sys/socket.h>
#include <sys/un.h>
#include <unistd.h>

const char *socket_path = "/tmp/ssh-agent-socket";

/* Create the socket */
int server_fd = socket(AF_UNIX, SOCK_STREAM, 0);

/* Remove any existing socket file at this path */
unlink(socket_path); /* unlink removes a filesystem path (like rm) */

/* Set up the address */
struct sockaddr_un addr;
memset(&addr, 0, sizeof(addr));
addr.sun_family = AF_UNIX;
strncpy(addr.sun_path, socket_path, sizeof(addr.sun_path) - 1);
/* strncpy copies at most n-1 characters, ensuring null termination if we set the last byte to 0 */
/* memset already set it to 0, so we are safe */

/* Bind and listen */
bind(server_fd, (struct sockaddr*)&addr, sizeof(addr));
listen(server_fd, 8);
```

`unlink()` is called before `bind()` because if a file already exists at that path (from a previous agent that did not clean up properly), `bind()` will fail with `EADDRINUSE`. Removing the old file first ensures a clean start.

The actual SSH agent sets the permissions on the socket file to be accessible only by the current user, providing security: other users on the same machine cannot access your agent.

### Connecting to a Unix Domain Socket (The SSH Client's Side)

```c
const char *auth_sock = getenv("SSH_AUTH_SOCK");
/* getenv reads an environment variable; returns NULL if not set */

if (auth_sock == NULL) {
    fprintf(stderr, "SSH_AUTH_SOCK not set: no agent running\n");
    /* cannot do agent forwarding without an agent */
    return -1;
}

int agent_fd = socket(AF_UNIX, SOCK_STREAM, 0);

struct sockaddr_un addr;
memset(&addr, 0, sizeof(addr));
addr.sun_family = AF_UNIX;
strncpy(addr.sun_path, auth_sock, sizeof(addr.sun_path) - 1);

int rc = connect(agent_fd, (struct sockaddr*)&addr, sizeof(addr));
if (rc == -1) {
    perror("connect to agent");
    close(agent_fd);
    return -1;
}

/* agent_fd is now connected to the SSH agent */
/* you can read() and write() agent protocol messages on it */
```

After `connect()` succeeds, `agent_fd` is a stream socket connected to the agent process. Reading and writing on it works exactly like TCP: `read()` gets bytes, `write()` sends bytes. The agent protocol defines the message format layered on top of this byte stream.


### Why Use Unix Domain Sockets Instead of TCP?

Three reasons:

Security: TCP sockets bound to localhost (127.0.0.1) are accessible by any process on the machine, including processes running as other users. A Unix domain socket file can have filesystem permissions set (readable/writable only by the owner). Other users cannot connect to your agent socket.

Speed: Unix domain sockets skip the network stack entirely. Data is copied directly between kernel buffers for the two processes. No IP headers, no TCP headers, no checksum calculation, no sequence number management. For the small messages of the agent protocol, this matters.

Path-based addressing: the agent path is communicated through an environment variable (`SSH_AUTH_SOCK`). Any child process inherits environment variables from its parent, so child SSH clients automatically know where to find the agent.