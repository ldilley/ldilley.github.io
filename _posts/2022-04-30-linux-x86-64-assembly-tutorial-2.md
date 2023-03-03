---
layout: post
title: Linux x86-64 Assembly Tutorial (part 2)
tags: [linux, assembly, c, tutorial]
comments: true
---

### Introduction
Welcome to the second installment of the 64-bit Linux assembly tutorial. If you have not yet read part one of this tutorial, you can do
so [here](https://blog.dilley.me/2022-04-20-linux-x86-64-assembly-tutorial-1/). If you have read it, I hope that you enjoyed it. We will be covering networking, debugging,
optimizing, endianness, and analogous C code in this tutorial. Let us get to it!

### Useful Links
* [Part I of this tutorial](https://blog.dilley.me/2022-04-20-linux-x86-64-assembly-tutorial-1/) (the basics)
* [Ryan A. Chapman's 64-bit Linux system call table](http://blog.rchapman.org/post/36801038863/linux-system-call-table-for-x86-64) (for reference)
* [Beej's Guide to Network Programming](http://beej.us/guide/bgnet/) (in C)
* [Valgrind](http://valgrind.org/) (debugger and profiler)
* [Understanding Big and Little Endian Byte Order](http://betterexplained.com/articles/understanding-big-and-little-endian-byte-order/)

### The C Way
We are going to start by looking at how you create a network program in C. See [Beej's Guide to Network Programming](http://beej.us/guide/bgnet/)
for more information. I am illustrating socket programming in a higher-level language to give you a better idea of the sequence of events
that occur. In order to accept network connections in a C program (or assembly), you must take the following steps:

1. Call `socket()` to obtain a file descriptor to be used for communication. We used file descriptors in the first tutorial
(`stdin`/`0` and `stdout`/`1` specifically.)
2. Call `bind()` to associate (or bind) the IP address of a network interface with the file descriptor returned by `socket()`.
3. Call `listen()` to make the file descriptor be receptive to incoming network connections.
4. Call `accept()` to handle incoming network connections.

`accept()` returns a file descriptor for the client which you can use to send and receive data to and from the remote end. You can
also call `close()` on the client file descriptor once you are done receiving or transmitting data. After putting it all together,
it would look something like:

```c
#include <stdio.h>        /* for printf() and puts() */
#include <stdlib.h>       /* for exit() and perror() */
#include <string.h>       /* for strlen() */
#include <sys/socket.h>   /* for AF_INET, SOCK_STREAM, and socket_t */
#include <netinet/in.h>   /* for INADDR_ANY and sockaddr_in */

#define PORT 9990         /* TCP port number to accept connections on */
#define BACKLOG 10        /* connection queue limit */

int main()
{
  /* server and connecting client file descriptors */
  int server_fd, client_fd;

  /* size of sockaddr_in structure */
  int addrlen;

  /* includes information for the server socket */
  struct sockaddr_in server_address;

  /* message we send to connecting clients */
  char *message = "Greetings!\n";

  /* socket() - returns a file descriptor we can use for our server
   * or -1 if there was a problem
   * Arguments:
   * AF_INET = address family Internet (for Internet addressing)
   * SOCK_STREAM = TCP (Transmission Control Protocol)
   * 0 = default protocol for this type of socket
   */
  server_fd = socket(AF_INET, SOCK_STREAM, 0);

  /* Check for an error */
  if(server_fd == -1)
  {
    perror("Unable to obtain a file descriptor for the server");
    exit(1);
  }

  server_address.sin_family = AF_INET;

  /* set the listen address to any/all available */
  server_address.sin_addr.s_addr = INADDR_ANY;

  /* The htons() function below deals with endian conversion which
   * we'll discuss later. This assignment sets the port number to
   * accept connections on. */
  server_address.sin_port = htons(PORT);

  /* bind() - binds the IP address to the server's file descriptor or
   * returns -1 if there was a problem */
  if(bind(server_fd, (struct sockaddr *)&server_address,
          sizeof(server_address)) == -1)
  {
    perror("Unable to bind");
    exit(1);
  }

  /* listen() - listen for incoming connections */
  if(listen(server_fd, BACKLOG) == -1)
  {
    puts("Failed to listen on server socket!");
    exit(1);
  }

  addrlen = sizeof(server_address);

  puts("Waiting for connections...");

  /* Infinite loop to accept connections forever */
  for(;;)
  {
    /* accept() - handle new client connections */
    client_fd = accept(server_fd, (struct sockaddr *)&server_address,
                       (socklen_t*)&addrlen);
    if(client_fd == -1)
    {
      perror("Unable to accept client connection");
      continue;
    }
    /* Send greeting to client and then disconnect them */
    send(client_fd, message, strlen(message), 0);
    close(client_fd);
  }

  return 0;
}
```

You should be able to copy and paste the above code into a text file.
Compile it with: `gcc <file>.c -o network_example`
After compiling the program, execute it with: `./network_example`
If all went well, you should see something similar to below:

```bash
[lloyd@lindev ~]$ ./network_example
Waiting for connections...
```

Open another terminal and issue: `telnet localhost 9990`
You should see something like the following:

```bash
[lloyd@lindev ~]$ telnet localhost 9990
Trying ::1...
telnet: connect to address ::1: Connection refused
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
Greetings!
Connection closed by foreign host.
```

You can read more about [bind()](http://linux.die.net/man/2/bind), [listen()](http://linux.die.net/man/2/listen), and
[accept()](http://linux.die.net/man/2/accept) if you're interested. Next up, we're going to replicate the above C program
in x86-64 assembly. Let's see how it looks...

### The Assembly Way

```asm
[BITS 64]

; Description: 64-bit Linux TCP server
; Author: Lloyd Dilley
; Date: 04/02/2014

struc sockaddr_in
  .sin_family resw 1
  .sin_port resw 1
  .sin_address resd 1
  .sin_zero resq 1
endstruc

section .bss
  peeraddr:
    istruc sockaddr_in
      at sockaddr_in.sin_family, resw 1
      at sockaddr_in.sin_port, resw 1
      at sockaddr_in.sin_address, resd 1
      at sockaddr_in.sin_zero, resq 1
    iend

section .data
  waiting:      db 'Waiting for connections...',0x0A
  waiting_len:  equ $-waiting
  greeting:     db 'Greetings!',0x0A
  greeting_len: equ $-greeting
  error:        db 'An error was encountered!',0x0A
  error_len:    equ $-error
  addr_len:     dq 16
  sockaddr:
    istruc sockaddr_in
      ; AF_INET
      at sockaddr_in.sin_family, dw 2
      ; TCP port 9990 (network byte order)
      at sockaddr_in.sin_port, dw 0x0627
      ; 127.0.0.1 (network byte order)
      at sockaddr_in.sin_address, dd 0x0100007F
      at sockaddr_in.sin_zero, dq 0
    iend

section .text
global _start
_start:
  ; Get a file descriptor for sys_bind
  mov rax, 41           ; sys_socket
  mov rdi, 2            ; AF_INET
  mov rsi, 1            ; SOCK_STREAM
  mov rdx, 0            ; protocol
  syscall
  mov r13, rax
  push rax              ; store return value (fd)
  test rax, rax         ; check if -1 was returned
  js exit_error

  ; Bind to a socket
  mov rax, 49           ; sys_bind
  pop rdi               ; file descriptor from sys_socket
  mov rbx, rdi          ; preserve server fd (rbx is saved across calls)
  mov rsi, sockaddr
  mov rdx, 16           ; size of sin_address is 16 bytes (64-bit address)
  syscall
  push rax
  test rax, rax
  js exit_error

  ; Listen for connections
  mov rax, 50           ; sys_listen
  mov rdi, rbx          ; fd
  mov rsi, 10           ; backlog
  syscall
  push rax
  test rax, rax
  js exit_error
  ; Notify user that we're ready to listen for incoming connections
  mov rax, 1            ; sys_write
  mov rdi, 1            ; file descriptor (1 is stdout)
  mov rsi, waiting
  mov rdx, waiting_len
  syscall
  call accept

accept:
  ; Accept connections
  mov rax, 43           ; sys_accept
  mov rdi, rbx          ; fd
  mov rsi, peeraddr
  lea rdx, [addr_len]
  syscall
  push rax
  test rax, rax
  js exit_error

  ; Send data
  mov rax, 1
  pop rdi               ; peer fd
  mov r15, rdi          ; preserve peer fd (r15 is saved across calls)
  mov rsi, greeting
  mov rdx, greeting_len
  syscall
  push rax
  test rax, rax
  js exit_error

  ; Close peer socket
  mov rax, 3            ; sys_close
  mov rdi, r15          ; fd
  syscall
  push rax
  test rax, rax
  js exit_error
  ;jz shutdown
  call accept           ; loop forever if preceding line is commented out

shutdown:
  ; Close server socket
  mov rax, 3
  mov rdi, rbx
  syscall
  push rax
  test rax, rax
  js exit_error

  ; Exit normally
  mov rax, 60           ; sys_exit
  xor rdi, rdi          ; return code 0
  syscall

exit_error:
  mov rax, 1
  mov rdi, 1
  mov rsi, error
  mov rdx, error_len
  syscall

  mov rax, 60
  pop rdi               ; stored error code
  syscall
```

Thank goodness for high-level languages, eh?
You can assemble and link just like you did from the first tutorial:
`nasm -f elf64 -o network_example.o network_example.asm && ld -o network_example network_example.o`

You can then execute the program and test it with telnet the same way you did with the C version. The functionality should be very
similar.

### Dissecting the Beast
NASM allows programmers to use structs, so we take advantage of this for better data organization. Just like in the C program,
a `sockaddr_in` structure is defined. This is essentially a template which holds various data members. For review, the `BSS` section
contains memory set aside for variable data during runtime. This makes sense considering it is not known what our connecting client
source addresses and ports will be. And since we know what address and port to use on the server side, the information can be set in
the `data` section as literals. I also touched on data types some in the first tutorial. The table below contains the types used in this
program along with their sizes and examples.

|Type|Size|Example|
|----|----|-------|
|`resb/db`|1 byte (8 bits)|A keyboard character such as the letter 'c'|
|`resw/dw`|2 bytes (16 bits) -- also called a "word"|A network port with a maximum value of 65,535|
|`resd/dd`|4 bytes (32 bits) -- also called a "double word"|An IPv4 address such as 192.168.1.1|
|`resq/dq`|8 bytes (64 bits) -- also called a "quad word"|A "long long" in C/C++ or represents a decimal number (float)|

An "octa word" (128 bits) is also worth mentioning, but is not used in this program. These are used for scientific calculations,
graphics, IPv6 addresses, globally unique IDs (GUIDs), etc. The dX variety are initialized and the 'd' stands for "data". So, `db` is
"data byte" and `dw` is "data word". The resX assortment is used for reserving space for uninitialized data. `resb` would be
"reserve byte" and `resq` is "reserve quad" for example. The "at" macro gets at each field and sets it with the specified data. `struc` 
and `endstruc` define a structure. `istruc` and `iend` declare an instance of a structure. You can see in the code how to refer to an
instance by using a label (peeraddr for example.)

In the `text` section (code), you should be able to get an idea of what is going on with the comments. The format is the same as the
program from the first tutorial. It is all a matter of putting bullets (data) in certain positions (registers) of a revolver and then
pulling the trigger with `syscall`. That is an analogy I like to use anyway. Again, you can refer to [Ryan A. Chapman's 64-bit Linux
system call table](http://blog.rchapman.org/post/36801038863/linux-system-call-table-for-x86-64) for reference. `sys_bind`, `sys_listen`, `sys_accept`, and other calls are all present there.


### Ten Little Endians
Endianness (name originates from Gulliver's Travels) refers to the way data is arranged in memory in the context of hardware
architectures. I bring this up because we needed to call `htons()` (short data from host to network order) in our C program on the
network port. We also needed to convert the loopback IP address and TCP port number to network byte order in the assembly program.

x86/x86-64 are considered little-endian architectures whereas SPARC is big endian. Some processors, such as PPC, can handle both modes
and are referred to as bi-endian. What does this mean exactly? Well, on little-endian machines, the most-significant byte (MSB) is stored
at the highest memory address. The least-significant byte (LSB) is stored at the lowest address. Big endian is the reverse of this.
An example would be storing three bytes that make up the word "BEEF". Using the ASCII values for each letter in hexadecimal: 'B' is 0x42,
'E' is 0x45, and 'F' is 0x46. On a big-endian system, the arrangement of bytes would appear as: 42 45 45 46. However, on a little-endian
system, they would appear as: 46 45 45 42. Obviously, debugging is easier on a big-endian system since data is still easily readable
by humans. Meanwhile, little endian has the advantage of programmers being able to determine if a number is even or odd by looking
at its LSB.

Due to these differences, the need for a common format for data being transmitted over a network was clear. Big endian or network byte order was decided on for this purpose. How can we convert? The easiest method is to use a calculator in programmer mode. Windows calculator supports this mode. The TCP port number 9990 in decimal is 2706 in hex. Since 0x27 is the most significant part, it goes in the right-most slot. 0x06 goes on the left resulting in 0x0627. This is similar for the IP address. Each octet of 127.0.0.1 must be converted to hex. This yields 7F 00 00 01. Again, 127 or 0x7F is the most significant part, so it goes on the far right (lowest memory address.) You end up with 0x0100007F.

### A Closer Look
You can use `gdb` or `valgrind` to debug of course, but this section is more about tracing program execution to demonstrate what is
going on from an OS perspective with system calls. If you have `strace` installed, issue:
`strace -f ./network_example`

You can actually see each system call from the assembly program and what arguments populate each function such as source port for
peer address. Try connecting with `telnet` with the trace still running and you can see `write()` and `close()` being called. Have a look:

```bash
[lloyd@lindev ~]$ strace -f ./network_example
execve("./network_example", ["./network_example"], [/* 26 vars */]) = 0
socket(PF_INET, SOCK_STREAM, IPPROTO_IP) = 3
bind(3, {sa_family=AF_INET, sin_port=htons(9990), sin_addr=inet_addr("127.0.0.1")}, 16) = 0
listen(3, 10)                           = 0
write(1, "Waiting for connections...\n", 27Waiting for connections...
) = 27
accept(3, {sa_family=AF_INET, sin_port=htons(47944), sin_addr=inet_addr("127.0.0.1")}, [16]) = 4
write(4, "Greetings!\n", 11)            = 11
close(4)                                = 0
accept(3, {sa_family=AF_INET, sin_port=htons(47946), sin_addr=inet_addr("127.0.0.1")}, [16]) = 4
write(4, "Greetings!\n", 11)            = 11
close(4)                                = 0
accept(3, ^CProcess 27238 detached
```

You can see from above that the server is assigned a file descriptor of 3 and the client is 4. 11 is the length of the greeting sent
to the client. `sin_port` and `sin_addr` from `accept()` contain the connecting client's source IP address and port. Pretty slick, huh?

### Compacting A Compact Program
As you can see, the size difference between the assembly program and the C program is significant. The functionally-equivalent C
program is over 4 times as large:

```bash
[lloyd@lindev ~]$ ls -lah network_example_*
-rwxr-xr-x. 1 lloyd linux_users 2.1K Dec 10 03:51 network_example_asm
-rwxr-xr-x. 1 lloyd linux_users 8.9K Dec 10 04:06 network_example_c
```

Let's see if we can squeeze both of these binaries a bit more...

```bash
[lloyd@lindev ~]$ strip -s network_example_*
[lloyd@lindev ~]$ ls -lah network_example_*
-rwxr-xr-x. 1 lloyd linux_users  888 Dec 10 04:28 network_example_asm
-rwxr-xr-x. 1 lloyd linux_users 6.2K Dec 10 04:28 network_example_c
```

Even after shaving off symbols from both binaries, the C program is now over 6 times larger than the assembly program. The assembly program isn't even 1K. This is a testament to assembly's efficiency. Yay for assembly!

### Sayonara
I apologize for the delay between the first tutorial and this one. Better late than never, right? I hope people still find this
information useful. If you have any questions or feedback, please drop me a line in the comments and I would be happy to reply.

### Author and Copyright Information
Copyright © 2015 Lloyd Dilley (originally written on 12/10/2015) under the terms of the [GNU Free Documentation License v1.3](https://www.gnu.org/licenses/fdl-1.3.txt)
