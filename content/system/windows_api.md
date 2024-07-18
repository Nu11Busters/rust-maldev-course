+++
title = "Windows API"
weight = 1
+++

## Introduction

There are several crates available for interacting with the Windows API.

The first is the winapi crate.
However, this crate is deprecated as it has not been updated since November 6, 2021.

The second is the `windows` crate developed by Microsoft, which is divided into 2 main crates:

* `windows`
* `windows-sys`

The `windows-sys` crate directly exposes API functions and structs, while the windows crate encompasses the API with an overlay to simplify its use with Rust.

To demonstrate the difference, here's a tangible example using the WinSock API:
In the `windows` library ([https://github.com/microsoft/windows-rs/blob/master/crates/libs/windows/src/Windows/Win32/Networking/MAX\_PREFERRED\_LENGTH WinSock/mod.rs](https://github.com/microsoft/windows-rs/blob/master/crates/libs/windows/src/Windows/Win32/Networking/MAX\_PREFERRED\_LENGTH WinSock/mod.rs)), line 12062 shows the implementation of the `fmt` trait for the `SOCKET_SECURITY_SETTINGS_IPSEC` structure, which is not the case in the `windows-sys` crate ([https://github.com/microsoft/windows-rs/blob/master/crates/libs/sys/src/Windows/Win32/Networking/WinSock/mod.rs##L5294](link)).

The link macro is declared in the `crates/libs/targets/src/lib.rs` file.

The argument put forward by Microsoft for the use of the `windows-sys` crate is compilation time, but as this argument is not important in this course, we will use the `windows` crate in this section.

You can find the crate documentation on this site.

## Preparing the environment

To include the crate in a project, add the crate to the dependencies in the `cargo.toml` file, for example:
```toml
[dependencies]
windows-sys = "0.52.0"
```

However, as the Windows API is so extensive in terms of functionality, this crate has many _features_. You can find them at [docs.rs](https://docs.rs/crate/windows-sys/latest/features).

Each _feature_ used in the examples will be indicated.

## Examples

#### Get user name and group

In this first example, we'll see how to retrieve the user name and group of the user who launched the program.

```rust
use windows::Win32::System::WindowsProgramming::GetUserNameA;
use windows::Win32::NetworkManagement::NetManagement::NetUserGetLocalGroups;
use windows::Win32::NetworkManagement::NetManagement::LG_INCLUDE_INDIRECT;
use windows::core::{PSTR, PCWSTR};
use windows::Win32::NetworkManagement::NetManagement::LOCALGROUP_USERS_INFO_0;
use windows::Win32::System::Diagnostics::Debug::{FORMAT_MESSAGE_FROM_SYSTEM, FORMAT_MESSAGE_IGNORE_INSERTS};
use windows::Win32::System::Diagnostics::Debug::FormatMessageA;
use windows::Win32::NetworkManagement::NetManagement::UNLEN;
use windows::Win32::NetworkManagement::NetManagement::MAX_PREFERRED_LENGTH;
use windows::Win32::NetworkManagement::NetManagement::NetApiBufferFree;
use std::ffi::CStr;
use std::ptr;
use std::ffi::c_void;

fn main() {
    let mut username_buf = vec![0u8; (UNLEN+1) as usize]; // creation of the buffer containing the user name
    let mut username_len: u32 = UNLEN+1;

    unsafe {
        let _ = GetUserNameA(PSTR {0: username_buf.as_mut_ptr()}, &mut username_len);
        let username_str = CStr::from_ptr(username_buf.as_mut_ptr() as *const i8).to_str().unwrap();
        println!("Current user: {}", username_str);
    }


    for i in (0..username_buf.len()).step_by(2) { // converts user name to utf-16
        username_buf.insert(i + 1, 0x00);
    }

    let res: u32;

    let mut groups_ptr: *mut u8 = ptr::null_mut();
    let mut entriesread: u32 = 0;
    let mut totalentries: u32 = 0;
    unsafe {
        res = NetUserGetLocalGroups(
            PCWSTR::null(), // local computer
            PCWSTR {
                0: username_buf.as_ptr() as *const u16
            },
            0,
            LG_INCLUDE_INDIRECT,
            &mut groups_ptr as *mut *mut u8,
            MAX_PREFERRED_LENGTH,
            &mut entriesread,
            &mut totalentries,
        );
    }

    if res != 0 {
        let mut err_msg_buffer: Vec<u8> = Vec::with_capacity(MAX_PREFERRED_LENGTH as usize);
        unsafe {
            FormatMessageA(
                FORMAT_MESSAGE_FROM_SYSTEM | FORMAT_MESSAGE_IGNORE_INSERTS,
                None,
                res,
                0,
                PSTR {
                    0: err_msg_buffer.as_mut_ptr()
                },
                MAX_PREFERRED_LENGTH as u32,
                None
            );
            eprint!("Unable to retrieve user group: {}", CStr::from_ptr(err_msg_buffer.as_mut_ptr() as *const i8).to_str().unwrap());
            NetApiBufferFree(Some(groups_ptr as *const c_void));
        }
        return ();
    }

    println!("User groups:");
    unsafe {
        let group_info: *const LOCALGROUP_USERS_INFO_0 = groups_ptr as *const LOCALGROUP_USERS_INFO_0;
        for i in 0..entriesread {
            let group_name = &(*group_info.offset(i as isize)).lgrui0_name;
            println!("- {}", group_name.to_string().unwrap());
        }
        NetApiBufferFree(Some(groups_ptr as *const c_void));
    }
}
```

## Functions used

In this example, we use the functions GetUserNameA and NetUserGetLocalGroups.

The parameters for the GetUserNameA function are:

1. `lbpbuffer`: A pointer to a buffer containing the string representing the user name. The pointer is of type `PSTR`, which is a type defined in WinAPI, and is a pointer to an ANSI string terminated by a null character ('\0'). This parameter is similar to a pointer to a C string (a `char**`). The maximum size of a username is `UNLEN`, which is 256. As indicated in the documentation, a buffer of size `UNLEN+1` can contain any username, the `+1` being for the null character.
2. `pcbbuffer`: This parameter has a dual function: when passed as a parameter to the function, it contains the size of the buffer that will hold the character string (passed in the `lbpbuffer` parameter). Once the function has been executed, the variable contains the size of the username plus the null character (the number of characters written in the buffer, in short). This double functionality is indicated in [Microsoft documentation](https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-getusernamea##parameters) by the words `[in, out]` to the left of the `pcbbuffer` parameter name.

This function may fail if the buffer is too small to hold the username, in which case it returns 0 and you need to use the `GetLastError` function to retrieve the error name. In this program, we have a buffer large enough to hold any user name.

The parameters for the `NetUserGetLocalGroups` function are:

1. `servername`: A pointer to a buffer which is a string encoded in UTF-16 indicating the name of the server on which we wish to execute the function. This string must be a constant, but this is not necessary here. The pointer is of type `PCWSTR`. However, as explained in the documentation, we give a pointer to null using `PCWSTR::null()` to indicate that we wish to execute the function on the local computer.
2. `username`: A pointer to a buffer which is a string encoded in UTF-16 indicating the username whose groups we wish to retrieve. As with `servername`, this parameter is of type `PCWSTR`.
3. `level`: A 32-bit unsigned integer (`u32`) indicating the level of information to be retrieved. In the context of this function, level 0 is the only level that can be used, indicating that basic information on local groups is required.
4. `flags`: A 32-bit unsigned integer (`u32`) specifying control flags. In this case, `LG_INCLUDE_INDIRECT` is often used, indicating that indirect groups (groups to which the user belongs via other groups) should also be included in the result.
5. `bufptr`: A pointer to an uninitialized buffer where groups will be stored. Here the type is `*mut *mut u8`, which can be seen as a `char**` in C. Here the result is converted into `LOCALGROUP_USERS_INFO_0`, which is a list of structures containing the groups. What interests us in these structures is the _LUID_, which we'll see later.
6. `prefmaxlen`: A 32-bit unsigned integer (`u32`) specifying the preferred buffer size initialized by the API. If the specified size is too small to receive the information, the function will return `ERROR_MORE_DATA`, and `prefmaxlen` will be updated with the required size, which can be handy in some cases. Here we use the constant `MAX_PREFERRED_LENGTH`, which indicates that we wish to have as much information as possible in the buffer.
7. `entriesread`: A pointer to a 32-bit unsigned integer (`*mut u32`) which will receive the number of entries read, in this case the number of groups.
8. `totalentries`: This is a pointer to a 32-bit unsigned integer (`*mut u32`) which contains the number of entries that could have been returned, in this case we want the totality of information.

This function can also fail, in which case it returns an error code. It is possible to have this error code as a character string using the `FormatMessageA` function.

This function takes as parameter:
1. `dwflags`: bitflags indicating what we wish to display, here thanks to the `FORMAT_MESSAGE_FROM_SYSTEM` and `FORMAT_MESSAGE_IGNORE_INSERTS` flags, we retrieve the raw character string from the system.
2. `lpsource`: the message location, not used here.
3. `dwmessageid`: A 32-bit unsigned integer (`u32`) representing the error identifier whose string representation we wish to retrieve. Here we pass the result of the `NetUserGetLocalGroups` function.
4. `dwlanguageid`: A 32-bit unsigned integer (`u32`) representing which language the string will be in. Here we specify 0 for the system language.
5. `lpbuffer`: A pointer to a buffer that will contain the string representing the error. The pointer is of type `PSTR`.
6. `nsize`: A 32-bit unsigned integer (`u32`) containing the size of the buffer passed in the `lpbuffer` argument.
7. `Arguments`: List of values used when formatting the message. This can be thought of as the arguments passed to `prinln!` after the first parameter. Here, we're not doing any formatting, so we don't put anything.

#### Code explanations

As you can see from the above description of the function parameters, the `GetUserNameA` function returns the username in ANSI, whereas the `NetUserGetLocalGroups` function requests the username encoded in UTF-16. This encoding difference is addressed by the following code, which adds a null character between each ANSI character in the username:

```rust
for i in (0..username_buf.len()).step_by(2) {
    username_buf.insert(i + 1, 0x00);
}
```

In the last part of the program, we have this part displaying the groups as strings:
```rust
let group_info: *const LOCALGROUP_USERS_INFO_0 = groups_ptr as *const LOCALGROUP_USERS_INFO_0;
for i in 0..entriesread {
    let group_name = &(*group_info.offset(i as isize)).lgrui0_name;
    println!("- {}", group_name.to_string().unwrap());
}
```

In this code, we retrieve the group name using the `lgrui0_name` attribute, which is of type `PWSTR`. However, as we're retrieving a pointer to an array, we need to use the offset method to move around the array.

You may also note that the `NetApiBufferFree` function is used to free the pointer to groups. This is necessary, as indicated in the documentation for the `bufptr` parameter of the `NetUserGetLocalGroups` function. This requirement marks the use of the C++ language in the API, as it is not present in Rust.

To complete this example, and as explained in the previous section, it is necessary to use _features_.

The _features_ used in this program are:
```
[dependencies.windows]
features = [
    "Win32_Foundation",
    "Win32_System_WindowsProgramming",
    "Win32_NetworkManagement_NetManagement",
    "Win32_System_Diagnostics_Debug"
]
```

You can see a certain patern in feature names, for example in the use `windows::Win32::NetworkManagement::NetManagement::UNLEN`, which is part of the "Win32\_NetworkManagement\_NetManagement" feature.

#### Recovering process privileges

This second example shows how to recover process privileges.
```
use windows::Win32::Foundation::HANDLE;
use windows::Win32::System::Threading::GetCurrentProcess;
use windows::Win32::System::Threading::OpenProcessToken;
use windows::Win32::Security::GetTokenInformation;
use windows::Win32::Security::TokenPrivileges;
use windows::Win32::Security::TOKEN_PRIVILEGES;
use windows::Win32::Security::TOKEN_QUERY;
use windows::Win32::Security::LookupPrivilegeNameA;
use windows::core::PSTR;
use std::ffi::CStr;
use std::ptr;

fn main() {
    unsafe {
        let current_process = GetCurrentProcess();

        let mut process_token = HANDLE::default();
        let ptr_token : *mut HANDLE = &mut process_token;

        let _ = OpenProcessToken(current_process, TOKEN_QUERY, ptr_token);

        let mut privileges_token : *mut TOKEN_PRIVILEGES = ptr::null_mut();
        let mut privileges_token_length = 0u32;

        // Here we retrieve the size of the token privileges buffer in the "privileges_token_length" variable.
        let _ = GetTokenInformation(process_token,
            TokenPrivileges,
            Some(privileges_token as *mut std::ffi::c_void),
            0,
            &mut privileges_token_length as *mut u32
        );

        // Create the buffer that will contain the privileges token
        let mut token_privileges_vec = Vec::with_capacity(privileges_token_length as usize);
        privileges_token = token_privileges_vec.as_mut_ptr() as *mut TOKEN_PRIVILEGES;

        match GetTokenInformation(process_token, TokenPrivileges, Some(privileges_token as *mut std::ffi::c_void), privileges_token_length, &mut privileges_token_length as *mut u32) {
            Err(_) => {
                eprintln!("Unable to recover process privileges");
                return ;
            },
            Ok(_) => {
                println!("Process privileges:");
                // For each privilege, we retrieve its name using its LUID
                for privilege in (*privileges_token).Privileges {
                    let mut priv_name = vec![0u8; 256 as usize];
                    let mut priv_name_length = 256;
                    let _ = LookupPrivilegeNameA(
                        None,
                        &(privilege.Luid),
                        PSTR {0:priv_name.as_mut_ptr()},
                        &mut priv_name_length
                    );
                    println!("- {:?}", CStr::from_ptr(priv_name.as_mut_ptr() as *const i8).to_str().unwrap());
                }
            }
        }
    }
}
```

#### The functions

In this example, we're using the following functions. The order of the list below is important, as it shows how the program works and how the token is used to retrieve process privileges:

* `GetCurrentProcess`: This function returns a pseudo-handle to the process calling this function (we'll see what a handle is next). This value is the constant `-1` converted into a `Handle` (we advise you not to raw this value for compatibility reasons). As you can see, this value is not the PID of the current process.
* `OpenProcessToken`: This function retrieves a token from a process via its `Handle`.
* `GetTokenInformation`: This function retrieves specific information about a token. In our case, we'll retrieve the privileges associated with the token.
* `LookupPrivilegeNameA`: This function allows us to convert a privilege in numeric format into a human-readable character string.

The `GetCurrentProcess` function has no parameters; it simply returns a pseudo-handle to the calling process.

The parameters for the `OpenProcessToken` function are:

1. `ProcessHandle`: The handle to the process from which we wish to retrieve a token.
2. `DesiredAccess`: A bitmask indicating which tokens we wish to retrieve, here we indicate only `TokenPrivileges`.
3. `TokenHandle`: Is a pointer to a handle where the handle to the token will be written (handles are used to point to tokens).

The parameters for the `GetTokenInformation` function are:

1. `TokenHandle`: This is a handle to the token whose information you wish to retrieve, in this case the token retrieved with the `OpenProcessToken` function.
2. `TokenInformationClass`: This is a value of the `TOKEN_INFORMATION_CLASS` enumeration, specifying the type of information you wish to retrieve. Several values are therefore possible, but in our case we use the TokenPrivileges value.
3. `TokenInformation`: A pointer to a buffer that will receive the requested information. It's important to note that the structure and size of this buffer depend on the type of information specified by `TokenInformationClass`.
4. `TokenInformationLength`: The size, in bytes, of the buffer pointed to by `TokenInformation`.
5. `ReturnLength`: A pointer to a 32-bit unsigned integer (`u32`) that receives the actual size of the token information. Even if the buffer indicated in the `TokenInformation` parameter is too small, this value will be filled with the size needed to store the information. We use this feature to dynamically obtain the right size for our buffer. Note that if the buffer is not large enough, the function will return an error.

The parameters for the `GetTokenInformation` function are:

1. `lpSystemName`: A pointer to a string specifying the name of the system in which to search for the privilege. None can be specified for the local system.
2. `lpLuid`: A pointer to a _LUID_ (Locally Unique Identifier) structure that identifies the privilege whose name you wish to obtain.
3. `lpName`: A pointer to a buffer that will receive the privilege name as a character string. This buffer must be large enough to hold the name.
4. `cchName`: A pointer to a 32-bit unsigned integer (`u32`) which specifies the size of the input buffer. After the call, this variable contains the number of characters written in the buffer to store the privilege name (hence the reference `[in, out]` to the left of this parameter name in the documentation).

#### Handles

A handle is a reference to a resource in or on the system, such as files, processes, threads, graphical objects or sockets.

These handles are used by applications to access and manipulate these resources (as shown in this program).

It should be noted that each time a resource is opened or created, the operating system allocates a handle to this resource and returns this identifier to the application. A handle to a resource will therefore be different on every process in the system, and is not a unique identifier.

#### Code explanation

This program works as follows:

1. A handle is retrieved from the current program.
2. Use the previously retrieved handle to retrieve the token of privileges associated with the process.
3. Retrieve the privileges associated with the retrieved token in the form of identifiers (each privilege is therefore an identifier).
4. Convert privilege identifiers into human-readable character strings

As you can see, in this code we make 2 calls to the `GetTokenInformation` function, the first of which retrieves the size of the token structure in bytes, as this is required to create a buffer of sufficient size.

To enable human-readable privileges, we use the `LookupPrivilegeNameA` function, which converts the privilege number (_LUID_) into a character string.

The features required in this program are:

```toml
[dependencies.windows]
features = [
    "Win32_Foundation",
    "Win32_System",
    "Win32_System_Threading",
    "Win32_Security",
    "Win32_System_Diagnostics_Debug"
]
```

## Conclusion

The `windows-sys` crate created by Microsoft makes it much easier to execute Windows API functions. This library provides links to Windows API functions and constants in Rust.

However, a thorough understanding of the official Windows API documentation in C++ is essential to create Rust programs with the `windows-sys` crate. C++ documentation is necessary to understand the specific details of the functions, data types and structures used in the API. In parallel, it is necessary to consult the equivalents available in the `windows-sys` crate in the documentation to see any differences and simplifications made by Microsoft.

In short, although the `windows-sys` crate simplifies the process of calling Windows API functions in Rust, a solid reference to the C++ documentation remains indispensable for using the API correctly in Rust.
