---
title: "LD_PRELOAD Hijacking: From the Linker to Shellcode Injection"
date: 2026-03-07 21:29:02 +0900
categories: [Blog]
tags: [linux, evasion, tradecraft, opsec, linker, ld_preload]
author: arnp
---

The `LD_PRELOAD` hijacking technique is one of the most known persistence techniques used in Linux environments. It has already been described in many blog articles but it is often difficult to understand how abusing library priority loading mechanism from the Linux linker may lead to process thread injection and persistence.

This post goes from first principles (how the dynamic linker actually works) through practical tradecraft and finishes with the forensic artifacts this techniques leaves behind and how to detect them.

## Disclaimer

This content is provided for educational and defensive security purposes only. 
The techniques described are presented to help security professionals understand threats and improve defensive capabilities.

All techniques should only be used in authorized testing environments where you have explicit permission. Unauthorized access to computer systems is illegal in most jurisdictions. The author assumes no liability for misuse of this information.

## 1. The Linux Linker

To understand how `LD_PRELOAD` works, it is important to first understand what problem the dynamic linker was designed to solve and how it can be potentially be abused.

### Why Is This Needed?

The simple answer to this question is: "Because programs are shared and expected to run in different environments".
What it means is that when one compiles a C program that calls `printf()`, the compiler does not copy the source code of `printf` out of `glibc`. If it did, every single program containing a call to this function would need to embed a private copy of the entire standard library (note that languages such as Rust or Go, compiled statically, embed the required part of the standard library into their compiled programs).

Instead, the compiler (such as `gcc`) produces what we call a dynamically linked ELF binary, in which the code for `printf` lives in a single shared file (`libc.so`) on the system, that every program references at runtime. This is more memory-efficient and allows to easily deploy security patches to every program using the standard library (in our example) without requiring to recompile them. 

### ELF's Dynamic Section

As just mentioned, the static linker (`ld`) does not actually embed the code for external functions at build time. Instead, it is going to write metadata into a special region of the ELF binary (Executable and **Linkable** Format, the standard binary file format for Linux executables) called the `Dynamic Section`. This section can be viewed with the `readelf -d $BINARY_PATH` command and contains the following two critical pieces of information:

* `NEEDED` entries: a list of `.so` files this binary depends on (such as `libc.so.6` or `libpthread.so.6`).
* A pointer to the `interpreter`: the path to the runtime linker itself, typically `/lib64/ld-linux-x86-64.so.2`, stored in the `.interp` section.

Using these two metadata, the ELF binary can inform the Kernel that it requires specific libraries to run and that the runtime linker specified in `.interp` section should load them.

For example, the `/bin/ssh` binary requires the following libraries to run:
![]({{ site.baseurl }}/assets/images/posts/ld_preload-abuse/dynamic_needed.png)

It also specifies the path of the linker to be used for loading the libraries:
![]({{ site.baseurl }}/assets/images/posts/ld_preload-abuse/dynamic_interpreter.png)

### The Runtime Linker's Job

When running an ELF executable, the Kernel will first map the ELF binary into memory, read its `.interp` section and hand control to the runtime dynamic linker set (such as `/lib64/ld-linux-x86-64.so.2`) instead of the program's `main()` function.

The runtime dynamic linker then performs the following operations:
* Load the dependencies: The linker reads the ELF `Dynamic Section`, checks the `NEEDED` entries and searches for the corresponding `.so` files on the filesystem.
The search follows a priority-ordered search path (in a similar way to the `$PATH` environment variable), that can be found in the [ld.so.8 man page](https://man7.org/linux/man-pages/man8/ld.so.8.html). The order is the following:
  * `LD_PRELOAD` environment variable.
  * `--preload` command-line option when invoking the dynamic linker.
  * `/etc/ld.so.preload` file.
  * `DT_RPATH` ELF binary tag (embedded in the binary's `Dynamic Section`), if `DT_RUNPATH` is absent.
  * `LD_LIBRARY_PATH` environment variable.
  * `DT_RUNPATH` ELF binary tag, if present (in this case, `DT_RPATH` is ignored).
  * `/etc/ld.so.cache` file.
  * `/lib64` directory (or `/lib` for 32-bit machines).
  * `/usr/lib64` (or `/usr/lib` for 32-bit machines).
* Relocating the dependencies: Even if the library files have been found, the linker does not know in advance where in memory each library will land (because of the `Address Space Layout Randomization` security feature randomizing memory locations). To remediate that, it leverages the two following ELF sections:
  * The `Procedure Linkage Table` (`.plt`): The PLT is a small stub of code generated by the compiler for each external function call. Taking the example of `printf()` again, a call to this function actually causes a jump to the PLT stub, which on its first call, triggers the linker's resolver to look up the real address of the loaded function.
  * The `Global Offset Table` (`.got`/`.got.plt`): The GOT is the actual address book used by the program at runtime. Once the linker resolves a symbol (such as `printf()`'s real memory address), it writes this memory address into the GOT. Future calls to the PLT stub skip the resolution process and directly jump to the GOT entry containing the memory address for the function. This mechanism is called `lazy binding` and is also the reason why the first call to a function is slightly slower than the next call.

Schematically, this mechanism may be represented as follows:
```
Code                  PLT                     GOT
-----------           -------                 -------
call printf   ---->   printf@plt:             printf@got:
                        jmp  [GOT entry]  --->  [1st call: linker resolver]
                                                [next calls: real printf()]
```

### Transitive Dependencies & Recursive Load Chain

Now that we have described the behaviour of the linker, we must note some exceptions to this general behaviour.

When the linker checks each entry from the `NEEDED` binary's list, it does not only load the corresponding library. Instead, it would also load each dependency library found in the library's (which is also an ELF shared object) `Dynamic Section` and its own `NEEDED` entries, in a recursive manner, until every dependency in the entire chain is loaded.
These recursive loadings have for particularity to be not be listed when inspecting the main binary with the `readelf` command.

Let's take the example of `/bin/ssh` again.

![]({{ site.baseurl }}/assets/images/posts/ld_preload-abuse/dynamic_needed.png)

We can see that `libgssapi_krb5.so.2` is listed as a dependency for the binary.

But `libgssapi_krb5.so.2` itself depends on other libraries, which may also themselves depend on others.

![]({{ site.baseurl }}/assets/images/posts/ld_preload-abuse/dynamic_libgssapi_krb5.png)

Checking the binary dependencies using `ldd`, we can see that all the dependencies from `libgssapi_krb5.so.2` are also loaded.

![]({{ site.baseurl }}/assets/images/posts/ld_preload-abuse/dynamic_ldd_ssh.png)

This has meaningful offensive implications as a defender inspecting the binary with `readelf` only would get an incomplete picture of its runtime attack surface.
If an attacker crafts a malicious `.so` library designed to shadow a symbol exported by a transitive dependency (a dependency from a dependecy, loaded recursively), a `readelf`-based inspection would show no evidence of the symbols from the malicious library being hooked. The malicious hook would only become visible when using the `ldd` command against the original binary or when inspecting the live process's `/proc/PID/maps` at runtime.

![]({{ site.baseurl }}/assets/images/posts/ld_preload-abuse/ping_proc_maps.png)

This step is also a useful reconnaissance step before planning any injection using `LD_PRELOAD` abuse. Rather than targeting an obvious `libc` function like `open()`, which may be actively watched, running `ldd` on the target binary may allow an attacker to get a clear overview of each library in the load chain and identify lower-profile hooking opportunities using transitive dependencies that would be less monitored.

### Trusting the Linker

The security of the entire library loading process rely on the implicit trust from applications against the linker. In other words, applications do not check whether or not the linker finds and loads the real `printf()` function. This is reinforced by the fact that the linker itself assumes that the libraries it finds are legitimate.

The concept of the `LD_PRELOAD` environment variable (and of any other loading order overriding technique) violates this assumption by design as this mechanism is mainly used by developers to load and test new library versions or intercept functions for debugging, also opening a way for attackers to establish persistence over a system. 

## 2. The Classic Symbol Shadowing Approach

The most common way of abusing the `LD_PRELOAD` mechanism is a technique named symbol shadowing. This technique consists of defining a malicious function with the exact same name as the target library function, building it as a library and adding the malicious library path to the `LD_PRELOAD` environment variable (or any other top-level entry from the linker priority-ordered search order) so the malicious symbol may be resolved instead of the original one.

The example below shows how to intercept calls to the `open()` function in a transparent way (the original function is still called for better obfuscation and system stability):
```c
#define _GNU_SOURCE
#include <dlfcn.h>
#include <fcntl.h>
#include <stdio.h>
#include <stdarg.h>

// Define a function pointer type that matches the signature of the real open() syscall wrapper.
typedef int (*orig_open_t)(const char *pathname, int flags, ...);

// Declare a function with the exact same name and signature as the standard open() in libc.
int open(const char *pathname, int flags, ...) {
    
    // Locate the next definition of "open" in the shared library search order (the real "open" function).
    orig_open_t real_open = (orig_open_t)dlsym(RTLD_NEXT, "open");

    // The malicious payload. In this PoC, we just print the pathname of the file to be opened.
    printf("open() called on : %s\n", pathname);

    // Initialize a va_list to iterate over the variadic arguments received (the extra arguments passed via the "..." part of the function's prototype).
    va_list args;
    va_start(args, flags);
    
    // Extract the mode argument (file permission bits, like 0644) from the variadic list as it is necessary to forward the call to the real open().
    mode_t mode = va_arg(args, mode_t);
    
    // Clean up the va_list to avoid undefined behavior after processing the variadic arguments.
    va_end(args);
    
    // Call the real open() function with all the original arguments intact and return its result to the caller, making the hook transparent.
    return real_open(pathname, flags, mode);
}
```

To implement the technique, we just need to compile the code as a shared library, set its path in the `LD_PRELOAD` environment variable before executing a binary that calls the `open()` syscall.

```bash
gcc -shared -fPIC -o symbol_shadowing_poc.so symbol_shadowing_poc.c -ldl

LD_PRELOAD=symbol_shadowing_poc.so cat /etc/hostname
```

![]({{ site.baseurl }}/assets/images/posts/ld_preload-abuse/symbol_shadowing.png)

Every call to `cat` (or any other command using the `open()` libc function) now routes through the hook and calls the malicious function, which then performs any action defined by the attacker (in our case, printing the filepath to be opened) before calling the real `open()` function by forwarding the arguments originally received and returning the function's result to the caller.

We can confirm that `symbol_shadowing_poc.so` is called using the `LD_DEBUG=libs` environment variable.

![]({{ site.baseurl }}/assets/images/posts/ld_preload-abuse/symbol_shadowing_ld_debug.png)

This technique is powerful but has two big weaknesses:
* The attacker needs to wait for the program to call a specific function, which could also be called more often than expected, preventing the attack to be implemented or on the contrary, causing some system instability and increasing the chances of being detected.
* The malicious library must be injected in the linker search order, which may be either very complex or easily detectable by security solutions.

### Delivering the Library

As mentioned in the previous section, building the malicious hook is only half of the problem.
For the technique to work, the library path must be inserted in the linker search order with a high priority using components such as the `LD_PRELOAD` environment variable or the `/etc/ld.so.preload` file.

This is also the most difficult part of the operation and where the practical aspect of the technique becomes more complex than the theory.

#### The Naive Way

The most naive and common way to set the `LD_PRELOAD` environment variable would be to use the `~/.bashrc` file. It is also one of the worst way to proceed in practice as this file is one of the first files an investigator would check on a compromised machine. Beyond the visibility problem, it fires on every new interactive shell, which may also lead to unwanted repeated triggers of the malicious payload.

Writing to `/etc/ld.so.preload` would allow to add the malicious library into the `ld` search path at a global level. It however requires `root` privileges and is probably worse by every metric expect persistence.
The `/etc/ld.so.preload` file is a globally-monitored file and systems such as the Wazuh Rootcheck regularly check its integrity. Furthermore, it injects the malicious library into every dynamically linked process on the machine, meaning that any call to the target function by any process would trigger the payload and leading to system instability. The amount of anomalous activity becomes that important that detection is near certain before being able to leverage the injected payload.

Hopefully, there are multiple other stealthier alternatives to these two naive approaches. Those will be covered in the next sections.

#### Virtual Environment Activation Scripts
Used by developers in multiple languages, activation scripts are an effective vector on developer machines. When a Python developer runs `source .venv/bin/activate` to enable their Python virtual environment, the `activate` file may include a line such as `export LD_PRELOAD=/path/to/malicious/lib.so` to important the malicious library. Targeting `.venv/bin/deatctivate` is even more counterintuitive as an investigator auditing a Python project's virtual environment for malicious persistence would surely check the activation script instead of the deactivation script. Also, the variable set on deactivation would also be inherited by any process spawned in that shell session afterwards and would not be reset by the activation script, making it a powerful vector.

#### Files Sourced by Other Files
In a similar way to transitive dependencies, setting the `LD_PRELOAD` environment variable in a script loaded or sourced by another script would make the trail harder to follow. For example, an investigator examining a `.bashrc` file and seeing `. "$HOME/.cargo/env` (which is a common setup when installing Rust on a Linux machine) will likely not open the `.cargo/env` file. Still, infecting this rarely monitored and legitimate file may increase the chance of the `LD_PRELOAD` variable setting location to remain hidden. We may also extend this concept to other legitimate files such as `.zshrc` sourcing `$ZSH/oh-my-zsh.sh`.

#### Profile Files
`~/.profile` (or `~/.bash_profile`) may be used as an alternative to `.bashrc` with the main advantage that it is only sourced by login shells such as SSH sessions, PAM-authenticated desktop logins or `su -` invocations. While this file may often be checked by investigators, it may reduce the execution frequency and help to ensure that the variable is only set for high-value sessions.

## 3. The Hookless Constructor Approach

We have just seen that the classic symbol shadowing technique has several weakness and relies on a specific function being call by a program in order to execute the malicious payload.

The constructor approach is fear more elegant and operationally useful as it allows to execute the malicious code at the instant the shared library is mapped into a process's address space, in other words, before `main()` is even called.

### The `.init_array` Mechanism

The ELF format has a special section named `.init_array` that stores an array of function points. When the runtime dynamic linker loads a shared library (such as the one we were previously adding to the `LD_PRELOAD` environment variable), it iterates over this array and calls every function it finds as part of the initialization sequence, before handing the control to the program's entry point.

This ELF section is mainly used to setup some global state before a library is used, such as registering signal handlers or initializing mutexes. For an attacker, it represents a chance to evade defensive measures that may read the memory of a program from its entry point (EDRs, anti-cheat systems, integrity checkers, sandboxes, ...), to guarantee the payload execution within a shorter time, and to reduce the risk of corrupting the program's and system's behavior.

In C, marking a function with the `__attribute__((constructor))` would make the compiler place it in the `.init_array` section of the compiled ELF binary. In Rust, the `ctor` crate provides the `#[ctor]` attribute that does the same thing.

For example, the following Rust code practically allows to register a function in the `.init_array` section of its compiled ELF file that writes the caller binary's PID into the `/tmp/preloader_log.txt`

```rust
// Cargo.toml dependencies:
// ctor = "0.2"
// [lib]
// crate-type = ["cdylib"]

use ctor::ctor;
use std::fs::OpenOptions;
use std::io::Write;

/// This function is placed in .init_array by the ctor macro, making the dynamic linker calling it automatically when the library is loaded.
#[ctor]
fn init() {
    
    // Generate some sample payload. In our case, we will be writing the PID of the calling process into /tmp/preloader_log.txt
    let payload = format!(
        "[+] Library loaded. PID: {}. Running before main().\n",
        std::process::id()
    );

    if let Ok(mut file) = OpenOptions::new()
        .create(true)
        .append(true)
        .open("/tmp/preloader_log.txt")
    {
        let _ = file.write_all(payload.as_bytes());
    }
}
```

To compile and inject the function, we may follow similar steps to the PoC code from the previous section:
```bash
cargo build --release
LD_PRELOAD=./target/release/libpreload.so ls
```

![]({{ site.baseurl }}/assets/images/posts/ld_preload-abuse/constructor_preload_poc.png)

### Verifying The Execution Order with GDB


## 4. Why `.init_array` Evades Defenses That Symbol Shadowing Cannot


### How Defenders Inspect Loaded Libraries


### The GOT Integrity Angle


### The Timing Advantage as an Evasion Property


### What `.init_array` Does NOT Evade


## 5. Avoiding Forb Bombs: Guard Clauses


### Process Filtering


### The Singleton Lock


### Environment Flag Guard


## 6. When LD_PRELOAD Fails: Understanding the Limits

### The Immunity of Static Binaries


### The Security Boundary: `AT_SECURE`


## 7. Why Use LD_PRELOAD? Tradecraft and Use Cases


### Process Masking: Hiding in Plain Sight


### Persistent Droppers: The Logic Bomb


### Data Exfiltration at Source: Bypassing TLS


### Privilege Inheritance: Context Stealing in Containers


## 8. The Shellcode Thread Injector PoC


## 9. The Forensic Trail: Detection and Evidence


### Artifact 1: The Environment Variable Itself


### Artifact 2: Deleted-but-Mapped Libraries (The Smoking Gun)


### Artifact 3: The `/etc/ld.so.preload` Global Hook


### Artifact 4: The Linker Debug Output


### Wazuh Detection


## 10. Opening: Toward LD_AUDIT

