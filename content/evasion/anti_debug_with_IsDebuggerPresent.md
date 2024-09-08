+++
title = "IsDebuggerPresent"
weight = 3
+++

Detecting a debugger attached to our process is very simple on Windows.

The Windows API contains a function called `IsDebuggerPresent`, which, as its name suggests, allows you to find out whether a debugger is present by returning a boolean.

To use it in our program, we need to use the `windows-sys` crate with the `Win32_Foundation` and `Win32_System_Diagnostics_Debug` _features_:
```toml
[dependencies]
windows-sys = "0.52.0"

[dependencies.windows]
features = ["Win32_Foundation", "Win32_System_Diagnostics_Debug"]
```

Its use is very simple, as the following code demonstrates (note that this function must be called in an unsafe block):

```rust
use windows::Win32::{System::Diagnostics::Debug::IsDebuggerPresent, Foundation::TRUE};

fn main() {
    unsafe {
        if IsDebuggerPresent() == TRUE {
            println!("Debugger attached!")
        } else {
            println!("No debugger attached.")
        }
    }
}
```

When we run our program without using a debugger, we can see that the string "No debugger attached." is displayed:

![](../without_windbg.png)

However, if you run it with WinDBG attached, you can see that the string "Debugger attached!" is displayed:

![](../windbg.png)
