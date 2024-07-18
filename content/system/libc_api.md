+++
title = "Libc API"
weight = 2
+++

## Introduction

#### LibC

The [**libc**](https://docs.rs/crate/libc/) crate in Rust simplifies the use of most of the functions available in libc.

The main features of this crate are :

* Declaration of data types: The crate defines data types such as **c_int**, **c_char** and **c_void**, which correspond to the *int*, *char* and *void* types respectively. These types are used to simplify the use and calling of functions written in C.
* Libc constants : The crate exposes a large number of constants present in the standard C library, such as error codes (`errno`) or signal constants. These constants facilitate the use of system APIs.
* Libc functions: libc exposes the most widely used functions of the standard C library, simplifying low-level operations such as file manipulation or process management.

Please note that not all libc functions are described in the crate, so please consult the documentation.

Unfortunately, it is not possible to use the functions available in this crate without using `unsafe` blocks.

This crate uses features that can be found in the crate documentation at [docs.rs](https://docs.rs/crate/libc/latest/features).

#### NIX

The [**Nix**](https://docs.rs/crate/nix/) crate exposes wrappers around the main features of libc to make it safer and easier to use.

The crate is divided into features. You can find a list of these features on [docs.rs](https://docs.rs/crate/nix/latest/features).

In this section, we'll look at how to use the two crates through three simple examples.

## Usage

To include one of the crates in a project, simply add the crate to the dependencies in the `Cargo.toml` file, as in the case of the `libc` crate:

```toml
[dependencies]
libc = "0.2.150"
```

For this same crate, it is possible to use it without using the standard library (**std**).
To do so, you need to specify the absence of default *features* (and possibly activate the other *features* one by one):

```toml
[dependencies]
libc = { version = "0.2.150", default-features = false }
```

#### Retrieving the user and group with which the program is running

You can retrieve the user and group with which the process is running using the [`geteuid`](https://linux.die.net/man/2/geteuid) function and the [`getegid`](https://linux.die.net/man/2/getegid) function respectively.

Using the `libc` crate:

```rust
extern crate libc;

fn main() {
    unsafe {
        let uid = libc::geteuid();
        println!("Current user's UID: {}", uid);

        let gid = libc::getegid();
        println!("GID of current group: {}", gid);
    }
}
```

For the **Nix** crate, the code is virtually the same except for the absence of the unsafe block:
```rust
extern crate nix;
use nix::unistd::{geteuid, getegid};
fn main() {
    let uid = geteuid();
    println!("Current user's UID: {}", uid);

    let gid = getegid();
    println!("GID of current group: {}", gid);
}
```

Note that you need to use the [user](https://docs.rs/crate/nix/latest/features##user) _feature_ to access both functions.

#### Displaying the current directory

In this example, we'll see how to retrieve the current directory by specifying the buffer where the string will be stored, passing it by reference.
```rust
extern crate libc;

fn main() {
    unsafe {
        let mut buffer: [libc::c_char; libc::PATH_MAX as usize] = [0; libc::PATH_MAX as usize];

        if libc::getcwd(buffer.as_mut_ptr(), buffer.len() as libc::size_t).is_null() {
            eprintln!("Erreur lors de l'obtention du répertoire de travail actuel");
        } else {
            let cwd = std::ffi::CStr::from_ptr(buffer.as_ptr());
            println!("Répertoire de travail actuel : {:?}", cwd);
        }
    }
}
```

You can see that we're using an array with the size `libc::PATH_MAX` and initialized to 0, just as you'd find in C.

The `std::ffi::CStr::from_ptr` function allows you to convert the string and more securely use the return string as it comes from a function written in C.

The version using the **Nix** crate is considerably simpler:
```rust
extern crate nix;

use nix::unistd::getcwd;

fn main() {
    match getcwd() {
        Ok(cwd) => {
            let current_dir = cwd.to_string_lossy();
            println!("Current working directory: {}", current_dir);
        }
        Err(_) => {
            eprintln!("Could not get current working directory.");
        }
    }
}
```

The _feature_ required to use the [`getcwd`](https://docs.rs/crate/nix/latest/features##fs) function is [fs](https://docs.rs/crate/nix/latest/features##fs).

#### TCP socket

Here's how to create a TCP socket using the **libc** crate to connect to address 127.0.0.1:12345 (works on x86 or x64 systems):

```rust
extern crate libc;
use std::net::Ipv4Addr;

fn main() {
    unsafe {
        // Create a TCP socket (SOCK_STREAM)
        let socket_fd = libc::socket(libc::AF_INET, libc::SOCK_STREAM, 0);

        if socket_fd == -1 {
            eprintln!("Error during socket creation");
            return;
        }

        // Fills struct sockaddr_in with server address (here '127.0.0.1:12345')
        let mut server_address: libc::sockaddr_in = std::mem::zeroed();
        server_address.sin_family = libc::AF_INET as libc::sa_family_t;
        server_address.sin_addr.s_addr = u32::from_le_bytes("127.0.0.1".parse::<Ipv4Addr>().unwrap().octets());
        server_address.sin_port = u16::to_be(12345);

        // connect system call
        if libc::connect(
            socket_fd,
            &server_address as *const _ as *const libc::sockaddr,
            std::mem::size_of_val(&server_address) as libc::socklen_t,
        ) == -1
        {
            eprintln!("Could not connect");
            libc::close(socket_fd);
            return;
        }

        println!("Connected to 127.0.0.1:12345");

        // Close connection
        libc::close(socket_fd);
    }
}
```

You can see that the `htons` and `inet_addr` functions are not used when creating the `sockaddr_in` structure.

This is because they are not available in the crate, so you have to use other language functions to achieve the same result.

Here, we retrieve the IP address required in `s_addr` by converting the IP address into an `IPv4Addr`, available in the `std::net` library, then retrieving the bytes that make it up to form a `u32`. You can replace the `u32::from_le_bytes` function with `u32::from_be_bytes` on big-endian systems like ARM.
To get the port in the right format, convert the port number into a `u16` in big-endian.

The Nix version is much simpler:
```rust
extern crate nix;

use std::os::fd::AsRawFd;
use nix::sys::socket::{socket, connect, AddressFamily, SockType, SockaddrIn};

fn main() {
    // Create a TCP socket
    let socket_fd = socket(
        AddressFamily::Inet,
        SockType::Stream,
        nix::sys::socket::SockFlag::empty(),
        None,
    );

    match socket_fd {
        Ok(fd) => {
            // Set IP address and port
            let server_addr = SockaddrIn::new(127, 0, 0, 1, 12345);

            // Connect to the server
            match connect(fd.as_raw_fd(), &server_addr) {
                Ok(_) => {
                    println!("Connected to 127.0.0.1:12345");
                }
                Err(e) => {
                    eprintln!("Server connection error : {}", e);
                }
            }
        }
        Err(e) => {
            eprintln!("Could not create socket : {}", e);
        }
    }
}
```

As you can see, no type conversions are required. The use of file descriptors defined in the os module of the standard library clearly shows how the libc fits into the language thanks to **Nix**, whereas the crate libc simply exposes the functions.
Conclusion


In conclusion, the crate **libc** in Rust is a simple interface to the standard C library. It is particularly useful for low-level operations and interoperability with C code, which is very useful for malware development.

The **Nix** crate is a wrapper around the library, facilitating development in Rust as well as program stability.

For detailed information on a libc function, you can consult the man-pages available on any Linux system. They provide detailed documentation of standard library functions.

To consult a man-page, use the command `man` followed by the function name. For example, to obtain information on the malloc function, you can run `man malloc` in the terminal.
There are also many online resources providing man-pages, which can be useful. You can find online versions on sites such as [man7](https://man7.org/linux/man-pages/).
