+++
title = "Anti-debug with PTRACE"
weight = 2
+++

In this section, we'll look at how to protect ourselves against programs that attempt to dynamically analyze our malware (debuggers, EDRs, etc.) on Linux.

On Linux, the `ptrace` system call allows you to attach yourself to a process and control its execution, modify its memory space and consult its registers. It is used by debuggers such as GDB. Attaching to a process with this system call is called 'tracing'.

It's important to remember that a process can only be traced by a single tracer, as any new tracing attempt will return a "Operation not permitted" error.

In this section, we're going to use a self-tracing technique. If we manage to trace ourselves, then other programs won't be able to trace us, and if we don't manage to trace ourselves, this means that a program is already doing so (if our program is launched with GDB, for example) and we quit immediately.

To do this, we're going to use the **Nix** library with the ptrace _feature_:
```toml
[dependencies]
nix = { version = "0.27.1", features = ["ptrace"] }
```

**Nix** wraps the various parameters available in ptrace in separate functions provided in the `nix::sys::ptrace` module.

We're going to use the `traceme` function. This function indicates that our process is being traced by its parent, even if the parent is really not tracing our process.

Here's an example calling `traceme`, if the function fails the program quits, otherwise it waits 10000 seconds:
```
use nix::sys::ptrace::traceme;
use std::time::Duration;

fn main() {
    match traceme() {
        Ok(_) => std::thread::sleep(Duration::new(10000, 0)),
        Err(e) => {
            eprintln!("traceme() failed: {}", e)
        }
    }
}
```

Now, when we try to attach GDB to the program, even as root, we get an error indicating that GDB can't attach because the program is already traced:

```bash
>>> ptrace_antidebug$ ./target/debug/ptrace_antidebug &
[1] 11850
>>> ptrace_antidebug$ gdb --pid=11850

...

Attaching to process 11850
warning: process 11850 is already traced by process 10323
ptrace: Operation not permitted.
(gdb) quit
>>> ptrace_antidebug$ sudo gdb --pid=11850

...

Attaching to process 11850
warning: process 11850 is already traced by process 10323
ptrace: Operation not permitted.
(gdb) quit
>>>
```
