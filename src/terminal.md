# NuttX RTOS for PinePhone: LVGL Terminal for NSH Shell

📝 _3 Feb 2023_

![LVGL Terminal App on PinePhone with Apache NuttX RTOS](https://lupyuen.github.io/images/terminal-title.jpg)

[__Apache NuttX RTOS__](https://lupyuen.github.io/articles/what) (Real-Time Operating System) now boots on [__Pine64 PinePhone__](https://wiki.pine64.org/index.php/PinePhone) and runs __Touchscreen Apps__!

Today we'll look inside a Touchscreen App that will be useful for NuttX Developers... Our __Terminal App for NSH Shell__. (Pic above)

[(Watch the Demo on YouTube)](https://www.youtube.com/watch?v=WdiXaMK8cNw)

_What's NSH Shell?_

[__NuttShell (NSH)__](https://nuttx.apache.org/docs/latest/applications/nsh/index.html) is the Command-Line Interface for NuttX. (Works like a Linux Shell)

Previously we needed a special [__Serial Cable__](https://lupyuen.github.io/articles/lvgl2#appendix-boot-apache-nuttx-rtos-on-pinephone) to access NSH Shell on PinePhone...

Now we can run NSH Commands __through the Touchscreen__! (Pic above)

(Super helpful for testing new NuttX Features on PinePhone!)

Read on to find out how we...

-   __Pipe a Command__ to NSH Shell

-   Poll for __NSH Output__

-   __Render the Terminal__ with LVGL Widgets

-   __Handle Input__ from LVGL Keyboard

-   __Handle Output__ from NSH Shell

And how we might simplify the LVGL coding with the __Zig Programming Language__.

_What's NuttX? Why run it on PinePhone?_

If we're new to NuttX, here's a gentle intro...

-   [__"NuttX RTOS for PinePhone: What is it?"__](https://lupyuen.github.io/articles/what)

![Flow of LVGL Terminal for PinePhone on Apache NuttX RTOS](https://lupyuen.github.io/images/terminal-flow.jpg)

# LVGL Terminal for NuttX

Before we dive in, let's walk through the internals of our __LVGL Terminal App for NuttX__ (pic above)...

1.  We start the __NSH Shell__ as a NuttX Task

    (Which will execute our NSH Commands)

1.  An NSH Command is entered through the __LVGL Keyboard Widget__

    (Which goes to the __Input Text Area Widget__)

    Let's say we type this NSH Command...

    ```text
    ls
    ```

1.  When the Enter Key is pressed, we send the NSH Command __`ls`__ to the __NSH Input Pipe__

1.  Which delivers the NSH Command to the __NSH Shell__

1.  NSH Shell __executes our NSH Command__...

    ```text
    nsh> ls
    ```

1.  NSH Shell produces some Text Output, which is pushed to the __NSH Output Pipe__

    ```text
    nsh> ls
    dev/
    var/
    ```

1.  We run an __LVGL Timer__ that periodically polls the NSH Output Pipe for Text Output

1.  When it detects the Text Output, the LVGL Timer reads the data...

    And renders the output in the __Output Text Area Widget__.

    ```text
    nsh> ls
    dev/
    var/
    ```

It looks like this...

![LVGL Terminal for PinePhone on Apache NuttX RTOS](https://lupyuen.github.io/images/terminal-demo.jpg)

_Whoa that looks complicated!_

Yeah. But we'll explain everything in this article...

-   How we start a __NuttX Task__

-   What are __NuttX Pipes__ and how we use them

-   How we render __LVGL Widgets__ and handle events

And eventually we'll understand the Source Code...

-   [__nuttx-apps/examples/lvglterm__](https://github.com/apache/nuttx-apps/blob/master/examples/lvglterm/lvglterm.c)

    [(How to compile LVGL Terminal)](https://github.com/lupyuen/lvglterm)

    [(Download the NuttX Image for PinePhone)](https://github.com/lupyuen2/wip-nuttx/releases/tag/nuttx-12.0.2)

We begin by starting the NSH Task and piping a command to NSH Shell...

![Pipe a Command to NSH Shell](https://lupyuen.github.io/images/terminal-flow2.jpg)

# Pipe a Command to NSH Shell

Our Terminal App needs to...

-   Start the __NuttX Task__ for NSH Shell

    (Which will execute our NSH Commands)

-   Redirect the __NSH Shell Input__

    (To receive the NSH Commands that we typed)

-   Redirect the __NSH Shell Output__

    (To render the output of NSH Commands)

We'll redirect the NSH Input and Output with __NuttX Pipes__.

(Which will work like Linux Pipes)

Let's find out how...

## Create the Pipes

_How will we create the NuttX Pipes?_

This is how we __create a NuttX Pipe__ for NSH Input: [lvglterm.c](https://github.com/apache/nuttx-apps/blob/master/examples/lvglterm/lvglterm.c#L166-L189)

```c
// Create the NuttX Pipe for NSH Input
int nsh_stdin[2];
int ret = pipe(nsh_stdin);

// Check for error
if (ret < 0) {
  _err("stdin pipe failed: %d\n", errno); return;
}
```

NSH Shell will receive NSH Commands through this Pipe.

_Why two elements in nsh_stdin?_

That's because a NuttX Pipe has __Two Endpoints__ (in and out)...

-   __`nsh_stdin[0]`__ reads from the Pipe

-   __`nsh_stdin[1]`__ writes to the Pipe

NuttX Pipes are __Unidirectional__... Don't mix up the endpoints!

To remind ourselves, we define the __Read and Write Endpoints__ like so...

```c
// pipe[0] for reading, pipe[1] for writing
#define READ_PIPE  0
#define WRITE_PIPE 1
```

We do the same to create the NuttX Pipes for __NSH Output__ and __NSH Error__...

```c
// Create the NuttX Pipe for NSH Output
int nsh_stdout[2];
ret = pipe(nsh_stdout);
if (ret < 0) { _err("stdout pipe failed: %d\n", errno); return; }

// Create the NuttX Pipe for NSH Error
int nsh_stderr[2];
ret = pipe(nsh_stderr);
if (ret < 0) { _err("stderr pipe failed: %d\n", errno); return; }
```

There's a reason why we call __`_err`__ instead of __`printf`__, we'll find out next...

## Connect the Pipes

_How will we connect the pipes to NSH Shell?_

In a while we'll start the NuttX Task for NSH Shell. But before that, we need some plumbing to __connect the NuttX Pipes__.

First we close the streams for __Standard Input, Output and Error__: [lvglterm.c](https://github.com/apache/nuttx-apps/blob/master/examples/lvglterm/lvglterm.c#L189-L201)

```c
// Close stdin, stdout and stderr
close(0);
close(1);
close(2);
```

That's because NSH Shell will __inherit our Standard I/O__ streams later.

Next we __redirect the Standard I/O__ streams to the NuttX Pipes that we've created earlier...

```c
// Redirect stdin, stdout and stderr to our NuttX Pipes.
// READ_PIPE is 0, WRITE_PIPE is 1
dup2(nsh_stdin[READ_PIPE],   0);  // Redirect stdin
dup2(nsh_stdout[WRITE_PIPE], 1);  // Redirect stdout
dup2(nsh_stderr[WRITE_PIPE], 2);  // Redirect stderr
```

When we do this, __Standard I/O will no longer work__ with the NuttX Console.

Instead, all Standard I/O will go to our NuttX Pipes.

_So printf will no longer print to the NuttX Console?_

Exactly! That's why we call __`_err`__ and __`_info`__ in this article.

These functions are __hardwired to the NuttX Console__. They will continue to work after we have redirected the Standard I/O streams.

## Create the Task

Our plumbing is done, let's __start the NuttX Task__ for NSH Shell: [lvglterm.c](https://github.com/apache/nuttx-apps/blob/master/examples/lvglterm/lvglterm.c#L201-L216)

```c
// Task ID will be returned here
pid_t pid;

// No arguments for the NuttX Task.
// argv[0] is always the Task Path.
static char * const argv[] = { "nsh", NULL };

// Start a NuttX Task for NSH Shell
int ret = posix_spawn(
  &pid,   // Returned Task ID
  "nsh",  // NSH Path
  NULL,   // Inherit stdin, stdout and stderr
  NULL,   // Default spawn attributes
  argv,   // Arguments
  NULL    // No environment
);

// Check for error
if (ret < 0) { _err("posix_spawn failed: %d\n", errno); return; }

// For Debugging: Wait a while for NSH Shell to start
sleep(1);
```

[(__posix_spawn__ should be enabled in menuconfig)](https://github.com/lupyuen/lvglterm/blob/main/README.md)

NSH Shell inherits our Standard I/O streams, which we've redirected to our NuttX Pipes.

We're ready to test this!

## Test the Pipes

Finally we add some __Test Code__ to verify that everything works: [lvgldemo.c](https://github.com/lupyuen2/wip-nuttx-apps/blob/a9d67c135c458088946ed35c1b24be1b4aee3553/examples/lvgldemo/lvgldemo.c#L292-L338)

```c
// Send a command to NSH stdin
const char cmd[] = "ls\r";
ret = write(   // Write to the stream...
  nsh_stdin[WRITE_PIPE],  // NSH stdin (WRITE_PIPE is 1)
  cmd,         // Data to be written
  sizeof(cmd)  // Number of bytes
);

// Wait a while for NSH Shell to execute our command
sleep(1);
```

The code above sends the __`ls`__ command to NSH Shell, by writing to our NuttX Pipe for __NSH Standard Input__.

NSH Shell __runs the command__ and generates the command output.

We __read the output__ from NSH Shell...

```c
// Read the output from NSH stdout.
// TODO: This will block if there's nothing to read.
static char buf[64];
ret = read(        // Read from the stream...
  nsh_stdout[READ_PIPE],  // NSH stdout (READ_PIPE is 0)
  buf,             // Buffer to be read
  sizeof(buf) - 1  // Buffer size (needs terminating null)
);

// Print the output
if (ret > 0) {
  buf[ret] = 0;
  _info("%s\n", buf);
}
```

And it works! Here's the NSH Shell auto-running the __`ls`__ command received via our NuttX Pipe...

```text
NuttShell (NSH) NuttX-12.0.0
nsh> ls
/:
 dev/
 var/
nsh>
```

[(See the Complete Log)](https://github.com/lupyuen2/wip-nuttx-apps/blob/a9d67c135c458088946ed35c1b24be1b4aee3553/examples/lvgldemo/lvgldemo.c#L340-L390)

_What about NSH Error Output?_

Normally we do this to read the __NSH Error Output__...

```c
// Warning: This will block!
#ifdef NOTUSED
  // Read the output from NSH stderr.
  // TODO: This will block if there's nothing to read.
  ret = read(        // Read from the stream...
    nsh_stderr[READ_PIPE],  // NSH stderr (READ_PIPE is 0)
    buf,             // Buffer to be read
    sizeof(buf) - 1  // Buffer size (needs terminating null)
  );

  // Print the output
  if (ret > 0) { buf[ret] = 0; _info("%s\n", buf); }
#endif
```

But there's a problem...

Calling __`read()`__ on __`nsh_stderr`__ will block the execution if there's __no NSH Error Output__ ready to be read!

(Same for __`nsh_stdout`__)

Instead let's check if there's NSH Output ready to be read. We do this by calling __`poll()`__...

![Poll for NSH Output](https://lupyuen.github.io/images/terminal-flow5.jpg)

# Poll for NSH Output

In the previous section we started an NSH Shell that will execute NSH Commands that we pipe to it...

But there's a problem: Calling __`read()`__ on __`nsh_stdout`__ will block if there's no NSH Output to be read.

(We can't block our LVGL App, since LVGL needs to handle User Interface Events periodically)

__Solution:__ We call __`has_input`__ to check if NSH Shell has data ready to be read, before we actually read the data: [lvglterm.c](https://github.com/apache/nuttx-apps/blob/master/examples/lvglterm/lvglterm.c#L318-L337)

```c
// If NSH stdout has data to be read...
if (has_input(nsh_stdout[READ_PIPE])) {

  // Read the data from NSH stdout
  static char buf[64];
  ret = read(
    nsh_stdout[READ_PIPE],
    buf,
    sizeof(buf) - 1
  );

  // Print the data
  if (ret > 0) { buf[ret] = 0; _info("%s\n", buf); }
}
```

[(We do the same for __`nsh_stderr`__)](https://github.com/apache/nuttx-apps/blob/master/examples/lvglterm/lvglterm.c#L337-L357)

__`has_input`__ calls __`poll()`__ on __`nsh_stdout`__ to check if NSH Shell has data ready to be read: [lvglterm.c](https://github.com/apache/nuttx-apps/blob/master/examples/lvglterm/lvglterm.c#L423-L485)

```c
// Return true if the File Descriptor has data to be read
static bool has_input(
  int fd  // File Descriptor to be checked
) {
  // Define the Poll Struct
  struct pollfd fdp;
  fdp.fd     = fd;      // File Descriptor to be checked
  fdp.events = POLLIN;  // Check for Input

  // Poll the File Descriptor for Input
  int ret = poll(
    &fdp,  // File Descriptors
    1,     // Number of File Descriptors
    0      // Poll Timeout (Milliseconds)
  );
```

Note that we set the __Poll Timeout__ to 0.

Thus __`poll()`__ returns immediately with the result, without blocking.

We decode the result of __`poll()`__ like so...

```c
  if (ret > 0) {
    // If Poll is OK and there's Input...
    if ((fdp.revents & POLLIN) != 0) {
      // Report that there's Input
      _info("has input: fd=%d\n", fd);
      return true;
    }

    // Else report No Input
    _info("no input: fd=%d\n", fd);
    return false;

  } else if (ret == 0) {
    // If Timeout, report No Input
    _info("timeout: fd=%d\n", fd);
    return false;

  } else if (ret < 0) {
    // Handle Error
    _err("poll failed: %d, fd=%d\n", ret, fd);
    return false;
  }
```

_What happens when we run this?_

If NSH Shell has data waiting to be read, __`has_input`__ returns True...

```text
has_input: has input: fd=8
```

And if there's nothing waiting to be read, __`has_input`__ returns False (due to timeout)...

```text
has_input: timeout: fd=8
```

[(See the Complete Log)](https://github.com/lupyuen2/wip-nuttx-apps/blob/c30e1968d5106794f435882af69dfb7b1858d694/examples/lvgldemo/lvgldemo.c#L403-L556)

We've solved our problem of Blocking Reads from NSH Output, by polling for NSH Output!

This polling for NSH Output needs to be done in an LVGL Timer, here's why...

![Timer for LVGL Terminal](https://lupyuen.github.io/images/terminal-flow6.jpg)

# Timer for LVGL Terminal

_How will we poll for NSH Output and display it?_

We started an NSH Shell that will execute NSH Commands that we pipe to it.

Now we need to periodically __poll for NSH Output__, and write the output to the LVGL display.

Every couple of milliseconds we...

  -   __Poll the NSH Shell__ to check if it has output data

  -   __Read the output__ from NSH Shell

  -   __Display the output__ in an LVGL Widget

We do this with an [__LVGL Timer__](https://docs.lvgl.io/master/overview/timer.html) that's triggered every 100 milliseconds: [lvglterm.c](https://github.com/apache/nuttx-apps/blob/master/examples/lvglterm/lvglterm.c#L216-L223)

```c
// Create an LVGL Terminal that will let us
// interact with NuttX NSH Shell
static void create_terminal(void) {

  // Create an LVGL Timer to poll for output from NSH Shell
  lv_timer_t *timer = lv_timer_create(
    timer_callback,  // Callback Function
    100,             // Timer Period (Milliseconds)
    &user_data       // Callback Data
  );
```

(__user_data__ is unused for now)

__timer_callback__ is our Callback Function for the LVGL Timer.

Inside the callback, we poll for NSH Output, read the output and display it: [lvglterm.c](https://github.com/apache/nuttx-apps/blob/master/examples/lvglterm/lvglterm.c#L295-L357)

```c
// Callback Function for LVGL Timer
static void timer_callback(lv_timer_t *timer) {

  // If NSH stdout has data to be read...
  if (has_input(nsh_stdout[READ_PIPE])) {

    // Read the output from NSH stdout
    static char buf[64];
    int ret = read(
      nsh_stdout[READ_PIPE],
      buf,
      sizeof(buf) - 1
    );

    // Add to NSH Output Text Area
    if (ret > 0) {
      buf[ret] = 0;
      remove_escape_codes(buf, ret);
      lv_textarea_add_text(output, buf);
    }
  }
```

[(We've seen __has_input__ earlier)](https://lupyuen.github.io/articles/terminal#poll-for-nsh-output)

[(__lv_textarea_add_text__ comes from LVGL)](https://docs.lvgl.io/master/widgets/textarea.html#_CPPv420lv_textarea_add_textP8lv_obj_tPKc)

We'll talk about __remove_escape_codes__ in a while.

_How do we test this Timer Callback?_

Without LVGL Widgets, testing the LVGL Timer Callback will be tricky. Here's how we tested by manipulating the LVGL Timer...

-   [__"Poll for NSH Output in LVGL Timer"__](https://github.com/lupyuen/pinephone-nuttx#poll-for-nsh-output-in-lvgl-timer)

_Why poll for NSH Output? Why not run a Background Thread that will block on NSH Output?_

Even if we ran a Background Thread that will block until NSH Output is available, we still need to write the NSH Output to an __LVGL Widget for display__.

But LVGL is [__NOT Thread-Safe__](https://docs.lvgl.io/master/porting/os.html#tasks-and-threads). Thus we need a Mutex to lock the LVGL Widget, which gets messy.

For now, it's simpler to run an LVGL Timer to poll for NSH Output.

Now that our Background Processing is ready, let's render the LVGL Widgets for our terminal...

![Render Terminal with LVGL Widgets](https://lupyuen.github.io/images/terminal-flow3.jpg)

# Render Terminal with LVGL Widgets

_How will we render the Terminal with LVGL?_

Our Terminal will have 3 LVGL Widgets...

-   [__LVGL Text Area Widget__](https://docs.lvgl.io/master/widgets/textarea.html) that shows the __NSH Output__

    (At the top)

-   [__LVGL Text Area Widget__](https://docs.lvgl.io/master/widgets/textarea.html) for __NSH Input__, to enter commands

    (At the middle)

-   [__LVGL Keyboard Widget__](https://docs.lvgl.io/master/widgets/keyboard.html) for typing commands into __NSH Input__

    (At the bottom)

Like this...

![LVGL Terminal App](https://lupyuen.github.io/images/lvgl2-terminal2.jpg)

This is how we create the 3 LVGL Widgets: [lvglterm.c](https://github.com/apache/nuttx-apps/blob/master/examples/lvglterm/lvglterm.c#L234-L295)

```c
// LVGL Text Area Widgets for NSH Input and Output
static lv_obj_t *input;
static lv_obj_t *output;

// Create the LVGL Widgets for the LVGL Terminal
static void create_widgets(void) {
  ...
  // Create an LVGL Text Area Widget for NSH Output
  output = lv_textarea_create(
    col  // Parent is LVGL Column Container
  );
```

(We'll explain __col__)

In the code above, we begin by creating the [__LVGL Text Area Widget__](https://docs.lvgl.io/master/widgets/textarea.html) to display the __NSH Output__.

Next we create another [__LVGL Text Area Widget__](https://docs.lvgl.io/master/widgets/textarea.html) to show the __NSH Input__...

```c
  // Create an LVGL Text Area Widget for NSH Input
  input = lv_textarea_create(
    col  // Parent is LVGL Column Container
  );
```

Then we create the [__LVGL Keyboard Widget__](https://docs.lvgl.io/master/widgets/keyboard.html) to allow touchscreen entry of NSH Commands...

```c
  // Create an LVGL Keyboard Widget
  lv_obj_t *kb = lv_keyboard_create(
    col  // Parent is LVGL Column Container
  );

  // No padding around the Keyboard Widget
  lv_obj_set_style_pad_all(kb, 0, 0);
```

We __register a Callback Function__ for NSH Input, to detect the pressing of the Enter Key...

```c
  // Register the Callback Function for NSH Input
  lv_obj_add_event_cb(
    input,  // LVGL Text Area Widget for NSH Input
    input_callback,  // Callback Function
    LV_EVENT_ALL,    // Callback for All Events
    NULL             // Callback Argument
  );
```

__input_callback__ is the Callback Function for NSH Input. Which we'll cover in a while.

Finally we set the __Keyboard Widget to populate__ the NSH Input Text Area...

```c
  // Set the Keyboard to populate the NSH Input Text Area
  lv_keyboard_set_textarea(
    kb,    // LVGL Keyboard Widget
    input  // LVGL Text Area Widget for NSH Input
  );
}
```

That's how we create the 3 LVGL Widgets for our Terminal App!

_What's col?_

__col__ is an __LVGL Column Container__ that contains our 3 LVGL Widgets: [lvglterm.c](https://github.com/apache/nuttx-apps/blob/master/examples/lvglterm/lvglterm.c#L234-L295)

```c
  // Create an LVGL Container with Column Flex Direction
  col = lv_obj_create(lv_scr_act());

  // Column is 100% of Screen Width, 100% of Screen Height
  // (720 x 1440 pixels)
  lv_obj_set_size(col, LV_PCT(100), LV_PCT(100));

  // Widgets in the column will have Flex Layout
  lv_obj_set_flex_flow(col, LV_FLEX_FLOW_COLUMN);

  // No padding around the column
  lv_obj_set_style_pad_all(col, 0, 0);
```

We're using [__LVGL Flex Layout__](https://docs.lvgl.io/master/layouts/flex.html) so that the LVGL Widgets will be __Auto-Positioned__, based on the Screen Size of our device.

(Hence the code will render correctly on devices other than PinePhone)

Remember we have 3 LVGL Widgets: NSH Output, NSH Input and Keyboard. NSH Input and Keyboard have __Fixed Height__.

But NSH Output has __Flexible Height__ and will __fill the vertical space__ in the column...

```c
  // Create an LVGL Text Area Widget for NSH Output
  output = lv_textarea_create(col);

  // Width of NSH Output is 100% of Column Width
  lv_obj_set_width(output, LV_PCT(100));

  // Height of NSH Output will fill the Column Height
  lv_obj_set_flex_grow(output, 1);
```

NSH Input and Keyboard have __Fixed Height__...

```c
  // Create an LVGL Text Area Widget for NSH Input
  input = lv_textarea_create(col);

  // Width of NSH Input is 100% of Column Width.
  // Height is fixed. (One line of text)
  lv_obj_set_size(input, LV_PCT(100), LV_SIZE_CONTENT);

  // Create an LVGL Keyboard Widget.
  // Use Fixed Width and Height.
  kb = lv_keyboard_create(col);
```

Thus we have arranged our LVGL Widgets neatly, to fit any Screen Size!

Note that we're using the LVGL Default Font for all 3 LVGL Widgets. Which has a problem...

![Set Default Font to Monospace](https://lupyuen.github.io/images/lvgl2-terminal2.jpg)

# Set Terminal Font to Monospace

Like any Terminal App, our LVGL Terminal looks nicer with a [__Monospaced Font__](https://en.wikipedia.org/wiki/Monospaced_font). (Instead of a Proportional Font)

_So we change the Default LVGL Font to a Monospaced Font?_

But watch what happens if we change the LVGL Default Font from Montserrat 20 (Proportional) to __UNSCII 16 (Monospaced)__...

The LVGL Keyboard has __missing symbols!__ Enter, Backspace, ...

The __symbols are undefined__ in the UNSCII 16 Font. (Pic above)

Thus we set the LVGL Default Font back to Montserrat 20.

And instead we set the __Font Style for NSH Input and Output__ to UNSCII 16: [lvglterm.c](https://github.com/apache/nuttx-apps/blob/master/examples/lvglterm/lvglterm.c#L234-L295)

```c
// Set the Font Style for NSH Input and Output
// to a Monospaced Font: UNSCII 16
static lv_style_t terminal_style;
lv_style_init(&terminal_style);
lv_style_set_text_font(&terminal_style, &lv_font_unscii_16);

// Create an LVGL Text Area Widget for NSH Output
output = lv_textarea_create(lv_scr_act());
// Set the Font Style for NSH Output
lv_obj_add_style(output, &terminal_style, 0);
...

// Create an LVGL Text Area Widget for NSH Input
input = lv_textarea_create(lv_scr_act());
// Set the Font Style for NSH Input
lv_obj_add_style(input, &terminal_style, 0);
...
```

Now we see the LVGL Keyboard without missing symbols (when rendered with Montserrat 20)...

![Set Terminal Font to Monospace](https://lupyuen.github.io/images/terminal-title.jpg)

Let's look at our Callback Function for NSH Input...

![Handle Input from LVGL Keyboard](https://lupyuen.github.io/images/terminal-flow4.jpg)

# Handle Input from LVGL Keyboard

_How will we check if the Enter Key has been pressed?_

Remember earlier we __registered a Callback Function__ for NSH Input Text Area, to detect the pressing of the Enter Key: [lvglterm.c](https://github.com/apache/nuttx-apps/blob/master/examples/lvglterm/lvglterm.c#L284-L288)

```c
// Register the Callback Function for NSH Input
lv_obj_add_event_cb(
  input,  // LVGL Text Area Widget for NSH Input
  input_callback,  // Callback Function
  LV_EVENT_ALL,    // Callback for All Events
  kb               // Callback Argument (Keyboard)
);
```

__input_callback__ is the Callback Function for NSH Input.

It waits for the Enter Key to be pressed, then it sends the typed command to __NSH Shell via the NuttX Pipe__: [lvglterm.c](https://github.com/apache/nuttx-apps/blob/master/examples/lvglterm/lvglterm.c#L357-L423)

```c
// Callback Function for NSH Input Text Area
static void input_callback(lv_event_t *e) {

  // Decode the LVGL Event
  const lv_event_code_t code = lv_event_get_code(e);

  // If NSH Input Text Area has been updated...
  if (code == LV_EVENT_VALUE_CHANGED) {

    // Get the Keyboard Widget from the LVGL Event
    const lv_obj_t *kb = lv_event_get_user_data(e);

    // Get the Button Index of the Key Pressed
    const uint16_t id = lv_keyboard_get_selected_btn(kb);

    // Get the Text of the Key Pressed
    const char *key = lv_keyboard_get_btn_text(kb, id);
```

The __Enter Key__ has a Text Label of [__`EF` `A2` `A2`__](https://github.com/lupyuen/lvglterm/blob/f3e9e7f4d53e49303e8dbeaa2b8a8497699404a0/lvglterm.c#L515-L517).

We match the Key Pressed with that label...

```c
    // If Key Pressed is Enter...
    if (key[0] == 0xef && key[1] == 0xa2 && key[2] == 0xa2) {

      // Read the NSH Input
      const char *cmd = lv_textarea_get_text(input);
      if (cmd == NULL || cmd[0] == 0) { return; }

      // Send the Command to NSH stdin
      int ret = write(
        nsh_stdin[WRITE_PIPE],
        cmd,
        strlen(cmd)
      );

      // Erase the NSH Input
      lv_textarea_set_text(input, "");
    }
  }
}
```

And we send the NSH Input Text to the NSH Shell for execution.

(The NSH Input Text is already terminated by a [__Newline Character `0x0A`__](https://github.com/lupyuen/lvglterm/blob/f3e9e7f4d53e49303e8dbeaa2b8a8497699404a0/lvglterm.c#L518-L519), which works fine with NSH Shell)

The command runs in NSH Shell and produces some Output Text. Which is handled by our LVGL Timer Callback Function...

![Timer for LVGL Terminal](https://lupyuen.github.io/images/terminal-flow6.jpg)

# Handle Output from NSH Shell

Earlier we've created an LVGL Timer that __polls periodically for output__ generated by NSH Shell...

-   [__"Timer for LVGL Terminal"__](https://lupyuen.github.io/articles/terminal#timer-for-lvgl-terminal)

If it detects NSH Output, the LVGL Timer Callback Function writes the output to the __NSH Output Text Area__: [lvglterm.c](https://github.com/apache/nuttx-apps/blob/master/examples/lvglterm/lvglterm.c#L295-L357)

```c
// Callback Function for LVGL Timer
static void timer_callback(lv_timer_t *timer) {

  // If NSH stdout has data to be read...
  if (has_input(nsh_stdout[READ_PIPE])) {

    // Read the output from NSH stdout
    static char buf[64];
    int ret = read(
      nsh_stdout[READ_PIPE],
      buf,
      sizeof(buf) - 1
    );

    // Add to NSH Output Text Area
    if (ret > 0) {
      buf[ret] = 0;
      remove_escape_codes(buf, ret);
      lv_textarea_add_text(output, buf);
    }
  }
```

[(We've seen __has_input__ earlier)](https://lupyuen.github.io/articles/terminal#poll-for-nsh-output)

[(__lv_textarea_add_text__ comes from LVGL)](https://docs.lvgl.io/master/widgets/textarea.html#_CPPv420lv_textarea_add_textP8lv_obj_tPKc)

_What's remove_escape_codes?_

NSH is configured to work with VT100 and ANSI Terminals. So the NSH Output will contain [__ANSI Escape Codes__](https://en.wikipedia.org/wiki/ANSI_escape_code) like...

```text
nsh> <ESC>[K
```

[(Source)](https://github.com/lupyuen/lvglterm/blob/f3e9e7f4d53e49303e8dbeaa2b8a8497699404a0/lvglterm.c#L453-L455)

Which is the ANSI Command for [__"Erase In Line"__](https://en.wikipedia.org/wiki/ANSI_escape_code#CSIsection). (Clear to the end of line)

[__remove_escape_codes__](https://github.com/apache/nuttx-apps/blob/master/examples/lvglterm/lvglterm.c#L485-L521) searches for Escape Codes in the NSH Output and removes them.

_But the NSH Output looks laggy..._

Yeah we used an [__LVGL Text Area__](https://docs.lvgl.io/latest/en/html/widgets/textarea.html) to render the NSH Output. Which is probably not optimal for scrollable, static text.

An [__LVGL Label Widget__](https://docs.lvgl.io/latest/en/html/widgets/label.html) might work better.

[(Remember to set __LV_LABEL_LONG_TXT_HINT__)](https://docs.lvgl.io/latest/en/html/widgets/label.html#very-long-texts)

Also we might have to change polling to __Multithreaded Blocking__.

Which means we need Mutexes to lock the LVGL Widgets.

![LVGL Programming in Zig](https://lupyuen.github.io/images/terminal-zig1.jpg)

# LVGL Programming in Zig

_As we add more features to our LVGL Terminal..._

_Will it become too complex to extend and maintain?_

It might! That's why we should consider coding our LVGL Terminal App in the [__Zig Programming Language__](https://ziglang.org/).

_LVGL works with Zig?_

Yep! The pic above shows an __LVGL Program in Zig__ that runs OK with PinePhone and NuttX.

To simpify the code, we created a simple __LVGL Wrapper in Zig__ that makes LVGL feel more "object-oriented".

[(More about this)](https://github.com/lupyuen/pinephone-lvgl-zig)

![Compiling an LVGL Program in Zig](https://lupyuen.github.io/images/terminal-zig2.jpg)

_How will we compile our LVGL Program in Zig?_

_Do we need to manually import every single LVGL Function from C?_

Zig Compiler automatically __imports all LVGL Functions__ from C into Zig!

The pic above shows how we compile our LVGL Program in Zig. Which will auto-import all LVGL Functions from the C Header Files.

[(More about this)](https://github.com/lupyuen/pinephone-lvgl-zig#build-lvgl-zig-app)

![Sorry ChatGPT... Please try harder](https://lupyuen.github.io/images/terminal-chatgpt.jpg)

[_Sorry ChatGPT... Please try harder_](https://lupyuen.github.io/articles/chatgpt)

# What's Next

Now we can run NuttX Console Apps on PinePhone, without a Serial Cable!

I hope this will make it a little easier to experiment with NuttX on PinePhone. Lemme know what you're building!

Meanwhile please check out the other articles on NuttX for PinePhone...

-   [__"Apache NuttX RTOS for PinePhone"__](https://github.com/lupyuen/pinephone-nuttx)

Many Thanks to my [__GitHub Sponsors__](https://lupyuen.github.io/articles/sponsor) for supporting my work! This article wouldn't have been possible without your support.

-   [__Sponsor me a coffee__](https://lupyuen.github.io/articles/sponsor)

-   [__Discuss this article on Reddit__](https://www.reddit.com/r/PINE64official/comments/10rfyip/nuttx_rtos_for_pinephone_lvgl_terminal_for_nsh/)

-   [__Discuss this article on Hacker News__](https://news.ycombinator.com/item?id=34621805)

-   [__My Current Project: "Apache NuttX RTOS for Ox64 BL808"__](https://github.com/lupyuen/nuttx-ox64)

-   [__My Other Project: "NuttX for Star64 JH7110"__](https://github.com/lupyuen/nuttx-star64)

-   [__Older Project: "NuttX for PinePhone"__](https://github.com/lupyuen/pinephone-nuttx)

-   [__Check out my articles__](https://lupyuen.github.io)

-   [__RSS Feed__](https://lupyuen.github.io/rss.xml)

_Got a question, comment or suggestion? Create an Issue or submit a Pull Request here..._

[__lupyuen.github.io/src/terminal.md__](https://github.com/lupyuen/lupyuen.github.io/blob/master/src/terminal.md)

# Notes

1.  NSH Architecture-Specific Initialization __CONFIG_NSH_ARCHINIT__ should be disabled...

    ```text
    Application Configuration
      > NSH Library 
        > Have architecture-specific initialization
    ```

    That's because LVGL Terminal starts a new task for NSH, which will redo the Architecture-Specific Initialization if __CONFIG_NSH_ARCHINIT__ is enabled.

    LVGL Terminal will handle the initialization when __CONFIG_NSH_ARCHINIT__ is disabled.

1.  On Arm64 Platforms: This (harmless) warning appears when starting the NSH Task...

    ```text
    mkfatfs: command not found
    ```
    
    This will be fixed in a future patch.
