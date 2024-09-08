
This tutorial will show you how to blink an LED using different methods in Rust. For this tutorial we will use the STM32F3DISCOVERY board.

## Setup the Project

ℹ [[Embedded Rust Setup Explained]] contains detailed explanation about how to do the setup.

### Create Project

First we have to create our Rust project, let's call it `led_blink`
``` Bash
cargo new led_blink
```

### Dependecies

let's add the necessary dependecies to our project
``` Bash
cargo add cortex-m-rt panic-halt cortex-m-semihosting
```

### Memory info

The STM32F3DISCOVERY uses the STM32F303VCT6 processor. According to the data sheet the size of the flash memory is 256 Kbyte and the RAM is 40 Kbyte. 

![[stm32f303_memory_map.png]]

According to the memory map the flash memory starts at address 0x0800 0000 and the RAM 0x2000 0000.

Create the `memory.x` file in the project root folder and add the memory information
```
MEMORY
{
  FLASH : ORIGIN = 0x08000000, LENGTH = 256K
  RAM : ORIGIN = 0x20000000, LENGTH = 40K
}
```

### OpenOCD and GDB configs

To make the flashing and debugging process smoother we will create some configuration files for [[OpenOCD]] and [[GDB]]. Add the following files to the project root directory.

`openocd.cfg`
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

### Cargo Config

configuring cargo with our target and rust flags
``` Bash
mkdir .cargo && touch .cargo/config.toml
```
with the following info
``` TOML
[build]
target = "thumbv7em-none-eabihf"

[target.thumbv7em-none-eabihf]
rustflags = ["-C", "link-arg=-Tlink.x"]
runner = "gdb-multiarch -x openocd.gdb"
```

Defining the runner will start gdb when we use `cargo run`.

### Changes in main.rs

We have to do changes in the `main.rs` file so that it would work on our microcontroller.
Change the `main.rs` file to this
``` Rust
#![no_std]  
#![no_main]  
  
use cortex_m_rt::entry;  
use panic_halt as _;  
use cortex_m_semihosting::hprintln;

#[entry]
fn main() -> ! {
	hprintln!("Hello, world!");
	loop {}
}
```

## Peripheral Control

First we have to identify which LED you want to use and how it's connected to our microcontroller. In this example we will toggle **LD10** on the STM32F3DISCOVERY board.
According to the datasheet of the board, *"User LD10: Red LED is a user LED connected to the I/O PE13 of the STM32F303VCT6"*.

To configure this pin as digital output, we'll neet the General Purpose IO ([[GPIO]]) peripheral.
All the microcontroller's peripherals are represented as register blocks whithin the memory map and configuring or interacting with them amounts to memory reads or writes at specific addresses.

### Identify the register addresses

Let's check the memory map of the STM32F303VCT6 in the reference manual. 
![[Pasted image 20240908163044.png]]
I/O `PE13` is part of the GPIOE port, which is located at the addresses `0x4800 1000` - `0x4800 13FF`. GPIOE has multiple registers, for this exercise we will use the `GPIOD port mode register` and `GPIOD port output data register`.

### Activate the GPIO port peripheral

Until you enable the clock for a peripheral, the peripheral is dead and it neither functions nor it takes any configuration values set by you. Once you activate the clock for a peripheral, the peripheral is ready to take your configuration and control-related commands or arguments (configuration values)

ℹ For some microcontrollers, the peripheral may be on by default, and you don't need to do any activation. Check the datasheet and reference manual!

### Procedure to turn on the LED




---




To make a GPIO toggling in STM32, we need to work with two peripherals: **RCC** (reset and clock control) and **GPIOx** (general-purpose input/output). The RCC is necessary because the GPIO has disabled clock by default.

https://makbit.com/web/firmware/tutorial-getting-started-with-stm32f3-discovery-board/

