# Rust Standard Library on Apache NuttX RTOS

📝 _26 Jan 2025_

![LED Blinky with Rust Standard Library on Apache NuttX RTOS (RustRover IDE)](https://lupyuen.github.io/images/rust7-title.png)

__Freshly Baked:__ Here's how we [__Blink the LED__](https://github.com/lupyuen2/wip-nuttx-apps/blob/rust-std/examples/rust/hello/src/lib.rs) with __Rust Standard Library__ on [__Apache NuttX RTOS__](https://nuttx.apache.org/docs/latest/index.html)...

```rust
// Open the LED Device for NuttX
let fd = open(      // Equivalent to NuttX open()
  "/dev/userleds",  // LED Device
  OFlag::O_WRONLY,  // Write Only
  Mode::empty()     // No Modes
).unwrap();         // Halt on Error

// Define the ioctl() function for Flipping LEDs
const ULEDIOC_SETALL: i32 = 0x1d03;  // ioctl() Command
ioctl_write_int_bad!(  // ioctl() will write One Int Value (LED Bit State)
  led_set_all,         // Name of our New Function
  ULEDIOC_SETALL       // ioctl() Command to send
);

// Flip LED 1 to On
unsafe {             // Be careful of ioctl()
  led_set_all(       // Set the LEDs for...
    fd.as_raw_fd(),  // LED Device
    1                // LED 1 (Bit 0) turns On
  ).unwrap();        // Halt on Error
}  // Equivalent to ioctl(fd, ULEDIOC_SETALL, 1)

// Flip LED 1 to Off: ioctl(fd, ULEDIOC_SETALL, 0)
unsafe { led_set_all(fd.as_raw_fd(), 0).unwrap(); }
```

Which requires the __`nix` Rust Crate__ / Library...

```bash
## Add the `nix` Rust Crate
## To our NuttX Rust App
$ cd apps/examples/rust/hello
$ cargo add nix --features fs,ioctl

Updating crates.io index
Adding nix v0.29.0 to dependencies
Features: + fs + ioctl
```
_(OK it's more complicated. Stay tuned)_

All this is now possible, thanks to the awesome work by [__Huang Qi__](https://github.com/apache/nuttx-apps/pull/2487)! 🎉

In today's article, we explain...

- How to build __NuttX + Rust Standard Library__

- __Handling JSON__ with the Serde Crate

- __Async Functions__ with the Tokio Crate

- __Blinking LEDs__ with the Nix Crate

- How we ported __Nix to NuttX__

- Why Nix? __Rustix__ might be better

- Why __File Descriptors are "Owned"__ in Rust

# Compile our Rust Hello App

_How to build NuttX + Rust Standard Library?_

Follow the instructions here...

- [__"Build NuttX for Rust Standard Library"__](https://lupyuen.github.io/articles/rust7#appendix-build-nuttx-for-rust-standard-library)

Then run the (thoroughly revamped) [__Rust Hello App__](https://github.com/apache/nuttx-apps/blob/master/examples/rust/hello/src/lib.rs) with __QEMU RISC-V Emulator__...

```bash
## Start NuttX on QEMU RISC-V 64-bit
$ qemu-system-riscv64 \
  -semihosting \
  -M virt,aclint=on \
  -cpu rv64 \
  -bios none \
  -kernel nuttx \
  -nographic

## Run the Rust Hello App
NuttShell (NSH) NuttX-12.8.0
nsh> hello_rust_cargo

{"name":"John","age":30}
{"name":"Jane","age":25}
Deserialized: Alice is 28 years old

Pretty JSON:
{
  "name": "Alice",
  "age": 28
}

Hello world from tokio!
```

Some bits are [__a little wonky__](https://lupyuen.github.io/articles/rust7#appendix-build-nuttx-for-rust-standard-library) (but will get better)

- Supports [__Arm and RISC-V__](https://nuttx.apache.org/docs/latest/guides/rust.html) architectures _(32-bit and 64-bit)_

- Works on __Rust Nightly Toolchain__ _(not Rust Stable)_

- Needs a tiny patch to __Local Toolchain__ _(pal/unix/fs.rs)_

- Sorry no __RISC-V Floating Point__ and no __Kernel Build__

What's inside the brand new Rust Hello App? We dive in...

![JSON with Serde on Apache NuttX RTOS (Neovim IDE)](https://lupyuen.github.io/images/rust7-json2.png)

# JSON with Serde

_What's this Serde?_

Think _"Serialize-Deserialize"_. [__Serde__](https://crates.io/crates/serde) is a Rust Crate / Library for Serializing and Deserializing our Data Structures. Works with [__JSON, CBOR, MessagePack, ...__](https://serde.rs/#data-formats)

This is how we __Serialize to JSON__ in our Hello Rust App: [nuttx-apps/lib.rs](https://github.com/apache/nuttx-apps/blob/master/examples/rust/hello/src/lib.rs#L1-L32)

```rust
// Allow Serde to Serialize and Deserialize a Person Struct
#[derive(Serialize, Deserialize)]
struct Person {
  name: String,  // Containing a Name (string)
  age:  u8,      // And Age (uint8_t)
}  // Note: Rust Strings live in Heap Memory!

// Main Function of our Hello Rust App
#[no_mangle]
pub extern "C" fn hello_rust_cargo_main() {

  // Create a Person Struct
  let john = Person {
    name: "John".to_string(),
    age:  30,
  };

  // Serialize our Person Struct
  let json_str = serde_json // Person Struct
    ::to_string(&john)  // Becomes a String
    .unwrap();          // Halt on Error
  println!("{}", json_str);
```

Which will print...

```bash
NuttShell (NSH) NuttX-12.8.0
nsh> hello_rust_cargo
{"name":"John","age":30}
```

Now we __Deserialize from JSON__: [lib.rs](https://github.com/apache/nuttx-apps/blob/master/examples/rust/hello/src/lib.rs#L32-L41)

```rust
// Declare a String with JSON inside
let json_data = r#"
  {
    "name": "Alice",
    "age": 28
  }"#;

// Deserialize our JSON String
// Into a Person Struct
let alice: Person = serde_json // Get Person Struct
  ::from_str(json_data)  // From JSON String
  .unwrap();             // Halt on Error
println!("Deserialized: {} is {} years old",
  alice.name, alice.age);
```

And we'll see...

```bash
Deserialized: Alice is 28 years old
```

Serde will also do __JSON Formatting__: [lib.rs](https://github.com/apache/nuttx-apps/blob/master/examples/rust/hello/src/lib.rs#L41-L44)

```rust
// Serialize our Person Struct
// But neatly please
let pretty_json_str = serde_json // Person Struct
  ::to_string_pretty(&alice)     // Becomes a Formatted String
  .unwrap();                     // Halt on Error
println!("Pretty JSON:\n{}", pretty_json_str);
```

Looks much neater...

```bash
Pretty JSON:
{
  "name": "Alice",
  "age": 28
}
```

[(Serde runs on __Rust Core Library__, though super messy)](https://bitboom.github.io/2020-10-22/serde-no-std)

![Async Functions with Tokio (Helix Editor + Zellij Workspace)](https://lupyuen.github.io/images/rust7-tokio.png)

# Async Functions with Tokio

_What's this Tokio? Sounds like a city?_

Indeed, "Tokio" is inspired by Tokyo (and [__Metal I/O__](https://crates.io/crates/mio))

> [__Tokio__](https://en.wikipedia.org/wiki/Tokio_(software)) ... provides a runtime and functions that enable the use of __Asynchronous I/O__, allowing for Concurrency in regards to Task Completion

Inside our __Rust Hello App__, here's how we run __Async Functions__ with Tokio: [nuttx-apps/lib.rs](https://github.com/apache/nuttx-apps/blob/master/examples/rust/hello/src/lib.rs#L44-L56)

```rust
// Use One Single Thread (Current Thread)
// To schedule Async Functions
tokio::runtime::Builder
  ::new_current_thread()  // Current Thread is the Single-Threaded Scheduler
  .enable_all()  // Enable the I/O and Time Functions
  .build()   // Create the Single-Threaded Scheduler
  .unwrap()  // Halt on Error
  .block_on( // Start the Scheduler
    async {  // With this Async Code
      println!("Hello world from tokio!");
  });

// Is it really async? Let's block and find out!
println!("Looping Forever...");
loop {}
```

We'll see...

```bash
nsh> hello_rust_cargo
Hello world from tokio!
Looping Forever...
```

[(Derived from __tokio::main__)](https://tokio.rs/tokio/topics/bridging)

_Yawn. Tokio looks underwhelming?_

Ah we haven't seen the full power of __Tokio Multi-Threaded Async Functions__ on NuttX...

```bash
nsh> hello_rust_cargo
pthread_create
nx_pthread_create

Task 0 sleeping for 1000 ms
Task 1 sleeping for  950 ms
Task 2 sleeping for  900 ms
Task 3 sleeping for  850 ms

Finished time-consuming task
Task 3 stopping
Task 2 stopping
Task 1 stopping
Task 0 stopping
```

Check this link for the __Tokio Async Demo__. And it works beautifully on NuttX! (Pic below)

- [__"Tokio Async Threading"__](https://lupyuen.github.io/articles/rust7#appendix-tokio-async-threading)

![Tokio Async Demo](https://lupyuen.github.io/images/rust7-vscode2.png)

_NuttX has POSIX Threads. Why use Async Functions?_

Think [__Node.js__](https://en.wikipedia.org/wiki/Node.js#Threading) and its _Single-Thread Event Loop_, making _Non-Blocking I/O Calls_. Supporting tens of thousands of concurrent connections. _(Without costly Thread Context Switching)_

Today we can (probably) do the same with __NuttX and Async Rust__. Assuming [__POSIX Async I/O__](https://nuttx.apache.org/docs/12.5.0/components/filesystem/aio.html) works OK with [__Tokio__](https://github.com/tokio-rs/tokio/blob/master/tokio/src/io/bsd/poll_aio.rs).

(Tokio calls them _"Async Tasks"_, sorry we won't. Because a [__Task in NuttX__](https://cwiki.apache.org/confluence/display/NUTTX/Tasks+vs.+Threads+FAQ) means something else)

_How will we use Tokio?_

> [__Tokio__](https://tokio.rs/tokio/tutorial) is designed for __I/O-Bound Applications__ where each individual task spends most of its time waiting for I/O.

Which means it's great for [__Network Servers__](https://tokio.rs/tokio/tutorial/io). Instead of spawning many __POSIX Threads__, we spawn a few threads and call __Async Functions__.

(Check out [__Tokio Select__](https://tokio.rs/tokio/tutorial/select) and [__Tokio Streams__](https://tokio.rs/tokio/tutorial/streams))

![LED Blinky with Rust Standard Library on Apache NuttX RTOS (RustRover IDE)](https://lupyuen.github.io/images/rust7-title.png)

# LED Blinky with Nix

_We're running nix on NuttX?_

Oh that's [__`nix` Crate__](https://crates.io/crates/nix) that provides __Safer Rust Bindings__ for POSIX / Unix / Linux. (It's not NixOS)

This is how we add the library to our __Rust Hello App__...

```bash
$ cd ../apps/examples/rust/hello
$ cargo add nix \
  --features fs,ioctl \
  --git https://github.com/lupyuen/nix.git \
  --branch nuttx

Updating git repository `https://github.com/lupyuen/nix.git`
Adding nix (git) to dependencies
Features: + fs + ioctl
34 deactivated features
```

_URL looks sus?_

Yep it's our Bespoke __`nix`__ Crate. That's because the Official __`nix`__ Crate doesn't support NuttX yet. We made [__a few tweaks__](https://github.com/lupyuen/nix/pull/1/files) to compile on NuttX. [(Explained in the __Appendix__)](https://lupyuen.github.io/articles/rust7#appendix-porting-nix-to-nuttx)

_Why call nix?_

We're __Blinking the LED__ on NuttX. We could call the [__POSIX API__](https://crates.io/crates/libc) direcly from Rust...

```rust
let fd = unsafe { libc::open("/dev/userleds", ...) };
unsafe { libc::ioctl(fd, ULEDIOC_SETALL, 1); }
unsafe { libc::close(fd); }
```

Though it doesn't look very... Safe. That's why we call the __Safer POSIX Bindings__ provided by __`nix`__. Like so: [wip-nuttx-apps/lib.rs](https://github.com/lupyuen2/wip-nuttx-apps/blob/rust-std/examples/rust/hello/src/lib.rs#L6-L39)

```rust
// Open the LED Device for NuttX
let fd = open(      // Equivalent to NuttX open()
  "/dev/userleds",  // LED Device
  OFlag::O_WRONLY,  // Write Only
  Mode::empty()     // No Modes
).unwrap();         // Halt on Error

// Define the ioctl() function for Flipping LEDs
const ULEDIOC_SETALL: i32 = 0x1d03;  // ioctl() Command
ioctl_write_int_bad!(  // ioctl() will write One Int Value (LED Bit State)
  led_set_all,         // Name of our New Function
  ULEDIOC_SETALL       // ioctl() Command to send
);
```

The code above opens the __LED Device__, returning an __Owned File Descriptor__ (explained below). It defines a function __led_set_all__, that will call _ioctl()_ to flip the LED.

Here's how we call __led_set_all__ to flip the LED: [lib.rs](https://github.com/lupyuen2/wip-nuttx-apps/blob/rust-std/examples/rust/hello/src/lib.rs#L39-L47)

```rust
// Flip LED 1 to On
unsafe {             // Be careful of ioctl()
  led_set_all(       // Set the LEDs for...
    fd.as_raw_fd(),  // LED Device
    1                // LED 1 (Bit 0) turns On
  ).unwrap();        // Halt on Error
}  // Equivalent to ioctl(fd, ULEDIOC_SETALL, 1)
```

We wait Two Seconds, then flip the __LED to Off__: [lib.rs](https://github.com/lupyuen2/wip-nuttx-apps/blob/rust-std/examples/rust/hello/src/lib.rs#L47-L53)

```rust
// Wait 2 seconds
sleep(2);

// Flip LED 1 to Off: ioctl(fd, ULEDIOC_SETALL, 0)
unsafe { led_set_all(fd.as_raw_fd(), 0).unwrap(); }
```

_ULEDIOC_SETALL looks familiar?_

We spoke about _ULEDIOC_SETALL_ in [__an earlier article__](https://lupyuen.github.io/articles/rust6#blink-the-led). And the Rust Code above mirrors the [__C Version__](https://github.com/lupyuen2/wip-nuttx-apps/blob/nim/examples/hello/hello_main.c#L40-L85) of our Blinky App.

_How to run the Rust Blinky App?_

1.  Copy the __Rust Blinky Files__ from here...

    [_lupyuen2/wip-nuttx-apps/examples/rust/hello_](https://github.com/lupyuen2/wip-nuttx-apps/tree/rust-std/examples/rust/hello)

    Specifically: [__Cargo.toml__](https://github.com/lupyuen2/wip-nuttx-apps/blob/rust-std/examples/rust/hello/Cargo.toml) and [__src/lib.rs__](https://github.com/lupyuen2/wip-nuttx-apps/blob/rust-std/examples/rust/hello/src/lib.rs)

1.  Overwrite our __Rust Hello App__...

    _apps/examples/rust/hello_

1.  [Rebuild our __NuttX Project__](https://lupyuen.github.io/articles/rust7#appendix-build-nuttx-for-rust-standard-library)

    ```bash
    make -j
    ```

1.  Then run it with __QEMU RISC-V Emulator__

    ```bash
    $ qemu-system-riscv64 \
      -semihosting \
      -M virt,aclint=on \
      -cpu rv64 \
      -bios none \
      -kernel nuttx \
      -nographic

    NuttShell (NSH) NuttX-12.8.0
    nsh> hello_rust_cargo

    board_userled: LED 1 set to 1
    board_userled: LED 1 set to 0
    ```

    NuttX blinks the __Emulated LED__ on QEMU Emulator!

    [(See the __Complete Log__)](https://gist.github.com/lupyuen/4321601a962589b531bf83b0032a94af#file-hello_rust_cargo-log-L496-L533)

_How to code Rust Apps for NuttX?_

We could open the __`apps`__ folder in VSCode, but __Rust Analyzer__ won't work.

Do this instead: _VSCode > File > Open Folder > apps/examples/rust/hello_. Then Rust Analyzer [__will work perfectly__](https://lupyuen.github.io/images/rust7-vscode2.png).

__cargo build__ seems to work, __cargo run__ won't. Remember to run [__cargo clippy__](https://doc.rust-lang.org/clippy/index.html)...

```bash
$ cargo clippy
Checking hello v0.1.0 (apps/examples/rust/hello)
Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.38s
```

Let's talk about Owned File Descriptors vs Raw File Descriptors...

![Owned File Descriptors vs Raw File Descriptors](https://lupyuen.github.io/images/rust7-fd2.jpg)

# Owned File Descriptors

__Safety Quiz:__ Why will this run OK...

```rust
// Copied from above: Open the LED Device
let owned_fd =
  open("/dev/userleds", ...)
  .unwrap();  // Returns an Owned File Descriptor

// Copied from above: Set the LEDs via ioctl()
led_set_all(
  owned_fd.as_raw_fd(),  // Extract the Raw File Descriptor
  1                      // Flip LED 1 to On
).unwrap();              // Yep runs OK
```

But __Not This__? (Pic above)

```rust
// Extract earlier the Raw File Descriptor (from the LED Device)
let raw_fd =
  open("/dev/userleds", ...)  // Open the LED Device
  .unwrap()      // Get the Owned File Descriptor
  .as_raw_fd();  // Which becomes a Raw File Descriptor

// Set the LEDs via ioctl()
led_set_all(
  raw_fd,    // Use the earlier Raw File Descriptor
  1          // Flip LED 1 to On
).unwrap();  // Oops will fail!
```

The Second Snippet will fail with __EBADF Error__...

```bash
nsh> hello_rust_cargo
thread '<unnamed>' panicked at src/lib.rs:32:33:
called `Result::unwrap()` on an `Err` value: EBADF
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

There's something odd about __Raw File Descriptors__ vs __Owned File Descriptors__... Fetching the Raw One too early might cause __EBADF Errors__. Here's why...

_What's a Raw File Descriptor?_

In NuttX and POSIX: [__Raw File Descriptor__](https://github.com/apache/nuttx/blob/master/include/stdio.h#L65-L71) is a __Plain Integer__ that specifies an I/O Stream...

|File Descriptor|I/O Stream|
|:---:|:----|
| 0 | Standard Input
| 1 | Standard Output
| 2 | Standard Error
| 3 | /dev/userleds <br> _(assuming we opened it)_

_What about Owned File Descriptor?_

In Rust: [__Owned File Descriptor__](https://doc.rust-lang.org/std/os/fd/struct.OwnedFd.html) is a __Rust Object__, wrapped around a Raw File Descriptor.

And Rust Objects shall be __Automatically Dropped__, when they go out of scope. (Unlike Integers)

_Causing the Second Snippet to fail?_

Exactly! _open()_ returns an __Owned File Descriptor__...

```rust
// Owned File Descriptor becomes Raw File Descriptor
let raw_fd =
  open("/dev/userleds", ...)  // Open the LED Device
  .unwrap()      // Get the Owned File Descriptor
  .as_raw_fd();  // Which becomes a Raw File Descriptor
```

And we turned it into __Raw File Descriptor__. (The Plain Integer, not the Rust Object)

Oops! Our Owned File Descriptor goes __Out Of Scope__ and gets dropped by Rust...

![Our Owned File Descriptor goes Out Of Scope and gets dropped by Rust](https://lupyuen.github.io/images/rust7-fd.jpg)

Thus Rust will helpfully close _/dev/userleds_. Since it's closed, our Raw File Descriptor __becomes invalid__...

```rust
// Set the LEDs via ioctl()
led_set_all(
  raw_fd,    // Use the (closed) Raw File Descriptor
  1          // Flip LED 1 to On
).unwrap();  // Oops will fail with EBADF Error!
```

Resulting in the [__EBADF Error__](https://man.freebsd.org/cgi/man.cgi?errno(2)). _ioctl()_ failed because _/dev/userleds_ is already closed!

__Lesson Learnt:__ Be careful with Owned File Descriptors. They're super helpful for Auto-Closing our files. But might have strange consequences.

Rustix is another popular POSIX Wrapper. We take a peek...

![Nix vs Rustix](https://lupyuen.github.io/images/rust7-compare.png)

# Nix vs Rustix

_Is there a Safer Way to call ioctl()?_

Calling _ioctl()_ from Rust will surely get messy: It's an __Unsafe Call__ that might cause bad writes into the NuttX Kernel! _(If we're not careful)_

At the top of the article, we saw __`nix`__ crate calling _ioctl()_. Now we look at [__Rustix__](https://crates.io/crates/rustix) calling _ioctl()_: [rustix/fs/ioctl.rs](https://github.com/bytecodealliance/rustix/blob/main/src/fs/ioctl.rs#L16-L32)

```rust
// Let's implement ioctl(fd, BLKSSZGET, &output)
// In Rustix: ioctl() is also unsafe
unsafe {
  // Create an "Ioctl Getter"
  // That will read data thru ioctl()
  let ctl = ioctl::Getter::<  // Ioctl Getter has 2 attributes...
    ioctl::BadOpcode<  // Attribute #1: Ioctl Command Code
      { c::BLKSSZGET } // Which is "Fetch the Logical Block Size of a Block Device"
    >,
    c::c_uint  // Attribute #2: Ioctl Getter will read a uint32_t thru ioctl()
  >::new();    // Create the Ioctl Getter

  // Now that we have the Ioctl Getter
  // We call ioctl() on the File Descriptor
  // Equivalent to: ioctl(fd, BLKSSZGET, &output) ; return output
  ioctl::ioctl(
    fd,  // Borrowed File Descriptor (safer than Raw)
    ctl  // Ioctl Getter
  ) // Returns the Value Read (Or Error)
}
```

[(Based on __Rustix Docs__)](https://docs.rs/rustix/latest/rustix/ioctl/index.html)

[(Rustix Ioctl passes a __Borrowed File Descriptor__, safer than Raw)](https://docs.rs/rustix/latest/rustix/ioctl/fn.ioctl.html)

_Nix vs Rustix: They feel quite similar?_

Actually Nix was previously a lot simpler, supporting only __Raw File Descriptors__. _(Instead of Owned File Descriptors)_

Today, Nix is moving to __Owned File Descriptors__ due to __I/O Safety__. Bummer it means Nix is becoming more [__Rustix-like__](https://crates.io/crates/rustix)...

- [__Nix: Implement I/O Safety__](https://github.com/nix-rust/nix/issues/1750)

- [__Rust: I/O Safety__](https://rust-lang.github.io/rfcs/3128-io-safety.html) _(used in Rustix and New Nix)_

_What's our preference: Nix or Rustix?_

Hmmm we're still pondering. __Rustix is newer__ (pic above), but it's also __more complex__ (based on Lines of Code). It might hinder our porting to NuttX.

Which would you choose? Lemme know! 🙏

![Nix vs Rustix: Lines of Code](https://lupyuen.github.io/images/rust7-loc.png)

[(__Rustix on NuttX:__ Will it run? Nope not yet)](https://lupyuen.github.io/articles/rust7#appendix-porting-rustix-to-nuttx)

[(__no1wudi/nuttx-rs__ shows potential)](https://github.com/no1wudi/nuttx-rs)

[(__Rust Embedded HAL__ might be a bad fit)](https://lupyuen.github.io/articles/rust6#appendix-nuttx-vs-rust-embedded-hal)

# What's Next

![Upcoming: Slint Rust GUI for NuttX 🎉](https://lupyuen.github.io/images/rust7-slint.jpg)

[__Upcoming:__ Slint Rust GUI for NuttX 🎉](https://github.com/apache/nuttx-apps/pull/2967)

_What platforms are supported for NuttX + Rust Standard Library? How about SBCs?_

Arm and RISC-V (32-bit and 64-bit). [__Check this doc__](https://nuttx.apache.org/docs/latest/guides/rust.html) for updates.

Sorry 64-bit __RISC-V Kernel Build__ is [__not supported yet__](https://github.com/apache/nuttx-apps/pull/2487#issuecomment-2601488835). So it __won't run on RISC-V SBCs__ like Ox64 BL808 and Oz64 SG2000.

_Sounds like we need plenty of Rust Testing? For every NuttX Platform?_

Yeah maybe we need [__Daily Automated Testing__](https://lupyuen.github.io/articles/rust6#appendix-daily-test-of-nuttx-qemu-risc-v) of NuttX + Rust Standard Library on [__NuttX Build Farm__](https://lupyuen.github.io/articles/ci4.html)?

With [__QEMU Emulator__](https://lupyuen.github.io/articles/rust6#appendix-daily-test-of-nuttx-qemu-risc-v) or a [__Real Device__](https://lupyuen.github.io/articles/sg2000a)?

And when the Daily Test fails: How to [__Auto-Rewind the Build__](https://lupyuen.github.io/articles/ci6.html) and discover the Breaking Commit? Hmmm...

<hr>

Many Thanks to the awesome __NuttX Admins__ and __NuttX Devs__! And [__My Sponsors__](https://lupyuen.org/articles/sponsor), for sticking with me all these years.

- [__Sponsor me a coffee__](https://lupyuen.org/articles/sponsor)

- [__Discuss this article on Hacker News__](https://news.ycombinator.com/item?id=42825198)

- [__My Current Project: "Apache NuttX RTOS for Sophgo SG2000"__](https://nuttx-forge.org/lupyuen/nuttx-sg2000)

- [__My Other Project: "NuttX for Ox64 BL808"__](https://nuttx-forge.org/lupyuen/nuttx-ox64)

- [__Older Project: "NuttX for Star64 JH7110"__](https://nuttx-forge.org/lupyuen/nuttx-star64)

- [__Olderer Project: "NuttX for PinePhone"__](https://nuttx-forge.org/lupyuen/pinephone-nuttx)

- [__Check out my articles__](https://lupyuen.org)

- [__RSS Feed__](https://lupyuen.org/rss.xml)

_Got a question, comment or suggestion? Create an Issue or submit a Pull Request here..._

[__lupyuen.org/src/rust7.md__](https://codeberg.org/lupyuen/lupyuen.org/src/branch/master/src/rust7.md)

![NuttX with Rust Standard Library](https://lupyuen.github.io/images/rust7-vscode.png)

# Appendix: Build NuttX for Rust Standard Library

Follow these steps to build __NuttX bundled with Rust Standard Library__...

[(Based on the __Official Doc__)](https://nuttx.apache.org/docs/latest/guides/rust.html)

[(Remember to install __RISC-V Toolchain__ and __RISC-V QEMU__)](https://nuttx.apache.org/docs/latest/platforms/risc-v/qemu-rv/boards/rv-virt/index.html)

```bash
## Install Rust: https://rustup.rs/
## Select "Standard Installation"
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
. "$HOME/.cargo/env"

## Switch to the Nightly Toolchain
rustup update
rustup toolchain install nightly
rustup default nightly

## Should show `rustc 1.86.0-nightly` or later
rustc --version

## Install the Nightly Toolchain
rustup component add rust-src --toolchain nightly-x86_64-unknown-linux-gnu
## For macOS: rustup component add rust-src --toolchain nightly-aarch64-apple-darwin

## Download the NuttX Kernel and Apps
git clone https://github.com/apache/nuttx
git clone https://github.com/apache/nuttx-apps apps
cd nuttx

## Configure NuttX for RISC-V 64-bit QEMU with LEDs
## (Alternatively: rv-virt:nsh64 or rv-virt:nsh or rv-virt:leds)
tools/configure.sh rv-virt:leds64

## Disable Floating Point: CONFIG_ARCH_FPU
kconfig-tweak --disable CONFIG_ARCH_FPU

## Enable CONFIG_SYSTEM_TIME64 / CONFIG_FS_LARGEFILE / CONFIG_DEV_URANDOM / CONFIG_TLS_NELEM = 16
kconfig-tweak --enable CONFIG_SYSTEM_TIME64
kconfig-tweak --enable CONFIG_FS_LARGEFILE
kconfig-tweak --enable CONFIG_DEV_URANDOM
kconfig-tweak --set-val CONFIG_TLS_NELEM 16

## Enable the Hello Rust Cargo App
## Increase the App Stack Size from 2 KB to 16 KB (especially for 64-bit platforms)
kconfig-tweak --enable CONFIG_EXAMPLES_HELLO_RUST_CARGO
kconfig-tweak --set-val CONFIG_EXAMPLES_HELLO_RUST_CARGO_STACKSIZE 16384

## Update the Kconfig Dependencies
make olddefconfig

## Build NuttX
make -j

## If it fails with "Mismatched Types":
## Patch the file `fs.rs` (see below)

## Start NuttX on QEMU RISC-V 64-bit
qemu-system-riscv64 \
  -semihosting \
  -M virt,aclint=on \
  -cpu rv64 \
  -bios none \
  -kernel nuttx \
  -nographic

## Inside QEMU: Run our Hello Rust App
hello_rust_cargo
```

We'll see this in __QEMU RISC-V Emulator__...

```bash
NuttShell (NSH) NuttX-12.8.0
nsh> hello_rust_cargo

{"name":"John","age":30}
{"name":"Jane","age":25}
Deserialized: Alice is 28 years old
Pretty JSON:
{
  "name": "Alice",
  "age": 28
}
Hello world from tokio!
```

__To Quit QEMU:__ Press __`Ctrl-a`__ then __`x`__

[(See the __Ubuntu Build Log__)](https://gist.github.com/lupyuen/6985933271f140db0dc6172ebba9bff5)

[(See the __macOS Build Log__)](https://gist.github.com/lupyuen/a2b91b5cc15824a31c287fbb6cda5fa2)

[(Also works for 32-bit __rv-virt:leds__)](https://gist.github.com/lupyuen/ccfae733657b864f2f9a24ce41808144)

<hr>

__Troubleshooting The Rust Build__

- If NuttX Build fails with __"Mismatched Types"__...

  <span style="font-size:80%">

  ```bash
  Compiling std v0.0.0 (.rustup/toolchains/nightly-x86_64-unknown-linux-gnu/lib/rustlib/src/rust/library/std)
  error[E0308]: mismatched types
      --> .rustup/toolchains/nightly-x86_64-unknown-linux-gnu/lib/rustlib/src/rust/library/std/src/sys/pal/unix/fs.rs:1037:33
  1037 |         unsafe { CStr::from_ptr(self.entry.d_name.as_ptr()) }
       |                  -------------- ^^^^^^^^^^^^^^^^^^^^^^^^^^ expected `*const u8`, found `*const i8`
       |                  |
       |                  arguments to this function are incorrect
       = note: expected raw pointer `*const u8`
                  found raw pointer `*const i8`
  note: associated function defined here
      --> .rustup/toolchains/nightly-x86_64-unknown-linux-gnu/lib/rustlib/src/rust/library/core/src/ffi/c_str.rs:264:25
  264  |     pub const unsafe fn from_ptr<'a>(ptr: *const c_char) -> &'a CStr {
       |                         ^^^^^^^^
  ```

  </span>

  Then edit this file...

  <span style="font-size:80%">

  ```bash
  ## For Ubuntu
  $HOME/.rustup/toolchains/nightly-x86_64-unknown-linux-gnu/lib/rustlib/src/rust/library/std/src/sys/pal/unix/fs.rs

  ## For macOS
  $HOME/.rustup/toolchains/nightly-aarch64-apple-darwin/lib/rustlib/src/rust/library/std/src/sys/pal/unix/fs.rs
  ```

  </span>

  Change the __name_cstr__ function at __Line 1036__...

  <span style="font-size:80%">

  ```rust
      fn name_cstr(&self) -> &CStr {
          unsafe { CStr::from_ptr(self.entry.d_name.as_ptr()) }
      }
  ```

  </span>

  To this...

  <span style="font-size:80%">

  ```rust
      fn name_cstr(&self) -> &CStr {
          unsafe { CStr::from_ptr(self.entry.d_name.as_ptr() as *const u8) }
      }
  ```

  </span>

  And verify the change...

  <span style="font-size:80%">

  ```bash
  ## For Ubuntu
  head -n 1049 $HOME/.rustup/toolchains/nightly-x86_64-unknown-linux-gnu/lib/rustlib/src/rust/library/std/src/sys/pal/unix/fs.rs \
    | tail -n 17

  ## For macOS
  head -n 1049 $HOME/.rustup/toolchains/nightly-aarch64-apple-darwin/lib/rustlib/src/rust/library/std/src/sys/pal/unix/fs.rs \
    | tail -n 17

  ## We should see
  ## fn name_cstr(&self) -> &CStr {
  ##   unsafe { CStr::from_ptr(self.entry.d_name.as_ptr() as *const u8) }
  ```

  </span>

  Finally rebuild with `make -j`

  [(Will be fixed in __Rust Toolchain__)](https://github.com/rust-lang/libc/pull/4222)

- If the build fails with __"-Z" Error__...

  ```bash
  error: the `-Z` flag is only accepted on the nightly channel of Cargo
  but this is the `stable` channel
  ```

  Then switch to the Nightly Toolchain...

  ```bash
  ## Switch to the Nightly Toolchain
  rustup update
  rustup toolchain install nightly
  rustup default nightly

  ## Should show `rustc 1.86.0-nightly` or later
  rustc --version
  ```

- If the build fails with __"Unable to build with the Standard Library"__...

  ```bash
  error: ".rustup/toolchains/nightly-x86_64-unknown-linux-gnu/lib/rustlib/src/rust/library/Cargo.lock" does not exist, unable to build with the standard library
  try: rustup component add rust-src --toolchain nightly-x86_64-unknown-linux-gnu
  ```

  Then install the Nightly Toolchain...

  ```bash
  ## Install the Nightly Toolchain
  rustup component add rust-src --toolchain nightly-x86_64-unknown-linux-gnu
  ## For macOS: rustup component add rust-src --toolchain nightly-aarch64-apple-darwin
  ```

- If the build fails with __"Error Loading Target"__...

  ```bash
  error: Error loading target specification:
  Could not find specification for target "riscv64imafdc-unknown-nuttx-elf"
  ```

  Then disable Floating Point...

  ```bash
  ## Disable Floating Point: CONFIG_ARCH_FPU
  kconfig-tweak --disable CONFIG_ARCH_FPU

  ## Update the Kconfig Dependencies
  make olddefconfig
  make -j
  ```

- _What if we're using Rust already? And we don't wish to change the Default Toolchain?_

  Use __rustup override__ to __Override the Folder Toolchain__. Do it in the __Parent Folder__ of __`nuttx`__ and __`apps`__...

  ```bash
  ## Set Rust to Nightly Build
  ## Apply this to the Parent Folder
  ## So it will work for `nuttx` and `apps`
  pushd ..
  rustup override list
  rustup override set nightly
  rustup override list
  popd
  ```

- _Rust App crashes in QEMU?_

  We might see a Stack Dump that __Loops Forever__. Or we might see __100% Full__ for the App Stack...

  <span style="font-size:80%">

  ```bash
  PID GROUP PRI POLICY   TYPE    NPX STATE   EVENT      SIGMASK          STACKBASE  STACKSIZE      USED   FILLED    COMMAND
    3     3 100 RR       Task    -   Running            0000000000000000 0x80071420      1856      1856   100.0%!   hello_rust_cargo
  ```

  </span>

  Then increase the App Stack Size...

  ```bash
  ## Increase the App Stack Size to 64 KB
  kconfig-tweak --set-val \
    CONFIG_EXAMPLES_HELLO_RUST_CARGO_STACKSIZE \
    65536

  ## Update the Kconfig Dependencies and rebuild
  make olddefconfig
  make -j
  ```

- _Rust Build seems to break sometimes?_

  We might need to clean up the __Rust Target Files__, if the Rust Build goes wonky...

  ```bash
  ## Erase the Rust Build and rebuild
  pushd ../apps/examples/rust/hello
  cargo clean
  popd
  make -j
  ```

- _How to code Rust Apps for NuttX?_

  We could open the __`apps`__ folder in VSCode, but __Rust Analyzer__ won't work.

  Do this instead: _VSCode > File > Open Folder > apps/examples/rust/hello_. Then Rust Analyzer will work perfectly. (Pic below)

  __cargo build__ seems to work, __cargo run__ won't. Remember to run [__cargo clippy__](https://doc.rust-lang.org/clippy/index.html)...

  ```bash
  $ cargo clippy
  Checking hello v0.1.0 (apps/examples/rust/hello)
  Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.38s
  ```

How did we port Rust Standard Library to NuttX? Details here...

- [__examples: New app to build Rust with Cargo__](https://github.com/apache/nuttx-apps/pull/2487)

- [__Rust: Add NuttX based targets for RISC-V and ARM__](https://github.com/rust-lang/rust/pull/127755)

![Tokio Async Threading](https://lupyuen.github.io/images/rust7-vscode2.png)

# Appendix: Tokio Async Threading

Earlier we saw Tokio's __Single-Threaded Scheduler__, running on the __Current Thread__...

- [__"Async Functions with Tokio"__](https://lupyuen.github.io/articles/rust7#async-functions-with-tokio)

```rust
// Use One Single Thread (Current Thread)
// To schedule Async Functions
tokio::runtime::Builder
  ::new_current_thread()  // Current Thread is the Single-Threaded Scheduler
  .enable_all()  // Enable the I/O and Time Functions
  .build()   // Create the Single-Threaded Scheduler
  .unwrap()  // Halt on Error
  .block_on( // Start the Scheduler
    async {  // With this Async Code
      println!("Hello world from tokio!");
  });

// Is it really async? Let's block and find out!
println!("Looping Forever...");
loop {}
```

And it ain't terribly exciting...

```bash
nsh> hello_rust_cargo
Hello world from tokio!
Looping Forever...
```

Now we try Tokio's __Multi-Threaded Scheduler__. And we create __One New POSIX Thread__ for the Scheduler: [wip-nuttx-apps/lib.rs](https://github.com/lupyuen2/wip-nuttx-apps/blob/rust-std/examples/rust/hello/src/lib.rs#L86-L140)

```rust
// Run 4 Async Functions in the Background
// By creating One New POSIX Thread
// Based on https://tokio.rs/tokio/topics/bridging
fn test_async() {

  // Create a Multi-Threaded Scheduler
  // Containing One New POSIX Thread
  let runtime = tokio::runtime::Builder
    ::new_multi_thread() // Multi-Threaded Scheduler
    .worker_threads(1)   // With One New POSIX Thread for our Scheduler
    .enable_all() // Enable the I/O and Time Functions
    .build()      // Create the Multi-Threaded Scheduler
    .unwrap();    // Halt on Error

  // Create 4 Async Functions
  // Remember their Async Handles
  let mut handles = Vec::with_capacity(4);
  for i in 0..4 {
    handles.push(        // Remember the Async Handles
      runtime.spawn(     // Start in the Background
        my_bg_task(i))); // Our Async Function
  }

  // Pretend to be busy while Async Functions execute (in the background)
  // We wait 750 milliseconds
  std::thread::sleep(
    tokio::time::Duration::from_millis(750));
  println!("Finished time-consuming task.");

  // Wait for All Async Functions to complete
  for handle in handles {
    runtime
      .block_on(handle)  // Wait for One Async Function to complete
      .unwrap();
  }
}

// Our Async Function that runs in the background...
// If i=0: Sleep for 1000 ms
// If i=1: Sleep for  950 ms
// If i=2: Sleep for  900 ms
// If i=3: Sleep for  850 ms
async fn my_bg_task(i: u64) {
  let millis = 1000 - 50 * i;
  println!("Task {} sleeping for {} ms.", i, millis);
  tokio::time::sleep(
    tokio::time::Duration::from_millis(millis)
  ).await;  // Wait for sleep to complete
  println!("Task {} stopping.", i);
}

// Needed by Tokio Multi-Threaded Scheduler
#[no_mangle]
pub extern "C" fn pthread_set_name_np() {}
```

_How to run the Tokio Demo?_

1.  Copy the __Tokio Demo Files__ from here...

    [_lupyuen2/wip-nuttx-apps/examples/rust/hello_](https://github.com/lupyuen2/wip-nuttx-apps/tree/rust-std/examples/rust/hello)

    Specifically: [__Cargo.toml__](https://github.com/lupyuen2/wip-nuttx-apps/blob/rust-std/examples/rust/hello/Cargo.toml) and [__src/lib.rs__](https://github.com/lupyuen2/wip-nuttx-apps/blob/rust-std/examples/rust/hello/src/lib.rs)

1.  Overwrite our __Rust Hello App__...

    _apps/examples/rust/hello_

1.  [Rebuild our __NuttX Project__](https://lupyuen.github.io/articles/rust7#appendix-build-nuttx-for-rust-standard-library)

    ```bash
    make -j
    ```

1.  Then run it with __QEMU RISC-V Emulator__

    ```bash
    $ qemu-system-riscv64 \
      -semihosting \
      -M virt,aclint=on \
      -cpu rv64 \
      -bios none \
      -kernel nuttx \
      -nographic

    NuttShell (NSH) NuttX-12.8.0
    nsh> hello_rust_cargo

1.  We'll see __Four Async Functions__, running on __One New POSIX Thread__...

    ```bash
    nsh> hello_rust_cargo
    pthread_create
    nx_pthread_create

    Task 0 sleeping for 1000 ms
    Task 1 sleeping for  950 ms
    Task 2 sleeping for  900 ms
    Task 3 sleeping for  850 ms

    Finished time-consuming task
    Task 3 stopping
    Task 2 stopping
    Task 1 stopping
    Task 0 stopping
    ```

    [(See the __Complete Log__)](https://gist.github.com/lupyuen/46db6d1baee0e589774cc43dd690da07#file-gistfile1-txt-L497-L535)

    [(Explained in __Tokio Docs__)](https://tokio.rs/tokio/topics/bridging)

1.  See the call to [__pthread_create__](https://github.com/apache/nuttx/blob/master/libs/libc/pthread/pthread_create.c#L88), which calls [__nx_pthread_create__](https://github.com/apache/nuttx/blob/master/sched/pthread/pthread_create.c#L179)? It means that Tokio is actually calling NuttX to create One POSIX Thread! (For the Multi-Threaded Scheduler)

1.  Yep it's consistent with our __Reverse Engineering of Tokio__...

    [__"Snooping Tokio on NuttX"__](https://lupyuen.github.io/articles/rust7#appendix-snooping-tokio-on-nuttx)

_What if we increase the Worker Threads? From 1 to 2?_

```rust
// Two Worker Threads instead of One
let runtime = tokio::runtime::Builder
  ::new_multi_thread() // New Multi-Threaded Scheduler
  .worker_threads(2)   // With Two New POSIX Threads for our Scheduler
```

The output looks exactly the same...

```bash
nsh> hello_rust_cargo
pthread_create
nx_pthread_create
pthread_create
nx_pthread_create

Task 0 sleeping for 1000 ms
Task 1 sleeping for  950 ms
Task 2 sleeping for  900 ms
Task 3 sleeping for  850 ms

Finished time-consuming task
Task 3 stopping
Task 2 stopping
Task 1 stopping
Task 0 stopping
```

Except that we see __Two Calls__ to [__pthread_create__](https://github.com/apache/nuttx/blob/master/libs/libc/pthread/pthread_create.c#L88) and [__nx_pthread_create__](https://github.com/apache/nuttx/blob/master/sched/pthread/pthread_create.c#L179). Tokio called NuttX to create Two POSIX Threads. (For the Multi-Threaded Scheduler)

_How did we log pthread_create?_

Inside NuttX Kernel: We added Debug Code to 
[__pthread_create__](https://github.com/apache/nuttx/blob/master/libs/libc/pthread/pthread_create.c#L88) and [__nx_pthread_create__](https://github.com/apache/nuttx/blob/master/sched/pthread/pthread_create.c#L179)

<span style="font-size:90%">

```c
// At https://github.com/apache/nuttx/blob/master/libs/libc/pthread/pthread_create.c#L88
#include <debug.h>
int pthread_create(...) {
  _info("pthread_entry=%p, arg=%p", pthread_entry, arg);

// At https://github.com/apache/nuttx/blob/master/sched/pthread/pthread_create.c#L179
#include <debug.h>
int nx_pthread_create(...) {
  _info("entry=%p, arg=%p", entry, arg);
```

</span>

![LED Blinky with Rust Standard Library on Apache NuttX RTOS (RustRover IDE)](https://lupyuen.github.io/images/rust7-title.png)

# Appendix: Porting Nix to NuttX

_What happens when we call nix crate as-is on NuttX?_

Earlier we said that we [__Customised the `nix` Crate__](https://lupyuen.github.io/articles/rust7#led-blinky-with-nix) to run on NuttX.

Why? Let's build our Rust Blinky App with the Original __`nix`__ Crate...

<span style="font-size:80%">

```bash
$ pushd ../apps/examples/rust/hello
$ cargo add nix --features fs,ioctl

Adding nix v0.29.0 to dependencies
Features: + fs + ioctl
33 deactivated features

$ popd
$ make -j

error[E0432]: unresolved import `self::const`
  -->   errno.rs:19:15
19 | pub use self::consts::*;
   |               ^^^^^^ could not find `consts` in `self`

error[E0432]: unresolved import `self::Errno`
   -->  errno.rs:198:15
198 |     use self::Errno::*;
    |               ^^^^^ could not find `Errno` in `self`

error[E0432]: unresolved import `crate::errno::Errno`
 -->  fcntl.rs:2:5
2 | use crate::errno::Errno;
  |     ^^^^^^^^^^^^^^-----
  |     no `Errno` in `errno`
```

</span>

Plus many errors. That's why we [__Customised the `nix` Crate__](https://github.com/lupyuen/nix/tree/nuttx) for NuttX...

```bash
$ cd ../apps/examples/rust/hello
$ cargo add nix \
  --features fs,ioctl \
  --git https://github.com/lupyuen/nix.git \
  --branch nuttx

Updating git repository `https://github.com/lupyuen/nix.git`
Adding nix (git) to dependencies
Features: + fs + ioctl
34 deactivated features
```

Here's how...

1.  __For Easier Porting:__ We cloned __`nix`__ locally...

    ```bash
    git clone \
      https://github.com/lupyuen/nix \
      --branch nuttx
    cd ../apps/examples/rust/hello
    cargo add nix \
      --features fs,ioctl \
      --path $HOME/nix
    ```

1.  We extended [__errno.rs__](https://github.com/lupyuen/nix/pull/1/files#diff-c64965cf18ab089e705398a750edb9b349ff3e0509454d801d6a150db7ff9b5e), copying the __FreeBSD Section__ _[cfg(target_os = "freebsd")]_ to __NuttX Section__ _[cfg(target_os = "nuttx")]_.

    (We removed the bits that don't exist on NuttX)

1.  NuttX seems to have a similar POSIX Profile to __Redox OS__? We changed plenty of code to look like this: [__sys/time.rs__](https://github.com/lupyuen/nix/pull/1/files#diff-7f322738311de78991dc089e6bcd3a89bcebc6d39b1a17508cf6c94bb170c9b0)

    ```rust
    // NuttX works like Redox OS
    #[cfg(not(any(target_os = "redox",
                  target_os = "nuttx")))]
    pub const UTIME_OMIT: TimeSpec = ...
    ```

1.  __For NuttX ioctl():__ It works more like BSD _(second parameter is __`int`__)_ than Linux _(second parameter is __`long`__)_: [sys/ioctl/mod.rs](https://github.com/lupyuen/nix/pull/1/files#diff-96785c020c81b7d3962a7ea3c4ec2f2b1388617a412c92b4d1f0437447f42af4)

    ```rust
    // NuttX ioctl() works like BSD
    #[cfg(any(bsd,
              solarish,
              target_os = "haiku",
              target_os = "nuttx"))]
    #[macro_use]
    mod bsd;

    // Nope, NuttX ioctl() does NOT work like Linux
    #[cfg(any(linux_android,
              target_os = "fuchsia",
              target_os = "redox"))]
    #[macro_use]
    mod linux;
    ```

1.  Here are the files we modified for NuttX...
    
    (Supporting __`fs`__ and __`ioctl`__ features only)

    [__All Modified Files__](https://github.com/lupyuen/nix/pull/1/files)

    [__errno.rs__](https://github.com/lupyuen/nix/pull/1/files#diff-c64965cf18ab089e705398a750edb9b349ff3e0509454d801d6a150db7ff9b5e)

    [__fcntl.rs__](https://github.com/lupyuen/nix/pull/1/files#diff-234e7e6580542ac96403821955043ffefa4cef1e0659216a9ee170cad6315c7d)

    [__unistd.rs__](https://github.com/lupyuen/nix/pull/1/files#diff-0223913fb22a7da0dcb64a51b192e5c049b4b276351c83bbaeb0cee0dbbd8a04)

    [__sys/stat.rs__](https://github.com/lupyuen/nix/pull/1/files#diff-5c119a000c85b1959421747235c671cc2f43b4f5fd2628daf1276f684a100ad8)

    [__sys/statvfs.rs__](https://github.com/lupyuen/nix/pull/1/files#diff-ed80a57034c9c336fb4516644f86cbd9ef75296fa76bdf9c7ca9adf251be0421)

    [__sys/mod.rs__](https://github.com/lupyuen/nix/pull/1/files#diff-db4000d9e8bf29c6719984245eeefdf7e0a9b4e525f37ac8c5d6a918d4dc3005)

    [__sys/time.rs__](https://github.com/lupyuen/nix/pull/1/files#diff-7f322738311de78991dc089e6bcd3a89bcebc6d39b1a17508cf6c94bb170c9b0)

    [__sys/ioctl/bsd.rs__](https://github.com/lupyuen/nix/pull/1/files#diff-48ef2619f99fe3916c145e82b718b5f2975d58992113203c51fb4315d8e3155b)

    [__sys/ioctl/mod.rs__](https://github.com/lupyuen/nix/pull/1/files#diff-96785c020c81b7d3962a7ea3c4ec2f2b1388617a412c92b4d1f0437447f42af4)

<hr>

__Troubleshooting nix ioctl() on NuttX__

To figure out if __`nix`__ passes ioctl() parameters correctly to NuttX: We insert __Ioctl Debug Code__ into NuttX Kernel...

```c
// At https://github.com/apache/nuttx/blob/master/fs/vfs/fs_ioctl.c#L261
#include <debug.h>
int ioctl(int fd, int req, ...) {
  _info("fd=0x%x, req=0x%x", fd, req);
```

Which [__Ioctl Macro__](https://docs.rs/nix/latest/nix/sys/ioctl/) shall we call in __`nix`__? We tried __ioctl_none__...

```rust
const ULEDIOC_SETALL: i32 = 0x1d03;
ioctl_none!(led_on, ULEDIOC_SETALL, 1);
unsafe { led_on(fd).unwrap(); }
```

But the __Ioctl Command Code__ got mangled up (`0x201d0301` should be `0x1d03`)

```bash
NuttShell (NSH) NuttX-12.8.0
nsh> hello_rust_cargo
fd=3
ioctl: fd=0x3, req=0x201d0301

thread '<unnamed>' panicked at src/lib.rs:31:25:
called `Result::unwrap()` on an `Err` value: ENOTTY
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

Then we tried __ioctl_write_int__...

```rust
const ULEDIOC_SETALL: i32 = 0x1d03;
ioctl_write_int!(led_on, ULEDIOC_SETALL, 1);
unsafe { led_on(fd, 1).unwrap(); }
```

Nope the __Ioctl Command Code__ is still mangled (`0x801d0301` should be `0x1d03`)

```bash
nsh> hello_rust_cargo
ioctl: fd=0x3, req=0x801d0301
thread '<unnamed>' panicked at src/lib.rs:30:28:
called `Result::unwrap()` on an `Err` value: ENOTTY
```

Finally this works: __ioctl_write_int_bad__...

```rust
const ULEDIOC_SETALL: i32 = 0x1d03;
ioctl_write_int_bad!(led_set_all, ULEDIOC_SETALL);

// Equivalent to ioctl(fd, ULEDIOC_SETALL, 1)
unsafe { led_set_all(fd, 1).unwrap(); }

// Equivalent to ioctl(fd, ULEDIOC_SETALL, 0)
unsafe { led_set_all(fd, 0).unwrap(); }
```

__Ioctl Command Code__ `0x1d03` is hunky dory yay!

```bash
NuttShell (NSH) NuttX-12.8.0
nsh> hello_rust_cargo
fd=3
ioctl: fd=0x3, req=0x1d03
board_userled: LED 1 set to 1
board_userled: LED 2 set to 0
board_userled: LED 3 set to 0

ioctl: fd=0x3, req=0x1d03
board_userled: LED 1 set to 0
board_userled: LED 2 set to 0
board_userled: LED 3 set to 0
```

![Nix vs Rustix](https://lupyuen.github.io/images/rust7-compare.png)

# Appendix: Porting Rustix to NuttX

_Will Rustix run on NuttX?_

Nope not yet...

```bash
$ cd ../apps/examples/rust/hello
$ cargo add rustix \
  --features fs \
  --git https://github.com/lupyuen/rustix.git \
  --branch nuttx

Updating git repository `https://github.com/lupyuen/rustix.git`
Adding rustix (git) to dependencies
Features: + alloc + fs + std + use-libc-auxv
29 deactivated features
```

We tried compiling this code...

```rust
#[no_mangle]
pub extern "C" fn hello_rust_cargo_main() {
  use rustix::fs::{Mode, OFlags};
  let file = rustix::fs::open(
    "/dev/userleds",
    OFlags::WRONLY,
    Mode::empty(),
  )
  .unwrap();
  println!("file={file:?}");
}
```

But it fails...

```bash
error[E0432]: unresolved import `libc::strerror_r`
  --> .cargo/registry/src/index.crates.io-1949cf8c6b5b557f/errno-0.3.10/src/unix.rs:16:33
   |
16 | use libc::{self, c_int, size_t, strerror_r, strlen};
   |                                 ^^^^^^^^^^
   |                                 |
   |                                 no `strerror_r` in the root
   |                                 help: a similar name exists in the module: `strerror`
```

Seems we need to fix __libc::strerror_r__ for NuttX? Or maybe the __errno__ crate.

![Async Functions with Tokio (Helix Editor + Zellij Workspace)](https://lupyuen.github.io/images/rust7-tokio.png)

# Appendix: Snooping Tokio on NuttX

In this section, we discover how __Tokio works under the hood__. Does it really call __POSIX Functions in NuttX__?

First we obtain the __RISC-V Disassembly__ of our NuttX Image, bundled with the Hello Rust App. We trace the NuttX Build: __`make V=1`__

```bash
make distclean
tools/configure.sh rv-virt:leds64

## Disable CONFIG_ARCH_FPU
kconfig-tweak --disable CONFIG_ARCH_FPU

## Enable CONFIG_SYSTEM_TIME64 / CONFIG_FS_LARGEFILE / CONFIG_DEV_URANDOM / CONFIG_TLS_NELEM = 16
kconfig-tweak --enable CONFIG_SYSTEM_TIME64
kconfig-tweak --enable CONFIG_FS_LARGEFILE
kconfig-tweak --enable CONFIG_DEV_URANDOM
kconfig-tweak --set-val CONFIG_TLS_NELEM 16

## Enable Hello Rust Cargo App, increase the Stack Size
kconfig-tweak --enable CONFIG_EXAMPLES_HELLO_RUST_CARGO
kconfig-tweak --set-val CONFIG_EXAMPLES_HELLO_RUST_CARGO_STACKSIZE 16384

## Update the Kconfig Dependencies
make olddefconfig

## Build NuttX with Tracing Enabled
make V=1
```

[(See the __Build Log__)](https://gist.github.com/lupyuen/b8f051c25e872fb8a444559c3dbf6374)

According to the Make Trace: __NuttX Build__ does this...

<span style="font-size:80%">

```bash
## Discard the Rust Debug Symbols
cd apps/examples/rust/hello
cargo build \
  --release \
  -Zbuild-std=std,panic_abort \
  --manifest-path apps/examples/rust/hello/Cargo.toml \
  --target riscv64imac-unknown-nuttx-elf

## Generate the Linker Script
riscv-none-elf-gcc \
  -E \
  -P \
  -x c \
  -isystem nuttx/include \
  -D__NuttX__ \
  -DNDEBUG \
  -D__KERNEL__  \
  -I nuttx/arch/risc-v/src/chip \
  -I nuttx/arch/risc-v/src/common \
  -I nuttx/sched \
  nuttx/boards/risc-v/qemu-rv/rv-virt/scripts/ld.script \
  -o  nuttx/boards/risc-v/qemu-rv/rv-virt/scripts/ld.script.tmp

## Link Rust App into NuttX
riscv-none-elf-ld \
  --entry=__start \
  -melf64lriscv \
  --gc-sections \
  -nostdlib \
  --cref \
  -Map=nuttx/nuttx.map \
  --print-memory-usage \
  -Tnuttx/boards/risc-v/qemu-rv/rv-virt/scripts/ld.script.tmp  \
  -L nuttx/staging \
  -L nuttx/arch/risc-v/src/board  \
  -o nuttx/nuttx   \
  --start-group \
  -lsched \
  -ldrivers \
  -lboards \
  -lc \
  -lmm \
  -larch \
  -lm \
  -lapps \
  -lfs \
  -lbinfmt \
  -lboard xpack-riscv-none-elf-gcc-13.2.0-2/lib/gcc/riscv-none-elf/13.2.0/rv64imac/lp64/libgcc.a apps/examples/rust/hello/target/riscv64imac-unknown-nuttx-elf/release/libhello.a \
  --end-group
```

</span>

Ah NuttX Build calls __cargo build `--`release__, stripping the Debug Symbols. We change it to __cargo build__ and dump the RISC-V Disassembly...

<span style="font-size:80%">

```bash
## Preserve the Rust Debug Symbols
pushd ../apps/examples/rust/hello
cargo build \
  -Zbuild-std=std,panic_abort \
  --manifest-path apps/examples/rust/hello/Cargo.toml \
  --target riscv64imac-unknown-nuttx-elf
popd

## Generate the Linker Script
riscv-none-elf-gcc \
  -E \
  -P \
  -x c \
  -isystem nuttx/include \
  -D__NuttX__ \
  -DNDEBUG \
  -D__KERNEL__  \
  -I nuttx/arch/risc-v/src/chip \
  -I nuttx/arch/risc-v/src/common \
  -I nuttx/sched \
  nuttx/boards/risc-v/qemu-rv/rv-virt/scripts/ld.script \
  -o  nuttx/boards/risc-v/qemu-rv/rv-virt/scripts/ld.script.tmp

## Link Rust App into NuttX
riscv-none-elf-ld \
  --entry=__start \
  -melf64lriscv \
  --gc-sections \
  -nostdlib \
  --cref \
  -Map=nuttx/nuttx.map \
  --print-memory-usage \
  -Tnuttx/boards/risc-v/qemu-rv/rv-virt/scripts/ld.script.tmp  \
  -L nuttx/staging \
  -L nuttx/arch/risc-v/src/board  \
  -o nuttx/nuttx   \
  --start-group \
  -lsched \
  -ldrivers \
  -lboards \
  -lc \
  -lmm \
  -larch \
  -lm \
  -lapps \
  -lfs \
  -lbinfmt \
  -lboard xpack-riscv-none-elf-gcc-13.2.0-2/lib/gcc/riscv-none-elf/13.2.0/rv64imac/lp64/libgcc.a apps/examples/rust/hello/target/riscv64imac-unknown-nuttx-elf/debug/libhello.a \
  --end-group

## Dump the disassembly to nuttx.S
riscv-none-elf-objdump \
  --syms --source --reloc --demangle --line-numbers --wide \
  --debugging \
  nuttx \
  >leds64-debug-nuttx.S \
  2>&1
```

</span>

[(See the __Build Log__)](https://gist.github.com/lupyuen/7b52d54725aaa831cb3dddc0b68bb41f)

Which produces the __Complete NuttX Disassembly__: [__leds64-debug-nuttx.S__](https://github.com/lupyuen2/wip-nuttx/releases/download/rust-std-1/leds64-debug-nuttx.S)

Whoa the Complete NuttX Disassembly is too huge to inspect!

Let's dump the RISC-V Disassembly of the __Rust Part__ only: __libhello.a__

```bash
## Dump the libhello.a disassembly to libhello.S
riscv-none-elf-objdump \
  --syms --source --reloc --demangle --line-numbers --wide \
  --debugging \
  apps/examples/rust/hello/target/riscv64imac-unknown-nuttx-elf/debug/libhello.a \
  >libhello.S \
  2>&1
```

Which produces the (much smaller) __Rust Disassembly__: [__libhello.S__](https://github.com/lupyuen2/wip-nuttx/releases/download/rust-std-1/libhello.S)

Is Tokio calling NuttX to create POSIX Threads? We search [__libhello.S__](https://github.com/lupyuen2/wip-nuttx/releases/download/rust-std-1/libhello.S) for __pthread_create__...

<span style="font-size:80%">

```bash
.rustup/toolchains/nightly-x86_64-unknown-linux-gnu/lib/rustlib/src/rust/library/std/src/sys/pal/unix/thread.rs:85

let ret = libc::pthread_create(&mut native, &attr, thread_start, p as *mut _);

auipc a0, 0x0 122: R_RISCV_PCREL_HI20 std::sys::pal::unix::thread::Thread::new::thread_start
mv    a2, a0 126: R_RISCV_PCREL_LO12_I .Lpcrel_hi254
add   a0, sp, 132
add   a1, sp, 136
sd    a1, 48(sp)
auipc ra, 0x0 130: R_RISCV_CALL_PLT pthread_create
```

</span>

OK that's the [__Rust Standard Library__](https://doc.rust-lang.org/src/std/sys/pal/unix/thread.rs.html#84) calling __pthread_create__ to create a new Rust Thread.

How are __Rust Threads__ created in Rust Standard Library? Like this: [std/thread/mod.rs](https://github.com/rust-lang/rust/blob/master/library/std/src/thread/mod.rs#L502)

```rust
// spawn_unchecked_ creates a new Rust Thread
unsafe fn spawn_unchecked_<'scope, F, T>(
  let my_thread = Thread::new(id, name);
```

And __spawn_unchecked__ is called by Tokio, according to our Rust Disassembly...

<span style="font-size:80%">

```bash
<core::ptr::drop_in_place<std::thread::Builder::spawn_unchecked_::MaybeDangling<tokio::runtime::blocking::pool::Spawner::spawn_thread::{{closure}}>>>:

.rustup/toolchains/nightly-x86_64-unknown-linux-gnu/lib/rustlib/src/rust/library/core/src/ptr/mod.rs:523

add   sp, sp, -16
sd    ra, 8(sp)
sd    a0, 0(sp)
auipc ra, 0x0 6: R_RISCV_CALL_PLT <std::thread::Builder::spawn_unchecked_::MaybeDangling<T> as core::ops::drop::Drop>::drop
```

</span>

Yep it checks out: Tokio calls Rust Standard Library, which calls NuttX to create POSIX Threads!

_Are we sure that Tokio creates a POSIX Thread? Not a NuttX Task?_

We run __`hello_rust_cargo &`__ to put it in the background...

<span style="font-size:80%">

```bash
nsh> hello_rust_cargo &
Hello world from tokio!

nsh> ps
  PID GROUP PRI POLICY   TYPE    NPX STATE    EVENT     SIGMASK            STACK    USED FILLED COMMAND
    0     0   0 FIFO     Kthread   - Ready              0000000000000000 0001904 0000712  37.3%  Idle_Task
    2     2 100 RR       Task      - Running            0000000000000000 0002888 0002472  85.5%! nsh_main
    4     4 100 RR       Task      - Ready              0000000000000000 0007992 0006904  86.3%! hello_rust_cargo
```

</span>

__`ps`__ says that there's only One Single NuttX Task __`hello_rust_cargo`__. And no other NuttX Tasks.

[(See the __Complete Log__)](https://gist.github.com/lupyuen/0377d9e015fee1d6a833c22e1b118961)
