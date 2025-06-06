# Creating the Unicorn Emulator for Avaota-A1 SBC (Apache NuttX RTOS)

📝 _13 Apr 2025_

![Avaota-A1 SBC: Shot on Sony NEX-7 with IKEA Ring Light, Yeelight Ring Light on Corelle Plate](https://lupyuen.github.io/images/unicorn4-title.jpg)

[__Avaota-A1 Arm64 SBC__](https://lupyuen.github.io/articles/avaota.html) is officially supported by [__Apache NuttX RTOS__](https://nuttx.apache.org/docs/latest/platforms/arm64/a527/boards/avaota-a1/index.html) (Allwinner A527 SoC). Let's take [__Unicorn Emulator__](https://www.unicorn-engine.org/) and create a __Software Emulator__ for Avaota SBC...

- We call __Unicorn Library__ to create our Barebones Emulator 

- Emulate the __16550 UART__ hardware by intercepting I/O Memory

- Recompile NuttX with __4 Tiny Tweaks__ and boot on Unicorn 

- NuttX makes a __Context Switch__ and fails

- Because Unicorn doesn't handle __Arm64 SysCalls?__

- No worries we'll __Emulate Arm64 SysCalls__ ourselves!

- By jumping into the __Arm64 Vector Table__

- NuttX on Unicorn boots to __NSH Shell!__ _(Almost)_

- How exactly does NuttX boot on Avaota SBC? We have a __Detailed Boot Flow__

_Why are we doing this?_

1.  So we can create __NuttX Drivers and Apps__ on Avaota SBC Emulator (without the actual hardware)

1.  Avaota Emulator is helpful for [__NuttX Continuous Integration__](https://lupyuen.org/articles/ci4.html), making sure that all Code Changes will run correctly on Avaota SBC

1.  The __Trade Tariffs__ are Terribly Troubling. Some of us NuttX Folks might need to hunker down and emulate Avaota SBC, for now.

1.  Or maybe we should provide [__Remote Access__](https://lupyuen.github.io/articles/avaota.html#microsd-multiplexer--smart-power-plug) to a Real Avaota SBC? 🤔

# NuttX for Avaota-A1

Weeks ago we ported NuttX to Avaota-A1 SBC...

- [__"Porting Apache NuttX RTOS to Avaota-A1 SBC (Allwinner A527 SoC)"__](https://lupyuen.github.io/articles/avaota.html)

To boot __NuttX on Unicorn__: We recompile NuttX with [__Four Tiny Tweaks__](https://github.com/lupyuen2/wip-nuttx/pull/106)...

1.  [__Set TCR_TG1_4K, Physical / Virtual Address to 32 Bits__](https://github.com/lupyuen2/wip-nuttx/pull/106/commits/640084e1fb1692887266716ecda52dc7ea4bf8e0)

    From the [__Previous Article__](https://lupyuen.github.io/articles/unicorn3.html): Unicorn Emulator requires __TCR_TG1_4K__. And the __Physical / Virtual Address Size__ should be 32 Bits.

1.  [__Disable PSCI__](https://github.com/lupyuen2/wip-nuttx/pull/106/commits/b3782b1ff989667df22b10d5c1023826e2211d88): We don't emulate the __PSCI Driver__ in Unicorn, so we disable this

1.  [__Enable Scheduler Logging__](https://github.com/lupyuen2/wip-nuttx/pull/106/commits/878e78eb40f334e6e128595dbb27ae08aed1e969): So we can see NuttX booting

1.  [__Enable SysCall Logging__](https://github.com/lupyuen2/wip-nuttx/pull/106/commits/c9f38c13eb5ac6f6bbcd4d3c1de218828f9f087d): To verify that NuttX SysCalls are OK

Here are the steps to compile __NuttX for Unicorn__...

```bash
## Compile Modified NuttX for Avaota-A1 SBC
git clone https://github.com/lupyuen2/wip-nuttx nuttx --branch unicorn-avaota
git clone https://github.com/lupyuen2/wip-nuttx-apps apps --branch unicorn-avaota
cd nuttx

## Build the NuttX Kernel
make -j distclean
tools/configure.sh avaota-a1:nsh
make -j
cp .config nuttx.config

## Build the NuttX Apps
make -j export
pushd ../apps
./tools/mkimport.sh -z -x ../nuttx/nuttx-export-*.tar.gz
make -j import
popd

## Combine NuttX Kernel and NuttX Apps into NuttX Image
genromfs -f nuttx-initrd -d ../apps/bin -V "NuttXBootVol"
head -c 65536 /dev/zero >/tmp/nuttx.pad
cat nuttx.bin /tmp/nuttx.pad nuttx-initrd >nuttx-Image

## Dump the NuttX Kernel disassembly to nuttx.S
aarch64-none-elf-objdump \
  --syms --source --reloc --demangle --line-numbers --wide --debugging \
  nuttx >nuttx.S 2>&1

## Dump the NSH Shell disassembly to nuttx-init.S
aarch64-none-elf-objdump \
  --syms --source --reloc --demangle --line-numbers --wide --debugging \
  ../apps/bin/init >nuttx-init.S 2>&1

## Dump the Hello disassembly to nuttx-hello.S
aarch64-none-elf-objdump \
  --syms --source --reloc --demangle --line-numbers --wide --debugging \
  ../apps/bin/hello >nuttx-hello.S 2>&1

## Copy NuttX Image to Unicorn Emulator
cp nuttx nuttx.S nuttx.config nuttx.hash nuttx-init.S nuttx-hello.S \
  $HOME/nuttx-arm64-emulator/nuttx
cp nuttx-Image \
  $HOME/nuttx-arm64-emulator/nuttx/Image
```

To boot NuttX in __Unicorn Emulator__...

```bash
## Boot NuttX in the Unicorn Emulator
git clone https://github.com/lupyuen/nuttx-arm64-emulator --branch avaota
cd nuttx-arm64-emulator
cargo run

## To see the Emulated UART Output
cargo run | grep "uart output"
```

We dive into the emulator code...

# Unicorn Emulator for Avaota-A1

_What's inside the Avaota-A1 Emulator?_

Inside our Avaota SBC Emulator: We begin by creating the __Unicorn Interface__: [main.rs](https://github.com/lupyuen/nuttx-arm64-emulator/blob/avaota/src/main.rs#L13-L36)

```rust
/// Memory Space for NuttX Kernel
const KERNEL_SIZE: usize = 0x1000_0000;
static mut KERNEL_CODE: [u8; KERNEL_SIZE] = [0; KERNEL_SIZE];

/// Emulate some Arm64 Machine Code
fn main() {

  // Init Emulator in Arm64 mode
  let mut unicorn = Unicorn::new(
    Arch::ARM64,
    Mode::LITTLE_ENDIAN
  ).unwrap();

  // Enable MMU Translation
  let emu = &mut unicorn;
  emu.ctl_tlb_type(unicorn_engine::TlbType::CPU)
    .unwrap();
```

Based on the [__Allwinner A527 Memory Map__](https://lupyuen.github.io/articles/avaota.html#fix-the-memory-map), we reserve __1 GB of I/O Memory__ for UART and other I/O Peripherals: [main.rs](https://github.com/lupyuen/nuttx-arm64-emulator/blob/avaota/src/main.rs#L36-L43)

```rust
  // Map 1 GB Read/Write Memory at 0x0000 0000 for Memory-Mapped I/O
  emu.mem_map(
    0x0000_0000,  // Address
    0x4000_0000,  // Size
    Permission::READ | Permission::WRITE  // Read/Write/Execute Access
  ).unwrap();
```

Next we load the __NuttX Image__ _(NuttX Kernel + NuttX Apps)_ into Unicorn Memory: [main.rs](https://github.com/lupyuen/nuttx-arm64-emulator/blob/avaota/src/main.rs#L43-L64)

```rust
  // Copy NuttX Image into memory
  let kernel = include_bytes!("../nuttx/Image");
  unsafe {
    assert!(KERNEL_CODE.len() >= kernel.len());
    KERNEL_CODE[0..kernel.len()].copy_from_slice(kernel);    
  }

  // Arm64 Memory Address where emulation starts.
  // Memory Space for NuttX Kernel also begins here.
  const ADDRESS: u64 = 0x4080_0000;

  // Map the NuttX Kernel to 0x4080_0000
  unsafe {
    emu.mem_map_ptr(
      ADDRESS, 
      KERNEL_CODE.len(), 
      Permission::READ | Permission::EXEC,
      KERNEL_CODE.as_mut_ptr() as _
    ).unwrap();
  }
```

Unicorn lets us hook into its internals, for emulating nifty things. We add the __Unicorn Hooks__ for...

- [__Block Hook:__](https://lupyuen.github.io/articles/unicorn.html#block-execution-hook) For each block of Arm64 Code, we render the [__Call Graph__](https://lupyuen.github.io/articles/unicorn2.html#intercept-code-execution-in-unicorn)

- [__Memory Hook:__](https://lupyuen.github.io/articles/unicorn.html#memory-access-hook) To emulate the [__UART Hardware__](https://lupyuen.github.io/articles/avaota.html#print-to-uart-in-arm64-assembly), we intercept the Memory Reads and Writes

- [__Interrupt Hook:__](https://lupyuen.github.io/articles/unicorn4.html#hook-the-unicorn-interrupt) We emulate [__Arm64 SysCalls__](https://lupyuen.github.io/articles/unicorn4.html#emulate-the-arm64-syscall) as Unicorn Interrupts

Like so: [main.rs](https://github.com/lupyuen/nuttx-arm64-emulator/blob/avaota/src/main.rs#L64-L88)

```rust
  // Add Hook for emulating each Basic Block of Arm64 Instructions
  emu.add_block_hook(1, 0, hook_block)
    .unwrap();

  // Add Hook for Arm64 Memory Access
  emu.add_mem_hook(
    HookType::MEM_ALL,  // Intercept Read and Write Accesses
    0,           // Begin Address
    u64::MAX,    // End Address
    hook_memory  // Hook Function
  ).unwrap();

  // Add Interrupt Hook
  emu.add_intr_hook(hook_interrupt)
    .unwrap();

  // Upcoming: Indicate that the UART Transmit FIFO is ready
```

[(__hook_block__ is explained here)](https://lupyuen.github.io/articles/unicorn2.html#intercept-code-execution-in-unicorn)

Finally we start the __Unicorn Emulator__: [main.rs](https://github.com/lupyuen/nuttx-arm64-emulator/blob/avaota/src/main.rs#L88-L106)

```rust
  // Emulate Arm64 Machine Code
  let err = emu.emu_start(
    ADDRESS,  // Begin Address
    ADDRESS + KERNEL_SIZE as u64,  // End Address
    0,  // No Timeout
    0   // Unlimited number of instructions
  );

  // Print the Emulator Error
  println!("err={:?}", err);
  println!("PC=0x{:x}", emu.reg_read(RegisterARM64::PC).unwrap());
}
```

That's it for our Barebones Emulator of Avaota SBC! We fill in the hooks...

![Avaota-A1 SBC with SDWire MicroSD Multiplexer and Smart Power Plug](https://lupyuen.github.io/images/avaota-title.jpg)

# Emulate 16550 UART

_What about Avaota I/O? How to emulate in Unicorn?_

Let's emulate the Bare Minimum for I/O: Printing output to the [__16550 UART__](https://lupyuen.github.io/articles/avaota.html#print-to-uart-in-arm64-assembly)...

1.  We intercept all writes to the [__UART Transmit Register__](https://lupyuen.github.io/articles/avaota.html#print-to-uart-in-arm64-assembly), and print them 

    _(So we can see the Boot Log from NuttX)_

1.  We signal to NuttX that [__UART Transmit FIFO__](https://lupyuen.github.io/articles/avaota.html#uart-driver-for-16550) is always ready to transmit

    _(Otherwise NuttX will wait forever for UART)_

This will tell NuttX that our __UART Transmit FIFO__ is forever ready: [main.rs](https://github.com/lupyuen/nuttx-arm64-emulator/blob/avaota/src/main.rs#L64-L73)

```rust
/// UART0 Base Address for Allwinner A527
const UART0_BASE_ADDRESS: u64 = 0x02500000;

fn main() {
  ...
  // Allwinner A527 UART Line Status Register (UART_LSR) is at Offset 0x14.
  // To indicate that the UART Transmit FIFO is ready:
  // We set Bit 5 to 1.
  emu.mem_write(
    UART0_BASE_ADDRESS + 0x14,  // UART Register Address
    &[0b10_0000]                // UART Register Value
  ).unwrap();
```

Our __Unicorn Memory Hook__ will intercept all writes to the __UART Transmit Register__, and print them: [main.rs](https://github.com/lupyuen/nuttx-arm64-emulator/blob/avaota/src/main.rs#L152-L178)

```rust
/// Hook Function for Memory Access.
/// Called once for every Arm64 Memory Access.
fn hook_memory(
  _: &mut Unicorn<()>,  // Emulator
  mem_type: MemType,    // Read or Write Access
  address: u64,  // Accessed Address
  size: usize,   // Number of bytes accessed
  value: i64     // Write Value
) -> bool {

  // If writing to UART Transmit Holding Register (THR):
  // We print the UART Output
  if address == UART0_BASE_ADDRESS {
    println!("uart output: {:?}", value as u8 as char);
  }

  // Always return true, value is unused by caller
  // https://github.com/unicorn-engine/unicorn/blob/dev/docs/FAQ.md#i-cant-recover-from-unmapped-readwrite-even-i-return-true-in-the-hook-why
  true
}
```

When we run this: Our Barebones Emulator will print the UART Output and show the __NuttX Boot Log__...

```bash
## To see the Emulated UART Output
$ cargo run | grep "uart output"
...
## NuttX begins booting...
- Ready to Boot Primary CPU
- Boot from EL1
- Boot to C runtime for OS Initialize
nx_start: Entry
```

We're ready to boot NuttX on Unicorn!

# NuttX Halts at SysCall

_Our Barebones Emulator: What happens when we run it?_

We boot NuttX on our [__Barebones Emulator__](https://lupyuen.github.io/articles/unicorn4.html#unicorn-emulator-for-avaota-a1). NuttX halts with an __Arm64 Exception__ at this curious address: _0x4080_6D60_...

```bash
$ cargo run
...
hook_block:  address=0x40806d4c, size=04, sched_unlock, sched/sched/sched_unlock.c:90:18
call_graph:  nxsched_merge_pending --> sched_unlock
call_graph:  click nxsched_merge_pending href "https://github.com/apache/nuttx/blob/master/sched/sched/sched_mergepending.c#L84" "sched/sched/sched_mergepending.c " _blank
hook_block:  address=0x40806d50, size=08, sched_unlock, sched/sched/sched_unlock.c:92:19
hook_block:  address=0x40806d58, size=08, sys_call0, arch/arm64/include/syscall.h:152:21
call_graph:  sched_unlock --> sys_call0
call_graph:  click sched_unlock href "https://github.com/apache/nuttx/blob/master/sched/sched/sched_unlock.c#L89" "sched/sched/sched_unlock.c " _blank
>> exception index = 2
>>> invalid memory accessed, STOP = 21!!!
err=Err(EXCEPTION)
PC=0x40806d60
```

_What's at 0x4080_6D60?_

We look up the [__Arm64 Disassembly__](https://github.com/lupyuen/nuttx-arm64-emulator/blob/avaota/nuttx/nuttx.S) for for NuttX Kernel. We see that _0x4080_6D60_ points to __Arm64 SysCall `SVC` `0`__...

```c
arch/arm64/include/syscall.h:152
// Execute an Arm64 SysCall SVC with SYS_ call number and no parameters
static inline uintptr_t sys_call0(unsigned int nbr) {
  register uint64_t reg0 __asm__("x0") = (uint64_t)(nbr);
    40806d58:	d2800040 	mov	x0, #0x2  // Parameter in Register X0 is 2
    40806d5c:	d4000001 	svc	#0x0      // Execute SysCall 0
    40806d60: ... //Next instruction to be executed on return from SysCall
```

[(__sys_call0__ is here)](https://github.com/apache/nuttx/blob/master/arch/arm64/include/syscall.h#L147-L164)

Somehow NuttX Kernel is making an Arm64 SysCall, and failing.

_Isn't Unicorn supposed to emulate Arm64 SysCalls?_

To find out: We step through Unicorn with [__CodeLLDB Debugger__](https://marketplace.visualstudio.com/items?itemName=vadimcn.vscode-lldb) (pic below). Unicorn triggers the Arm64 Exception here: [qemu/accel/tcg/cpu-exec.c](https://github.com/unicorn-engine/unicorn/blob/master/qemu/accel/tcg/cpu-exec.c#L335-L428)

```c
// When Unicorn handles a CPU Exception...
static inline bool cpu_handle_exception(CPUState *cpu, int *ret) {
  ...
  // Unicorn: call registered interrupt callbacks
  catched = false;
  HOOK_FOREACH_VAR_DECLARE;
  HOOK_FOREACH(uc, hook, UC_HOOK_INTR) {
    if (hook->to_delete) { continue; }
    JIT_CALLBACK_GUARD(((uc_cb_hookintr_t)hook->callback)(uc, cpu->exception_index, hook->user_data));
    catched = true;
  }

  // Unicorn: If interrupt is uncaught, stop the execution
  if (!catched) {
    if (uc->invalid_error == UC_ERR_OK) {
      // OOPS! EXCEPTION HAPPENS HERE
      uc->invalid_error = UC_ERR_EXCEPTION;
    }
    cpu->halted = 1;
    *ret = EXCP_HLT;
    return true;
  }
```

[(Set these __Debug Breakpoints__)](https://github.com/lupyuen/nuttx-arm64-emulator/blob/avaota/.vscode/bookmarks.json)

[(Compare with __Original QEMU__)](https://github.com/qemu/qemu/blob/master/accel/tcg/cpu-exec.c#L704-L769)

Aha! Unicorn is expecting us to __Hook This Interrupt__ and emulate the Arm64 SysCall, inside our Interrupt Callback.

Before hooking the interrupt, we track down the origin of the SysCall...

![Stepping through Unicorn with CodeLLDB Debugger](https://lupyuen.github.io/images/unicorn3-avaota.jpg)

# SysCall for Context Switch

_Why is NuttX Kernel making an Arm64 SysCall? Aren't SysCalls used by NuttX Apps?_

Let's find out! NuttX passes a __Parameter to SysCall__ in Register X0. The Parameter Value is __`2`__: [nuttx.S](https://github.com/lupyuen/nuttx-arm64-emulator/blob/avaota/nuttx/nuttx.S)

```c
sched/sched/sched_unlock.c:92
// To unlock the Task Scheduler...
void sched_unlock(void) {
  ...
  up_switch_context(this_task(), rtcb);
    40807230:	d538d080 	mrs	x0, tpidr_el1
    40807234:	37000060 	tbnz	w0, #0, 40807240 <sched_unlock+0x80>

arch/arm64/include/syscall.h:152
// We execute an Arm64 SysCall SVC with SYS_ call number and no parameters
static inline uintptr_t sys_call0(unsigned int nbr) {
  register uint64_t reg0 __asm__("x0") = (uint64_t)(nbr);
    40807238:	d2800040 	mov	x0, #0x2  // Parameter in Register X0 is 2
    4080723c:	d4000001 	svc	#0x0      // Execute SysCall 0
```

[(__sched_unlock__ is here)](https://github.com/apache/nuttx/blob/master/sched/sched/sched_unlock.c#L80-L95)

What's the NuttX SysCall with Parameter 2? It's for __Switching The Context__ between NuttX Tasks: [syscall.h](https://github.com/apache/nuttx/blob/master/arch/arm64/include/syscall.h#L78-L83)

```c
// NuttX SysCall 2 will Switch Context:
// void arm64_switchcontext(void **saveregs, void *restoreregs)
#define SYS_switch_context (2)
```

Which is implemented here: [arm64_syscall.c](https://github.com/apache/nuttx/blob/master/arch/arm64/src/common/arm64_syscall.c#L201-L216)

```c
// NuttX executes the Arm64 SysCall...
uint64_t *arm64_syscall(uint64_t *regs) {
  ...
  // If SysCall is for Switch Context...
  case SYS_switch_context:

    // Switch the Running Task
    nxsched_suspend_scheduler(*running_task);
    nxsched_resume_scheduler(tcb);
    *running_task = tcb;

    // Switch the Address Enviroment
    restore_critical_section(tcb, cpu);
    addrenv_switch(tcb);
    break;
```

Ah we see the light...

1.  NuttX Kernel makes an __Arm64 SysCall__ during Startup

1.  To trigger the __Very First Context Switch__

1.  Which will start the __NuttX Tasks__ and boot successfully

1.  This means we must emulate the __Arm64 SysCall__!

FYI __NuttX SysCalls__ are defined here...

- [__SysCall Defines:__ syscall_lookup.h](https://github.com/apache/nuttx/blob/master/include/sys/syscall_lookup.h)

- [__SysCall CSV:__ syscall.csv](https://github.com/apache/nuttx/blob/master/syscall/syscall.csv)

# Hook The Unicorn Interrupt

_To Boot NuttX: We need to Emulate the Arm64 SysCall. How?_

We saw earlier that Unicorn expects us to...

1.  [__Hook the Unicorn Interrupt__](https://lupyuen.github.io/articles/unicorn4.html#nuttx-halts-at-syscall)

1.  Then __Emulate the Arm64 SysCall__

This is how we __Hook the Interrupt__: [main.rs](https://github.com/lupyuen/nuttx-arm64-emulator/blob/avaota/src/main.rs#L85-L152)

```rust
/// Main Function of Avaota Emulator
fn main() {
  ...
  // Add the Interrupt Hook
  emu.add_intr_hook(hook_interrupt)
    .unwrap();

  // Emulate Arm64 Machine Code
  let err = emu.emu_start(
    ADDRESS,  // Begin Address
    ADDRESS + KERNEL_SIZE as u64,  // End Address
    0,  // No Timeout
    0   // Unlimited number of instructions
  );
  ...
}

/// Hook Function to handle the Unicorn Interrupt
fn hook_interrupt(
  emu: &mut Unicorn<()>,  // Emulator
  intno: u32,             // Interrupt Number
) {
  let pc = emu.reg_read(RegisterARM64::PC).unwrap();
  let x0 = emu.reg_read(RegisterARM64::X0).unwrap();
  println!("hook_interrupt: intno={intno}");
  println!("PC=0x{pc:08x}");
  println!("X0=0x{x0:08x}");
  println!("ESR_EL0={:?}", emu.reg_read(RegisterARM64::ESR_EL0));
  println!("ESR_EL1={:?}", emu.reg_read(RegisterARM64::ESR_EL1));

  // Upcoming: Handle the SysCall
  ...
}
```

Our Interrupt Hook is super barebones, barely sufficient for making it past the Arm64 SysCall...

```bash
$ cargo run
...
## NuttX Scheduler calls Arm64 SysCall...
call_graph:  sched_unlock --> sys_call0
call_graph:  click sched_unlock href "https://github.com/apache/nuttx/blob/master/sched/sched/sched_unlock.c#L89" "sched/sched/sched_unlock.c " _blank

## Unicorn calls our Interrupt Hook!
>> exception index = 2
hook_interrupt: intno=2
PC=0x40806d60

## Our Interrupt Hook returns to Unicorn,
## without handling the Arm64 SysCall...
call_graph:  sys_call0 --> sched_unlock
call_graph:  up_irq_restore --> sched_unlock
call_graph:  click up_irq_restore href "https://github.com/apache/nuttx/blob/master/arch/arm64/include/irq.h#L382" "arch/arm64/include/irq.h " _blank

## NuttX tries to continue booting, but fails...
call_graph:  nx_start --> up_idle
call_graph:  click nx_start href "https://github.com/apache/nuttx/blob/master/sched/init/nx_start.c#L781" "sched/init/nx_start.c " _blank

## Unicorn says that NuttX has halted at WFI
>> exception index = 65537
>>> stop with r = 10001, HLT=10001
>>> got HLT!!!
err=Ok(())
PC=0x408169d0
```

[(__0x4081_69D0__ points to WFI)](https://github.com/lupyuen/nuttx-arm64-emulator#handle-nuttx-syscall-in-unicorn)

But we're not done yet! Unicorn halts because we haven't emulated the Arm64 SysCall. Let's do it...

# Arm64 Vector Table

_How to emulate the Arm64 SysCall?_

Here's our plan...

1.  System Register [__VBAR_EL1__](https://developer.arm.com/documentation/ddi0601/2025-03/AArch64-Registers/VBAR-EL1--Vector-Base-Address-Register--EL1-) points to the __Arm64 Vector Table__

    _(Exception Level 1, for NuttX Kernel)_

1.  We read __VBAR_EL1__ to fetch the __Arm64 Vector Table__

1.  We __jump into__ the Vector Table, at the right spot

1.  Which will execute the __NuttX Exception Handler__ for Arm64 SysCall

_Where exactly in the Arm64 Vector Table?_

__VBAR_EL1__ points to this Vector Table: [arm64_vector_table.S](https://github.com/apache/nuttx/blob/master/arch/arm64/src/common/arm64_vector_table.S#L103-L145)

```c
/* Four types of exceptions:
 * - synchronous: aborts from MMU, SP/CP alignment checking, unallocated
 *   instructions, SVCs/SMCs/HVCs, ...)
 * - IRQ: group 1 (normal) interrupts
 * - FIQ: group 0 or secure interrupts
 * - SError: fatal system errors
 *
 * Four different contexts:
 * - from same exception level, when using the SP_EL0 stack pointer
 * - from same exception level, when using the SP_ELx stack pointer
 * - from lower exception level, when this is AArch64
 * - from lower exception level, when this is AArch32
 *
 * +------------------+------------------+-------------------------+
 * |     Address      |  Exception type  |       Description       |
 * +------------------+------------------+-------------------------+
 * | VBAR_ELn + 0x000 | Synchronous      | Current EL with SP0     |
 * |          + 0x080 | IRQ / vIRQ       |                         |
 * |          + 0x100 | FIQ / vFIQ       |                         |
 * |          + 0x180 | SError / vSError |                         |
 * +------------------+------------------+-------------------------+
 * |          + 0x200 | Synchronous      | Current EL with SPx     |
 * |          + 0x280 | IRQ / vIRQ       |                         |
 * |          + 0x300 | FIQ / vFIQ       |                         |
 * |          + 0x380 | SError / vSError |                         |
 * +------------------+------------------+-------------------------+
 * |          + 0x400 | Synchronous      | Lower EL using AArch64  |
 * |          + 0x480 | IRQ / vIRQ       |                         |
 * |          + 0x500 | FIQ / vFIQ       |                         |
 * |          + 0x580 | SError / vSError |                         |
 * +------------------+------------------+-------------------------+
 * |          + 0x600 | Synchronous      | Lower EL using AArch32  |
 * |          + 0x680 | IRQ / vIRQ       |                         |
 * |          + 0x700 | FIQ / vFIQ       |                         |
 * |          + 0x780 | SError / vSError |                         |
 * +------------------+------------------+-------------------------+
```

We're doing __SVC SysCall__ _(Synchronous Exception)_ at Exception Level 1...

Which means Unicorn Emulator should jump to __VBAR_EL1 + 0x200__. Here's how...

# Emulate the Arm64 SysCall

Inside our __Interrupt Hook__: This is how we jump to __VBAR_EL1 + 0x200__: [main.rs](https://github.com/lupyuen/nuttx-arm64-emulator/blob/avaota/src/main.rs#L115-L152)

```rust
/// Hook Function to Handle Unicorn Interrupt
fn hook_interrupt(
    emu: &mut Unicorn<()>,  // Emulator
    intno: u32,             // Interrupt Number
) {
  let pc = emu.reg_read(RegisterARM64::PC).unwrap();
  let x0 = emu.reg_read(RegisterARM64::X0).unwrap();
  println!("hook_interrupt: intno={intno}");
  println!("PC=0x{pc:08x}");
  println!("X0=0x{x0:08x}");
  println!("ESR_EL0={:?}", emu.reg_read(RegisterARM64::ESR_EL0));
  println!("ESR_EL1={:?}", emu.reg_read(RegisterARM64::ESR_EL1));

  // SysCall from NuttX Apps: We don't handle it yet
  if pc >= 0xC000_0000 { println!("TODO: Handle SysCall from NuttX Apps"); finish(); }

  // SysCall from NuttX Kernel: Handle it here...
  if intno == 2 {

    // We are doing SVC (Synchronous Exception) at EL1.
    // Which means Unicorn Emulator should jump to VBAR_EL1 + 0x200.
    let esr_el1 = 0x15 << 26;  // Exception is SVC
    let vbar_el1 = emu.reg_read(RegisterARM64::VBAR_EL1).unwrap();
    let svc = vbar_el1 + 0x200;

    // Update the ESR_EL1 and Program Counter
    emu.reg_write(RegisterARM64::ESR_EL1, esr_el1).unwrap();
    emu.reg_write(RegisterARM64::PC, svc).unwrap();

    // Print the values
    println!("esr_el1=0x{esr_el1:08x}");
    println!("vbar_el1=0x{vbar_el1:08x}");
    println!("jump to svc=0x{svc:08x}");
  }
}
```

[(Why we need __ESR_EL1__)](https://github.com/lupyuen/nuttx-arm64-emulator#esr_el1-is-missing)

And it works: NuttX on Unicorn boots _(almost)_ to __NSH Shell__. Yay!

```bash
$ cargo run | grep "uart output"
...
## NuttX begins booting...
- Ready to Boot Primary CPU
- Boot from EL1
- Boot to C runtime for OS Initialize
nx_start: Entry
up_allocate_kheap: heap_start=0x0x40849000, heap_size=0x77b7000
gic_validate_dist_version: No GIC version detect
arm64_gic_initialize: no distributor detected, giving up ret=-19
uart_register: Registering /dev/console
uart_register: Registering /dev/ttyS0
work_start_highpri: Starting high-priority kernel worker thread(s)
nxtask_activate: hpwork pid=1,TCB=0x40849e78
work_start_lowpri: Starting low-priority kernel worker thread(s)
nxtask_activate: lpwork pid=2,TCB=0x4084c008
nxtask_activate: AppBringUp pid=3,TCB=0x4084c190

## Unicorn calls our Interrupt Hook...
>> exception index = 2
hook_interrupt: intno=2
PC=0x40807300
X0=0x00000002

## We jump to VBAR_EL1 + 0x200
## Which points to NuttX Exception Handler for Arm64 SysCall
esr_el1=0x54000000
vbar_el1=0x40827000
jump to svc=0x40827200

## Unicorn executes the NuttX Exception Handler for Arm64 SysCall
>> exception index = 65536
>>> stop with r = 10000, HLT=10001
>> exception index = 4294967295

## NuttX dumps the Arm64 SysCall
arm64_dump_syscall: SYSCALL arm64_syscall: regs: 0x408483c0 cmd: 2
arm64_dump_syscall: x0:  0x2                 x1:  0x0
arm64_dump_syscall: x2:  0x4084c008          x3:  0x408432b8
arm64_dump_syscall: x4:  0x40849e78          x5:  0x2
arm64_dump_syscall: x6:  0x40843000          x7:  0x3

## NuttX continues booting yay!
nx_start_application: Starting init task: /system/bin/init
nxtask_activate: /system/bin/init pid=4,TCB=0x4084c9f0
nxtask_exit: AppBringUp pid=3,TCB=0x4084c190
...
## More Arm64 SysCalls, handled correctly...
arm64_dump_syscall: SYSCALL arm64_syscall: regs: 0x40853c70 cmd: 1
arm64_dump_syscall: x0:  0x1                 x1:  0x40843000
arm64_dump_syscall: x2:  0x0                 x3:  0x1
arm64_dump_syscall: x4:  0x3                 x5:  0x40844000
arm64_dump_syscall: x6:  0x4                 x7:  0x0
...
arm64_dump_syscall: SYSCALL arm64_syscall: regs: 0x4084bc20 cmd: 2
arm64_dump_syscall: x0:  0x2                 x1:  0xc0
arm64_dump_syscall: x2:  0x4084c008          x3:  0x0
arm64_dump_syscall: x4:  0x408432d0          x5:  0x0
arm64_dump_syscall: x6:  0x0                 x7:  0x0
...
arm64_dump_syscall: SYSCALL arm64_syscall: regs: 0x4084fc20 cmd: 2
arm64_dump_syscall: x0:  0x2                 x1:  0x64
arm64_dump_syscall: x2:  0x4084c9f0          x3:  0x0
arm64_dump_syscall: x4:  0x408432d0          x5:  0x0
arm64_dump_syscall: x6:  0x0                 x7:  0x0
```

[(See the __Unicorn Log__)](https://github.com/lupyuen/nuttx-arm64-emulator/releases/download/avaota-1/unicorn.log)

[(See the __NuttX Log__)](https://github.com/lupyuen/nuttx-arm64-emulator/releases/download/avaota-1/nuttx.log)

But NSH Shell won't start correctly, here's why...

```bash
## Our Emulator stops at SysCall Command 9
>> exception index = 2
hook_interrupt: intno=2
PC=0xc0003f00
X0=0x00000009
ESR_EL1=Ok(1409286144)
TODO: Handle SysCall from NuttX Apps
```

[(Qiling emulates __Linux SysCalls__ with Unicorn)](https://github.com/qilingframework/qiling/blob/master/qiling/os/linux/syscall.py)

[(__Qiling__ is also a Mythical Beast)](https://en.wikipedia.org/wiki/Qilin)

![NuttX Boot Flow for ROM FS Filesystem](https://lupyuen.github.io/images/unicorn4-bootflow.png)

# SysCall from NuttX App

_What's SysCall Command 9? Where in NSH Shell is 0xC000_3F00?_

```bash
$ cargo run
...
## Our Emulator stops at SysCall Command 9
hook_interrupt: intno=2
PC=0xc0003f00
X0=0x00000009
ESR_EL0=Ok(0)
ESR_EL1=Ok(1409286144)
```

According to Arm64 Disassembly of NSH Shell, __SysCall Command 9__ happens inside the `gettid` function: [nuttx-init.S](https://github.com/lupyuen/nuttx-arm64-emulator/blob/avaota/nuttx/nuttx-init.S)

```c
// NSH Shell calls gettid() to fetch Thread ID.
// Exception Level 0 (NuttX App) calls Exception Level 1 (NuttX Kernel).
gettid():
  2ef4:	d2800120 	mov	x0,  #0x9  // SysCall Command 9 (Register X0)
  2ef8:	f81f0ffe 	str	x30, [sp, #-16]!
  2efc:	d4000001 	svc	#0x0       // Execute the SysCall
  2f00:	f84107fe 	ldr	x30, [sp], #16
  2f04:	d65f03c0 	ret
```

This says that...

1.  __NSH Shell__ is starting as a NuttX App

    _(Exception Level 0)_

1.  NSH Shall calls `gettid()` to fetch the __Current Thread ID__

1.  Which triggers an Arm64 SysCall from __NuttX App__ into __NuttX Kernel__

    _(Exception Level 0 calls Exception Level 1)_

1.  Which we haven't implemented yet

    _(Nope, no SysCalls across Exception Levels)_

1.  We'll implement this SysCall soon!

_What about the Call Graph?_

Yep our Avaota Emulator has helpfully generated a __Detailed Call Graph__ (pic above) that describes the NuttX Boot Flow...

- [__NuttX Boot Flow in PDF__](https://github.com/lupyuen/nuttx-arm64-emulator/blob/avaota/nuttx-boot-flow.pdf) / [__SVG__](https://github.com/lupyuen/nuttx-arm64-emulator/blob/avaota/nuttx-boot-flow.svg) / [__PNG__](https://github.com/lupyuen/nuttx-arm64-emulator/blob/avaota/nuttx-boot-flow.png)

- Based on the [__Mermaid Flowchat__](https://raw.githubusercontent.com/lupyuen/nuttx-arm64-emulator/refs/heads/avaota/nuttx-boot-flow.mmd)

_Whoa our eyes are hurting!_

This might help: We feed the above [__Mermaid Flowchat__](https://raw.githubusercontent.com/lupyuen/nuttx-arm64-emulator/refs/heads/avaota/nuttx-boot-flow.mmd) to an LLM to get these...

- [__LLM explains NuttX Boot Flow (Short Version)__](https://gist.github.com/lupyuen/b7d937c302d1926f62cea3411ca0b3c6)

- [__LLM explains NuttX Boot Flow (Long Version)__](https://docs.google.com/document/d/1qYkBu3ca3o5BXdwtUpe0EirMv9PpMOdmf7QBnqGFJkA/edit?tab=t.0)

![Avaota-A1 SBC: Shot on Sony NEX-7 with IKEA Ring Light, Yeelight Ring Light on Corelle Plate](https://lupyuen.github.io/images/unicorn4-title.jpg)

<span style="font-size:80%">

_Shot on Sony NEX-7 with IKEA Ring Light, Yeelight Ring Light on Corelle Plate_

</span>

# What's Next

_Anything else we need for our Avaota Emulator?_

- __Arm64 Timer:__ We need emulate the timer for triggering time-based tasks

- __UART Input:__ So we can enter commands into the NSH Shell

- __GIC v3:__ Arm64 Timer and I/O Interrupts shall be triggered through the GIC Interrupt Controller

- __Emulated Peripherals:__ For testing the NuttX Drivers for GPIO, I2C, SPI, ... Maybe even the Onboard LCD Display (pic above)

Special Thanks to [__My Sponsors__](https://lupyuen.org/articles/sponsor) for supporting my writing. Your support means so much to me 🙏

- [__Sponsor me a coffee__](https://lupyuen.org/articles/sponsor)

- [__Discuss this article on Hacker News__](https://news.ycombinator.com/item?id=43668912)

- [__My Current Project: "Apache NuttX RTOS for StarPro64 EIC7700X"__](https://github.com/lupyuen/nuttx-starpro64)

- [__My Other Project: "NuttX for Oz64 SG2000"__](https://nuttx-forge.org/lupyuen/nuttx-sg2000)

- [__Older Project: "NuttX for Ox64 BL808"__](https://nuttx-forge.org/lupyuen/nuttx-ox64)

- [__Olderer Project: "NuttX for PinePhone"__](https://nuttx-forge.org/lupyuen/pinephone-nuttx)

- [__Check out my articles__](https://lupyuen.org)

- [__RSS Feed__](https://lupyuen.org/rss.xml)

_Got a question, comment or suggestion? Create an Issue or submit a Pull Request here..._

[__lupyuen.org/src/unicorn4.md__](https://codeberg.org/lupyuen/lupyuen.org/src/branch/master/src/unicorn4.md)
