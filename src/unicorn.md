# (Possibly) Emulate PinePhone with Unicorn Emulator

📝 _24 Feb 2023_

![Emulating Arm64 Machine Code in Unicorn](https://lupyuen.github.io/images/unicorn-title.jpg)

[_Emulating Arm64 Machine Code in Unicorn_](https://github.com/lupyuen/pinephone-emulator/blob/bc5643dea66c70f57a150955a12884f695acf1a4/src/main.rs#L1-L55)

[__Unicorn__](https://www.unicorn-engine.org/) is a lightweight __CPU Emulator Framework__ based on [__QEMU__](http://www.qemu.org/).

(Programmable with C, Rust, Python and [__many other languages__](https://github.com/unicorn-engine/unicorn/tree/master/bindings))

We're porting a new operating system [__Apache NuttX RTOS__](https://lupyuen.github.io/articles/what) to [__Pine64 PinePhone__](https://wiki.pine64.org/index.php/PinePhone). And I wondered...

_To make PinePhone testing easier... Can we emulate Arm64 PinePhone with Unicorn Emulator?_

Let's find out! In this article we'll call __Unicorn Emulator__ to...

-   __Emulate Arm64__ Machine Code

-   __Attach Hooks__ to intercept Memory Access and Code Execution

-   __Boot Apache NuttX RTOS__ in the emulator

-   __Simulate the UART Controller__ for PinePhone

-   __Track an Exception__ in Arm64 Memory Management

We'll do all this in __basic Rust__, instead of classic C.

(That's because I'm too old to write meticulous C... But I'm OK to get nagged by Rust Compiler if I miss something!)

We begin by emulating some machine code...

# Emulate Arm64 Machine Code

Suppose we wish to emulate this __Arm64 Machine Code__...

```rust
// Start Address: 0x10000

// str  w11, [x13], #0
AB 05 00 B8

// ldrb w15, [x13], #0
AF 05 40 38

// End Address: 0x10008
```

With these __Arm64 Register Values__...

| Register | Value |
|:--------:|:------|
| `X11` | `0x12345678`
| `X13` | `0x10008`
| `X15` | `0x33`

Which means...

1.  __Store `X11`__ (value __`0x12345678`__)

    Into the address referenced by __`X13`__

    (Address __`0x10008`__)

1.  __Load `X15`__ as a __Single Byte__

    From the address referenced by __`X13`__

    (Address __`0x10008`__)

1.  Which sets __`X15`__ to __`0x78`__

    (Because __`0x10008`__ contains byte __`0x78`__)

    [(__`X`__ Registers are __64-bit__, __`W`__ Registers are __32-bit__)](https://developer.arm.com/documentation/102374/0100/Registers-in-AArch64---general-purpose-registers)

This is how we __call Unicorn Emulator__ to emulate the Arm64 Machine Code: [main.rs](https://github.com/lupyuen/pinephone-emulator/blob/bc5643dea66c70f57a150955a12884f695acf1a4/src/main.rs#L1-L55)

```rust
use unicorn_engine::{Unicorn, RegisterARM64};
use unicorn_engine::unicorn_const::{Arch, Mode, Permission};

fn main() {
  // Arm64 Memory Address where emulation starts
  const ADDRESS: u64 = 0x10000;

  // Arm64 Machine Code for the above address
  let arm64_code: Vec<u8> = vec![
    0xab, 0x05, 0x00, 0xb8,  // str w11,  [x13], #0
    0xaf, 0x05, 0x40, 0x38,  // ldrb w15, [x13], #0
  ];
```

We begin by defining the __Arm64 Machine Code__.

Then we __initialise the emulator__...

```rust
  // Init Emulator in Arm64 mode
  let mut unicorn = Unicorn::new(
    Arch::ARM64,
    Mode::LITTLE_ENDIAN
  ).expect("failed to init Unicorn");

  // Magical horse mutates to bird
  let emu = &mut unicorn;
```

Unicorn needs some __Emulated Memory__ to run our code.

We map __2MB of Executable Memory__...

```rust
  // Map 2MB of Executable Memory at 0x10000
  // for Arm64 Machine Code
  emu.mem_map(
    ADDRESS,          // Address is 0x10000
    2 * 1024 * 1024,  // Size is 2MB
    Permission::ALL   // Read, Write and Execute Access
  ).expect("failed to map code page");
```

And we __populate the Executable Memory__ with our Arm64 Machine Code...

```rust
  // Write Arm64 Machine Code to emulated Executable Memory
  emu.mem_write(
    ADDRESS,     // Address is 0x10000
    &arm64_code  // Arm64 Machine Code
  ).expect("failed to write instructions");
```

We __set the Arm64 Registers__: X11, X13 and X15...

```rust
  // Register Values
  const X11: u64 = 0x12345678;    // X11 value
  const X13: u64 = ADDRESS + 0x8; // X13 value
  const X15: u64 = 0x33;          // X15 value
  
  // Set the Arm64 Registers
  emu.reg_write(RegisterARM64::X11, X11)
    .expect("failed to set X11");
  emu.reg_write(RegisterARM64::X13, X13)
    .expect("failed to set X13");
  emu.reg_write(RegisterARM64::X15, X15)
    .expect("failed to set X15");
```

We __start the emulator__...

```rust
  // Emulate Arm64 Machine Code
  let err = emu.emu_start(
    ADDRESS,  // Begin Address is 0x10000
    ADDRESS + arm64_code.len() as u64,  // End Address is 0x10008
    0,  // No Timeout
    0   // Unlimited number of instructions
  );

  // Print the Emulator Error
  println!("err={:?}", err);
```

Finally we __read Register X15__ and verify the result...

```rust
  // Read the X15 Register
  assert_eq!(
    emu.reg_read(RegisterARM64::X15),  // Register X15
    Ok(0x78)  // Expected Result
  );
}
```

And we're done!

Remember to add [__unicorn-engine__](https://crates.io/crates/unicorn-engine) to the dependencies: [Cargo.toml](https://github.com/lupyuen/pinephone-emulator/blob/bc5643dea66c70f57a150955a12884f695acf1a4/Cargo.toml#L8-L9)

```text
[dependencies]
unicorn-engine = "2.0.0"
```

When we run our [__Rust Program__](https://github.com/lupyuen/pinephone-emulator/blob/bc5643dea66c70f57a150955a12884f695acf1a4/src/main.rs)...

```text
→ cargo run --verbose

Fresh cc v1.0.79
Fresh cmake v0.1.49
Fresh pkg-config v0.3.26
Fresh bitflags v1.3.2
Fresh libc v0.2.139
Fresh unicorn-engine v2.0.1
Fresh pinephone-emulator v0.1.0
Finished dev [unoptimized + debuginfo] target(s) in 0.08s
Running `target/debug/pinephone-emulator`

err=Ok(())
```

Unicorn is hunky dory!

Let's talk about Memory-Mapped Input / Output...

![Memory Access Hook for Arm64 Emulation](https://lupyuen.github.io/images/unicorn-code2.png)

[_Memory Access Hook for Arm64 Emulation_](https://github.com/lupyuen/pinephone-emulator/blob/3655ac2875664376f42ad3a3ced5cbf067790782/src/main.rs#L59-L95)

# Memory Access Hook

To emulate our gadget (like PinePhone), we need to handle [__Memory-Mapped Input / Output__](https://en.wikipedia.org/wiki/Memory-mapped_I/O_and_port-mapped_I/O).

(Like for printing to the Serial or UART Port)

We do this in Unicorn Emulator with a __Memory Access Hook__ that will be called to __intercept every Memory Access__.

Here's a sample __Hook Function__ that will be called to intercept every Arm64 Read / Write Access: [main.rs](https://github.com/lupyuen/pinephone-emulator/blob/3655ac2875664376f42ad3a3ced5cbf067790782/src/main.rs#L83-L95)

```rust
// Hook Function for Memory Access.
// Called once for every Arm64 Memory Access.
fn hook_memory(
  _: &mut Unicorn<()>,  // Emulator
  mem_type: MemType,    // Read or Write Access
  address:  u64,    // Accessed Address
  size:     usize,  // Number of bytes accessed
  value:    i64     // Write Value
) -> bool {         // Always return true

  // TODO: Emulate Memory-Mapped Input/Output (UART Controller)
  println!("hook_memory: mem_type={:?}, address={:#x}, size={:?}, value={:#x}", mem_type, address, size, value);

  // Always return true, value is unused by caller
  // https://github.com/unicorn-engine/unicorn/blob/dev/docs/FAQ.md#i-cant-recover-from-unmapped-readwrite-even-i-return-true-in-the-hook-why
  true
}
```

Our Hook Function prints __every Read / Write Access__ to the Emulated Arm64 Memory.

This is how we __attach the Memory Hook Function__ to Unicorn Emulator: [main.rs](https://github.com/lupyuen/pinephone-emulator/blob/3655ac2875664376f42ad3a3ced5cbf067790782/src/main.rs#L59-L74)

```rust
// Add Hook for Arm64 Memory Access
let _ = emu.add_mem_hook(
  HookType::MEM_ALL,  // Intercept Read and Write Access
  0,           // Begin Address
  u64::MAX,    // End Address
  hook_memory  // Hook Function
).expect("failed to add memory hook");
```

When we run this, we see the Read and Write Memory Accesses made by our [__Emulated Arm64 Code__](https://lupyuen.github.io/articles/unicorn#emulate-arm64-machine-code)...

```text
hook_memory: 
  mem_type=WRITE, 
  address=0x10008, 
  size=4, 
  value=0x12345678

hook_memory: 
  mem_type=READ, 
  address=0x10008, 
  size=1, 
  value=0x0
```

(Value is not relevant for Memory Reads)

Later we'll implement UART Output with a Memory Access Hook. But first we intercept some code...

![Code Execution Hook for Arm64 Emulation](https://lupyuen.github.io/images/unicorn-code3.png)

[_Code Execution Hook for Arm64 Emulation_](https://github.com/lupyuen/pinephone-emulator/blob/3655ac2875664376f42ad3a3ced5cbf067790782/src/main.rs#L108-L117)

# Code Execution Hook

_Can we intercept every Arm64 Instruction that will be emulated?_

Yep we can attach a __Code Execution Hook__ to Unicorn Emulator.

Here's a sample Hook Function that will be called for __every Arm64 Instruction emulated__: [main.rs](https://github.com/lupyuen/pinephone-emulator/blob/3655ac2875664376f42ad3a3ced5cbf067790782/src/main.rs#L108-L117)

```rust
// Hook Function for Code Emulation.
// Called once for each Arm64 Instruction.
fn hook_code(
  _: &mut Unicorn<()>,  // Emulator
  address: u64,  // Instruction Address
  size: u32      // Instruction Size
) {
  // TODO: Handle special Arm64 Instructions
  println!("hook_code: address={:#x}, size={:?}", address, size);
}
```

And this is how we call Unicorn Emulator to __attach the Code Hook Function__: [main.rs](https://github.com/lupyuen/pinephone-emulator/blob/3655ac2875664376f42ad3a3ced5cbf067790782/src/main.rs#L52-L57)

```rust
// Add Hook for emulating each Arm64 Instruction
let _ = emu.add_code_hook(
  ADDRESS,  // Begin Address
  ADDRESS + arm64_code.len() as u64,  // End Address
  hook_code  // Hook Function for Code Emulation
).expect("failed to add code hook");
```

When we run this with our [__Arm64 Machine Code__](https://lupyuen.github.io/articles/unicorn#emulate-arm64-machine-code), we see the Address of __every Arm64 Instruction emulated__ (and its size)...

```text
hook_code:
  address=0x10000,
  size=4

hook_code:
  address=0x10004,
  size=4
```

We might use this to emulate [__Special Arm64 Instructions__](https://developer.arm.com/documentation/102374/0101/Registers-in-AArch64---system-registers).

If we don't need to intercept every single instruction, try the Block Execution Hook...

# Block Execution Hook

_Is there something that works like a Code Execution Hook..._

_But doesn't stop at every single Arm64 Instruction?_

Yep Unicorn Emulator supports __Block Execution Hooks__.

This Hook Function will be called once when executing a __Block of Arm64 Instructions__: [main.rs](https://github.com/lupyuen/pinephone-emulator/blob/3655ac2875664376f42ad3a3ced5cbf067790782/src/main.rs#L97-L106)

```rust
// Hook Function for Block Emulation.
// Called once for each Basic Block of Arm64 Instructions.
fn hook_block(
  _: &mut Unicorn<()>,  // Emulator
  address: u64,  // Block Address
  size: u32      // Block Size
) {
  // TODO: Trace the flow of emulated code
  println!("hook_block: address={:#x}, size={:?}", address, size);
}
```

This is how we __attach the Block Execution Hook__: [main.rs](https://github.com/lupyuen/pinephone-emulator/blob/3655ac2875664376f42ad3a3ced5cbf067790782/src/main.rs#L48-L50)

```rust
// Add Hook for emulating each Basic Block of Arm64 Instructions
let _ = emu.add_block_hook(hook_block)
  .expect("failed to add block hook");
```

Block Execution Hooks are __less granular__ (called less often) than Code Execution Hooks...

```text
hook_block: address=0x10000, size=8
hook_code:  address=0x10000, size=4
hook_code:  address=0x10004, size=4
```

Which means that Unicorn Emulator calls our Hook Function only once for the [__entire Block of two Arm64 Instructions__](https://lupyuen.github.io/articles/unicorn#emulate-arm64-machine-code).

[(What's a Block of Arm64 Instructions?)](https://github.com/lupyuen/pinephone-emulator#what-is-a-block-of-arm64-instructions)

_How is this useful?_

The Block Execution Hook is super helpful for monitoring the __Execution Flow__ of our emulated code. We can...

-   Read the __ELF Debug Symbols__ (from the NuttX Image)

-   Match with the __Block Execution Addresses__

-   And print the __Name of the Function__ that's being emulated

This is how we do it...

-   [__"Map Address to Function with ELF File"__](https://lupyuen.github.io/articles/unicorn#appendix-map-address-to-function-with-elf-file)

# Unmapped Memory

_What happens when Unicorn Emulator tries to access memory that isn't mapped?_

Unicorn Emulator will call our Memory Access Hook with __mem_type__ set to __READ_UNMAPPED__...

```text
hook_memory:
  address=0x01c28014,
  size=2,
  mem_type=READ_UNMAPPED,
  value=0x0
```

[(Source)](https://github.com/lupyuen/pinephone-emulator/blob/b842358ba457b67ffa9f4c1a362b0386cfd97c4a/README.md#block-execution-hooks-for-arm64-emulation)

The log says that our Arm64 Machine Code attempted to read address `01C2` `8014`, which is unmapped.

This is how we map the memory: [rust.rs](https://github.com/lupyuen/pinephone-emulator/blob/cd030954c2ace4cf0207872f275abc3ffb7343c6/src/main.rs#L26-L32)

```rust
// Map 16 MB at 0x0100 0000 for Memory-Mapped I/O by Allwinner A64 Peripherals
// https://github.com/apache/nuttx/blob/master/arch/arm64/src/a64/hardware/a64_memorymap.h#L33-L51
emu.mem_map(
  0x0100_0000,       // Address
  16 * 1024 * 1024,  // Size
  Permission::READ | Permission::WRITE  // Read and Write Access
).expect("failed to map memory mapped I/O");
```

We'll see this later when we handle Memory-Mapped Input / Output.

_Can we map Memory Regions during emulation?_

Yep we may use a Memory Access Hook to __map memory regions on the fly__.

[(Like this)](https://github.com/unicorn-engine/unicorn/blob/dev/docs/FAQ.md#i-cant-recover-from-unmapped-readwrite-even-i-return-true-in-the-hook-why)

![Running Apache NuttX RTOS in Unicorn](https://lupyuen.github.io/images/unicorn-code4.png)

[_Running Apache NuttX RTOS in Unicorn_](https://github.com/lupyuen/pinephone-emulator/blob/aa24d1c61256f38f92cf627d52c3e9a0c189bfc6/src/main.rs#L6-L78)

# Apache NuttX RTOS in Unicorn

We're ready to run Apache NuttX RTOS in Unicorn Emulator!

We've compiled [__Apache NuttX RTOS for PinePhone__](https://github.com/lupyuen/pinephone-emulator/blob/main/nuttx) into an Arm64 Binary Image: [__nuttx.bin__](https://github.com/lupyuen/pinephone-emulator/blob/main/nuttx/nuttx.bin)

This is how we __load the NuttX Binary Image__ into Unicorn: [main.rs](https://github.com/lupyuen/pinephone-emulator/blob/aa24d1c61256f38f92cf627d52c3e9a0c189bfc6/src/main.rs#L6-L40)

```rust
// Arm64 Memory Address where emulation starts
const ADDRESS: u64 = 0x4008_0000;

// Arm64 Machine Code for the above address
let arm64_code = include_bytes!("../nuttx/nuttx.bin");
```

[_(Rustle... Whoosh!)_](https://doc.rust-lang.org/std/macro.include_bytes.html)

We __initialise the emulator__ the same way...

```rust
// Init Emulator in Arm64 mode
let mut unicorn = Unicorn::new(
  Arch::ARM64,
  Mode::LITTLE_ENDIAN
).expect("failed to init Unicorn");

// Magical horse mutates to bird
let emu = &mut unicorn;
```

Based on the [__NuttX Memory Map__](https://github.com/apache/nuttx/blob/master/arch/arm64/include/a64/chip.h#L44-L52) for PinePhone, we map two Memory Regions for NuttX...

-   __Executable Memory__ (128 MB) at __`4000` `0000`__

    (For Arm64 Machine Code, Data and BSS)

-   __Read / Write Memory__ (512 MB) at __`0000` `0000`__

    (For Memory-Mapped I/O by Allwinner A64 Peripherals)

```rust
// Map 128 MB Executable Memory at 0x4000 0000 for Arm64 Machine Code
// https://github.com/apache/nuttx/blob/master/arch/arm64/include/a64/chip.h#L44-L52
emu.mem_map(
  0x4000_0000,        // Address
  128 * 1024 * 1024,  // Size
  Permission::ALL     // Read, Write and Execute Access
).expect("failed to map code page");

// Map 512 MB Read/Write Memory at 0x0000 0000 for
// Memory-Mapped I/O by Allwinner A64 Peripherals
// https://github.com/apache/nuttx/blob/master/arch/arm64/include/a64/chip.h#L44-L52
emu.mem_map(
  0x0000_0000,        // Address
  512 * 1024 * 1024,  // Size
  Permission::READ | Permission::WRITE  // Read and Write Access
).expect("failed to map memory mapped I/O");
```

We __load the NuttX Machine Code__ into Emulated Memory...

```rust
// Write Arm64 Machine Code to emulated Executable Memory
emu.mem_write(
  ADDRESS,    // Address is 4008 0000
  arm64_code  // NuttX Binary Image
).expect("failed to write instructions");
```

And we __run NuttX RTOS__!

```rust
// Omitted: Attach Code, Block and Memory Hooks
...
// Emulate Arm64 Machine Code
let err = emu.emu_start(
  ADDRESS,  // Begin Address
  ADDRESS + arm64_code.len() as u64,  // End Address
  0,  // No Timeout
  0   // Unlimited number of instructions
);
```

Unicorn happily __boots Nuttx RTOS__ (yay!)...

```text
→ cargo run 
hook_block:  address=0x40080000, size=8
hook_block:  address=0x40080040, size=4
hook_block:  address=0x40080044, size=12
hook_block:  address=0x40080118, size=16
...
```

[(See the Arm64 Disassembly)](https://github.com/lupyuen/pinephone-emulator/blob/a1fb82d829856d86d6845c477709c2be24373aca/nuttx/nuttx.S)

But our legendary creature gets stuck in mud. Let's find out why...

![Emulating the UART Controller](https://lupyuen.github.io/images/unicorn-code5.png)

[_Emulating the UART Controller_](https://github.com/lupyuen/pinephone-emulator/blob/4d78876ad6f40126bf68cb2da4a43f56d9ef6e76/src/main.rs#L27-L76)

# Wait for UART Controller

Unicorn gets __stuck in a curious loop__ while booting NuttX RTOS...

```text
hook_code:   address=0x400801f8, size=4
hook_code:   address=0x400801fc, size=4
hook_block:  address=0x400801f4, size=12
hook_code:   address=0x400801f4, size=4
hook_memory: address=0x01c28014, size=2, mem_type=READ, value=0x0

hook_code:   address=0x400801f8, size=4
hook_code:   address=0x400801fc, size=4
hook_block:  address=0x400801f4, size=12
hook_code:   address=0x400801f4, size=4
hook_memory: address=0x01c28014, size=2, mem_type=READ, value=0x0
...
```

[(Source)](https://github.com/lupyuen/pinephone-emulator/blob/045fa5da84d9e07ead5a820a075c1445661328b6/README.md#unicorn-emulator-waits-forever-for-uart-controller-ready)

See the pattern? Unicorn Emulator loops forever at address __`4008` `01F4`__...

While reading the data from address __`01C2` `8014`__.

_What's at 4008 01F4?_

Here's the __NuttX Arm64 Disassembly__ at address __`4008` `01F4`__: [nuttx.S](https://github.com/lupyuen/pinephone-emulator/blob/a1fb82d829856d86d6845c477709c2be24373aca/nuttx/nuttx.S#L3398-L3411)

```text
SECTION_FUNC(text, up_lowputc)
  ldr   x15, =UART0_BASE_ADDRESS
  400801f0:	580000cf 	ldr	x15, 40080208 <up_lowputc+0x18>
nuttx/arch/arm64/src/chip/a64_lowputc.S:89
  early_uart_ready x15, w2
  400801f4:	794029e2 	ldrh	w2, [x15, #20]
  400801f8:	721b005f 	tst	w2, #0x20
  400801fc:	54ffffc0 	b.eq	400801f4 <up_lowputc+0x4>  // b.none
nuttx/arch/arm64/src/chip/a64_lowputc.S:90
  early_uart_transmit x15, w0
  40080200:	390001e0 	strb	w0, [x15]
nuttx/arch/arm64/src/chip/a64_lowputc.S:91
  ret
  40080204:	d65f03c0 	ret
```

Which comes from this __NuttX Source Code__: [a64_lowputc.S](https://github.com/apache/nuttx/blob/master/arch/arm64/src/a64/a64_lowputc.S#L61-L71)

```text
/* Wait for A64 UART to be ready to transmit
 * xb: Register that contains the UART Base Address
 * wt: Scratch register number
 */
.macro early_uart_ready xb, wt
1:
  ldrh  \wt, [\xb, #0x14] /* UART_LSR (Line Status Register) */
  tst   \wt, #0x20        /* Check THRE (TX Holding Register Empty) */
  b.eq  1b                /* Wait for the UART to be ready (THRE=1) */
.endm
```

_NuttX is printing something to the UART Port?_

Yep! NuttX prints __Debug Messages__ to the (Serial) UART Port when it boots...

And it's waiting for the __UART Controller to be ready__, before printing!

[(As explained here)](https://lupyuen.github.io/articles/uboot#wait-for-uart-ready)

_What's at 01C2 8014?_

__`01C2` `8014`__ is the __UART Line Status Register__ (UART_LSR) for the Allwinner A64 UART Controller inside PinePhone.

__Bit 5__ needs to be set to 1 to indicate that the __UART Transmit FIFO__ is ready. Or NuttX will wait forever!

[(As explained here)](https://lupyuen.github.io/articles/serial#wait-to-transmit)

_How to fix the UART Ready Bit at 01C2 8014?_

This is how we __emulate the UART Ready Bit__ with our Input / Output Memory: [main.rs](https://github.com/lupyuen/pinephone-emulator/blob/4d78876ad6f40126bf68cb2da4a43f56d9ef6e76/src/main.rs#L42-L49)

```rust
// Allwinner A64 UART Line Status Register (UART_LSR) at Offset 0x14.
// To indicate that the UART Transmit FIFO is ready:
// Set Bit 5 to 1.
// https://lupyuen.github.io/articles/serial#wait-to-transmit
emu.mem_write(
  0x01c2_8014,  // UART Register Address
  &[0b10_0000]  // UART Register Value
).expect("failed to set UART_LSR");
```

And Unicorn Emulator stops looping!

Unicorn continues booting NuttX, which fills the [__BSS Section__](https://en.wikipedia.org/wiki/.bss) with 0...

```text
hook_block:  address=0x40089328, size=8
hook_memory: address=0x400b6a52, size=1, mem_type=WRITE, value=0x0
hook_block:  address=0x40089328, size=8
hook_memory: address=0x400b6a53, size=1, mem_type=WRITE, value=0x0
...
```

[(Source)](https://github.com/lupyuen/pinephone-emulator/blob/045fa5da84d9e07ead5a820a075c1445661328b6/README.md#unicorn-emulator-waits-forever-for-uart-controller-ready)

But we don't see any NuttX Boot Messages. Let's print the UART Output...

![Emulating UART Output in Unicorn](https://lupyuen.github.io/images/unicorn-code6.png)

[_Emulating UART Output in Unicorn_](https://github.com/lupyuen/pinephone-emulator/blob/aa6dd986857231a935617e8346978d7750aa51e7/src/main.rs#L89-L111)

# Emulate UART Output

_We expect to see Boot Messages from NuttX..._

_How do we print the UART Output?_

NuttX RTOS will write the UART Output to Allwinner A64's __UART Transmit Holding Register__ (THR) at __`01C2` `8000`__...

-   [__"Transmit UART"__](https://lupyuen.github.io/articles/serial#transmit-uart)

To emulate the UART Output, we use Unicorn's __Memory Access Hook__...

-   [__"Memory Access Hook"__](https://lupyuen.github.io/articles/unicorn#memory-access-hook)

In our Memory Access Hook, we intercept all writes to __`01C2` `8000`__ and dump the bytes written: [main.rs](https://github.com/lupyuen/pinephone-emulator/blob/aa6dd986857231a935617e8346978d7750aa51e7/src/main.rs#L89-L111)

```rust
// Hook Function for Memory Access.
// Called once for every Arm64 Memory Access.
fn hook_memory(
  _: &mut Unicorn<()>,  // Emulator
  mem_type: MemType,    // Read or Write Access
  address: u64,  // Accessed Address
  size: usize,   // Number of bytes accessed
  value: i64     // Write Value
) -> bool {      // Always return true

  // If writing to UART Transmit Holding Register (THR):
  // Print the output
  // https://lupyuen.github.io/articles/serial#transmit-uart
  if address == 0x01c2_8000 {
    println!("uart output: {:?}", value as u8 as char);
  }

  // Always return true, value is unused by caller
  // https://github.com/unicorn-engine/unicorn/blob/dev/docs/FAQ.md#i-cant-recover-from-unmapped-readwrite-even-i-return-true-in-the-hook-why
  true
}
```

When we run this, we see a long chain of UART Output...

```text
→ cargo run | grep uart
uart output: '-'
uart output: ' '
uart output: 'R'
uart output: 'e'
uart output: 'a'
uart output: 'd'
uart output: 'y'
...
```

[(Source)](https://gist.github.com/lupyuen/587dbeb9329d9755e4d007dd8e1246cd)

Which reads as...

```text
- Ready to Boot CPU
- Boot from EL2
- Boot from EL1
- Boot to C runtime for OS Initialize
```

[(Similar to this)](https://lupyuen.github.io/articles/uboot#pinephone-boots-nuttx)

Yep NuttX RTOS is booting on Unicorn Emulator! But we have a problem...

![Debugging an Arm64 Exception](https://lupyuen.github.io/images/unicorn-debug.jpg)

[_Debugging an Arm64 Exception_](https://github.com/lupyuen/pinephone-emulator#dump-the-arm64-exception)

# Emulator Halts with MMU Fault

_NuttX RTOS boots OK on Unicorn Emulator?_

Not quite. Unicorn Emulator halts with an __Arm64 Exception__ at address __`4008` `0EF8`__...

```text
hook_block:  address=0x40080eec, size=16
hook_code:   address=0x40080eec, size=4
hook_code:   address=0x40080ef0, size=4
hook_code:   address=0x40080ef4, size=4
hook_code:   address=0x40080ef8, size=4
err=Err(EXCEPTION)
```

[(See the Complete Log)](https://gist.github.com/lupyuen/778f15875edf632ccb5a093a656084cb)

Here's the __NuttX Arm64 Disassembly__ at address __`4008` `0EF8`__: [nuttx.S](https://github.com/lupyuen/pinephone-emulator/blob/a1fb82d829856d86d6845c477709c2be24373aca/nuttx/nuttx.S)

```text
nuttx/arch/arm64/src/common/arm64_mmu.c:544
  write_sysreg((value | SCTLR_M_BIT | SCTLR_C_BIT), sctlr_el1);
    40080ef0:	d28000a1 	mov	x1, #0x5                   	// #5
    40080ef4:	aa010000 	orr	x0, x0, x1
    40080ef8:	d5181000 	msr	sctlr_el1, x0
```

Which comes from this __NuttX Source Code__: [arm64_mmu.c](https://github.com/apache/nuttx/blob/master/arch/arm64/src/common/arm64_mmu.c#L541-L544)

```c
// Enable the MMU and data cache:
// Read from System Control Register EL1
value = read_sysreg(sctlr_el1);

// Write to System Control Register EL1
write_sysreg(  // Write to System Register...
  value | SCTLR_M_BIT | SCTLR_C_BIT,  // Enable Address Translation and Caching
  sctlr_el1    // System Control Register EL1
);
```

The code above sets these flags in Arm64 [__System Control Register EL1__](https://developer.arm.com/documentation/ddi0595/2021-06/AArch64-Registers/SCTLR-EL1--System-Control-Register--EL1-) (SCTLR_EL1)...

- __SCTLR_M_BIT__ (Bit 0): Enable __Address Translation__ for EL0 and EL1 Stage 1

- __SCTLR_C_BIT__ (Bit 2): Enable __Caching__ for EL0 and EL1 Stage 1

  [(What's EL1?)](https://lupyuen.github.io/articles/interrupt#exception-levels)

Thus the __Address Translation__ (or Caching) has failed in our Emulated Arm64 Memory Management Unit, inside [__enable_mmu_el1__](https://github.com/apache/nuttx/blob/master/arch/arm64/src/common/arm64_mmu.c#L541-L544)...

![Call Graph for Apache NuttX RTOS](https://lupyuen.github.io/images/unicorn-callgraph.jpg)

[(Source)](https://github.com/lupyuen/pinephone-emulator#call-graph-for-apache-nuttx-rtos)

We won't chase the Unicorn into the Rabbit Hole, but the details are covered here...

-   [__"Unicorn Emulator Halts in NuttX MMU"__](https://github.com/lupyuen/pinephone-emulator#unicorn-emulator-halts-in-nuttx-mmu)

-   [__"Dump the Arm64 Exception"__](https://github.com/lupyuen/pinephone-emulator#dump-the-arm64-exception)

-   [__"Arm64 MMU Exception"__](https://github.com/lupyuen/pinephone-emulator#arm64-mmu-exception)

-   [__"Debug the Unicorn Emulator"__](https://github.com/lupyuen/pinephone-emulator#debug-the-unicorn-emulator)

-   [__"Map Address to Function with ELF File"__](https://github.com/lupyuen/pinephone-emulator#map-address-to-function-with-elf-file)

-   [__"Call Graph for Apache NuttX RTOS"__](https://github.com/lupyuen/pinephone-emulator#call-graph-for-apache-nuttx-rtos)

![Apache NuttX RTOS on PinePhone](https://lupyuen.github.io/images/lvgl2-title.jpg)

[_Apache NuttX RTOS on PinePhone_](https://lupyuen.github.io/articles/what)

# Emulation Concerns

_So are we happy with Unicorn Emulator?_

Yep! Unicorn Emulator is sufficient for __Automated Daily Build and Test__ for NuttX on PinePhone. (Via GitHub Actions)

Which will be similar to this BL602 setup, except that we'll boot the Daily Build on Unicorn Emulator (instead of Real Hardware)...

-   [__"Auto Flash and Test NuttX on RISC-V BL602"__](https://lupyuen.github.io/articles/auto)

Also Unicorn Emulator has produced a __Call Graph__ for NuttX on PinePhone, which is extremely valuable for troubleshooting...

-   [__"(Clickable) Call Graph for Apache NuttX RTOS"__](https://lupyuen.github.io/articles/unicorn2)

_But our PinePhone Emulator doesn't handle Console Input..._

Yeah we'll do that later. We have a long wishlist of features to build: Interrupts, Memory Protection, Multiple CPUs, Cortex A53, GIC v2, ...

-   [__"Wishlist for PinePhone Emulator"__](https://github.com/lupyuen/pinephone-emulator#todo)

_NuttX runs graphical apps on PinePhone right?_

Yep someday we'll render the __NuttX Graphics Framebuffers__ in Unicorn.

(Maybe with a Rust GUI Library. Or [__Unicorn.js__](https://alexaltea.github.io/unicorn.js/) and WebAssembly)

-   [__"NuttX RTOS for PinePhone: Framebuffer"__](https://lupyuen.github.io/articles/fb)

_What about emulating other operating systems: Linux / macOS / Windows / Android?_

Check out the __Qiling Binary Emulation Framework__...

-   [__qilingframework/qiling__](https://github.com/qilingframework/qiling)

_How about other hardware platforms: STM32 Blue Pill and ESP32?_

Check out __QEMU Emulator__...

-   [__"Unit Testing with QEMU Blue Pill Emulator"__](https://lupyuen.github.io/articles/stm32-blue-pill-unit-testing-with-qemu-blue-pill-emulator)

-   [__"NuttX on an emulated ESP32 using QEMU"__](https://medium.com/@lucassvaz/nuttx-on-an-emulated-esp32-using-qemu-8d8d93d24c63)

# What's Next

Check out the next article...

-   [__"(Clickable) Call Graph for Apache NuttX RTOS"__](https://lupyuen.github.io/articles/unicorn2)

This has been a fun educational exercise. Now we have a way to run __Automated Daily Tests__ for Apache NuttX RTOS on PinePhone... Kudos to the __Maintainers of Unicorn Emulator__!

Many Thanks to my [__GitHub Sponsors__](https://github.com/sponsors/lupyuen) for supporting my work! This article wouldn't have been possible without your support.

-   [__Sponsor me a coffee__](https://github.com/sponsors/lupyuen)

-   [__Discuss this article on Reddit__](https://www.reddit.com/r/rust/comments/119unud/possibly_emulate_pinephone_with_unicorn_emulator/)

-   [__Discuss this article on Hacker News__](https://news.ycombinator.com/item?id=34908923)

-   [__My Current Project: "Apache NuttX RTOS for Ox64 BL808"__](https://github.com/lupyuen/nuttx-ox64)

-   [__My Other Project: "NuttX for Star64 JH7110"__](https://github.com/lupyuen/nuttx-star64)

-   [__Older Project: "NuttX for PinePhone"__](https://github.com/lupyuen/pinephone-nuttx)

-   [__Check out my articles__](https://lupyuen.github.io)

-   [__RSS Feed__](https://lupyuen.github.io/rss.xml)

_Got a question, comment or suggestion? Create an Issue or submit a Pull Request here..._

[__lupyuen.github.io/src/unicorn.md__](https://github.com/lupyuen/lupyuen.github.io/blob/master/src/unicorn.md)

# Appendix: Map Address to Function with ELF File

Check out the new article...

-   [__"(Clickable) Call Graph for Apache NuttX RTOS"__](https://lupyuen.github.io/articles/unicorn2)

Our __Block Execution Hook__ now prints the __Function Name__ and the __Filename__...

```text
hook_block:  
  address=0x40080eb0, 
  size=12, 
  setup_page_tables, 
  arch/arm64/src/common/arm64_mmu.c:516:25

hook_block:  
  address=0x40080eec, 
  size=16, 
  enable_mmu_el1, 
  arch/arm64/src/common/arm64_mmu.c:543:11

err=Err(EXCEPTION)
```

[(Source)](https://gist.github.com/lupyuen/f2e883b2b8054d75fbac7de661f0ee5a)

Our Hook Function looks up the Address in the [__DWARF Debug Symbols__](https://crates.io/crates/gimli) of the [__NuttX ELF File__](https://github.com/lupyuen/pinephone-emulator/blob/main/nuttx/nuttx), like so: [main.rs](https://github.com/lupyuen/pinephone-emulator/blob/465a68a10e3fdc23c5897c3302eb0950cc4db614/src/main.rs#L127-L156)

```rust
/// Hook Function for Block Emulation.
/// Called once for each Basic Block of Arm64 Instructions.
fn hook_block(
  _: &mut Unicorn<()>,  // Emulator
  address: u64,  // Block Address
  size: u32      // Block Size
) {
  print!("hook_block:  address={:#010x}, size={:02}", address, size);

  // Print the Function Name
  let function = map_address_to_function(address);
  if let Some(ref name) = function {
    print!(", {}", name);
  }

  // Print the Source Filename
  let loc = map_address_to_location(address);
  if let Some((ref file, line, col)) = loc {
    let file = file.clone().unwrap_or("".to_string());
    let line = line.unwrap_or(0);
    let col = col.unwrap_or(0);
    print!(", {}:{}:{}", file, line, col);
  }
  println!();
}
```

We map the Block Address to Function Name and Source File in __map_address_to_function__ and __map_address_to_location__:  [main.rs](https://github.com/lupyuen/pinephone-emulator/blob/465a68a10e3fdc23c5897c3302eb0950cc4db614/src/main.rs#L172-L219)

```rust
/// Map the Arm64 Code Address to the Function Name by looking up the ELF Context
fn map_address_to_function(
  address: u64         // Code Address
) -> Option<String> {  // Function Name
  // Lookup the Arm64 Code Address in the ELF Context
  let context = ELF_CONTEXT.context.borrow();
  let mut frames = context.find_frames(address)
    .expect("failed to find frames");

  // Return the Function Name
  if let Some(frame) = frames.next().unwrap() {
    if let Some(func) = frame.function {
      if let Ok(name) = func.raw_name() {
        let s = String::from(name);
        return Some(s);
      }
    }    
  }
  None
}

/// Map the Arm64 Code Address to the Source Filename, Line and Column
fn map_address_to_location(
  address: u64     // Code Address
) -> Option<(      // Returns...
  Option<String>,  // Filename
  Option<u32>,     // Line
  Option<u32>      // Column
)> {
  // Lookup the Arm64 Code Address in the ELF Context
  let context = ELF_CONTEXT.context.borrow();
  let loc = context.find_location(address)
    .expect("failed to find location");

  // Return the Filename, Line and Column
  if let Some(loc) = loc {
    if let Some(file) = loc.file {
      let s = String::from(file)
        .replace("/private/tmp/nuttx/nuttx/", "")
        .replace("arch/arm64/src/chip", "arch/arm64/src/a64");  // TODO: Handle other chips
      Some((Some(s), loc.line, loc.column))
    } else {
      Some((None, loc.line, loc.column))
    }
  } else {
    None
  }
}
```

To run this, we need the [__addr2line__](https://crates.io/crates/addr2line), [__gimli__](https://crates.io/crates/gimli) and [__once_cell__](https://crates.io/crates/once_cell) crates: [Cargo.toml](https://github.com/lupyuen/pinephone-emulator/blob/465a68a10e3fdc23c5897c3302eb0950cc4db614/Cargo.toml#L8-L12)

```text
[dependencies]
addr2line = "0.19.0"
gimli = "0.27.2"
once_cell = "1.17.1"
unicorn-engine = "2.0.0"
```

At startup, we load the [__NuttX ELF File__](https://github.com/lupyuen/pinephone-emulator/blob/main/nuttx/nuttx) into __ELF_CONTEXT__ as a [__Lazy Static__](https://docs.rs/once_cell/latest/once_cell/): [main.rs](https://github.com/lupyuen/pinephone-emulator/blob/465a68a10e3fdc23c5897c3302eb0950cc4db614/src/main.rs#L288-L322)

```rust
use std::rc::Rc;
use std::cell::RefCell;
use once_cell::sync::Lazy;

/// ELF File for mapping Addresses to Function Names and Filenames
const ELF_FILENAME: &str = "nuttx/nuttx";

/// ELF Context for mapping Addresses to Function Names and Filenames
static ELF_CONTEXT: Lazy<ElfContext> = Lazy::new(|| {
  // Open the ELF File
  let path = std::path::PathBuf::from(ELF_FILENAME);
  let file_data = std::fs::read(path)
    .expect("failed to read ELF");
  let slice = file_data.as_slice();

  // Parse the ELF File
  let obj = addr2line::object::read::File::parse(slice)
    .expect("failed to parse ELF");
  let context = addr2line::Context::new(&obj)
    .expect("failed to parse debug info");

  // Set the ELF Context
  ElfContext {
    context: RefCell::new(context),
  }
});

/// Wrapper for ELF Context. Needed for `Lazy`
struct ElfContext {
  context: RefCell<
    addr2line::Context<
      gimli::EndianReader<
        gimli::RunTimeEndian, 
        Rc<[u8]>  // Doesn't implement Send / Sync
      >
    >
  >
}

/// Send and Sync for ELF Context. Needed for `Lazy`
unsafe impl Send for ElfContext {}
unsafe impl Sync for ElfContext {}
```
