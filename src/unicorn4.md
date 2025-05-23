# Creating the Unicorn Emulator for Avaota-A1 SBC (Apache NuttX RTOS)

📝 _30 Apr 2025_

![TODO](https://lupyuen.org/images/unicorn4-title.jpg)

TODO

- Unicorn doesn't seem to emulate Arm64 SysCalls?

- No worries we'll emulate Arm64 SysCalls ourselves!

![Avaota-A1 SBC with SDWire MicroSD Multiplexer and Smart Power Plug](https://lupyuen.org/images/avaota-title.jpg)

[NuttX Boot Flow in PDF](nuttx-boot-flow.pdf) / [SVG](nuttx-boot-flow.svg) / [PNG](nuttx-boot-flow.png)

[qiling/core_hooks.py](https://github.com/qilingframework/qiling/blob/master/qiling/core_hooks.py)

[qiling/os/linux/syscall.py](https://github.com/qilingframework/qiling/blob/master/qiling/os/linux/syscall.py)

[Qilin](https://en.wikipedia.org/wiki/Qilin)

Trade Tariffs

Emulator -> Driver

Or driver -> emulator?

Maybe Emulator + Device Farm

[“Attached is the Mermaid Flowchart for the Boot Flow for Apache NuttX RTOS. Please explain how NuttX boots.”](https://docs.google.com/document/d/1qYkBu3ca3o5BXdwtUpe0EirMv9PpMOdmf7QBnqGFJkA/edit?tab=t.0)

https://gist.github.com/lupyuen/b7d937c302d1926f62cea3411ca0b3c6

# NuttX for Avaota-A1

Earlier we ported NuttX to Avaota-A1 SBC...

- TODO: Article

To boot __NuttX on Unicorn__: We recompiled NuttX with [__Four Tiny Tweaks__](https://github.com/lupyuen2/wip-nuttx/pull/106)...

1.  [__Set TCR_TG1_4K, Physical / Virtual Address to 32 Bits__](https://github.com/lupyuen2/wip-nuttx/pull/106/commits/640084e1fb1692887266716ecda52dc7ea4bf8e0)

    From the [__Previous Article__](TODO): Unicorn Emulator requires __TCR_TG1_4K__ (TODO what?). And the __Physical / Virtual Address Size__ should be 32 Bits.

1.  [__Disable PSCI__](https://github.com/lupyuen2/wip-nuttx/pull/106/commits/b3782b1ff989667df22b10d5c1023826e2211d88)

    We don't have the __PSCI Driver__ in Unicorn, so we disable this.

1.  [__Enable Scheduler Logging__](https://github.com/lupyuen2/wip-nuttx/pull/106/commits/878e78eb40f334e6e128595dbb27ae08aed1e969)

    So we can see NuttX booting.

1.  [__Enable SysCall Logging__](https://github.com/lupyuen2/wip-nuttx/pull/106/commits/c9f38c13eb5ac6f6bbcd4d3c1de218828f9f087d)

    So we can verify that NuttX SysCalls are OK.

Here are the steps to compile __NuttX for Unicorn__...

```bash
## Compile Modified NuttX for Avaota-A1 SBC
git clone https://github.com/lupyuen2/wip-nuttx nuttx --branch unicorn-avaota
git clone https://github.com/lupyuen2/wip-nuttx-apps apps --branch unicorn-avaota
cd nuttx

## Build NuttX
make -j distclean
tools/configure.sh avaota-a1:nsh
make -j
cp .config nuttx.config

## Build Apps Filesystem
make -j export
pushd ../apps
./tools/mkimport.sh -z -x ../nuttx/nuttx-export-*.tar.gz
make -j import
popd

## Generate the Initial RAM Disk
## Prepare a Padding with 64 KB of zeroes
## Append Padding and Initial RAM Disk to the NuttX Kernel
genromfs -f nuttx-initrd -d ../apps/bin -V "NuttXBootVol"
head -c 65536 /dev/zero >/tmp/nuttx.pad
cat nuttx.bin /tmp/nuttx.pad nuttx-initrd \
  >nuttx-Image

## Dump the NuttX Kernel disassembly to nuttx.S
aarch64-none-elf-objdump \
  --syms --source --reloc --demangle --line-numbers --wide --debugging \
  nuttx \
  >nuttx.S \
  2>&1

## Dump the NSH Shell disassembly to nuttx-init.S
aarch64-none-elf-objdump \
  --syms --source --reloc --demangle --line-numbers --wide --debugging \
  ../apps/bin/init \
  >nuttx-init.S \
  2>&1

## Dump the Hello disassembly to nuttx-hello.S
aarch64-none-elf-objdump \
  --syms --source --reloc --demangle --line-numbers --wide --debugging \
  ../apps/bin/hello \
  >nuttx-hello.S \
  2>&1

## Copy NuttX Image to Unicorn Emulator
cp nuttx nuttx.S nuttx.config nuttx.hash \
  nuttx-init.S nuttx-hello.S \
  $HOME/nuttx-arm64-emulator/nuttx
cp nuttx-Image \
  $HOME/nuttx-arm64-emulator/nuttx/Image
```

To boot NuttX in Unicorn Emulator...

```bash
## Boot NuttX in the Unicorn Emulator
git clone https://github.com/lupyuen/nuttx-arm64-emulator --branch avaota
cd nuttx-arm64-emulator
cargo run

## To see the Emulated UART Output:
cargo run | grep "uart output"
```

TODO: Log

# Unicorn Emulator for Avaota-A1

_What's inside the Avaota-A1 Emulator?_

Inside our Avaota-A1 Emulator: This is how we create the __Unicorn Interface__: [main.rs](TODO)

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

Based on the [__Allwinner A527 Memory Map__](TODO), we reserve __1 GB of I/O Memory__ for GIC (Interrupt Controller), UART and other Peripherals: [main.rs](TODO)

```rust
    // Map 1 GB Read/Write Memory at 0x0000 0000 for Memory-Mapped I/O
    emu.mem_map(
        0x0000_0000,  // Address
        0x4000_0000,  // Size
        Permission::READ | Permission::WRITE  // Read/Write/Execute Access
    ).unwrap();
```

Next we load the __NuttX Image__ _(NuttX Kernel + NuttX Apps)_ into Unicorn Memory: [main.rs](TODO)

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

Unicorn lets us hook into its internals, for special processing. We add the Unicorn Hooks for...

- __Block Hook:__ To draw the Call Graph

- __Memory Hook:__ To emulate the UART Hardware

- __Interrupt Hook:__ To emulate Arm64 SysCalls

Like so: [main.rs](TODO)

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

    // Omitted: Indicate that the UART Transmit FIFO is ready
```

[(__hook_block__ will draw the Call Graph)](TODO)

[(__hook_memory__ will be used for UART Emulation)](TODO)

[(__hook_interrupt__ shall be explained)](TODO)

Finally we start the __Unicorn Emulator__...

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

Thanks to Unicorn: We have a Barebones Emulator for Avaota-A1! Now we fill in the hooks...

# Emulate 16550 UART

_What about I/O? How to emulate in Unicorn?_

Let's emulate the Bare Minimum for I/O: Printing output to the __16550 UART__...

1.  We intercept all writes to the __UART Transmit Register__, and print them 

1.  We signal to NuttX that __UART Transmit FIFO__ is always ready for output

This will tell NuttX that __UART Transmit FIFO__ is always ready: [main.rs](TODO)

```rust
/// UART Base Address
const UART0_BASE_ADDRESS: u64 = 0x02500000;

fn main() {
    ...
    // TODO Allwinner A64 UART Line Status Register (UART_LSR) at Offset 0x14.
    // To indicate that the UART Transmit FIFO is ready:
    // Set Bit 5 to 1.
    // TODO https://lupyuen.github.io/articles/serial#wait-to-transmit
    emu.mem_write(
        UART0_BASE_ADDRESS + 0x14,  // UART Register Address
        &[0b10_0000]  // UART Register Value
    ).unwrap();
```

And this will intercept all writes to the __UART Transmit Register__: [main.rs](TODO)

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
    // Ignore RAM access, we only intercept Memory-Mapped Input / Output
    if address >= 0x4000_0000 { return true; }
    // println!("hook_memory: address={address:#010x}, size={size:02}, mem_type={mem_type:?}, value={value:#x}");

    // If writing to UART Transmit Holding Register (THR):
    // Print the UART Output
    // https://lupyuen.github.io/articles/serial#transmit-uart
    if address == UART0_BASE_ADDRESS {
        println!("uart output: {:?}", value as u8 as char);
        // print!("{}", value as u8 as char);
    }

    // Always return true, value is unused by caller
    // https://github.com/unicorn-engine/unicorn/blob/dev/docs/FAQ.md#i-cant-recover-from-unmapped-readwrite-even-i-return-true-in-the-hook-why
    true
}
```

This means our Barebones Emulator will print the UART Output and show the __NuttX Boot Log__...

```bash
## To see the Emulated UART Output:
$ cargo run | grep "uart output"
TODO
```

![TODO](https://lupyuen.org/images/unicorn3-avaota.jpg)

# NuttX Halts at SysCall

_What happens when we run this?_

We run the [__Barebones Emulator__](TODO) (from earlier).

NuttX halts with an __Arm64 Exception__ at this curious address: _0x4080_6D60_...

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
AAAAAAAAAAAA
>>> invalid memory accessed, STOP = 21!!!
err=Err(EXCEPTION)
PC=0x40806d60
```

_What's at 0x4080_6D60?_

We look up the [__NuttX Kernel Disassembly__](TODO). We see that _0x4080_6D60_ points to an __Arm64 SysCall `svc 0`__...

```c
sys_call0():
/Users/luppy/avaota/nuttx/include/arch/syscall.h:152
/* SVC with SYS_ call number and no parameters */
static inline uintptr_t sys_call0(unsigned int nbr)
{
  register uint64_t reg0 __asm__("x0") = (uint64_t)(nbr);
    40806d58:	d2800040 	mov	x0, #0x2                   	// #2
/Users/luppy/avaota/nuttx/include/arch/syscall.h:154
  __asm__ __volatile__
    40806d5c:	d4000001 	svc	#0x0
// 0x40806d60 is the next instruction to be executed on return from SysCall
```

_Isn't Unicorn supposed to handle Arm64 SysCalls?_

We step through Unicorn with the excellent [__CodeLLDB Debugger__](https://marketplace.visualstudio.com/items?itemName=vadimcn.vscode-lldb) (pic above). Unicorn triggers the Arm64 Exception here: [unicorn-engine-2.1.3/qemu/accel/tcg/cpu-exec.c](TODO)

```c
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
  // Unicorn: If un-catched interrupt, stop executions.
  if (!catched) {
    printf("AAAAAAAAAAAA\n"); // qq
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

Aha! Unicorn is expecting us to __Hook This Interrupt__ and handle the Arm64 SysCall via our Interrupt Callback. Let's do it...

# NuttX SysCall

_Why is NuttX doing an Arm64 SysCall? Aren't SysCalls used by NuttX Apps?_

Earlier we saw NuttX making an Arm64 SysCall, let's find out why.

NuttX passes a __Parameter to SysCall__ in Register X0. The value is 2...

```c
/Users/luppy/avaota/nuttx/sched/sched/sched_unlock.c:92
                {
                  up_switch_context(this_task(), rtcb);
    40807230:	d538d080 	mrs	x0, tpidr_el1
    40807234:	37000060 	tbnz	w0, #0, 40807240 <sched_unlock+0x80>
sys_call0():
/Users/luppy/avaota/nuttx/include/arch/syscall.h:152
/* SVC with SYS_ call number and no parameters */
static inline uintptr_t sys_call0(unsigned int nbr)
{
  register uint64_t reg0 __asm__("x0") = (uint64_t)(nbr);
    40807238:	d2800040 	mov	x0, #0x2                   	// #2
/Users/luppy/avaota/nuttx/include/arch/syscall.h:154
  __asm__ __volatile__
    4080723c:	d4000001 	svc	#0x0
```

What is SysCall with Parameter 2? It's for __Switching The Context__ between NuttX Tasks...

https://github.com/apache/nuttx/blob/master/arch/arm64/include/syscall.h#L78-L83

```c
/* SYS call 2:
 * void arm64_switchcontext(void **saveregs, void *restoreregs);
 */
#define SYS_switch_context        (2)
```

Which is implemented here...

https://github.com/apache/nuttx/blob/master/arch/arm64/src/common/arm64_syscall.c#L201-L216

```c
uint64_t *arm64_syscall(uint64_t *regs) {
  ...
      case SYS_switch_context:

        /* Update scheduler parameters */

        nxsched_suspend_scheduler(*running_task);
        nxsched_resume_scheduler(tcb);
        *running_task = tcb;

        /* Restore the cpu lock */

        restore_critical_section(tcb, cpu);
#ifdef CONFIG_ARCH_ADDRENV
        addrenv_switch(tcb);
#endif
        break;
```

Ah now we see the light. NuttX makes a SysCall during startup, to trigger the __Very First Context Switch__! Which will start the other NuttX Tasks and boot successfully.

FYI NuttX SysCalls are defined here...

https://github.com/apache/nuttx/blob/master/include/sys/syscall_lookup.h

```c
SYSCALL_LOOKUP(getpid,                     0)
SYSCALL_LOOKUP(gettid,                     0)
SYSCALL_LOOKUP(sched_getcpu,               0)
SYSCALL_LOOKUP(sched_lock,                 0)
SYSCALL_LOOKUP(sched_lockcount,            0)
SYSCALL_LOOKUP(sched_unlock,               0)
SYSCALL_LOOKUP(sched_yield,                0)
```

TODO: SysCall Spreadsheet

# Hook The Unicorn Interrupt

_To boot NuttX: We need to Emulate the SysCall. How?_

We saw earlier that Unicorn expects us to [__Hook The Interrupt__](TODO) and emulate the SysCall. This is how we Hook the Interrupt: TODO

```rust
fn main() {
    ...
    // Add Interrupt Hook
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

/// Hook Function to Handle Interrupt
fn hook_interrupt(
    emu: &mut Unicorn<()>,  // Emulator
    intno: u32, // Interrupt Number
) {
    let pc = emu.reg_read(RegisterARM64::PC).unwrap();
    let x0 = emu.reg_read(RegisterARM64::X0).unwrap();
    println!("hook_interrupt: intno={intno}");
    println!("PC=0x{pc:08x}");
    println!("X0=0x{x0:08x}");
    println!("ESR_EL0={:?}", emu.reg_read(RegisterARM64::ESR_EL0));
    println!("ESR_EL1={:?}", emu.reg_read(RegisterARM64::ESR_EL1));
    println!("ESR_EL2={:?}", emu.reg_read(RegisterARM64::ESR_EL2));
    println!("ESR_EL3={:?}", emu.reg_read(RegisterARM64::ESR_EL3));

    // Upcoming: Handle the SysCall
    ...
}
```

With this Barebones Interrupt Hook: Now Unicorn calls our Interrupt Handler...

```bash
$ cargo run
...
hook_block:  address=0x40806d50, size=08, sched_unlock, sched/sched/sched_unlock.c:92:19
hook_block:  address=0x40806d58, size=08, sys_call0, arch/arm64/include/syscall.h:152:21
call_graph:  sched_unlock --> sys_call0
call_graph:  click sched_unlock href "https://github.com/apache/nuttx/blob/master/sched/sched/sched_unlock.c#L89" "sched/sched/sched_unlock.c " _blank
>> exception index = 2
hook_interrupt: intno=2
PC=0x40806d60
WARNING: Your register accessing on id 290 is deprecated and will get UC_ERR_ARG in the future release (2.2.0) because the accessing is either no-op or not defined. If you believe the register should be implemented or there is a bug, please submit an issue to https://github.com/unicorn-engine/unicorn. Set UC_IGNORE_REG_BREAK=1 to ignore this warning.
CP_REG=Ok(0)
ESR_EL0=Ok(0)
ESR_EL1=Ok(0)
ESR_EL2=Ok(0)
ESR_EL3=Ok(0)
hook_block:  address=0x40806d60, size=16, sched_unlock, sched/sched/sched_unlock.c:104:28
call_graph:  sys_call0 --> sched_unlock
call_graph:  click sys_call0 href "https://github.com/apache/nuttx/blob/master/arch/arm64/include/syscall.h#L151" "arch/arm64/include/syscall.h " _blank
hook_block:  address=0x40806d90, size=04, up_irq_restore, arch/arm64/include/irq.h:383:3
hook_block:  address=0x40806d94, size=12, sched_unlock, sched/sched/sched_unlock.c:168:1
call_graph:  up_irq_restore --> sched_unlock
call_graph:  click up_irq_restore href "https://github.com/apache/nuttx/blob/master/arch/arm64/include/irq.h#L382" "arch/arm64/include/irq.h " _blank
hook_block:  address=0x408062b4, size=04, nx_start, sched/init/nx_start.c:782:7
hook_block:  address=0x408169c8, size=08, up_idle, arch/arm64/src/common/arm64_idle.c:62:3
call_graph:  nx_start --> up_idle
call_graph:  click nx_start href "https://github.com/apache/nuttx/blob/master/sched/init/nx_start.c#L781" "sched/init/nx_start.c " _blank
>> exception index = 65537
>>> stop with r = 10001, HLT=10001
>>> got HLT!!!
err=Ok(())
PC=0x408169d0
WARNING: Your register accessing on id 290 is deprecated and will get UC_ERR_ARG in the future release (2.2.0) because the accessing is either no-op or not defined. If you believe the register should be implemented or there is a bug, please submit an issue to https://github.com/unicorn-engine/unicorn. Set UC_IGNORE_REG_BREAK=1 to ignore this warning.
CP_REG=Ok(0)
ESR_EL0=Ok(0)
ESR_EL1=Ok(0)
ESR_EL2=Ok(0)
ESR_EL3=Ok(0)
call_graph:  up_idle --> ***_HALT_***
call_graph:  click up_idle href "https://github.com/apache/nuttx/blob/master/arch/arm64/src/common/arm64_idle.c#L61" "arch/arm64/src/common/arm64_idle.c " _blank
```

[(PC 0x408169d0 points to WFI)](TODO)

But we're not done yet! Unicorn halts because we haven't emulated the Arm64 SysCall. Let's do it...

# Arm64 Vector Table

_How to emulate the Arm64 SysCall?_

Here's our plan...

1.  System Register [__VBAR_EL1__](TODO) points to the Arm64 Vector Table for [__Exception Level 1__](TODO)

1.  We read VBAR_EL1 to fetch the Arm64 Vector Table

1.  Then we jump to the proper place in the Vector Table

1.  Which will execute the Exception Handler for Arm64 SysCall

_What's inside the Arm64 Vector Table?_

VBAR_EL1 points to this: [arm64_vector_table.S](https://github.com/apache/nuttx/blob/master/arch/arm64/src/common/arm64_vector_table.S#L103-L145)

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

We are doing SVC (Synchronous Exception) at EL1. Which means Unicorn Emulator should jump to VBAR_EL1 + 0x200.

# Emulate the Arm64 SysCall

Inside our Interrupt Hook: This is how we jump to VBAR_EL1 + 0x200: [main.rs](TODO)

```rust
/// Hook Function to Handle Interrupt
fn hook_interrupt(
    emu: &mut Unicorn<()>,  // Emulator
    intno: u32, // Interrupt Number
) {
    let pc = emu.reg_read(RegisterARM64::PC).unwrap();
    let x0 = emu.reg_read(RegisterARM64::X0).unwrap();
    println!("hook_interrupt: intno={intno}");
    println!("PC=0x{pc:08x}");
    println!("X0=0x{x0:08x}");
    println!("ESR_EL0={:?}", emu.reg_read(RegisterARM64::ESR_EL0));
    println!("ESR_EL1={:?}", emu.reg_read(RegisterARM64::ESR_EL1));
    println!("ESR_EL2={:?}", emu.reg_read(RegisterARM64::ESR_EL2));
    println!("ESR_EL3={:?}", emu.reg_read(RegisterARM64::ESR_EL3));

    // TODO
    // We don't handle SysCalls from NuttX Apps yet
    if pc >= 0xC000_0000 {
        println!("TODO: Handle SysCall from NuttX Apps");
        finish();
    }

    // Handle the SysCall
    if intno == 2 {
        // We are doing SVC (Synchronous Exception) at EL1.
        // Which means Unicorn Emulator should jump to VBAR_EL1 + 0x200.
        let esr_el1 = 0x15 << 26;  // Exception is SVC
        let vbar_el1 = emu.reg_read(RegisterARM64::VBAR_EL1).unwrap();
        let svc = vbar_el1 + 0x200;
        println!("esr_el1=0x{esr_el1:08x}");
        println!("vbar_el1=0x{vbar_el1:08x}");
        println!("jump to svc=0x{svc:08x}");
        emu.reg_write(RegisterARM64::ESR_EL1, esr_el1).unwrap();
        emu.reg_write(RegisterARM64::PC, svc).unwrap();
    } else {
        sleep(time::Duration::from_secs(10));
    }
}
```

And it works!

TODO

NuttX on Unicorn now boots to NSH Shell. Yay!

```bash
- Ready to Boot Primary CPU
- Boot from EL1
- Boot to C runtime for OS Initialize
\rnx_start: Entry
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
>> exception index = 2
hook_interrupt: intno=2
PC=0x40807300
X0=0x00000002
ESR_EL0=Ok(0)
ESR_EL1=Ok(0)
ESR_EL2=Ok(0)
ESR_EL3=Ok(0)
esr_el1=0x54000000
vbar_el1=0x40827000
jump to svc=0x40827200
>> exception index = 65536
>>> stop with r = 10000, HLT=10001
>> exception index = 4294967295

arm64_dump_syscall: SYSCALL arm64_syscall: regs: 0x408483c0 cmd: 2
arm64_dump_syscall: x0:  0x2                 x1:  0x0
arm64_dump_syscall: x2:  0x4084c008          x3:  0x408432b8
arm64_dump_syscall: x4:  0x40849e78          x5:  0x2
arm64_dump_syscall: x6:  0x40843000          x7:  0x3

nx_start_application: Starting init task: /system/bin/init
nxtask_activate: /system/bin/init pid=4,TCB=0x4084c9f0
nxtask_exit: AppBringUp pid=3,TCB=0x4084c190

>> exception index = 2
hook_interrupt: intno=2
PC=0x40816be8
X0=0x00000001
ESR_EL0=Ok(0)
ESR_EL1=Ok(1409286144)
ESR_EL2=Ok(0)
ESR_EL3=Ok(0)
esr_el1=0x54000000
vbar_el1=0x40827000
jump to svc=0x40827200
>> exception index = 65536
>>> stop with r = 10000, HLT=10001
>> exception index = 4294967295

arm64_dump_syscall: SYSCALL arm64_syscall: regs: 0x40853c70 cmd: 1
arm64_dump_syscall: x0:  0x1                 x1:  0x40843000
arm64_dump_syscall: x2:  0x0                 x3:  0x1
arm64_dump_syscall: x4:  0x3                 x5:  0x40844000
arm64_dump_syscall: x6:  0x4                 x7:  0x0

>> exception index = 2
hook_interrupt: intno=2
PC=0x4080b35c
X0=0x00000002
ESR_EL0=Ok(0)
ESR_EL1=Ok(1409286144)
ESR_EL2=Ok(0)
ESR_EL3=Ok(0)
esr_el1=0x54000000
vbar_el1=0x40827000
jump to svc=0x40827200
>> exception index = 65536
>>> stop with r = 10000, HLT=10001
>> exception index = 4294967295

arm64_dump_syscall: SYSCALL arm64_syscall: regs: 0x4084bc20 cmd: 2
arm64_dump_syscall: x0:  0x2                 x1:  0xc0
arm64_dump_syscall: x2:  0x4084c008          x3:  0x0
arm64_dump_syscall: x4:  0x408432d0          x5:  0x0
arm64_dump_syscall: x6:  0x0                 x7:  0x0

>> exception index = 2
hook_interrupt: intno=2
PC=0x4080b35c
X0=0x00000002
ESR_EL0=Ok(0)
ESR_EL1=Ok(1409286144)
ESR_EL2=Ok(0)
ESR_EL3=Ok(0)
esr_el1=0x54000000
vbar_el1=0x40827000
jump to svc=0x40827200
>> exception index = 65536
>>> stop with r = 10000, HLT=10001
>> exception index = 4294967295

arm64_dump_syscall: SYSCALL arm64_syscall: regs: 0x4084fc20 cmd: 2
arm64_dump_syscall: x0:  0x2                 x1:  0x64
arm64_dump_syscall: x2:  0x4084c9f0          x3:  0x0
arm64_dump_syscall: x4:  0x408432d0          x5:  0x0
arm64_dump_syscall: x6:  0x0                 x7:  0x0

>> exception index = 2
hook_interrupt: intno=2
PC=0xc0003f00
X0=0x00000009
ESR_EL0=Ok(0)
ESR_EL1=Ok(1409286144)
ESR_EL2=Ok(0)
ESR_EL3=Ok(0)
TODO: Handle SysCall from NuttX Apps
```

But NSH Shell won't start correctly, here's why...

TODO: Why ESR_EL1?

# SysCall from NuttX App

_What is SysCall Command 9? Where in NSH Shell is 0xC000_3F00?_

```bash
hook_interrupt: intno=2
PC=0xc0003f00
X0=0x00000009
ESR_EL0=Ok(0)
ESR_EL1=Ok(1409286144)
```

According to the RISC-V Disassembly of NSH Shell, SysCall Command 9 happens inside `gettid()`: [nuttx-init.S](TODO)

```c
0000000000002ef4 <gettid>:
gettid():
    2ef4:	d2800120 	mov	x0, #0x9                   	// #9
    2ef8:	f81f0ffe 	str	x30, [sp, #-16]!
    2efc:	d4000001 	svc	#0x0
    2f00:	f84107fe 	ldr	x30, [sp], #16
    2f04:	d65f03c0 	ret
```

Which means that NSH Shell is starting up and calling `gettid()` to fetch the __Current Thread ID__. But it triggers a SysCall from the __NuttX App (EL0)__ into __NuttX Kernel (EL1)__, which we haven't implemented.

We'll implement this soon!

TODO: GIC

TODO: Timer

TODO: Other Peripherals

# What's Next

Special Thanks to [__My Sponsors__](https://lupyuen.org/articles/sponsor) for supporting my writing. Your support means so much to me 🙏

- [__Sponsor me a coffee__](https://lupyuen.org/articles/sponsor)

- [__Discuss this article on Hacker News__](TODO)

- [__My Current Project: "Apache NuttX RTOS for StarPro64 EIC7700X"__](https://github.com/lupyuen/nuttx-starpro64)

- [__My Other Project: "NuttX for Oz64 SG2000"__](https://nuttx-forge.org/lupyuen/nuttx-sg2000)

- [__Older Project: "NuttX for Ox64 BL808"__](https://nuttx-forge.org/lupyuen/nuttx-ox64)

- [__Olderer Project: "NuttX for PinePhone"__](https://nuttx-forge.org/lupyuen/pinephone-nuttx)

- [__Check out my articles__](https://lupyuen.org)

- [__RSS Feed__](https://lupyuen.org/rss.xml)

_Got a question, comment or suggestion? Create an Issue or submit a Pull Request here..._

[__lupyuen.org/src/unicorn4.md__](https://codeberg.org/lupyuen/lupyuen.org/src/branch/master/src/unicorn4.md)

![TODO](https://lupyuen.org/images/unicorn4-title.jpg)

<span style="font-size:80%">

_Shot on Sony NEX-7 with IKEA Ring Light, Yeelight Ring Light on Corelle Plate :-)_

</span>
