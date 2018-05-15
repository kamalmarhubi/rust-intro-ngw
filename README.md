# Rust 101

We're going to build a tiny HTTP client, and then use that to build a tiny
multithreaded web scraper.

Our HTTP client is going to use the socket interface directly, bypassing Rust's standard library. This

You'll need Rust installed, which you can do by following the instructions at
  https://www.rust-lang.org/en-US/install.html.

This will install the Rust compiler and standard library, as well as the Cargo build and dependency management tool.


### A word on docs

There are a couple of ways to get documentation for Rust packages, which are
called crates. There's a publicly hosted site at https://docs.rs with docs for
all the crates (packages or libraries) on crates.io. You can get a crate's docs
by going to https://docs.rs/cratename, which will redirect you to the latest
version of its docs.

Alternatively, Cargo has a subcommand for building docs for your project and
all its dependencies: `cargo doc --open`. This generates the docs, and then
opens them in a browser. It's really handy to have all the docs for the
libraries you're using in one place.

## Setting up your project

Cargo can set up a project for you, including version control:

```
$ cargo new rust-ngw
     Created binary (application) `rust-ngw` project
$ cd rust-ngw
$ ls -a
./  ../  Cargo.toml  .git/  .gitignore  src/
$ cat src/main.rs
fn main() {
    println!("Hello, world!");
}
```

Note it's created a skeleton project, including initilising a git repository. (If you prefer another version control system, check `cargo new --help` for docs on the `--vcs` flag.)

Now, if we run `cargo run`, it will bulid our binary and run it:

```
$ cargo run
   Compiling rust-ngw v0.1.0 (file:///tmp/asss)
    Finished dev [unoptimized + debuginfo] target(s) in 0.87 secs
     Running `target/debug/rust-ngw`
Hello, world!
```

Since we'll be using the `nix` library, let's add it to our project right away! Open up `Cargo.toml`, and this under `[dependencies]`:

```toml
nix = "0.10"
```

If you've used something like `npm` or `bundler`, Cargo plays a similar role: it manages dependencies, and figures out which versions to install and so on. We just told Cargo that we want `nix` at version 0.10.x.

If you run `cargo run` again, you'll see Cargo download and compile `nix` and its transitive dependencies.

### A quick change to `main`

The default `main` doesn't return anything. Because we'll be doing all kinds of
IO and network stuff that can fail, we want to signal this in our `main`
functions's signature. So change it to return a `Result`, which is Rust's way
of representing tihngs that might fail. So let's change `fn main()` to

```rust
fn main() -> Result<(), Box<std::error::Error>> {
    println!("Hello, world!");
    Ok(())
}
```

The `Ok(())` is how we say that everything completed fine, and there's no useful result.

<small>(If you've used a typed funcitonal language, `()` is sometimes called "unit".)</small>

We'll get a chance to look at `Result` and `Error` in more detail later. For now let's just move on to...

## Opening a TCP connection with sockets

To start off, we're going to learn a tiny bit about socket programming. A
socket is an endpoint for communication. There are a bunch of different types,
but the ones we're interested in here are TCP sockets.

First, we'll need to create a socket! This means we have to tell the OS what
kind of socket we want. There are a couple of axes:

- the address family, which for us will either be AF_INET (IPv4) or AF_INET (IPv6).
- the type of socket, which for TCP sockets os SOCK_STREAM

The C function for creating a socket is `socket()` in the `sys/socket.h`
header. The modules in the nix bindings mirror the layout of the C headers, so
we can get it by

```rust
extern crate nix;
use nix::sys::socket::socket;
```

`extern crate nix` is telling Rust to bring the nix library in.

You can take a look at the [docs for socket][docs-nix-socket] to see how to
call it. To create a TCP/IPv4 socket, we'll pass `AddressFamily::Inet` as the
address family, and `SockType::Stream` as the socket type. This means we need
to `use` (ie import) those types as well, so change the `use` statement to

```rust
use nix::sys::socket::{socket, AddressFamily, SockFlag, SockType};
```

Now we can create the socket!


```rust
let sock = socket(AddressFamily::Inet, SockType::Stream, SockFlag::empty(), None)?;
```

[docs-nix-socket]: https://docs.rs/nix/0.10.0/nix/sys/socket/fn.socket.html

To break this down, we're asking the OS to create a socket for us using the
IPv4 address family, and with type stream, ie, a TCP socket. The
`SockFlag::empty()` and `None` arguments are the flags and protocol options,
and we don't need to set any here.

Great! Now, at this point, all we've done is create the socket. It's not
currently connected to anything.

To connect to a server, we call the aptly named `connect` function, passing it
the socket and an address to connect to:

```rust
let ip_addr = IpAddr::new_v4(1, 1, 1, 1);
let sockaddr = SockAddr::new_inet(InetAddr::new(ip_addr, port));
connect(sock, &sockaddr)?;
```

You'll need to add a few more imports for `IpAddr`, `SockAddr`, and `connect`. At this point it might be worth changing the `use` line to

```rust
use nix::sys::socket::*;
```

to import everything from the socket module.

Building up the address is kind of tedious. That's what we get for bypassing the standard library!

__Exercise.__ Open a TCP connection to 1.1.1.1 port 80.


### What's up with those question marks?

You might be wondering what the deal is with the `?`s. They're a shorthand to say "hey we know this call might fail. If it does, just return the error immediately, and stop running this function."

__Exercise.__ See what happens if you connect to an IP / port combo that isn't listening for connections. Eg. try 127.0.0.3 port 123.


## Reading from a socket: Star Wars in the terminal

To receive bytes from the socket, we need somewhere to put them. We'll use a 1024-byte buffer:

```rust
let mut buf = [0u8; 1024];
```

This defines an array with 1024 entries, all of which are 0u8, ie 0 as an unsigned 8-bit number, aka a byte.

Now we're ready to receive using [recv][docs-nix-recv]. We'll need to add it to the `use` statemnt up top.  This function takes a flags argument, but we don't actually want to pass any flags, so we'll pass in empty flags.

```rust
let len = recv(sock, &mut buf, MsgFlags::empty())?;
```

`recv` returns the number of bytes it received and wrote into `buf`. So if we want just the newly received bytes, we can take a `len`-long slice of `buf`:

```rust
let new_bytes = &buf[..len];
```

To put this together, we're going to watch Star Wars Episode IV in our terminal! If we connect to `towel.blinkenlights.nl` at TCP port 23, we can have some fun.

But! We don't know how to turn a host name into an IP address yet. You'll have to look it up with `dig` or `nslookup` and hardcode the IP address. (Or, just take my word that it's 94.142.241.111!)

__Exercise.__ Write a loop that repeatedly calls `recv` and then writes those bytes to stdout. 

*Hint.* An infinite loop in Rust looks like this:
```
loop {
  // do stuff for a long long long long long time
}
```

*Hint.* Here's how to write bytes out to stdout:

```rust
io::stdout().write(bytes);
```

(You'll need to `use std::io` up top to bring the IO module in, and `use std::io::Write` to bring in the `write` method.)


## Writing to a socket, with some help from from a cat

Great, now we can connect out, and receive bytes. But we need to be able to make requests over th socket, so we'll have to be able to send bytes as well. The function that does that is `send`.

To help us see this in effect, we'll use netcat. For a quick demo, open up two terminals. In on of them, run

```
nc -lp 12345
```

This tells netcat to listen on port 12345. In the other one, run `nc localhost 12345`, which tells netcat to connect to port 12345 on your machine. You should now be able to type stuff in either terminal and have it show up in the other one. :-)

For our purposes, we'll just have the listening netcat. The role of the connecting netcat will be played by our program.

__Exercise.__ Change your program to connect to 127.0.0.1 port 12345, and write some bytes! They should show up in your listening netcat.

*Hint.* You can turn a string into bytes by writing `mystr.as_bytes()`, so pass something like `"never graduate!".as_bytes()` to `send`.

## Time for a tiny bit of abstraction!

We've got a handy little set of tools now. We can
- connect to a TCP address
- receive bytes
- send bytes

Let's extract this and package it up into a type with some methods. We'll define a struct! Here's a basic struct definition:

```rust
struct MyStruct {
    a_number: u32,
    a_string: String,
}
```

Now we can define some methods using an `impl` block:

```rust
impl MyStruct {
    fn print_it(&self) {
        for i in 0..self.a_number {
	    println!("{}", self.a_string);
	}
    }

    fn number(&self) -> u32 {
        self.a_number
    }

    fn set_number(&mut self, n: u32) {
        self.a_number = n;
    }
}
```

__Exercise.__ Define a `TcpSocket` struct, with one field `fd` of type `RawFd`. This is the type that `socket` returns: a raw file descriptor.

__Exercise.__ Define a `connect` method that takes an IP and port, and returns a `TcpSocket` connected to them. Since this might fail, we'll have it return a `Result` type like `main` does. Here's a signature:

```rust
    fn connect(ip: [u8; 4], port: u16) -> Result<TcpSocket, Box<std::error::Error>> {
        // STUFF
    }
```

__Exercise.__ Add a `recv` method that takes a buffer and receives bytes into it, returning the number of bytes received. Here's a signature:

```rust
    fn recv(&self, buf: &mut [u8]) -> Result<usize, Box<std::error::Error>> {
        // STUFF
    }
```

The `buf` is `&mut [u8]` which means a mutable "slice" of bytes. The `mut` means that we can write bytes in there, which is needed to be able to put the bytes we receive there. :-)

__Exercise.__ Add a `send` method that takes a buffer and sends bytes from it, returning the number of bytes sent. I bet you can figure out a signature for this!

*Hint* For sending bytes, we don't need to change the buffer, so you can leave off the `mut`!


## Learning a bit about traits via Read and Write

Remember further up where we wrote `use std::io::Write` to be able to write to stdout? `Write` is the interface for... writing. There's analogous one for reading called `Read`. Both `Read` and `Write` are traits, which is Rust-speak for an interface.

To make our little fledgling socket wrapper interoperate with Rust libraries, we can implement `Read` and `Write` for it ourselves!

To implement a trait, you make a special `impl` block, and then defined the required methods.

```rust
impl SomeTrait for MyStruct {
    // Required method definitions here!
}
```

For `Read` and `Write`, these are

```
    fn read(&mut self, buf: &mut [u8]) -> Result<usize, io::Error>
```

and
```
    fn write(&mut self, buf: &[u8]) -> Result<usize, io::Error>
```
respectively.

These signatures are really close to our own `send` and `recv`, so we can actually just implement them using the methods we already defined.

__Exercise.__ Implement `Read` and `Write` for `TcpSocket`.

*Hint* The big difference is the error type. To get an `io::Error`, call `io::Error::new(io::ErrorKind::Other, error).


__Exercise.__ Look up `io::copy` and go back to the Start Wars example and use it to replace the loop.


## Making an HTTP request

All right, let's make an HTTP request! We're going to make some requests to httpbin.org, a super handy service for messing around with HTTP. Here's where we'll start:

```
concat!(
"GET /get HTTP/1.1\r\n",
"Host: httpbin.org",
"\r\n\r\n");
```

__Exercise.__ Use your skillz to find an IP address for httpbin.org, and then use your `TcpSocket` to connect and send this HTTP request. Then `io::copy` the result to stdout.
