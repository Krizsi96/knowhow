
This tutorial will show you how to blink an LED using different methods in Rust. For this tutorial we will use the STM32F3DISCOVERY board.

ℹ The source code available at [RustyBlinky](https://github.com/Krizsi96/rusty_blinky/tree/main)

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
cargo add cortex-m-rt panic-halt cortex-m-semihosting cortex_m
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
  
use core::ptr::{read_volatile, write_volatile};  
use cortex_m::asm::nop;  
use cortex_m_rt::entry;  
use cortex_m_semihosting::hprintln;  
use panic_halt as _;

#[entry]
fn main() -> ! {
	hprintln!("Hello, world!");
	loop {}
}
```

## Blinky With Peripheral Control

First we have to identify which LED you want to use and how it's connected to our microcontroller. In this example we will toggle **LD10** on the STM32F3DISCOVERY board.
According to the datasheet of the board, *"User LD10: Red LED is a user LED connected to the I/O PE13 of the STM32F303VCT6"*.

To configure this pin as digital output, we'll neet the General Purpose IO ([[GPIO]]) peripheral.
All the microcontroller's peripherals are represented as register blocks whithin the memory map and configuring or interacting with them amounts to memory reads or writes at specific addresses.

### Identify the register addresses

Let's check the memory map of the STM32F303VCT6 in the reference manual. 
![[peripheral_register_boundary_addresses.png]]
I/O `PE13` is part of the GPIOE port, which is located at the addresses `0x4800 1000` - `0x4800 13FF`. GPIOE has multiple registers, for this exercise we will use the `GPIOD port mode register` and `GPIOD port output data register`.

### Activate the GPIO port peripheral

Until you enable the clock for a peripheral, the peripheral is dead and it neither functions nor it takes any configuration values set by you. Once you activate the clock for a peripheral, the peripheral is ready to take your configuration and control-related commands or arguments (configuration values)

ℹ For some microcontrollers, the peripheral may be on by default, and you don't need to do any activation. Check the datasheet and reference manual!

#### Enable the peripheral clock

We have to enable the peripheral clock through the `peripheral clock control registers` of the microcontroller. 
![[rcc_address.png]]

![[gpio_bus_architecture.png]]

The GPIO ports are on the AHB bus, so we have to work with the `AHB peripheral clock enable restister` (`RCC_AHBENR`).
![[ahb_peripheral_clock_enable_register.png]]

Bit 21 is the enable bit for the GPIOE port
![[description_gpioeen.png]]

**Address of the clock control register** (`RCC_AHBENR`): 0x4002 1000 + 0x14 = **0x4002 1014**

``` Rust
// Enable the clock for GPIOE peripheral on the AHB bus
const RCC_AHBENR: *mut u32 = 0x4002_1014 as *mut u32;  
const GPIOEEN: u32 = 21;  
const ENABLE_GPIO_E_MASK: u32 = 1 << GPIOEEN;  
unsafe {  
    write_volatile(RCC_AHBENR, read_volatile(RCC_AHBENR) | ENABLE_GPIO_E_MASK);  
    hprintln!("updated RCC_AHBENR: {}", read_volatile(RCC_AHBENR));  
}
```
### Configure the GPIO pin mode as output

Since we are driving an LED, the operation mode of the GPIO pin has to be configured as output. We can do that with the `GPIO port mode register` (`GPIOx_MODER`)

![[gpio_port_mode_register.png]]

**Address of the GPIOE mode register** (`GPIOE_MODER`): 0x4800 1000 + 0x00 = **0x4800 1000**

To configure I/O `PE13` as output pin, we have to set `MODER13` value to `01`.

``` Rust
// Configure the mode of the IO pin as output  
const GPIOE_MODER: *mut u32 = 0x4800_1000 as *mut u32;
const GEN_OUTPUT_MODE: u32 = 1;  
let mut gpioe_mode_r: u32 = 0;  
unsafe {  
    gpioe_mode_r = read_volatile(GPIOE_MODER);  
}

// Clear MODER13 (bit 27 and 26)  
const MODE_R_13_CLEAR_MASK: u32 = !(0x3 << 26);  
gpioe_mode_r &= MODE_R_13_CLEAR_MASK;  
  
// Set MODER13 to general purpose output mode  
const MODE_R_13_MASK: u32 = GEN_OUTPUT_MODE << 26;  
gpioe_mode_r |= MODE_R_13_MASK;  
unsafe {  
    write_volatile(GPIOE_MODER, gpioe_mode_r);  
    hprintln!("updated GPIOE_MODER: {}", read_volatile(GPIOE_MODER));  
}
```

### Write to the GPIO pin

We can control the output of the `PE13` pin with the `GPIO port output data register` (`GPIOx_ODR`)

- 1 (HIGH) to make the GPIO pin state HIGH (3.3V)
- 0 (LOW) to make the GPIO pin state LOW (0V)

![[gpio_port_output_data_register.png]]

**Address of the GPIOE output data register** (`GPIOE_ODR`): 0x4800 1000 + 0x14 = **0x4800 1014**

To set the I/O `PE13` output to high, we have to set the `ODR13` value to `1`.

``` Rust
// Toggle the LED via setting the output data register    
const GPIOE_ODR: *mut u32 = 0x4800_1014 as *mut u32;
const PIN_POS: u32 = 13;  
let mut is_on: bool = true;  
loop {  
    unsafe {  
        write_volatile(GPIOE_ODR, (is_on as u32) << PIN_POS);  
        hprintln!("updated GPIOE_ODR: {}", read_volatile(GPIOE_ODR));  
    }  
    for _ in 0..500 {  
        nop();  
    }  
    is_on = !is_on;  
}
```

## Blinky With PAC

Peripheral Access Crate (PAC) is the lowest layer of our embedded rust stack. It provides an interface to access all of our microcontrollers peripheral registers. It knows their names, their addresses, their bit fields and even has some description information. 
Arm developed the CMSIS System View Description ([[CMSIS-SVD]]) format, that formalizes the description of the system contained in [[arm cortex M]] processor-based microcontrollers. This machine readable format contains all the register information and the [svd2rust](https://crates.io/crates/svd2rust) crate does the rest. It is a command line tool that transforms that manufacturer defined SVD file into a rust crate that exposes a type safe API to access the peripherals of the device. 

### The way to use an svd2rust generated PAC

The peripherals are held as register blocks inside the PAC's peripheral singleton, so our application first needs to take ownership of that instance. Then we can access the peripheral registers by name reading their bit fields directly or writing to them through a [[Rust#Closure]].

Example:
``` Rust
let mut peripherals = stm32f30x::Peripherals::take().unwrap();
peripherals.GPIOA.odr().write(|w| w.bits(1));
```

### Add the PAC

For the `STM32F303` microcontroller there is a crate called `stm32f3`. This crate provides an autogenerated API for access to `STM32F3` peripherals. The API is generated using [svd2rust](https://github.com/rust-embedded/svd2rust) with patched svd files containing extensive type-safe support.

To add it to the project:
``` Bash
cargo add stm32f3
```

To use, in `Cargo.toml`
``` TOML
[dependencies.stm32f3]
version = "0.15.1"
features = ["stm32f303"]
```

In the `main.rs`
``` Rust
use stm32f3::stm32f303;

#[entry]  
fn main() -> ! {  
    // Taking ownership of the PAC's peripheral singleton  
    let mut peripherals = stm32f303::Peripherals::take().unwrap();
    loop {}
}
```

###  Identify the register addresses

See [[Embedded Rust Blinking an LED#Identify the register addresses]] about the registers.

### Enable the peripheral clock

As mentioned in the [[Embedded Rust Blinking an LED#Activate the GPIO port peripheral]] we have to enable the peripheral clock for the GPIOE port on the AHB bus.

``` Rust
// Enable the clock for GPIOE peripheral on the AHB bus  
let rcc = &peripherals.RCC;  
rcc.ahbenr.modify(|_, w| w.iopeen().set_bit());
```

### Configure the GPIO pin mode as output

We can do that with the `GPIO port mode register` (`GPIOx_MODER`)
``` Rust
// Configure the mode of the IO pin as output  
let gpioe = &peripherals.GPIOE;  
gpioe.moder.modify(|_, w| w.moder13().output());
```

### Write to the GPIO pin

We can control the output of the `PE13` pin with the `GPIO port output data register` (`GPIOx_ODR`)
``` Rust
// Toggle the LED via setting the output data register  
let mut is_on: bool = true;  
loop {  
    gpioe.odr.modify(|_, w| w.odr13().bit(is_on));  
    for _ in 0..500_000 {  
        nop();  
    }  
    is_on = !is_on;  
}
```