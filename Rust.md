## rustc

The Rust C compiler is a front end for [[LLVM]].

### rustc/LLVM target triple

`<arch><sub>-<vendor>-<sys>-<env>`

- arch ▶ core architecture (x86_64, i386, arm, ...)
- sub ▶ sub architecture (ex. arm v5, v6, v7m, ...)
- vendor (pc, apple, ibm, ...)
- sys ▶ operating system (none, linux, win32, darwin, ...)
- env ▶ environment or ABI (eabi, gnu, elf, ...)

## Dependency Management

The packages in rust are called `crates`. You can search for the available open source crates on [crates.io](https://crates.io/).

## Panic

In Rust, for normal recoverable errors you would just return a result type in the error state from whatever function you're in. Panic is reserved for fatal errors, it's when you've hit some intractable issue like failing an assertion and the `panic handler` is where your code goes to die. 💀