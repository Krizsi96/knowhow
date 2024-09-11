
# Introduction

Working on embedded systems over 10 years ago, the developer experience was often determined by your microcontroller vendor. Which meant: writing in [[C]] using the vendor's compiler and IDE, with their in circuit debugger hardware and working in Windows. ☹
[[Rust]], the memory safe alternative to [[C]] has started making inroads into the embedded space and technologies like [[language server protocol]] and [[LLVM]] have made it easier than ever to move away from vendor tooling for bare metal projects.

## Tooling

### Rustup

If you don't have it already, you have to install `rustup`, which is rust toolchain manager.
``` Bash
open https://rust-lang.org/tools/install
```
Follow the instructions for the installation.

You can update `rustup` with the following command:
``` Bash
rustup update
```

### IDE

For IDE, you have plenty of options. A decent editor for [[Rust]], ideally with [[LSP]] support, is sufficient. For example:

- JetBrains Rust Rover
- Neo Vim
- VS Code

## Hardware

The [[Rust#rustc]] compiler is a front end for [[LLVM]] which means it's very easy to cross compile for various core architectures and environments (especially for microcontrollers that have [[arm cortex M]] or [[RISK-V]] processors).

It's nice to work with project or evaluation boards (like the [[STM32F3DISCOVERY]]), because they usually include a secondary microcontroller that acts as a USB debugging and programming interface to the host machine

## Cross Compiling

The way to specify a different target in [[Rust#rustc]] is through a `target triple` ([[Rust#rustc/LLVM target triple]]).

1. Find out the specific microcontroller that your board has.
2. Check the chip developer website for the architecture
3. Open [The rustc book - Platform Support](https://doc.rust-lang.org/beta/rustc/platform-support.html) page
4. Search for your target architecture and check if it is supported by the compiler
5. Install the target compiler with the following command `rustup target add <target>`
6. You can verify the installed targets with `rustup show`

## Bare Metal Rust

Create a new project
``` Bash
cargo new project_name
```

This will create the following files:

- `.git` repository files
- `Cargo.toml` that contains some meta data about our project and dependencies
- `src/main.rs` is the main source file

You can see additional files:

- `Cargo.lock` the very specific dependencies we are using in our project
- `target/` were our compiled artifacts are

First and very important step is to tell the compiler that we don't want to use the standard library and its main function. You have to add the following attributes to the top of your `main.rs` file.

``` Rust
#![no_std]
#![no_main]

fn main() {}
```


## Dependency Management

The Rust programming language comes with its own [[Rust#dependency management]] tool.

Let's see an example. Our code has to be properly located in our microcontroller's memory map, luckily there is a crate that addresses this issue called `cortex-m-rt`. This include a linker script that'll work with any arm cortex M based microcontroller. It will take care of the following:

- populating the vector table so the device can boot correctly, and properly dispatch exceptions and interrupts.
- initializing `static` variables before the program entry point.
- enabling the [[FPU]] before the program entry point

The `cortex-m-rt` requires a `memory.x` file which minimally needs to contain both the start address and size of the flash and RAM regions of memory for our specific microcontroller. Check the datasheet of the microcontroller for these informations.

Adding the dependency:
``` Bash
cargo add cortex-m-rt
```

This will download and add the crate to the `Cargo.toml` file.

Create the `memory.x` file and add the memory information.
```
MEMORY
{
  FLASH : ORIGIN = 0x08000000, LENGTH = 64K
  RAM : ORIGIN = 0x20000000, LENGTH = 20K
}
```

We have to modify the  `main` function as follows
``` Rust
use cortex_m_rt::entry;

#[entry]
fn main() -> ! {
	loop {}
}
```

- `use cortex_m_rt::entry;` imports the entry functionality from the crate
- `#[entry]` indicates the entry point for the program
- `fn main() -> !` Our program will be the _only_ process running on the target hardware so we don't want it to end! We use a [divergent function](https://doc.rust-lang.org/rust-by-example/fn/diverging.html) (the `-> !` bit in the function signature) to ensure at compile time that'll be the case.
- `loop {}` puts the program into an infinite loop

To build the project we have to provide the target and linker options every time. It is better to put this information into a cargo config file. 

Create a `.cargo/config.toml` with the following data

``` toml
[build]
target = "thumbv7em-none-eabihf"

[target.thumbv7em-none-eabihf]
rustflags = ["-C", "link-arg=-Tlink.x"]
```

Now every time you run `cargo build` it will automatically this target and rust compiler flags.

## Don't Panic!

You have to define your [[Rust#Panic]] handler. It is just another function that is marked with the panic handler attribute that has a very specific function signature. It takes a reference to a panic info argument and it never returns. It is needed to be defined even if it is not linked in.

The easiest option is to use the `panic-halt` crate which does the absolute bare mininum.
⚠ Not something that you want to use in a product, but it's okay for bench debugging!

``` Bash
cargo add panic-halt
```

Add it to the `main.rs` file
``` Rust
use panic_halt as _;
```

## Build & Flash

### Analyzing the binary

Install the following tools:

- `llvm-tools` ▶ gives binutils like size, readobj, info about the binary and disassembly

``` Bash
rustup component add llvm-tools
```

- `cargo-binutils` ▶ adds binutils subcommand to cargo

``` Bash
cargo install cargo-binutils
```

To see the size information of the binary:
``` Bash
cargo size -- -Ax
```
Output:
``` Bash
app  :  
section                size         addr  
.vector_table         0x400    0x8000000  
.text                  0x8c    0x8000400  
.rodata                   0    0x800048c  
.data                     0   0x20000000  
.gnu.sgstubs              0    0x80004a0  
.bss                      0   0x20000000  
.uninit                   0   0x20000000  
.debug_abbrev        0x1198          0x0  
.debug_info         0x22642          0x0  
.debug_aranges       0x1330          0x0  
.debug_ranges       0x19c80          0x0  
.debug_str          0x3b594          0x0  
.comment               0x40          0x0  
.ARM.attributes        0x3a          0x0  
.debug_frame         0x4120          0x0  
.debug_line         0x1ee2c          0x0  
.debug_loc             0x29          0x0  
Total               0x9d199
```

With this you can confirm if the `memory.x` is set up correctly.

### Flashing

#### Probe-rs toolchain

###### Install `cargo-embed`

``` Bash
curl --proto '=https' --tlsv1.2 -LsSf https://github.com/probe-rs/probe-rs/releases/latest/download/probe-rs-tools-installer.sh | sh
```

This is going to both rebuild our code and talk with our secondary debug microcontroller to send over our binary and have it program our primary microcontroller. It's also allow us to connect a debugger so that we can set break points, modify memory and step through the code.

###### Flashing 

``` Bash
cargo embed --chip <device_chip>
```

❗This is failing currently, maybe it will be fixed in the future❗

#### ST-Link

There are other ways to program the board with the ST-Link, such as [openocd](https://github.com/rogerclarkmelbourne/Arduino_STM32/wiki/Programming-an-STM32F103XXX-with-a-generic-%22ST-Link-V2%22-programmer-from-Linux), but now we will use the [open-source stlink](https://github.com/texane/stlink) tools.

##### Install

Unfortunately, this software is not available via the apt package manager, therefore we have to [compile it from source](https://github.com/texane/stlink/blob/master/doc/compiling.md).

``` Bash
# in a directory of your choice:
sudo apt install libusb-1.0 libusb-1.0-0-dev
git clone https://github.com/texane/stlink
cd stlink
make all
```

This creates the executables `st-flash` and `st-info`, which reside now in `build/Release/`. There are different approaches to “install” these executables. We will now copy them into a system folder.

``` Bash
sudo cp build/Release/st-{flash,info} \
  build/Release/src/gdbserver/st-util /usr/local/bin
```

###### Flashing

``` Bash
cargo build --release
arm-none-eabi-objcopy -O binary \
  target/thumbv7m-none-eabi/release/<binary_name> <binary_name>.bin
> st-flash write <binary_name>.bin <flash_memory_address>
```

###### Optional: Udev Rules

To access the ST-Link without root permissions, copy the udev rules from the source code to `/etc/udev/rules.d/`

``` Bash
sudo cp etc/udev/rules.d/*.rules /etc/udev/rules.d
sudo udevadm control --reload-rules && udevadm trigger
```

##### Optional: Erase existing software/bootloaders:

This is done by executing the following command _right after pressing the reset button_ on the board:

``` Bash
st-flash erase
```

## Debugging

### Debugging with RTT

If your code is trying to maintain communication with one or more connected devices then hitting a breakpoint that halts the CPU can be massively disruptive to the to the timing of those interfaces and ironically can make debugging more difficult by introducing new issues. In these cases a "print line" style debugging could be more beneficial.

**Real Time Transfer** (RTT) is a way to exchange data between your host computer and your target microcontroller using its debug interface. For example if you want to print out messages to the host you're basically just writing to a in-memory ring buffer that gets read by the debugger (it is very fast).

You can use the `rtt-target` crate implements this protocol. However, to use the global `rprintln!` macro, a platform-specific `critical-section` implementation is needed for locking. 
ℹ for cortex M processor, this is covered by the `cortex-m` crate, so add it with
``` Bash
cargo add cortex-m --features critical-section-single-core
```

Hello World example:
``` Rust
#![no_std]  
#![no_main]  
  
use cortex_m_rt::entry;  
use panic_halt as _;  
use rtt_target::rtt_init_print;  
  
#[entry]  
fn main() -> ! {  
	rtt_init_print!();  
    rprintln!("Hello, world!");  
    loop {  
	    rprintln!("Echo...");  
        for _ in 0..100_000 {  
	        nop();  
        }  
    }  
}
```

❗It uses `cargo embed` which doesn't work in the current moment❗
### Debugging with GDB

We will do remote debugging and the client will be a [[GDB]] process, the server will be [[OpenOCD]].

#### Semihosting

*“Semihosting is a mechanism that enables code running on an ARM target to communicate and use the Input/Output facilities on a host computer that is running a debugger.”* - ARM

For cortex M processors you can use the `cortex-m-semihosting` crate.

For example:
``` Rust
use cortex_m_semihosting::{debug, hprintln};

#[entry]
fn main() -> ! {
	hprintln!("Hello, world!").unwrap();
	loop {}
}
```


#### Run OpenOCD

On a terminal run `openocd` from your project root directory to connect the probe on your board. You have to specify the interface and the target. for example, the STM32F3 config:
``` Bash
openocd -f interface/stlink.cfg -f target/stm32f3.cfg
```

Alternatively, you can create a configuration file for openocd, called `openocd.cfg`
``` cfg
# Sample OpenOCD configuration for the STM32F3DISCOVERY development board 

# Depending on the hardware revision you got you'll have to pick ONE of these 
# interfaces. At any time only one interface should be commented out. 

# Revision C (newer revision) 
source [find interface/stlink.cfg] 

# Revision A and B (older revisions) 
# source [find interface/stlink-v2.cfg] 

source [find target/stm32f3x.cfg]
```

#### Run GDB

On another terminal run GDB also from your project root directory.
``` Bash
gdb-multiarch -q <your-binary>
```

Next connect GDB to OpenOCD, which is waiting for a TCP connection on port 3333.
```
(gdb) target remote :3333
```

Flash the program onto the microcontroller using the `load` command.
```
(gdb) load
```

To use semihosting, we have to tell OpenOCD to enable semihosting. You can send commands to OpenOCD using the `monitor` command.
```
(gdb) monitor arm semihosting enable
```

See more info about debugging [here](https://docs.rust-embedded.org/book/start/hardware.html)

To exit GDB use the `quit` command.

You can use a config file for the setup of the connection like this:
`openocd.gdb`
``` gdb
target extended-remote :3333 

# print demangled symbols 
set print asm-demangle on 

# detect unhandled exceptions, hard faults and panics 
break DefaultHandler 
break HardFault 
break rust_begin_unwind 

monitor arm semihosting enable 

load 

# start the process but immediately halt the processor 
stepi
```

Now running `<gdb> -x openocd.gdb <your-binary>` will immediately connect GDB to OpenOCD, enable semihosting, load the program and start the process.



