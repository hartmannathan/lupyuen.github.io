# Build a Linux Driver for PineDio LoRa SX1262 USB Adapter

📝 _28 Oct 2021_

[__UPDATE:__ This PineDio USB driver is incomplete. Please use __JF002/loramac-node__ instead](https://codeberg.org/JF002/loramac-node)

_What if our Laptop Computer could talk to other devices..._

_Over a Long Range, Low Bandwidth wireless network like LoRa?_

[(Up to 5 km or 3 miles in urban areas... 15 km or 10 miles in rural areas!)](https://lora-developers.semtech.com/documentation/tech-papers-and-guides/lora-and-lorawan/)

Yep that's possible today... With [__Pinebook Pro__](https://wiki.pine64.org/wiki/Pinebook_Pro) and the [__PineDio LoRa SX1262 USB Adapter__](https://wiki.pine64.org/wiki/Pinedio#USB_adapter)! (Pic below)

This article explains how we built the __LoRa SX1262 Driver__ for PineDio USB Adapter and tested it on Pinebook Pro (Manjaro Linux Arm64)...

-   [__github.com/lupyuen/lora-sx1262__](https://github.com/lupyuen/lora-sx1262)

Our LoRa SX1262 Driver is __still incomplete__ (it's not a Kernel Driver yet), but the driver __talks OK to other LoRa Devices__. (With some limitations)

Read on to learn more...

![PineDio LoRa SX1262 USB Adapter](https://lupyuen.github.io/images/usb-title.jpg)

# PineDio LoRa USB Adapter

PineDio LoRa USB Adapter looks like a simple dongle...

1.  Take a [__CH341 USB-to-Serial Interface Module__](http://www.wch-ic.com/products/CH341.html)

    (Top half of pic below)

1.  Connect it to a [__Semtech SX1262 LoRa Module__](https://www.semtech.com/products/wireless-rf/lora-core/sx1262) over SPI

    (Bottom half of pic below)

And we get the PineDio LoRa USB Adapter!

![Schematic for PineDio LoRa SX1262 USB Adapter](https://lupyuen.github.io/images/usb-schematic.jpg)

[(Source)](https://wiki.pine64.org/wiki/Pinedio#USB_LoRa_adapter)

_So CH341 exposes the SPI Interface for SX1262 over USB?_

Yep Pinebook Pro shall __control SX1262 over SPI__, bridged by CH341.

Which means that we need to install a __CH341 SPI Driver__ on Pinebook Pro.

(More about this in a while)

_What about other pins on SX1262: DIO1, BUSY and NRESET?_

__DIO1__ is used by SX1262 to signal that a LoRa Packet has been received.

__BUSY__ is read by our computer to check if SX1262 is busy.

__NRESET__ is toggled by our computer to reset the SX1262 module.

Pinebook Pro shall control these pins via the __GPIO Interface on CH341__, as we'll see in a while.

[(More about PineDio USB)](https://wiki.pine64.org/wiki/Pinedio#USB_adapter)

[(CH341 Datasheet)](https://wiki.pine64.org/wiki/Pinedio#USB_LoRa_adapter)

![PineDio Stack BL604 RISC-V Board (foreground) talking to The Things Network via RAKWireless RAK7248 LoRaWAN Gateway (background)](https://lupyuen.github.io/images/ttn-title.jpg)

# LoRa SX1262 Driver for PineDio USB

_Where did the PineDio USB LoRa Driver come from?_

Believe it or not... The PineDio USB LoRa Driver is the exact same driver running on __PineCone BL602__ and __PineDio Stack BL604__! (Pic above)

-   [__"PineCone BL602 Talks LoRaWAN"__](https://lupyuen.github.io/articles/lorawan)

-   [__"LoRaWAN on PineDio Stack BL604 RISC-V Board"__](https://lupyuen.github.io/articles/lorawan2)

But modified to talk to __CH341 SPI for PineDio USB__.

(And compiled for Arm64 instead of RISC-V 32-bit)

The BL602 / BL604 LoRa Driver was ported from __Semtech's Reference Implementation__ of SX1262 Driver...

-   [__LoRaMac-node/radio/sx126x__](https://github.com/Lora-net/LoRaMac-node/tree/master/src/radio/sx126x)

![The Things Network in Singapore](https://lupyuen.github.io/images/lorawan2-ttn3.png)

[(Source)](https://lupyuen.github.io/articles/lorawan2#seeking-volunteers)

## LoRaWAN Support

_There are many LoRa Drivers out there, why did we port Semtech's Reference Driver?_

That's because Semtech's Reference Driver __supports LoRaWAN__, which adds security features to low-level LoRa.

[(Like for authentication and encryption)](https://lupyuen.github.io/articles/lorawan2#security)

_How useful is LoRaWAN? Will we be using it?_

Someday we might connect PineDio USB to a __LoRaWAN Network__ like...

-   [__The Things Network__](https://lupyuen.github.io/articles/ttn): Free-to-use public global LoRaWAN Network for IoT devices. (Pic above)

-   [__Helium__](https://www.helium.com/lorawan): Commercial global LoRaWAN Network for IoT devices.

Thus it's good to build a LoRa Driver for PineDio USB that will support LoRaWAN in future.

[(I tried porting this new driver by Semtech... But gave up when I discovered it doesn't support LoRaWAN)](https://github.com/Lora-net/sx126x_driver)

[(Seeking security on LoRa without LoRaWAN? Check out the LoRaWAN alternatives)](https://lupyuen.github.io/articles/lorawan2#lorawan-alternatives)

## NimBLE Porting Layer

_Do we call any open source libraries in our PineDio USB Driver?_

Yes we call __NimBLE Porting Layer__, the open source library for Multithreading Functions...

-   [__Multitask with NimBLE Porting Layer__](https://lupyuen.github.io/articles/lora2#multitask-with-nimble-porting-layer)

To transmit and receive LoRa Messages we need __Timers and Background Threads__. Which are provided by NimBLE Porting Layer.

_Have we used NimBLE Porting Layer before?_

Yep we used NimBLE Porting Layer in the __LoRa SX1262 and SX1276 Drivers__ for BL602...

-   [__"PineCone BL602 RISC-V Board Receives LoRa Packets"__](https://lupyuen.github.io/articles/lora2)

So we're really fortunate that NimBLE Porting Layer complies on Arm64 Linux as well.

[(It's part of PineTime InfiniTime too!)](https://lupyuen.github.io/pinetime-rust-mynewt/articles/dfu#nimble-stack-for-bluetooth-le-on-pinetime)

![Pinebook Pro with PineDio USB Adapter](https://lupyuen.github.io/images/usb-pinedio3.jpg)

# Read SX1262 Registers

_What's the simplest way to test our USB PineDio Driver?_

To test whether our USB PineDio Driver is working with CH341 SPI, we can read the __LoRa SX1262 Registers__.

Here's how: [main.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/main.c#L74-L81)

```c
/// Main Function
int main(void) {
  //  Read SX1262 registers 0x00 to 0x0F
  read_registers();
  return 0;
}

/// Read SX1262 registers
static void read_registers(void) {
  //  Init the SPI port
  SX126xIoInit();

  //  Read and print the first 16 registers: 0 to 15
  for (uint16_t addr = 0; addr < 0x10; addr++) {
    //  Read the register
    uint8_t val = SX126xReadRegister(addr);

    //  Print the register value
    printf("Register 0x%02x = 0x%02x\r\n", addr, val);
  }
}
```

[(__SX126xIoInit__ is defined here)](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x-linux.c#L65-L77)

In our Main Function we call __read_registers__ and __SX126xReadRegister__ to read a bunch of SX1262 Registers. (`0x00` to `0x0F`)

In our PineDio USB Driver, __SX126xReadRegister__ calls __SX126xReadRegisters__ and __sx126x_read_register__ to read each register: [sx126x-linux.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x-linux.c#L268-L281)

```c
/// Read an SX1262 Register at the specified address
uint8_t SX126xReadRegister(uint16_t address) {
  //  Read one register and return the value
  uint8_t data;
  SX126xReadRegisters(address, &data, 1);
  return data;
}

/// Read one or more SX1262 Registers at the specified address.
/// `size` is the number of registers to read.
void SX126xReadRegisters(uint16_t address, uint8_t *buffer, uint16_t size) {
  //  Wake up SX1262 if sleeping
  SX126xCheckDeviceReady();

  //  Read the SX1262 registers
  int rc = sx126x_read_register(NULL, address, buffer, size);
  assert(rc == 0);

  //  Wait for SX1262 to be ready
  SX126xWaitOnBusy();
}
```

(We'll see __SX126xCheckDeviceReady__ and __SX126xWaitOnBusy__ in a while)

__sx126x_read_register__ reads a register by sending the Read Register Command to SX1262 over SPI: [sx126x-linux.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x-linux.c#L486-L495)

```c
/// Send a Read Register Command to SX1262 over SPI
/// and return the results in `buffer`. `size` is the
/// number of registers to read.
static int sx126x_read_register(const void* context, const uint16_t address, uint8_t* buffer, const uint8_t size) {
  //  Reserve 4 bytes for our SX1262 Command Buffer
  uint8_t buf[SX126X_SIZE_READ_REGISTER] = { 0 };

  //  Init the SX1262 Command Buffer
  buf[0] = RADIO_READ_REGISTER;       //  Command ID
  buf[1] = (uint8_t) (address >> 8);  //  MSB of Register ID
  buf[2] = (uint8_t) (address >> 0);  //  LSB of Register ID
  buf[3] = 0;                         //  Unused

  //  Transmit the Command Buffer over SPI 
  //  and receive the Result Buffer
  int status = sx126x_hal_read( 
    context,  //  Context (unsued)
    buf,      //  Command Buffer
    SX126X_SIZE_READ_REGISTER,  //  Command Buffer Size: 4 bytes
    buffer,   //  Result Buffer
    size,     //  Result Buffer Size
    NULL      //  Status not required
  );
  return status;
}
```

And the values of the registers are returned by SX1262 over SPI.

(More about __sx126x_hal_read__ later)

## Run the Driver

Follow the instructions to __install the CH341 SPI Driver__...

-   [__"Install CH341 SPI Driver"__](https://lupyuen.github.io/articles/usb#appendix-install-ch341-spi-driver)

Follow the instructions to __download, build and run__ the PineDio USB Driver...

-   [__"Appendix: Build PineDio USB Driver"__](https://lupyuen.github.io/articles/usb#appendix-build-pinedio-usb-driver)

Remember to edit [__src/main.c__](https://github.com/lupyuen/lora-sx1262/blob/master/src/main.c) and uncomment...

```c
#define READ_REGISTERS
```

Build and run the PineDio USB Driver...

```bash
## Build PineDio USB Driver
make

## Run PineDio USB Driver
sudo ./lora-sx1262
```

And watch for these __SX1262 Register Values__...

```text
Register 0x00 = 0x00
...
Register 0x08 = 0x80
Register 0x09 = 0x00
Register 0x0a = 0x01
```

[(See the Output Log)](https://github.com/lupyuen/lora-sx1262#read-registers)

[(See the dmesg Log)](https://github.com/lupyuen/lora-sx1262#read-registers-1)

If we see these values... Our PineDio USB Driver is talking correctly to CH341 SPI and SX1262!

Note that the values above will change when we __transmit and receive LoRa Messages__.

Let's do that next.

![Reading SX1262 Registers on PineDio USB](https://lupyuen.github.io/images/usb-registers3.png)

## Source Files for Linux

_We're seeing layers of code, like an onion? (Or Shrek)_

Yep we have __layers of Source Files__ in our SX1262 Driver...

1.  Source Files __specific to Linux__

    (For PineDio USB and Pinebook Pro)

1.  Source Files __specific to BL602 and BL604__

    (For PineCone BL602 and PineDio Stack BL604)

1.  Source Files __common to all platforms__

    (For Linux, BL602 and BL604)

The Source Files __specific to Linux__ are...

-   [__src/main.c__](https://github.com/lupyuen/lora-sx1262/blob/master/src/main.c)

    (Main Program for Linux)

-   [__src/sx126x-linux.c__](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x-linux.c)

    (Linux Interface for SX1262 Driver)

-   [__npl/linux/src__](https://github.com/lupyuen/lora-sx1262/tree/master/npl/linux/src)

    (NimBLE Porting Layer for Linux)

All other Source Files are shared by Linux, BL602 and BL604.

(Except [__sx126x-board.c__](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x-board.c) which is the BL602 / BL604 Interface for SX1262)

# LoRa Parameters

Before we transmit and receive LoRa Messages on PineDio USB, let's talk about the __LoRa Parameters__.

To find out which __LoRa Frequency__ we should use for our region...

-  [__LoRa Frequencies by Country__](https://www.thethingsnetwork.org/docs/lorawan/frequencies-by-country.html)

We set the __LoRa Frequency__ like so: [main.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/main.c#L10-L25)

```c
/// TODO: We are using LoRa Frequency 923 MHz 
/// for Singapore. Change this for your region.
#define USE_BAND_923
```

Change __USE_BAND_923__ to __USE_BAND_433__, __780__, __868__ or __915__.

[(See the complete list)](https://github.com/lupyuen/lora-sx1262/blob/master/src/main.c#L10-L25)

Below are the other __LoRa Parameters__: [main.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/main.c#L27-L46)

```c
/// LoRa Parameters
#define LORAPING_TX_OUTPUT_POWER            14        /* dBm */

#define LORAPING_BANDWIDTH                  0         /* [0: 125 kHz, */
                                                      /*  1: 250 kHz, */
                                                      /*  2: 500 kHz, */
                                                      /*  3: Reserved] */
#define LORAPING_SPREADING_FACTOR           7         /* [SF7..SF12] */
#define LORAPING_CODINGRATE                 1         /* [1: 4/5, */
                                                      /*  2: 4/6, */
                                                      /*  3: 4/7, */
                                                      /*  4: 4/8] */
#define LORAPING_PREAMBLE_LENGTH            8         /* Same for Tx and Rx */
#define LORAPING_SYMBOL_TIMEOUT             5         /* Symbols */
#define LORAPING_FIX_LENGTH_PAYLOAD_ON      false
#define LORAPING_IQ_INVERSION_ON            false

#define LORAPING_TX_TIMEOUT_MS              3000    /* ms */
#define LORAPING_RX_TIMEOUT_MS              10000    /* ms */
#define LORAPING_BUFFER_SIZE                64      /* LoRa message size */
```

[(More about LoRa Parameters)](https://www.thethingsnetwork.org/docs/lorawan/spreading-factors/)

During testing, these should __match the LoRa Parameters__ used by the LoRa Transmitter / Receiver.

These are __LoRa Transmitter and Receiver__ programs based on [__RAKwireless WisBlock__](https://lupyuen.github.io/articles/wisblock) (pic below) that I used for testing PineDio USB...

-   [__wisblock-lora-transmitter__](https://github.com/lupyuen/wisblock-lora-transmitter/tree/pinedio)

    [(LoRa Parameters for Transmitter)](https://github.com/lupyuen/wisblock-lora-transmitter/blob/pinedio/src/main.cpp#L38-L58)

-   [__wisblock-lora-receiver__](https://github.com/lupyuen/wisblock-lora-receiver)

    [(LoRa Parameters for Receiver)](https://github.com/lupyuen/wisblock-lora-receiver/blob/main/src/main.cpp#L37-L56)

Thus the LoRa Parameters for PineDio USB should match the above.

_Are there practical limits on the LoRa Parameters?_

Yes we need to comply with the __Local Regulations__ on the usage of [__ISM Radio Bands__](https://en.wikipedia.org/wiki/ISM_radio_band): FCC, ETSI, ...

-   [__"Regional Parameters"__](https://www.thethingsnetwork.org/docs/lorawan/regional-parameters/)

(Blasting LoRa Messages non-stop is no-no!)

When we connect PineDio USB to __The Things Network__, we need to comply with their Fair Use Policy...

-   [__"Fair Use of The Things Network"__](https://lupyuen.github.io/articles/ttn#fair-use-of-the-things-network)

![RAKwireless WisBlock LPWAN Module mounted on WisBlock Base Board](https://lupyuen.github.io/images/wisblock-title.jpg)

## Initialise LoRa SX1262

Our __init_driver__ function takes the above LoRa Parameters and initialises LoRa SX1262 like so: [main.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/main.c#L149-L203)

```c
/// Command to initialise the LoRa Driver.
/// Assume that create_task has been called to init the Event Queue.
static void init_driver(char *buf, int len, int argc, char **argv) {
  //  Set the LoRa Callback Functions
  RadioEvents_t radio_events;
  memset(&radio_events, 0, sizeof(radio_events));  //  Must init radio_events to null, because radio_events lives on stack!
  radio_events.TxDone    = on_tx_done;     //  Packet has been transmitted
  radio_events.RxDone    = on_rx_done;     //  Packet has been received
  radio_events.TxTimeout = on_tx_timeout;  //  Transmit Timeout
  radio_events.RxTimeout = on_rx_timeout;  //  Receive Timeout
  radio_events.RxError   = on_rx_error;    //  Receive Error
```

Here we set the __Callback Functions__ that will be called when a LoRa Message has been transmitted or received, also when we encounter a transmit / receive timeout or error.

(We'll see the Callback Functions in a while)

Next we initialise the LoRa Transceiver and set the __LoRa Frequency__...

```c
  //  Init the SPI Port and the LoRa Transceiver
  Radio.Init(&radio_events);

  //  Set the LoRa Frequency
  Radio.SetChannel(RF_FREQUENCY);
```

We set the __LoRa Transmit Parameters__...

```c
  //  Configure the LoRa Transceiver for transmitting messages
  Radio.SetTxConfig(
    MODEM_LORA,
    LORAPING_TX_OUTPUT_POWER,
    0,        //  Frequency deviation: Unused with LoRa
    LORAPING_BANDWIDTH,
    LORAPING_SPREADING_FACTOR,
    LORAPING_CODINGRATE,
    LORAPING_PREAMBLE_LENGTH,
    LORAPING_FIX_LENGTH_PAYLOAD_ON,
    true,     //  CRC enabled
    0,        //  Frequency hopping disabled
    0,        //  Hop period: N/A
    LORAPING_IQ_INVERSION_ON,
    LORAPING_TX_TIMEOUT_MS
  );
```

Finally we set the __LoRa Receive Parameters__...

```c
  //  Configure the LoRa Transceiver for receiving messages
  Radio.SetRxConfig(
    MODEM_LORA,
    LORAPING_BANDWIDTH,
    LORAPING_SPREADING_FACTOR,
    LORAPING_CODINGRATE,
    0,        //  AFC bandwidth: Unused with LoRa
    LORAPING_PREAMBLE_LENGTH,
    LORAPING_SYMBOL_TIMEOUT,
    LORAPING_FIX_LENGTH_PAYLOAD_ON,
    0,        //  Fixed payload length: N/A
    true,     //  CRC enabled
    0,        //  Frequency hopping disabled
    0,        //  Hop period: N/A
    LORAPING_IQ_INVERSION_ON,
    true      //  Continuous receive mode
  );    
}
```

The __Radio__ functions are Platform-Independent (Linux and BL602), defined in [radio.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/radio.c)

-   [__RadioInit:__](https://lupyuen.github.io/articles/usb#radioinit-initialise-lora-module) Init LoRa SX1262

    [(__RadioInit__ is explained here)](https://lupyuen.github.io/articles/usb#radioinit-initialise-lora-module)

-   [__RadioSetChannel:__](https://lupyuen.github.io/articles/usb#radiosetchannel-set-lora-frequency) Set LoRa Frequency

    [(__RadioSetChannel__ is explained here)](https://lupyuen.github.io/articles/usb#radiosetchannel-set-lora-frequency)

-   [__RadioSetTxConfig:__](https://lupyuen.github.io/articles/usb#radiosettxconfig-set-transmit-configuration) Set LoRa Transmit Configuration

    [(__RadioSetTxConfig__ is explained here)](https://lupyuen.github.io/articles/usb#radiosettxconfig-set-transmit-configuration)

-   [__RadioSetRxConfig:__](https://lupyuen.github.io/articles/usb#radiosetrxconfig-set-receive-configuration) Set LoRa Receive Configuration

    [(__RadioSetRxConfig__ is explained here)](https://lupyuen.github.io/articles/usb#radiosetrxconfig-set-receive-configuration)

(The __Radio__ functions will also be called later when we implement LoRaWAN)

![Transmitting a LoRa Message](https://lupyuen.github.io/images/usb-transmit2.png)

# Transmit LoRa Message

Now we're ready to __transmit a LoRa Message__! Here's how: [main.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/main.c#L74-L119)

```c
/// Main Function
int main(void) {
  //  Init SX1262 driver
  init_driver();

  //  TODO: Do we need to wait?
  sleep(1);

  //  Send a LoRa message
  send_message();
  return 0;
}
```

We begin by calling __init_driver__ to set the LoRa Parameters and the Callback Functions.

(We've seen __init_driver__ in the previous section)

To transmit a LoRa Message, __send_message__ calls __send_once__: [main.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/main.c#L205-L211)

```c
/// Send a LoRa message. Assume that SX1262 driver has been initialised.
static void send_message(void) {
  //  Send the "PING" message
  send_once(1);
}
```

__send_once__ prepares a 64-byte LoRa Message containing the string "`PING`": [demo.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/main.c#L213-L239)

```c
/// We send a "PING" message and expect a "PONG" response
const uint8_t loraping_ping_msg[] = "PING";
const uint8_t loraping_pong_msg[] = "PONG";

/// 64-byte buffer for our LoRa message
static uint8_t loraping_buffer[LORAPING_BUFFER_SIZE];

/// Send a LoRa message. If is_ping is 0, send "PONG". Otherwise send "PING".
static void send_once(int is_ping) {
  //  Copy the "PING" or "PONG" message 
  //  to the transmit buffer
  if (is_ping) {
    memcpy(loraping_buffer, loraping_ping_msg, 4);
  } else {
    memcpy(loraping_buffer, loraping_pong_msg, 4);
  }
```

Then we __pad the 64-byte message__ with values 0, 1, 2, ...

```c
  //  Fill up the remaining space in the 
  //  transmit buffer (64 bytes) with values 
  //  0, 1, 2, ...
  for (int i = 4; i < sizeof loraping_buffer; i++) {
    loraping_buffer[i] = i - 4;
  }
```

And we __transmit the LoRa Message__...

```c
  //  We compute the message length, up to max 29 bytes.
  //  CAUTION: Anything more will cause message corruption!
  #define MAX_MESSAGE_SIZE 29
  uint8_t size = sizeof loraping_buffer > MAX_MESSAGE_SIZE
    ? MAX_MESSAGE_SIZE 
    : sizeof loraping_buffer;

  //  We send the transmit buffer, limited to 29 bytes.
  //  CAUTION: Anything more will cause message corruption!
  Radio.Send(loraping_buffer, size);

  //  TODO: Previously we send 64 bytes, which gets garbled consistently.
  //  Does CH341 limit SPI transfers to 31 bytes?
  //  (Including 2 bytes for SX1262 SPI command header)
  //  Radio.Send(loraping_buffer, sizeof loraping_buffer);
}
```

[(__RadioSend__ is explained here)](https://lupyuen.github.io/articles/usb#radiosend-transmit-message)

Our PineDio USB Driver has an issue with __CH341 SPI Transfers__...

__Transmitting a LoRa Message on PineDio USB longer than 29 bytes will cause message corruption!__

Thus we limit the Transmit LoRa Message Size to __29 bytes__.

(There's a way to fix this... More about CH341 later)

When the LoRa Message has been transmitted, the LoRa Driver calls our Callback Function __on_tx_done__ defined in [main.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/main.c#L253-L266)

```c
/// Callback Function that is called when our LoRa message has been transmitted
static void on_tx_done(void) {
  //  Log the success status
  loraping_stats.tx_success++;

  //  Switch the LoRa Transceiver to 
  //  low power, sleep mode
  Radio.Sleep();
}
```

[(__RadioSleep__ is explained here)](https://lupyuen.github.io/articles/usb#radiosleep-switch-to-sleep-mode)

Here we log the number of packets transmitted, and put LoRa SX1262 into __low power, sleep mode__.

Note: __on_tx_done__ won't actually be called in our current driver, because we haven't implemented Multithreading. (More about this later)

To handle Transmit Timeout Errors, we define the Callback Function __on_tx_timeout__: [main.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/main.c#L300-L312)

```c
/// Callback Function that is called when our LoRa message couldn't be transmitted due to timeout
static void on_tx_timeout(void) {
  //  Switch the LoRa Transceiver to 
  //  low power, sleep mode
  Radio.Sleep();

  //  Log the timeout
  loraping_stats.tx_timeout++;
}
```

## Run the Driver

Follow the instructions to __install the CH341 SPI Driver__...

-   [__"Install CH341 SPI Driver"__](https://lupyuen.github.io/articles/usb#appendix-install-ch341-spi-driver)

Follow the instructions to __download, build and run__ the PineDio USB Driver...

-   [__"Appendix: Build PineDio USB Driver"__](https://lupyuen.github.io/articles/usb#appendix-build-pinedio-usb-driver)

Remember to edit [__src/main.c__](https://github.com/lupyuen/lora-sx1262/blob/master/src/main.c) and uncomment...

```c
#define SEND_MESSAGE
```

Also edit [__src/main.c__](https://github.com/lupyuen/lora-sx1262/blob/master/src/main.c#L10-L46) and set the __LoRa Parameters__. (As explained earlier)

Build and run the PineDio USB Driver...

```bash
## Build PineDio USB Driver
make

## Run PineDio USB Driver
sudo ./lora-sx1262
```

We should see __PineDio USB transmitting__ our 29-byte LoRa Message...

```text
send_message
RadioSend: size=29
50 49 4e 47 00 01 02 03 04 05 06 07 08 09 0a 0b 0c 0d 0e 0f 10 11 12 13 14 15 16 17 18 
```

[("`PING`" followed by 0, 1, 2, ...)](https://github.com/lupyuen/lora-sx1262/blob/master/src/main.c#L213-L239)

[(See the Output Log)](https://github.com/lupyuen/lora-sx1262#send-message)

[(See the dmesg Log)](https://github.com/lupyuen/lora-sx1262#send-message-1)

On [__RAKwireless WisBlock__](https://github.com/lupyuen/wisblock-lora-receiver) we should see the same 29-byte LoRa Message received...

```text
LoRaP2P Rx Test
Starting Radio.Rx
OnRxDone: Timestamp=18, RssiValue=-28 dBm, SnrValue=13, 
Data=50 49 4E 47 00 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F 10 11 12 13 14 15 16 17 18 
```

[(See the WisBlock Log)](https://github.com/lupyuen/lora-sx1262#wisblock-receiver-log)

PineDio USB has successfully transmitted a 29-byte LoRa Message to RAKwireless WisBlock!

![RAKwireless WisBlock receives 29-byte LoRa Message from RAKwireless WisBlock](https://lupyuen.github.io/images/usb-wisblock5.jpg)

## Spectrum Analysis with SDR

_What if nothing appears in our LoRa Receiver?_

Use a __Spectrum Analyser__ (like a __Software Defined Radio__) to sniff the airwaves and check whether our LoRa Message is transmitted...

1.  At the right __Radio Frequency__

    (923 MHz below)

1.  With __sufficient power__

    (Red stripe below)

![LoRa Message captured with Software Defined Radio](https://lupyuen.github.io/images/usb-chirp2.jpg)

LoRa Messages have a characteristic criss-cross shape: __LoRa Chirp__. (Like above)

More about LoRa Chirps and Software Defined Radio...

-   [__"Visualise LoRa with Software Defined Radio"__](https://lupyuen.github.io/articles/lora#visualise-lora-with-software-defined-radio)

![Receiving a LoRa Message with PineDio USB](https://lupyuen.github.io/images/usb-receive5.png)

# Receive LoRa Message

Let's __receive a LoRa Message__ on PineDio USB!

This is how we do it: [main.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/main.c#L74-L119)

```c
/// Main Function
int main(void) {
  //  TODO: Create a Background Thread 
  //  to handle LoRa Events
  create_task();
```

We start by creating a Background Thread to handle LoRa Events.

(__create_task__ doesn't do anything because we haven't implemented Multithreading. More about this later)

Next we set the __LoRa Parameters__ and the __Callback Functions__...

```c
  //  Init SX1262 driver
  init_driver();

  //  TODO: Do we need to wait?
  sleep(1);
```

(Yep the same __init_driver__ we've seen earlier)

For the next __10 seconds__ we poll and handle LoRa Events (like Message Received)...

```c
  //  Handle LoRa events for the next 10 seconds
  for (int i = 0; i < 10; i++) {
    //  Prepare to receive a LoRa message
    receive_message();

    //  Process the received LoRa message, if any
    RadioOnDioIrq(NULL);
    
    //  Sleep for 1 second
    usleep(1000 * 1000);
  }
  return 0;
}
```

We call __receive_message__ to get SX1262 ready to receive a single LoRa Message.

Then we call __RadioOnDioIrq__ to handle the Message Received Event. (If any)

[(__RadioOnDioIrq__ is explained here)](https://lupyuen.github.io/articles/usb#radioondioirq)

__receive_message__ is defined like so: [main.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/main.c#L241-L248)

```c
/// Receive a LoRa message. Assume that SX1262 driver has been initialised.
/// Assume that create_task has been called to init the Event Queue.
static void receive_message(void) {
  //  Receive a LoRa message within the timeout period
  Radio.Rx(LORAPING_RX_TIMEOUT_MS);
}
```

[(__RadioRx__ is explained here)](https://lupyuen.github.io/articles/usb#radiorx-receive-message)

When the LoRa Driver receives a LoRa Message, it calls our Callback Function __on_rx_done__ defined in [main.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/main.c#L268-L298)

```c
/// Callback Function that is called when a LoRa message has been received
static void on_rx_done(
  uint8_t *payload,  //  Buffer containing received LoRa message
  uint16_t size,     //  Size of the LoRa message
  int16_t rssi,      //  Signal strength
  int8_t snr) {      //  Signal To Noise ratio

  //  Switch the LoRa Transceiver to low power, sleep mode
  Radio.Sleep();

  //  Log the signal strength, signal to noise ratio
  loraping_rxinfo_rxed(rssi, snr);
```

__on_rx_done__ switches the LoRa Transceiver to low power, sleep mode and logs the received packet.

Next we __copy the received packet__ into a buffer...

```c
  //  Copy the received packet
  if (size > sizeof loraping_buffer) {
    size = sizeof loraping_buffer;
  }
  loraping_rx_size = size;
  memcpy(loraping_buffer, payload, size);
```

Finally we __dump the buffer__ containing the received packet...

```c
  //  Dump the contents of the received packet
  for (int i = 0; i < loraping_rx_size; i++) {
    printf("%02x ", loraping_buffer[i]);
  }
  printf("\r\n");
}
```

_What happens when we don't receive a packet in 10 seconds? (LORAPING_RX_TIMEOUT_MS)_

The LoRa Driver calls our Callback Function __on_rx_timeout__ defined in [main.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/main.c#L314-L327)

```c
/// Callback Function that is called when no LoRa messages could be received due to timeout
static void on_rx_timeout(void) {
  //  Switch the LoRa Transceiver to low power, sleep mode
  Radio.Sleep();

  //  Log the timeout
  loraping_stats.rx_timeout++;
}
```

We switch the LoRa Transceiver into sleep mode and log the timeout.

Note: __on_rx_timeout__ won't actually be called in our current driver, because we haven't implemented Multithreading. (More about this later)

To handle Receive Errors, we define the Callback Function __on_rx_error__: [main.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/main.c#L329-L341)

```c
/// Callback Function that is called when we couldn't receive a LoRa message due to error
static void on_rx_error(void) {
  //  Log the error
  loraping_stats.rx_error++;

  //  Switch the LoRa Transceiver to low power, sleep mode
  Radio.Sleep();
}
```

## Run the Driver

Follow the instructions to __install the CH341 SPI Driver__...

-   [__"Install CH341 SPI Driver"__](https://lupyuen.github.io/articles/usb#appendix-install-ch341-spi-driver)

Follow the instructions to __download, build and run__ the PineDio USB Driver...

-   [__"Appendix: Build PineDio USB Driver"__](https://lupyuen.github.io/articles/usb#appendix-build-pinedio-usb-driver)

Remember to edit [__src/main.c__](https://github.com/lupyuen/lora-sx1262/blob/master/src/main.c) and uncomment...

```c
#define RECEIVE_MESSAGE
```

Also edit [__src/main.c__](https://github.com/lupyuen/lora-sx1262/blob/master/src/main.c#L10-L46) and set the __LoRa Parameters__. (As explained earlier)

Build and run the PineDio USB Driver...

```bash
## Build PineDio USB Driver
make

## Run PineDio USB Driver
sudo ./lora-sx1262
```

Switch over to [__RAKwireless WisBlock__](https://github.com/lupyuen/wisblock-lora-transmitter/tree/pinedio) and transmit a 28-byte LoRa Message...

```text
LoRap2p Tx Test
send: 48 65 6c 6c 6f 00 01 02 03 04 05 06 07 08 09 0a 0b 0c 0d 0e 0f 10 11 12 13 14 15 16 
OnTxDone
```

[("`Hello`" followed by 0, 1, 2, ...)](https://github.com/lupyuen/wisblock-lora-transmitter/blob/pinedio/src/main.cpp#L124-L148)

[(See the WisBlock Log)](https://github.com/lupyuen/lora-sx1262#wisblock-transmitter-log)

On __PineDio USB__ we should see the same 28-byte LoRa Message...

```text
IRQ_RX_DONE
03 04 05 06 07 08 09 0a 0b 0c 0d 0e 0f 10 11 12 13 14 15 16 48 65 6c 6c 6f 00 01 02 
IRQ_PREAMBLE_DETECTED
IRQ_HEADER_VALID
receive_message
```

[(See the Output Log)](https://github.com/lupyuen/lora-sx1262#receive-message)

[(See the dmesg Log)](https://github.com/lupyuen/lora-sx1262#receive-message-1)

PineDio USB has successfully received a 28-byte LoRa Message from RAKwireless WisBlock!

![PineDio USB receives a 28-byte LoRa Message from RAKwireless WisBlock](https://lupyuen.github.io/images/usb-wisblock6.jpg)

_Why 28 bytes?_

Our PineDio USB Driver has an issue with __CH341 SPI Transfers__...

__Receiving a LoRa Message on PineDio USB longer than 28 bytes will cause message corruption!__

Thus we limit the Receive LoRa Message Size to __28 bytes__.

There's a way to fix this... Coming up next!

![Schematic for PineDio LoRa SX1262 USB Adapter](https://lupyuen.github.io/images/usb-schematic.jpg)

[(Source)](https://wiki.pine64.org/wiki/Pinedio#USB_LoRa_adapter)

# CH341 SPI Interface

Remember that PineDio USB Dongle contains a [__CH341 USB-to-Serial Interface Module__](http://www.wch-ic.com/products/CH341.html) that talks to LoRa SX1262 (over SPI)...

-   [__CH341 Datasheet__](https://wiki.pine64.org/wiki/Pinedio#USB_LoRa_adapter)

-   [__CH341 Interfaces (Chinese)__](http://www.wch.cn/downloads/CH341DS2_PDF.html)

Pinebook Pro (Manjaro Linux Arm64) has a built-in driver for CH341... But it __doesn't support SPI__.

Thus for our PineDio USB Driver we're calling this __CH341 SPI Driver__...

-   [__JF002/spi-ch341-usb__](https://codeberg.org/JF002/spi-ch341-usb)

We install the CH341 SPI Driver with these steps...

-   [__"Install CH341 SPI Driver"__](https://lupyuen.github.io/articles/usb#appendix-install-ch341-spi-driver)

Now let's call the CH341 SPI Driver from our PineDio USB Driver.

[(Note: PineDio Wiki recommends _dimich-dmb/spi-ch341-usb_, but it didn't transmit LoRa packets during my testing)](https://github.com/dimich-dmb/spi-ch341-usb.git)

## Initialise SPI

Here's how our PineDio USB Driver calls CH341 SPI Driver to __initialise the SPI Bus__: [sx126x-linux.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x-linux.c#L651-L667)

```c
/// SPI Bus
static int spi = 0;

/// Init the SPI Bus. Return 0 on success.
static int init_spi(void) {
  //  Open the SPI Bus
  spi = open("/dev/spidev1.0", O_RDWR);
  assert(spi > 0);

  //  Set to SPI Mode 0
  uint8_t mmode = SPI_MODE_0;
  int rc = ioctl(spi, SPI_IOC_WR_MODE, &mmode);
  assert(rc == 0);

  //  Set LSB/MSB Mode
  uint8_t lsb = 0;
  rc = ioctl(spi, SPI_IOC_WR_LSB_FIRST, &lsb);
  assert(rc == 0);
  return 0;
}
```

__init_spi__ is called by [__SX126xIoInit__](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x-linux.c#L65-L77), which is called by [__RadioInit__](https://lupyuen.github.io/articles/usb#radioinit-initialise-lora-module) and [__init_driver__](https://github.com/lupyuen/lora-sx1262/blob/master/src/main.c#L149-L203)

(We've seen __init_driver__ earlier)

[(__RadioInit__ is explained here)](https://lupyuen.github.io/articles/usb#radioinit-initialise-lora-module)

## Transfer SPI

To __transfer SPI Data__ between PineDio USB and CH341 / SX1262, we do this: [sx126x-linux.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x-linux.c#L669-L691)

```c
/// Blocking call to transmit and receive buffers on SPI. Return 0 on success.
static int transfer_spi(const uint8_t *tx_buf, uint8_t *rx_buf, uint16_t len) {
  assert(spi > 0);
  assert(len > 0);
  assert(len <= 31);  //  CAUTION: CH341 SPI doesn't seem to support 32-byte SPI transfers 

  //  Prepare SPI Transfer
  struct spi_ioc_transfer spi_trans;
  memset(&spi_trans, 0, sizeof(spi_trans));
  spi_trans.tx_buf = (unsigned long) tx_buf;  //  Transmit Buffer
  spi_trans.rx_buf = (unsigned long) rx_buf;  //  Receive Buffer
  spi_trans.cs_change = true;   //  Set SPI Chip Select to Low
  spi_trans.len       = len;    //  How many bytes
  printf("spi tx: "); for (int i = 0; i < len; i++) { printf("%02x ", tx_buf[i]); } printf("\n");

  //  Transfer and receive the SPI buffers
  int rc = ioctl(spi, SPI_IOC_MESSAGE(1), &spi_trans);
  assert(rc >= 0);
  assert(rc == len);

  printf("spi rx: "); for (int i = 0; i < len; i++) { printf("%02x ", rx_buf[i]); } printf("\n");
  return 0;
}
```

(__transfer_spi__ will be called by our PineDio USB Driver, as we'll see later)

__transfer_spi__ has a strange assertion that stops large SPI transfers...

```c
//  CAUTION: CH341 SPI doesn't seem to 
//  support 32-byte SPI transfers 
assert(len <= 31);
```

We'll learn why in a while.

![PineDio USB transmits a garbled 64-byte LoRa Message to RAKwireless WisBlock](https://lupyuen.github.io/images/usb-wisblock4.jpg)

## Long Messages are Garbled

_What happens when we transmit a LoRa Message longer than 29 bytes?_

The pic above shows what happens when we __transmit a long message__ (64 bytes) from PineDio USB to RAKwireless WisBlock...

1.  Our __64-byte message is garbled__ when received

    (By RAKwireless WisBlock)

1.  But the message is __consistently garbled__

    (RAKwireless WisBlock receives the same garbled message twice, not any random message)

1.  Which means it's __not due to Radio Interference__

    (Radio Interference would garble the messages randomly)

By tweaking our PineDio USB Driver, we discover two shocking truths...

1.  __Transmitting a LoRa Message on PineDio USB longer than 29 bytes will cause message corruption!__

1.  __Receiving a LoRa Message on PineDio USB longer than 28 bytes will cause message corruption!__

Let's trace the code and solve this mystery.

## Transmit Long Message

Our PineDio USB Driver calls this function to __transmit a LoRa Message__: [sx126x-linux.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x-linux.c#L497-L503)

```c
static int sx126x_write_buffer(const void* context, const uint8_t offset, const uint8_t* buffer, const uint8_t size) {
  //  Prepare the Write Buffer Command (2 bytes)
  uint8_t buf[SX126X_SIZE_WRITE_BUFFER] = { 0 };
  buf[0] = RADIO_WRITE_BUFFER;  //  Write Buffer Command
  buf[1] = offset;              //  Write Buffer Offset

  //  Transfer the Write Buffer Command to SX1262 over SPI
  return sx126x_hal_write(
    context,  //  Context
    buf,      //  Command Buffer
    SX126X_SIZE_WRITE_BUFFER,  //  Command Buffer Size (2 bytes)
    buffer,   //  Write Data Buffer
    size      //  Write Data Buffer Size
  );
}
```

In this code we prepare a __SX1262 Write Buffer Command__ (2 bytes) and pass the Command Buffer (plus Data Buffer) to __sx126x_hal_write__.

(Data Buffer contains the LoRa Message to be transmitted)

Note that __Write Buffer Offset is always 0__, because of [__SX126xSetPayload__](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x.c#L144-L147) and [__SX126xWriteBuffer__](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x-linux.c#L283-L289).

[(__SX126xSetPayload__ and __SX126xWriteBuffer__ are explained here)](https://lupyuen.github.io/articles/usb#radiosend-transmit-message)

__sx126x_hal_write__ transfers the Command Buffer and Data Buffer over SPI: [sx126x-linux.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x-linux.c#L532-L569)

```c
/**
 * Radio data transfer - write
 *
 * @remark Shall be implemented by the user
 *
 * @param [in] context          Radio implementation parameters
 * @param [in] command          Pointer to the buffer to be transmitted
 * @param [in] command_length   Buffer size to be transmitted
 * @param [in] data             Pointer to the buffer to be transmitted
 * @param [in] data_length      Buffer size to be transmitted
 *
 * @returns Operation status
 */
static int sx126x_hal_write( 
  const void* context, const uint8_t* command, const uint16_t command_length,
  const uint8_t* data, const uint16_t data_length ) {
  printf("sx126x_hal_write: command_length=%d, data_length=%d\n", command_length, data_length);

  //  Total length is command + data length
  uint16_t len = command_length + data_length;
  assert(len > 0);
  assert(len <= SPI_BUFFER_SIZE);

  //  Clear the SPI Transmit and Receive buffers
  memset(&spi_tx_buf, 0, len);
  memset(&spi_rx_buf, 0, len);

  //  Copy command bytes to SPI Transmit Buffer
  memcpy(&spi_tx_buf, command, command_length);

  //  Copy data bytes to SPI Transmit Buffer
  memcpy(&spi_tx_buf[command_length], data, data_length);

  //  Transmit and receive the SPI buffers
  int rc = transfer_spi(spi_tx_buf, spi_rx_buf, len);
  assert(rc == 0);
  return 0;
}
```

We use an internal __1024-byte buffer for SPI Transfers__, so we're hunky dory here: [sx126x-linux.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x-linux.c#L521-L528)

```c
/// Max size of SPI transfers
#define SPI_BUFFER_SIZE 1024

/// SPI Transmit Buffer
static uint8_t spi_tx_buf[SPI_BUFFER_SIZE];

/// SPI Receive Buffer
static uint8_t spi_rx_buf[SPI_BUFFER_SIZE];
```

__sx126x_hal_write__ calls __transfer_spi__ to transfer the SPI Data.

(We've seen __transfer_spi__ earlier)

Thus __transfer_spi__ looks highly sus for transmitting Long LoRa Messages.

What about receiving Long LoRa Messages?

## Receive Long Message

Our PineDio USB Driver calls this function to __receive a LoRa Message__: [sx126x-linux.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x-linux.c#L505-L513)

```c
static int sx126x_read_buffer(const void* context, const uint8_t offset, uint8_t* buffer, const uint8_t size) {
  //  Prepare the Read Buffer Command (3 bytes)
  uint8_t buf[SX126X_SIZE_READ_BUFFER] = { 0 };
  buf[0] = RADIO_READ_BUFFER;  //  Read Buffer Command
  buf[1] = offset;             //  Read Buffer Offset
  buf[2] = 0;                  //  NOP

  //  Transfer the Read Buffer Command to SX1262 over SPI
  int status = sx126x_hal_read( 
    context,  //  Context
    buf,      //  Command Buffer
    SX126X_SIZE_READ_BUFFER,  //  Command Buffer Size (3 bytes)
    buffer,   //  Read Data Buffer
    size,     //  Read Data Buffer Size
    NULL      //  Ignore the status
  );
  return status;
}
```

In this code we prepare a __SX1262 Read Buffer Command__ (3 bytes) and pass the Command Buffer (plus Data Buffer) to __sx126x_hal_read__.

(Data Buffer will contain the received LoRa Message)

Note that __Read Buffer Offset is always 0__, because of [__SX126xGetPayload__](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x.c#L149-L160) and [__SX126xReadBuffer__](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x-linux.c#L291-L297).

[(__SX126xGetPayload__ and __SX126xReadBuffer__ are explained here)](https://lupyuen.github.io/articles/usb#receive-done)

__sx126x_hal_read__ transfers the Command Buffer over SPI: [sx126x-linux.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x-linux.c#L571-L615)

```c
/**
 * Radio data transfer - read
 *
 * @remark Shall be implemented by the user
 *
 * @param [in] context          Radio implementation parameters
 * @param [in] command          Pointer to the buffer to be transmitted
 * @param [in] command_length   Buffer size to be transmitted
 * @param [in] data             Pointer to the buffer to be received
 * @param [in] data_length      Buffer size to be received
 * @param [out] status          If not null, return the second SPI byte received as status
 *
 * @returns Operation status
 */
static int sx126x_hal_read( 
  const void* context, const uint8_t* command, const uint16_t command_length,
  uint8_t* data, const uint16_t data_length, uint8_t *status ) {
  printf("sx126x_hal_read: command_length=%d, data_length=%d\n", command_length, data_length);

  //  Total length is command + data length
  uint16_t len = command_length + data_length;
  assert(len > 0);
  assert(len <= SPI_BUFFER_SIZE);

  //  Clear the SPI Transmit and Receive buffers
  memset(&spi_tx_buf, 0, len);
  memset(&spi_rx_buf, 0, len);

  //  Copy command bytes to SPI Transmit Buffer
  memcpy(&spi_tx_buf, command, command_length);

  //  Transmit and receive the SPI buffers
  int rc = transfer_spi(spi_tx_buf, spi_rx_buf, len);
  assert(rc == 0);

  //  Copy SPI Receive buffer to data buffer
  memcpy(data, &spi_rx_buf[command_length], data_length);

  //  Return the second SPI byte received as status
  if (status != NULL) {
    assert(len >= 2);
    *status = spi_rx_buf[1];
  }
  return 0;
}
```

And returns the Data Buffer that has been read over SPI.

__sx126x_hal_read__ also calls __transfer_spi__ to transfer the SPI Data.

Now __transfer_spi__ is doubly sus... The same function is called to transmit AND receive Long LoRa Messages!

![CH341 SPI Driver fails when transferring 32 bytes](https://lupyuen.github.io/images/usb-spi8.png)

[(Source)](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x-linux.c#L669-L691)

## SPI Transfer Fails with 32 Bytes

_Does __transfer_spi__ impose a limit on the size of SPI Transfers?_

With some tweaking, we discover that __transfer_spi garbles the data when transferring 32 bytes or more__!

This seems to be a limitation of the __CH341 SPI Driver__.

[(Due to __CH341_USB_MAX_BULK_SIZE__ maybe?)](https://codeberg.org/JF002/spi-ch341-usb/src/branch/remove-calls-to-spi_busnum_to_master/spi-ch341-usb.c#L68)

Hence we limit all SPI Transfers to __31 bytes__: [sx126x-linux.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x-linux.c#L669-L691)

```c
/// Blocking call to transmit and receive buffers on SPI. Return 0 on success.
static int transfer_spi(const uint8_t *tx_buf, uint8_t *rx_buf, uint16_t len) {
  //  CAUTION: CH341 SPI doesn't seem to 
  //  support 32-byte SPI transfers 
  assert(len <= 31);
```

_Why 29 bytes for the max transmit size? And 28 bytes for the max receive size?_

That's because...

-   SX1262 Write Buffer Command (for transmit) occupies __2 SPI bytes__

-   SX1262 Read Buffer Command (for receive) occupies __3 SPI bytes__

But wait! We might have a fix for Long LoRa Messages...

![SX1262 Commands for WriteBuffer and ReadBuffer](https://lupyuen.github.io/images/usb-buffer.png)

[(From Semtech SX1262 Datasheet)](https://www.semtech.com/products/wireless-rf/lora-core/sx1262)

## Fix Long Messages

_Is there a way to fix Long LoRa Messages on PineDio USB?_

Let's look back at our code in [__sx126x_write_buffer__](https://lupyuen.github.io/articles/usb#transmit-long-message).

To transmit a LoRa Message, we send the __WriteBuffer Command__ to SX1262 over SPI...

1.  __WriteBuffer Command:__ `0x0E`

1.  __WriteBuffer Offset:__ `0x00`

1.  __WriteBuffer Data:__ Transfer 29 bytes (max)

This copies the __entire LoRa Message__ into the SX1262 Transmit Buffer as a __single (huge) chunk__.

If we try to transmit a LoRa Message that's __longer than 29 bytes__, the SPI Transfer fails.

This appears in the [__Output Log__](https://github.com/lupyuen/lora-sx1262#send-message) as...

```text
sx126x_hal_write: 
  command_length=2, 
  data_length=29
spi tx: 
  0e 00 
  50 49 4e 47 00 01 02 03 04 05 06 07 08 09 0a 0b 0c 0d 0e 0f 10 11 12 13 14 15 16 17 18 
spi rx: 
  a2 a2 
  a2 a2 a2 a2 a2 a2 a2 a2 a2 a2 a2 a2 a2 a2 a2 a2 a2 a2 a2 a2 a2 a2 a2 a2 a2 a2 a2 a2 a2 
```

[("`50 49 4e 47`" is "`PING`" followed by 0, 1, 2, ...)](https://github.com/lupyuen/lora-sx1262/blob/master/src/main.c#L213-L239)

_Can we transfer in smaller chunks instead?_

Yes! According to the [__SX1262 Datasheet__](https://www.semtech.com/products/wireless-rf/lora-core/sx1262) (pic above), we can copy the LoRa Message in __smaller chunks__ (29 bytes), by changing the __WriteBuffer Offset__...

1.  __WriteBuffer Command:__ `0x0E`

1.  __WriteBuffer Offset:__ `0x00`

1.  __WriteBuffer Data:__ Transfer first 29 bytes

1.  __WriteBuffer Command:__ `0x0E`

1.  __WriteBuffer Offset:__ `0x1D` (29 decimal)

1.  __WriteBuffer Data:__ Transfer next 29 bytes

We need to mod the code in [__sx126x_write_buffer__](https://lupyuen.github.io/articles/usb#transmit-long-message) to copy the LoRa Message in __29-byte chunks__.

_Awesome! Will this work for receiving Long LoRa Messages?_

Yep! To receive a LoRa Message, [__sx126x_read_buffer__](https://lupyuen.github.io/articles/usb#receive-long-message) sends this __ReadBuffer Command__ to SX1262 over SPI...

1.  __ReadBuffer Command:__ `0x1E`

1.  __ReadBuffer Offset:__ `0x00`

1.  __ReadBuffer NOP:__ `0x00`

1.  __ReadBuffer Data:__ Transfer 28 bytes (max)

Which appears in the [__Output Log__](https://github.com/lupyuen/lora-sx1262#receive-message) as...

```text
sx126x_hal_read: 
  command_length=3, 
  data_length=28
spi tx: 
  1e 00 00 
  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 
spi rx: 
  d2 d2 d2 
  48 65 6c 6c 6f 00 01 02 03 04 05 06 07 08 09 0a 0b 0c 0d 0e 0f 10 11 12 13 14 15 16 
```

[("`48 65 6c 6c 6f`" is "`Hello`" followed by 0, 1, 2, ...)](https://github.com/lupyuen/wisblock-lora-transmitter/blob/pinedio/src/main.cpp#L124-L148)

Instead of reading the __entire LoRa Message__ (from SX1262 Receive Buffer) in a single chunk, we should read it in __28-byte chunks__...

1.  __ReadBuffer Command:__ `0x1E`

1.  __ReadBuffer Offset:__ `0x00`

1.  __ReadBuffer NOP:__ `0x00`

1.  __ReadBuffer Data:__ Transfer first 28 bytes

1.  __ReadBuffer Command:__ `0x1E`

1.  __ReadBuffer Offset:__ `0x1C` (28 decimal)

1.  __ReadBuffer NOP:__ `0x00`

1.  __ReadBuffer Data:__ Transfer next 28 bytes

We need to fix [__sx126x_read_buffer__](https://lupyuen.github.io/articles/usb#receive-long-message) to read the LoRa Message in __28-byte chunks__.

_Is this fix for Long LoRa Messages really necessary?_

Maybe not!

Remember we need to comply with the __Local Regulations__ on the usage of [__ISM Radio Bands__](https://en.wikipedia.org/wiki/ISM_radio_band): FCC, ETSI, ...

-   [__"Regional Parameters"__](https://www.thethingsnetwork.org/docs/lorawan/regional-parameters/)

(Blasting Long LoRa Messages non-stop is no-no!)

When we connect PineDio USB to __The Things Network__, we need to comply with their Fair Use Policy...

-   [__"Fair Use of The Things Network"__](https://lupyuen.github.io/articles/ttn#fair-use-of-the-things-network)

With __CBOR Encoding__, we can compress simple LoRa Messages (Sensor Data) into 12 bytes roughly. [(See this)](https://lupyuen.github.io/articles/cbor)

Thus __28 bytes might be sufficient__ for many LoRa Applications.

(Long LoRa Messages are more prone to Radio Interference and Collisions as well)

[(But lemme know if you would like me to fix this!)](https://github.com/lupyuen/lora-sx1262/issues)

# CH341 GPIO Interface

_Besides SPI, what Interfaces do we need to control LoRa SX1262?_

PineDio USB needs a __GPIO Interface__ to control these __SX1262 Pins__...

-   __DIO1__: Used by SX1262 to signal that a LoRa Packet has been transmitted or received

    (__DIO1__ shifts from Low to High when that happens)

-   __BUSY__: Read by our driver to check if SX1262 is busy

    (__BUSY__ is High when SX1262 is busy)

-   __NRESET__: Toggled by our driver to reset the SX1262 module

We may call the GPIO Interface that's provided by the __CH341 SPI Driver__...

-   [__"Driver Development (GPIO)"__](https://wiki.pine64.org/wiki/Pinedio#Driver_development)

-   [__"Using GPIOs (CH341)"__](https://codeberg.org/JF002/spi-ch341-usb#using-gpios)

But the current PineDio USB Driver __doesn't use GPIO yet__.

![Controlling SX1262 without GPIO](https://lupyuen.github.io/images/usb-sleep3.png)

_Huh? SX1262 works without GPIO control?_

We found some sneaky workarounds to __control LoRa SX1262 without GPIO__...

-   __DIO1__: Because we don't support GPIO Interrupts (yet), we __poll the SX1262 Status every second__ to check if a LoRa Packet has been received.

    [(See this)](https://github.com/lupyuen/lora-sx1262/blob/master/src/main.c#L102-L112)

-   __BUSY__: Instead of reading this pin to check if SX1262 is busy, we __sleep 10 milliseconds__.

    [(See this)](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x-linux.c#L171-L182)

-   __NRESET__: To reset the SX1262 module, we __manually unplug PineDio USB__ and plug it back in.

    [(See this)](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x-linux.c#L149-L169)

These sneaky hacks will need to be fixed by calling the GPIO Interface.

_What needs to be fixed for GPIO?_

We need to mod these functions to call the __CH341 GPIO Interface__...

1.  Initialise the GPIO Pins: [__SX126xIoInit__](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x-linux.c#L65-L77)

    [(Explained here)](https://lupyuen.github.io/articles/lorawan#appendix-bl602-spi-functions)

1.  Register GPIO Interrupt Handler for DIO1: [__SX126xIoIrqInit__](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x-linux.c#L79-L91)

    [(Explained here)](https://lupyuen.github.io/articles/lorawan#appendix-bl602-gpio-interrupts)

1.  Reset SX1262 via GPIO: [__SX126xReset__](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x-linux.c#L149-L169)

    ```c
    void SX126xReset(void) {
        //  TODO: Set Reset pin to Low
        //  rc = bl_gpio_output_set(SX126X_NRESET, 1);
        //  assert(rc == 0);

        //  Wait 1 ms
        DelayMs(1);

        //  TODO: Configure Reset pin as a GPIO Input Pin, no pullup, no pulldown
        //  rc = bl_gpio_enable_input(SX126X_NRESET, 0, 0);
        //  assert(rc == 0);

        //  Wait 6 ms
        DelayMs(6);
    }
    ```

1.  Check SX1262 Busy State via GPIO: [__SX126xWaitOnBusy__](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x-linux.c#L171-L182)

    (__SX126xWaitOnBusy__ is called by [__SX126xCheckDeviceReady__](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x.c#L133-L142), which wakes up SX1262 before checking if SX1262 is busy)

    ```c
    void SX126xWaitOnBusy(void) {
      //  TODO: Fix the GPIO check for busy state.
      //  while( bl_gpio_input_get_value( SX126X_BUSY_PIN ) == 1 );

      //  Meanwhile we sleep 10 milliseconds
      usleep(10 * 1000);
    }
    ```

1.  Get DIO1 Pin State: [__SX126xGetDio1PinState__](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x-linux.c#L337-L344)

    ```c
    uint32_t SX126xGetDio1PinState(void) {    
      //  TODO: Read and return DIO1 Pin State
      //  return bl_gpio_input_get_value( SX126X_DIO1 );

      //  Meanwhile we always return 0
      return 0;
    }
    ```

When we have implemented [__GPIO Interrupts__](https://codeberg.org/JF002/spi-ch341-usb#reacting-on-gpio-input-interrupt) in our driver, we can remove the [__Event Polling__](https://github.com/lupyuen/lora-sx1262/blob/master/src/main.c#L102-L112). And we run a __Background Thread__ to handle LoRa Events.

Here's how we'll do multithreading...

![Multithreading with NimBLE Porting Layer](https://lupyuen.github.io/images/usb-handler.jpg)

# Multithreading with NimBLE Porting Layer

_How will we receive LoRa Messages with GPIO Interrupts?_

After we have implemented [__GPIO Interrupts__](https://codeberg.org/JF002/spi-ch341-usb#reacting-on-gpio-input-interrupt) in our PineDio USB Driver, this is how we'll __receive LoRa Messages__ (see pic above)...

1.  When SX1262 receives a LoRa Message, it triggers a __GPIO Interrupt__ on Pin DIO1

1.  CH341 Driver forwards the GPIO Interrupt to our Interrupt Handler Function [__handle_gpio_interrupt__](https://lupyuen.github.io/articles/lora2#gpio-interrupt-handler)

1.  __handle_gpio_interrupt__ enqueues an Event into our __Event Queue__

1.  Our __Background Thread__ removes the Event from the Event Queue and calls [__RadioOnDioIrq__](https://lupyuen.github.io/articles/usb#radioondioirq) to process the received LoRa Message

We handle GPIO Interrupts the same way in our __LoRa SX1262 Driver for BL602__...

-   [__"BL602 GPIO Interrupts"__](https://lupyuen.github.io/articles/lorawan#appendix-bl602-gpio-interrupts)

_Why do we need a Background Thread?_

This will allow our LoRa Application to __run without blocking__ (waiting) on incoming LoRa Messages.

This is especially useful when we implement __LoRaWAN on PineDio USB__, because LoRaWAN needs to handle __asynchronous messages__ in the background.

[(Like when we join a LoRaWAN Network)](https://lupyuen.github.io/articles/lorawan#join-network-request)

_How will we implement the Background Thread and Event Queue?_

We'll call __NimBLE Porting Layer__, the open source library for Multithreading Functions...

-   [__Multitask with NimBLE Porting Layer__](https://lupyuen.github.io/articles/lora2#multitask-with-nimble-porting-layer)

Which has been compiled into our PineDio USB Driver...

-   [__npl/linux/src__](https://github.com/lupyuen/lora-sx1262/tree/master/npl/linux/src)

The code below shall be updated to __start the Background Thread__: [main.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/main.c#L343-L413)

```c
/// TODO: Create a Background Thread to handle LoRa Events
static void create_task(void) {
  //  Init the Event Queue
  ble_npl_eventq_init(&event_queue);

  //  Init the Event
  ble_npl_event_init(
    &event,        //  Event
    handle_event,  //  Event Handler Function
    NULL           //  Argument to be passed to Event Handler
  );

  //  TODO: Create a Background Thread to process the Event Queue
  //  nimble_port_freertos_init(task_callback);
}
```

And we shall implement the GPIO Interrupt Handler Function [__handle_gpio_interrupt__](https://lupyuen.github.io/articles/lora2#gpio-interrupt-handler) for Linux.

[(We don't need to code the Event Queue, it has been done here)](https://github.com/lupyuen/lora-sx1262/blob/master/src/main.c#L343-L413)

![PineDio LoRa Family: PineDio Gateway, PinePhone Backplate and USB Adapter](https://lupyuen.github.io/images/lorawan2-pine64.jpg)

_PineDio LoRa Family: PineDio Gateway, PinePhone Backplate and USB Adapter_

# What's Next

Now that we have a __Basic LoRa Driver for PineDio USB__, we can explore all kinds of fun possibilities...

1.  Merge the PineDio USB Driver back into [__BL602 IoT SDK__](https://github.com/lupyuen/bl_iot_sdk)

    (So we can maintain a single LoRa Driver for for PineDio USB and [__PineDio Stack BL604__](https://lupyuen.github.io/articles/lorawan2))

    [__UPDATE__: We have merged PineDio USB Driver into BL602 IoT SDK!](https://github.com/lupyuen/bl_iot_sdk/commit/caae2d5219700df297411641be7286ada07f7804?diff=unified)

1.  Implement __LoRaWAN on PineDio USB__

    (So we can connect Pinebook Pro to [__The Things Network__](https://lupyuen.github.io/articles/ttn) and [__Helium__](https://www.helium.com/lorawan))

    [(We'll port the BL602 LoRaWAN Driver to Linux)](https://github.com/lupyuen/lorawan)

1.  Explore __LoRa Mesh Networks__ for PineDio USB...

    [__Meshtastic__](https://meshtastic.org/) (Data Mesh), [__QMesh__](https://hackaday.io/project/161491-qmesh-a-lora-based-voice-mesh-network) (Voice Mesh), [__Mycelium Mesh__](https://mycelium-mesh.net/) (Text Mesh)
 
1.  Test PineDio USB with __PineDio LoRa Gateway__...

    [__"PineDio LoRa Gateway: Testing The Prototype"__](https://lupyuen.github.io/articles/gateway)

1.  [__PinePhone LoRa Backplate__](https://wiki.pine64.org/wiki/Pinedio#Pinephone_backplate) (pic above) is an intriguing accessory...

    The Backplate connects PinePhone to __LoRa SX1262__ via __ATtiny84__... Which runs an [__Arduino I2C-To-SPI Bridge__](https://github.com/zschroeder6212/tiny-i2c-spi)!

    (Our PineDio USB Driver might run on PinePhone if we talk to I2C instead of SPI)

    [(Check the updates here)](https://github.com/lupyuen/pinephone-lora)

Lemme know what you would like to see!

Many Thanks to my [__GitHub Sponsors__](https://lupyuen.github.io/articles/sponsor) for supporting my work! This article wouldn't have been possible without your support.

-   [Sponsor me a coffee](https://lupyuen.github.io/articles/sponsor)

-   [Discuss this article on Reddit](https://www.reddit.com/r/PINE64official/comments/qhiuer/build_a_linux_driver_for_pinedio_lora_sx1262_usb/)

-   [Read "The RISC-V BL602 / BL604 Book"](https://lupyuen.github.io/articles/book)

-   [Check out my articles](https://lupyuen.github.io)

-   [RSS Feed](https://lupyuen.github.io/rss.xml)

_Got a question, comment or suggestion? Create an Issue or submit a Pull Request here..._

[`lupyuen.github.io/src/usb.md`](https://github.com/lupyuen/lupyuen.github.io/blob/master/src/usb.md)

# Notes

1.  This article is the expanded version of [this Twitter Thread](https://twitter.com/MisterTechBlog/status/1451548895461326858)

# Appendix: Install CH341 SPI Driver

To install __CH341 SPI Driver__ on __Pinebook Pro Manjaro Arm64__...

```bash
## Install DKMS so that we may load Kernel Drivers dynamically
sudo pacman -Syu dkms base-devel --needed

## Install Kernel Headers for Manjaro: https://linuxconfig.org/manjaro-linux-kernel-headers-installation
uname -r 
## Should show "5.14.12-1-MANJARO-ARM" or similar
sudo pacman -S linux-headers
pacman -Q | grep headers
## Should show "linux-headers 5.14.12-1" or similar

## Reboot to be safe
sudo reboot

## Install CH341 SPI Driver
git clone https://codeberg.org/JF002/spi-ch341-usb
pushd spi-ch341-usb
## TODO: Edit Makefile and change...
##   KERNEL_DIR  = /usr/src/linux-headers-$(KVERSION)/
## To...
##   KERNEL_DIR  = /lib/modules/$(KVERSION)/build
make
sudo make install
popd

## Unload the CH341 Non-SPI Driver if it has been automatically loaded
lsmod | grep ch341
sudo rmmod ch341

## Load the CH341 SPI Driver
sudo modprobe spi-ch341-usb
```

Then do this...

1.  Plug in PineDio USB

1.  Check that the CH341 SPI Driver has been correctly loaded...

    ```bash
    dmesg
    ```

1.  We should see...

    ```text
    ch341_gpio_probe: 
    registered GPIOs from 496 to 511
    ```

    The CH341 SPI Driver has been loaded successfully.

    [(See the complete log)](https://github.com/lupyuen/lora-sx1262#connect-usb)

1.  If we see...

    ```text
    spi_ch341_usb: 
    loading out-of-tree module taints kernel
    ```

    It means the __older CH341 Non-SPI Driver__ has been loaded.

    [(See the complete log)](https://github.com/lupyuen/lora-sx1262#connect-usb-failed)

To remove the __older CH341 Non-SPI Driver__...

1.  Unplug PineDio USB

1.  Enter...

    ```bash
    ## Unload the CH341 Non-SPI Driver
    sudo rmmod ch341
    ```

1.  Plug in PineDio USB

1.  Enter...

    ```bash
    dmesg
    ```

    And recheck the messages.

[(More about CH341 SPI Driver)](https://wiki.pine64.org/wiki/Pinedio#USB_adapter)

## When Rebooting

Whenever we reboot our computer...

1.  Plug in PineDio USB

1.  Check that the CH341 SPI Driver has been correctly loaded...

    ```bash
    dmesg
    ```

1.  We should see...

    ```text
    ch341_gpio_probe: 
    registered GPIOs from 496 to 511
    ```

    The CH341 SPI Driver has been loaded successfully.

    [(See the complete log)](https://github.com/lupyuen/lora-sx1262#connect-usb)

1.  If we see...

    ```text
    spi_ch341_usb: 
    loading out-of-tree module taints kernel
    ```

    It means the __older CH341 Non-SPI Driver__ has been loaded.

    [(See the complete log)](https://github.com/lupyuen/lora-sx1262#connect-usb-failed)

To remove the __older CH341 Non-SPI Driver__...

1.  Unplug PineDio USB

1.  Enter...

    ```bash
    ## Unload the CH341 Non-SPI Driver
    sudo rmmod ch341
    ```

1.  Plug in PineDio USB

1.  Enter...

    ```bash
    dmesg
    ```

    And recheck the messages.

# Appendix: Build PineDio USB Driver

To build __PineDio USB Driver__ on __Pinebook Pro Manjaro Arm64__...

1.  Follow the instructions in the previous section to install CH341 SPI Driver.

1.  Check that the CH341 SPI Driver has been loaded. (`dmesg`)

1.  Enter at the command prompt...

```bash
## Download PineDio USB Driver
git clone --recursive https://github.com/lupyuen/lora-sx1262
cd lora-sx1262

## TODO: Edit src/main.c and uncomment 
## READ_REGISTERS, SEND_MESSAGE or RECEIVE_MESSAGE.
## See "PineDio USB Operations" below.

## TODO: Edit src/main.c and update the LoRa Parameters.
## See "LoRa Parameters" above.

## Build PineDio USB Driver
make

## Run PineDio USB Driver Demo
sudo ./lora-sx1262
```

[(See the Output Log)](https://github.com/lupyuen/lora-sx1262#pinedio-usb-output-log)

[(More about PineDio USB)](https://wiki.pine64.org/wiki/Pinedio#USB_adapter)

## PineDio USB Operations

The PineDio USB Demo supports 3 operations...

1.  Read SX1262 Registers:

    Edit [__src/main.c__](https://github.com/lupyuen/lora-sx1262/blob/master/src/main.c) and uncomment...

    ```c
    #define READ_REGISTERS
    ```

    [(See the Read Register Log)](https://github.com/lupyuen/lora-sx1262#read-registers)

1.  Send LoRa Message:

    Edit [__src/main.c__](https://github.com/lupyuen/lora-sx1262/blob/master/src/main.c) and uncomment...

    ```c
    #define SEND_MESSAGE
    ```

    [(See the Send Message Log)](https://github.com/lupyuen/lora-sx1262#send-message)

1.  Receive LoRa Message:

    Edit [__src/main.c__](https://github.com/lupyuen/lora-sx1262/blob/master/src/main.c) and uncomment...

    ```c
    #define RECEIVE_MESSAGE
    ```

    [(See the Receive Message Log)](https://github.com/lupyuen/lora-sx1262#receive-message)

# Appendix: Radio Functions

In this section we explain the Platform-Independent (Linux and BL602) __Radio Functions__, which are defined in [radio.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/radio.c)

-   [__RadioInit:__](https://github.com/lupyuen/lora-sx1262/blob/master/src/radio.c#L523-L559) Initialise LoRa SX1262

-   [__RadioSetChannel:__](https://github.com/lupyuen/lora-sx1262/blob/master/src/radio.c#L600-L604) Set LoRa Frequency

-   [__RadioSetTxConfig:__](https://github.com/lupyuen/lora-sx1262/blob/master/src/radio.c#L788-L908) Set LoRa Transmit Configuration

-   [__RadioSetRxConfig:__](https://github.com/lupyuen/lora-sx1262/blob/master/src/radio.c#L661-L786) Set LoRa Receive Configuration

-   [__RadioSend:__](https://github.com/lupyuen/lora-sx1262/blob/master/src/radio.c#L1069-L1098) Transmit a LoRa Message

-   [__RadioRx:__](https://github.com/lupyuen/lora-sx1262/blob/master/src/radio.c#L1117-L1138) Receive one LoRa Message

-   [__RadioIrqProcess:__](https://github.com/lupyuen/lora-sx1262/blob/master/src/radio.c#L1314-L1460) Process Transmit and Receive Interrupts

-   [__RadioSleep:__](https://github.com/lupyuen/lora-sx1262/blob/master/src/radio.c#L1100-L1109) Switch SX1262 to low-power sleep mode

## RadioInit: Initialise LoRa Module

__RadioInit__ initialises the LoRa SX1262 Module: [radio.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/radio.c#L523-L559)

```c
void RadioInit( RadioEvents_t *events ) {
  //  We copy the Event Callbacks from "events", because
  //  "events" may be stored on the stack
  assert(events != NULL);
  memcpy(&RadioEvents, events, sizeof(RadioEvents));
  //  Previously: RadioEvents = events;
```

The function begins by copying the list of __Radio Event Callbacks__...

-   __TxDone__: Called when a LoRa Message has been transmitted

-   __RxDone__: Called when a LoRa Message has been received

-   __TxTimeout__: Called upon timeout when transmitting a LoRa Message

-   __RxTimeout__: Called upon timeout when receiving a LoRa Message

-   __RxError__: Called when a LoRa Message has been received with CRC Error

This differs from the Semtech Reference Implementation, which copies the pointer to __RadioEvents_t__ instead of the entire __RadioEvents_t__.

(Which causes problems when __RadioEvents_t__ lives on the stack)

Next we __init the SPI and GPIO Ports__, wake up the LoRa Module, and init the TCXO Control and RF Switch Control.

```c
  //  Init SPI and GPIO Ports, wake up the LoRa Module,
  //  init TCXO Control and RF Switch Control.
  SX126xInit( RadioOnDioIrq );
```

[(__SX126xInit__ is defined here)](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x.c#L112-L131)

[(__RadioOnDioIrq__ is explained here)](https://lupyuen.github.io/articles/usb#radioondioirq)

We set the LoRa Module to __Standby Mode__...

```c
  //  Set LoRa Module to standby mode
  SX126xSetStandby( STDBY_RC );
```

We set the __Power Regulation: LDO or DC-DC__...

```c
  //  TODO: Declare the power regulation used to power the device
  //  This command allows the user to specify if DC-DC or LDO is used for power regulation.
  //  Using only LDO implies that the Rx or Tx current is doubled

  //  #warning SX126x is set to LDO power regulator mode (instead of DC-DC)
  //  SX126xSetRegulatorMode( USE_LDO );   //  Use LDO

  //  #warning SX126x is set to DC-DC power regulator mode (instead of LDO)
  SX126xSetRegulatorMode( USE_DCDC );  //  Use DC-DC
```

[(__SX126xSetRegulatorMode__ is defined here)](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x.c#L390-L393)

This depends on how our LoRa Module is wired for power.

For now we're using __DC-DC__ Power Regulation. (To be verified)

[(More about LDO vs DC-DC Power Regulation)](https://lupyuen.github.io/articles/lorawan#dc-dc-vs-ldo)

We set the __Base Addresses__ of the Read and Write Buffers to 0...

```c
  //  Set the base addresses of the Read and Write Buffers to 0
  SX126xSetBufferBaseAddress( 0x00, 0x00 );
```

[(__SX126xSetBufferBaseAddress__ is defined here)](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x.c#L697-L704)

The Read and Write Buffers are accessed by [__sx126x_read_buffer__](https://lupyuen.github.io/articles/usb#receive-long-message) and [__sx126x_write_buffer__](https://lupyuen.github.io/articles/usb#transmit-long-message).

We set the __Transmit Power__ and the __Ramp Up Time__...

```c
  //  TODO: Set the correct transmit power and ramp up time
  SX126xSetTxParams( 22, RADIO_RAMP_3400_US );
  //  TODO: Previously: SX126xSetTxParams( 0, RADIO_RAMP_200_US );
```

[(__SX126xSetTxParams__ is defined here)](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x.c#L528-L576)

__Ramp Up Time__ is the duration (in microseconds) we need to wait for SX1262's Power Amplifier to ramp up (charge up) to the configured Transmit Power.

For easier testing we have set the Transmit Power to __22 dBm__ (highest power) and Ramp Up Time to __3400 microseconds__ (longest duration).

(To give sufficient time for the Power Amplifier to ramp up to the highest Transmit Power)

After testing we should revert to the __Default Transmit Power__ (0) and __Ramp Up Time__ (200 microseconds).

[(More about the Transmit Power)](https://lupyuen.github.io/articles/lorawan#transmit-power)

[(Over Current Protection in __SX126xSetTxParams__)](https://lupyuen.github.io/articles/lorawan#over-current-protection)

We configure which __LoRa Events will trigger interrupts__ on each DIO Pin...

```c
  //  Set the DIO Interrupt Events:
  //  All LoRa Events will trigger interrupts on DIO1
  SX126xSetDioIrqParams(
    IRQ_RADIO_ALL,   //  Interrupt Mask
    IRQ_RADIO_ALL,   //  Interrupt Events for DIO1
    IRQ_RADIO_NONE,  //  Interrupt Events for DIO2
    IRQ_RADIO_NONE   //  Interrupt Events for DIO3
  );
```

[(__SX126xSetDioIrqParams__ is defined here)](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x.c#L457-L470)

(All LoRa Events will trigger interrupts on DIO1)

We define the SX1262 Registers that will be restored from __Retention Memory__ when waking up from Warm Start Mode...

```c
  //  Add registers to the retention list (4 is the maximum possible number)
  RadioAddRegisterToRetentionList( REG_RX_GAIN );
  RadioAddRegisterToRetentionList( REG_TX_MODULATION );
```

[(__RadioAddRegisterToRetentionList__ is defined here)](https://github.com/lupyuen/lora-sx1262/blob/master/src/radio.c#L1167-L1195)

Finally we init the Timeout Timers (from NimBLE Porting Layer) for __Transmit Timeout__ and __Receive Timeout__...

```c
  //  Initialize driver timeout timers
  TimerInit( &TxTimeoutTimer, RadioOnTxTimeoutIrq );
  TimerInit( &RxTimeoutTimer, RadioOnRxTimeoutIrq );

  //  Interrupt not fired yet
  IrqFired = false;
}
```

[(__TimerInit__ is defined here)](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x-linux.c#L361-L380)

## RadioSetChannel: Set LoRa Frequency

__RadioSetChannel__ sets the LoRa Frequency: [radio.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/radio.c#L600-L604)

```c
void RadioSetChannel( uint32_t freq ) {
  SX126xSetRfFrequency( freq );
}
```

__RadioSetChannel__ passes the LoRa Frequency (like `923000000` for 923 MHz) to __SX126xSetRfFrequency__.

__SX126xSetRfFrequency__ is defined as follows: [sx126x.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x.c#L497-L514)

```c
void SX126xSetRfFrequency( uint32_t frequency ) {
  uint8_t buf[4];
  if( ImageCalibrated == false ) {
    SX126xCalibrateImage( frequency );
    ImageCalibrated = true;
  }
  uint32_t freqInPllSteps = SX126xConvertFreqInHzToPllStep( frequency );
  buf[0] = ( uint8_t )( ( freqInPllSteps >> 24 ) & 0xFF );
  buf[1] = ( uint8_t )( ( freqInPllSteps >> 16 ) & 0xFF );
  buf[2] = ( uint8_t )( ( freqInPllSteps >> 8 ) & 0xFF );
  buf[3] = ( uint8_t )( freqInPllSteps & 0xFF );
  SX126xWriteCommand( RADIO_SET_RFFREQUENCY, buf, 4 );
}
```

[(__SX126xCalibrateImage__ is defined here)](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x.c#L408-L438)

[(__SX126xConvertFreqInHzToPllStep__ is defined here)](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x.c#L812-L826)

[(__SX126xWriteCommand__ is defined here)](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x-linux.c#L204-L217)

## RadioSetTxConfig: Set Transmit Configuration

__RadioSetTxConfig__ sets the LoRa Transmit Configuration: [radio.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/radio.c#L788-L908)

```c
void RadioSetTxConfig( RadioModems_t modem, int8_t power, uint32_t fdev,
  uint32_t bandwidth, uint32_t datarate,
  uint8_t coderate, uint16_t preambleLen,
  bool fixLen, bool crcOn, bool freqHopOn,
  uint8_t hopPeriod, bool iqInverted, uint32_t timeout ) {

  //  LoRa Modulation or FSK Modulation?
  switch( modem ) {
    case MODEM_FSK:
      //  Omitted: FSK Modulation
      ...
```

Since we're using __LoRa Modulation__ instead of FSK Modulation, we skip the section on FSK Modulation.

We begin by populating the __Modulation Parameters__: Spreading Factor, Bandwidth and Coding Rate...

```c
    case MODEM_LORA:
      //  LoRa Modulation
      SX126x.ModulationParams.PacketType = 
        PACKET_TYPE_LORA;
      SX126x.ModulationParams.Params.LoRa.SpreadingFactor = 
        ( RadioLoRaSpreadingFactors_t ) datarate;
      SX126x.ModulationParams.Params.LoRa.Bandwidth =  
        Bandwidths[bandwidth];
      SX126x.ModulationParams.Params.LoRa.CodingRate = 
        ( RadioLoRaCodingRates_t )coderate;
```

Depending on the LoRa Parameters, we optimise for __Low Data Rate__...

```c
      //  Optimise for Low Data Rate
      if( ( ( bandwidth == 0 ) && ( ( datarate == 11 ) || ( datarate == 12 ) ) ) ||
      ( ( bandwidth == 1 ) && ( datarate == 12 ) ) ) {
        SX126x.ModulationParams.Params.LoRa.LowDatarateOptimize = 0x01;
      } else {
        SX126x.ModulationParams.Params.LoRa.LowDatarateOptimize = 0x00;
      }
```

Next we populate the __Packet Parameters__: Preamble Length, Header Type, Payload Length, CRC Mode and Invert IQ...

```c
      //  Populate Packet Type
      SX126x.PacketParams.PacketType = PACKET_TYPE_LORA;

      //  Populate Preamble Length
      if( ( SX126x.ModulationParams.Params.LoRa.SpreadingFactor == LORA_SF5 ) ||
        ( SX126x.ModulationParams.Params.LoRa.SpreadingFactor == LORA_SF6 ) ) {
        if( preambleLen < 12 ) {
          SX126x.PacketParams.Params.LoRa.PreambleLength = 12;
        } else {
          SX126x.PacketParams.Params.LoRa.PreambleLength = preambleLen;
        }
      } else {
        SX126x.PacketParams.Params.LoRa.PreambleLength = preambleLen;
      }

      //  Populate Header Type, Payload Length, CRC Mode and Invert IQ
      SX126x.PacketParams.Params.LoRa.HeaderType = 
        ( RadioLoRaPacketLengthsMode_t )fixLen;
      SX126x.PacketParams.Params.LoRa.PayloadLength = 
        MaxPayloadLength;
      SX126x.PacketParams.Params.LoRa.CrcMode = 
        ( RadioLoRaCrcModes_t )crcOn;
      SX126x.PacketParams.Params.LoRa.InvertIQ = 
        ( RadioLoRaIQModes_t )iqInverted;
```

We set the LoRa Module to __Standby Mode__ and configure it for __LoRa Modulation__ (or FSK Modulation)...

```c
      //  Set LoRa Module to Standby Mode
      RadioStandby( );

      //  Configure LoRa Module for LoRa Modulation (or FSK Modulation)
      RadioSetModem( 
        ( SX126x.ModulationParams.PacketType == PACKET_TYPE_GFSK ) 
        ? MODEM_FSK 
        : MODEM_LORA
      );
```

[(__RadioStandby__ is defined here)](https://github.com/lupyuen/lora-sx1262/blob/master/src/radio.c#L1111-L1115)

[(__RadioSetModem__ is defined here)](https://github.com/lupyuen/lora-sx1262/blob/master/src/radio.c#L576-L598)

We configure the LoRa Module with the __Modulation Parameters__ and __Packet Parameters__...

```c
      //  Configure Modulation Parameters
      SX126xSetModulationParams( &SX126x.ModulationParams );

      //  Configure Packet Parameters
      SX126xSetPacketParams( &SX126x.PacketParams );
      break;
  }
```

[(__SX126xSetModulationParams__ is defined here)](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x.c#L578-L621)

[(__SX126xSetPacketParams__ is defined here)](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x.c#L623-L680)

This is a Workaround for __Modulation Quality__ with __500 kHz Bandwidth__...

```c
  // WORKAROUND - Modulation Quality with 500 kHz LoRa Bandwidth, see DS_SX1261-2_V1.2 datasheet chapter 15.1
  if( ( modem == MODEM_LORA ) && ( SX126x.ModulationParams.Params.LoRa.Bandwidth == LORA_BW_500 ) ) {
    SX126xWriteRegister( 
      REG_TX_MODULATION, 
      SX126xReadRegister( REG_TX_MODULATION ) & ~( 1 << 2 ) 
    );
  } else {
    SX126xWriteRegister( 
      REG_TX_MODULATION, 
      SX126xReadRegister( REG_TX_MODULATION ) | ( 1 << 2 ) 
    );
  }
  // WORKAROUND END
```

[(__SX126xWriteRegister__ is defined here)](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x-linux.c#L263-L266)

[(__SX126xReadRegister__ is defined here)](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x-linux.c#L276-L281)

We finish by setting the __Transmit Power__ and __Transmit Timeout__...

```c
  //  Set Transmit Power
  SX126xSetRfTxPower( power );

  //  Set Transmit Timeout
  TxTimeout = timeout;
}
```

__SX126xSetRfTxPower__ is defined in [sx126x-linux.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x-linux.c#L299-L304)...

```c
void SX126xSetRfTxPower( int8_t power ) {
  //  TODO: Previously: SX126xSetTxParams( power, RADIO_RAMP_40_US );
  SX126xSetTxParams( power, RADIO_RAMP_3400_US );  //  TODO
}
```

For easier testing we have set the Ramp Up Time to __3400 microseconds__ (longest duration).

After testing we should revert to the __Default Ramp Up Time__ (40 microseconds).

## RadioSetRxConfig: Set Receive Configuration

__RadioSetRxConfig__ sets the LoRa Receive Configuration: [radio.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/radio.c#L661-L786)

```c
void RadioSetRxConfig( RadioModems_t modem, uint32_t bandwidth,
  uint32_t datarate, uint8_t coderate,
  uint32_t bandwidthAfc, uint16_t preambleLen,
  uint16_t symbTimeout, bool fixLen,
  uint8_t payloadLen,
  bool crcOn, bool freqHopOn, uint8_t hopPeriod,
  bool iqInverted, bool rxContinuous ) {

  //  Set Symbol Timeout
  RxContinuous = rxContinuous;
  if( rxContinuous == true ) {
    symbTimeout = 0;
  }

  //  Set Max Payload Length
  if( fixLen == true ) {
    MaxPayloadLength = payloadLen;
  }
  else {
    MaxPayloadLength = 0xFF;
  }
```

We begin by setting the __Symbol Timeout__ and __Max Payload Length__.

Since we're using __LoRa Modulation__ instead of FSK Modulation, we skip the section on FSK Modulation...

```c
  //  LoRa Modulation or FSK Modulation?
  switch( modem )
  {
    case MODEM_FSK:
      //  Omitted: FSK Modulation
      ...
```

We populate the __Modulation Parameters__: Spreading Factor, Bandwidth and Coding Rate...

```c
    case MODEM_LORA:
      //  LoRa Modulation
      SX126xSetStopRxTimerOnPreambleDetect( false );
      SX126x.ModulationParams.PacketType = 
        PACKET_TYPE_LORA;
      SX126x.ModulationParams.Params.LoRa.SpreadingFactor = 
        ( RadioLoRaSpreadingFactors_t )datarate;
      SX126x.ModulationParams.Params.LoRa.Bandwidth = 
        Bandwidths[bandwidth];
      SX126x.ModulationParams.Params.LoRa.CodingRate = 
        ( RadioLoRaCodingRates_t )coderate;
```

Depending on the LoRa Parameters, we optimise for __Low Data Rate__...

```c
      //  Optimise for Low Data Rate
      if( ( ( bandwidth == 0 ) && ( ( datarate == 11 ) || ( datarate == 12 ) ) ) ||
      ( ( bandwidth == 1 ) && ( datarate == 12 ) ) ) {
        SX126x.ModulationParams.Params.LoRa.LowDatarateOptimize = 0x01;
      } else {
        SX126x.ModulationParams.Params.LoRa.LowDatarateOptimize = 0x00;
      }
```

We populate the __Packet Parameters__: Preamble Length, Header Type, Payload Length, CRC Mode and Invert IQ...

```c
      //  Populate Packet Type
      SX126x.PacketParams.PacketType = PACKET_TYPE_LORA;

      //  Populate Preamble Length
      if( ( SX126x.ModulationParams.Params.LoRa.SpreadingFactor == LORA_SF5 ) ||
          ( SX126x.ModulationParams.Params.LoRa.SpreadingFactor == LORA_SF6 ) ){
        if( preambleLen < 12 ) {
          SX126x.PacketParams.Params.LoRa.PreambleLength = 12;
        } else {
          SX126x.PacketParams.Params.LoRa.PreambleLength = preambleLen;
        }
      } else {
        SX126x.PacketParams.Params.LoRa.PreambleLength = preambleLen;
      }

      //  Populate Header Type, Payload Length, CRC Mode and Invert IQ
      SX126x.PacketParams.Params.LoRa.HeaderType = 
        ( RadioLoRaPacketLengthsMode_t )fixLen;
      SX126x.PacketParams.Params.LoRa.PayloadLength = 
        MaxPayloadLength;
      SX126x.PacketParams.Params.LoRa.CrcMode = 
        ( RadioLoRaCrcModes_t )crcOn;
      SX126x.PacketParams.Params.LoRa.InvertIQ = 
        ( RadioLoRaIQModes_t )iqInverted;
```

We set the LoRa Module to __Standby Mode__ and configure it for __LoRa Modulation__ (or FSK Modulation)...

```c
      //  Set LoRa Module to Standby Mode
      RadioStandby( );

      //  Configure LoRa Module for LoRa Modulation (or FSK Modulation)
      RadioSetModem( 
          ( SX126x.ModulationParams.PacketType == PACKET_TYPE_GFSK ) 
          ? MODEM_FSK 
          : MODEM_LORA 
      );
```

[(__RadioStandby__ is defined here)](https://github.com/lupyuen/lora-sx1262/blob/master/src/radio.c#L1111-L1115)

[(__RadioSetModem__ is defined here)](https://github.com/lupyuen/lora-sx1262/blob/master/src/radio.c#L576-L598)

We configure the LoRa Module with the __Modulation Parameters__, __Packet Parameters__ and Symbol Timeout...

```c
      //  Configure Modulation Parameters
      SX126xSetModulationParams( &SX126x.ModulationParams );

      //  Configure Packet Parameters
      SX126xSetPacketParams( &SX126x.PacketParams );

      //  Configure Symbol Timeout
      SX126xSetLoRaSymbNumTimeout( symbTimeout );
```

[(__SX126xSetModulationParams__ is defined here)](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x.c#L578-L621)

[(__SX126xSetPacketParams__ is defined here)](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x.c#L623-L680)

[(__SX126xSetLoRaSymbNumTimeout__ is defined here)](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x.c#L366-L388)

This is a Workaround that __optimises the Inverted IQ Operation__...

```c
      // WORKAROUND - Optimizing the Inverted IQ Operation, see DS_SX1261-2_V1.2 datasheet chapter 15.4
      if( SX126x.PacketParams.Params.LoRa.InvertIQ == LORA_IQ_INVERTED ) {
        SX126xWriteRegister( 
          REG_IQ_POLARITY, 
          SX126xReadRegister( REG_IQ_POLARITY ) & ~( 1 << 2 ) 
        );
      } else {
        SX126xWriteRegister( 
          REG_IQ_POLARITY, 
          SX126xReadRegister( REG_IQ_POLARITY ) | ( 1 << 2 ) 
        );
      }
      // WORKAROUND END
```

[(__SX126xWriteRegister__ is defined here)](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x-linux.c#L263-L266)

[(__SX126xReadRegister__ is defined here)](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x-linux.c#L276-L281)

We finish by setting the __Receive Timeout__ to No Timeout (always receiving)...

```c
      // Timeout Max, Timeout handled directly in SetRx function
      RxTimeout = 0xFFFF;
      break;
  }
}
```

## RadioSend: Transmit Message

__RadioSend__ transmits a LoRa Message: [radio.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/radio.c#L1069-L1098)

```c
void RadioSend( uint8_t *buffer, uint8_t size ) {

  //  Set the DIO Interrupt Events:
  //  Transmit Done and Transmit Timeout
  //  will trigger interrupts on DIO1
  SX126xSetDioIrqParams( 
    IRQ_TX_DONE | IRQ_RX_TX_TIMEOUT,  //  Interrupt Mask
    IRQ_TX_DONE | IRQ_RX_TX_TIMEOUT,  //  Interrupt Events for DIO1
    IRQ_RADIO_NONE,  //  Interrupt Events for DIO2
    IRQ_RADIO_NONE   //  Interrupt Events for DIO3
  );
```

[(__SX126xSetDioIrqParams__ is defined here)](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x.c#L457-L470)

We begin by configuring which __LoRa Events will trigger interrupts__ on each DIO Pin.

(Transmit Done and Transmit Timeout will trigger interrupts on DIO1)

Next we configure the __Packet Parameters__...

```c
  //  Populate the payload length
  if( SX126xGetPacketType( ) == PACKET_TYPE_LORA ) {
    SX126x.PacketParams.Params.LoRa.PayloadLength = size;
  } else {
    SX126x.PacketParams.Params.Gfsk.PayloadLength = size;
  }
  //  Configure the packet parameters
  SX126xSetPacketParams( &SX126x.PacketParams );
```

[(__SX126xGetPacketType__ is defined here)](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x.c#L523-L526)

[(__SX126xSetPacketParams__ is defined here)](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x.c#L623-L680)

We finish by sending the __Message Payload__ and starting the __Transmit Timer__...

```c
  //  Send message payload
  SX126xSendPayload( buffer, size, 0 );

  //  Start Transmit Timer
  TimerStart( &TxTimeoutTimer, TxTimeout );
}
```

[(__TimerStart__ is defined here)](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x-linux.c#L396-L421)

__SX126xSendPayload__ is defined below: [sx126x.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x.c#L162-L166)

```c
///  Send message payload
void SX126xSendPayload( uint8_t *payload, uint8_t size, uint32_t timeout ) {
  //  Copy message payload to Transmit Buffer
  SX126xSetPayload( payload, size );

  //  Transmit the buffer
  SX126xSetTx( timeout );
}
```

[(__SX126xSetTx__ is defined here)](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x.c#L289-L299)

This code copies the Message Payload to the SX1262 __Transmit Buffer__ and transmits the message.

__SX126xSetPayload__ copies to the Transmit Buffer by calling __SX126xWriteBuffer__: [sx126x.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x.c#L144-L147)

```c
/// Copy message payload to Transmit Buffer
void SX126xSetPayload( uint8_t *payload, uint8_t size ) {
  //  Copy message payload to Transmit Buffer
  SX126xWriteBuffer( 0x00, payload, size );
}
```

__SX126xWriteBuffer__ wakes up the LoRa Module, writes to the Transmit Buffer and waits for the operation to be completed: [sx126x.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x-linux.c#L283-L289)

```c
/// Copy message payload to Transmit Buffer
void SX126xWriteBuffer( uint8_t offset, uint8_t *buffer, uint8_t size ) {
  //  Wake up SX1262 if sleeping
  SX126xCheckDeviceReady( );

  //  Copy message payload to Transmit Buffer
  int rc = sx126x_write_buffer(NULL, offset, buffer, size);
  assert(rc == 0);

  //  Wait for SX1262 to be ready
  SX126xWaitOnBusy( );
}
```

[(__SX126xCheckDeviceReady__ is defined here)](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x.c#L133-L142)

[(__sx126x_write_buffer__ is explained here)](https://lupyuen.github.io/articles/usb#transmit-long-message)

[(__SX126xWaitOnBusy__ is defined here)](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x-linux.c#L171-L182)

When the LoRa Message is transmitted (successfully or unsuccessfully), the LoRa Module triggers a __DIO1 Interrupt__.

Our driver calls __RadioIrqProcess__ to process the interrupt. [(See this)](https://lupyuen.github.io/articles/usb#radioirqprocess-process-transmit-and-receive-interrupts)

## RadioRx: Receive Message

__RadioRx__ receives a single LoRa Message: [radio.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/radio.c#L1117-L1138)

```c
void RadioRx( uint32_t timeout ) {

  //  Set the DIO Interrupt Events:
  //  All LoRa Events will trigger interrupts on DIO1
  SX126xSetDioIrqParams(
    IRQ_RADIO_ALL,   //  Interrupt Mask
    IRQ_RADIO_ALL,   //  Interrupt Events for DIO1
    IRQ_RADIO_NONE,  //  Interrupt Events for DIO2
    IRQ_RADIO_NONE   //  Interrupt Events for DIO3
  );
```

[(__SX126xSetDioIrqParams__ is defined here)](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x.c#L457-L470)

We begin by configuring which __LoRa Events will trigger interrupts__ on each DIO Pin.

(All LoRa Events will trigger interrupts on DIO1)

We start the __Receive Timer__ to catch Receive Timeouts...

```c
  //  Start the Receive Timer
  if( timeout != 0 ) {
    TimerStart( &RxTimeoutTimer, timeout );
  }
```

[(__TimerStart__ is defined here)](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x-linux.c#L396-L421)

Now we begin to __receive a LoRa Message__ continuously, or until a timeout occurs...

```c
  if( RxContinuous == true ) {
    //  Receive continuously
    SX126xSetRx( 0xFFFFFF ); // Rx Continuous
  } else {
    //  Receive with timeout
    SX126xSetRx( RxTimeout << 6 );
  }
}
```

__SX126xSetRx__ enters Receive Mode like so: [sx126x.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x.c#L301-L313)

```c
void SX126xSetRx( uint32_t timeout ) {
  uint8_t buf[3];

  //  Remember we're in Receive Mode
  SX126xSetOperatingMode( MODE_RX );

  //  Configure Receive Gain
  SX126xWriteRegister( REG_RX_GAIN, 0x94 ); // default gain

  //  Enter Receive Mode
  buf[0] = ( uint8_t )( ( timeout >> 16 ) & 0xFF );
  buf[1] = ( uint8_t )( ( timeout >> 8 ) & 0xFF );
  buf[2] = ( uint8_t )( timeout & 0xFF );
  SX126xWriteCommand( RADIO_SET_RX, buf, 3 );
}
```

[(__SX126xSetOperatingMode__ is defined here)](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x-linux.c#L126-L147)

[(__SX126xWriteRegister__ is defined here)](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x-linux.c#L263-L266)

[(__SX126xWriteCommand__ is defined here)](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x-linux.c#L204-L217)

When a LoRa Message is received (successfully or unsuccessfully), the LoRa Module triggers a __DIO1 Interrupt__.

Our driver calls __RadioIrqProcess__ to process the interrupt, which is explained next...

## RadioIrqProcess: Process Transmit and Receive Interrupts

__RadioIrqProcess__ processes the interrupts that are triggered when a LoRa Message is transmitted and received: [radio.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/radio.c#L1314-L1460)

```c
/// Process Transmit and Receive Interrupts.
/// For BL602: Must be run in the Application
/// Task Context, not Interrupt Context because 
/// we will call printf and SPI Functions here.
void RadioIrqProcess( void ) {

  //  Remember and clear Interrupt Flag
  CRITICAL_SECTION_BEGIN( );
  const bool isIrqFired = IrqFired;
  IrqFired = false;
  CRITICAL_SECTION_END( );
```

[(Note: Critical Sections are not yet implemented)](https://github.com/lupyuen/lora-sx1262/blob/master/include/sx126x-board.h#L58-L60)

The function begins by copying the __Interrupt Flag__ and clearing the flag.

[(The Interrupt Flag is set by __RadioOnDioIrq__)](https://lupyuen.github.io/articles/usb#radioondioirq)

The rest of the function will run only if the __Interrupt Flag was originally set__...

```c
  //  IrqFired must be true to process interrupts
  if( isIrqFired == true ) {
    //  Get the Interrupt Status
    uint16_t irqRegs = SX126xGetIrqStatus( );

    //  Clear the Interrupt Status
    SX126xClearIrqStatus( irqRegs );
```

[(__SX126xGetIrqStatus__ is defined here)](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x.c#L472-L478)

[(__SX126xClearIrqStatus__ is defined here)](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x.c#L803-L810)

This code fetches the __Interrupt Status__ from the LoRa Module and clears the Interrupt Status.

If DIO1 is still High, we set the __Interrupt Flag__ for future processing...

```c
    //  Check if DIO1 pin is High. If it is the case revert IrqFired to true
    CRITICAL_SECTION_BEGIN( );
    if( SX126xGetDio1PinState( ) == 1 ) {
      IrqFired = true;
    }
    CRITICAL_SECTION_END( );
```

__Interrupt Status__ tells us which LoRa Events have just occurred. We handle the LoRa Events accordingly...

-   Transmit Done

-   Receive Done

-   CAD Done

-   Transmit / Receive Timeout

-   Preamble Detected

-   Sync Word Valid

-   Header Valid

-   Header Error

### Transmit Done

When the LoRa Module has transmitted a LoRa Message successfully, we stop the Transmit Timer and call the __Callback Function for Transmit Done__: [radio.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/radio.c#L1339-L1349)

```c
    //  If a LoRa Message was transmitted successfully...
    if( ( irqRegs & IRQ_TX_DONE ) == IRQ_TX_DONE ) {

      //  Stop the Transmit Timer
      TimerStop( &TxTimeoutTimer );

      //!< Update operating mode state to a value lower than \ref MODE_STDBY_XOSC
      SX126xSetOperatingMode( MODE_STDBY_RC );

      //  Call the Callback Function for Transmit Done
      if( ( RadioEvents.TxDone != NULL ) ) {
        RadioEvents.TxDone( );
      }
    }
```

[(__TimerStop__ is defined here)](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x-linux.c#L382-L394)

[(__SX126xSetOperatingMode__ is defined here)](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x-linux.c#L126-L147)

__TxDone__ points to the __on_tx_done__ Callback Function that we've seen earlier.

### Receive Done

When the LoRa Module receives a LoRa Message, we stop the Receive Timer: [radio.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/radio.c#L1351-L1389)

```c
    //  If a LoRa Message was received...
    if( ( irqRegs & IRQ_RX_DONE ) == IRQ_RX_DONE ) {

      //  Stop the Receive Timer
      TimerStop( &RxTimeoutTimer );
```

In case of CRC Error, we call the __Callback Function for Receive Error__...

```c
      if( ( irqRegs & IRQ_CRC_ERROR ) == IRQ_CRC_ERROR ) {

        //  If the received message has CRC Error...
        if( RxContinuous == false ) {
          //!< Update operating mode state to a value lower than \ref MODE_STDBY_XOSC
          SX126xSetOperatingMode( MODE_STDBY_RC );
        }

        //  Call the Callback Function for Receive Error
        if( ( RadioEvents.RxError ) ) {
          RadioEvents.RxError( );
        }
```

__RxError__ points to the __on_rx_error__ Callback Function that we've seen earlier.

If the received message has no CRC Error, we do this Workaround for __Implicit Header Mode Timeout Behavior__...

```c
      } else {
        //  If the received message has no CRC Error...
        uint8_t size;

        //  If we are receiving continuously...
        if( RxContinuous == false ) {
          //!< Update operating mode state to a value lower than \ref MODE_STDBY_XOSC
          SX126xSetOperatingMode( MODE_STDBY_RC );

          // WORKAROUND - Implicit Header Mode Timeout Behavior, see DS_SX1261-2_V1.2 datasheet chapter 15.3
          SX126xWriteRegister( REG_RTC_CTRL, 0x00 );
          SX126xWriteRegister( 
            REG_EVT_CLR, 
            SX126xReadRegister( REG_EVT_CLR ) | ( 1 << 1 ) 
          );
          // WORKAROUND END
        }
```

[(__SX126xWriteRegister__ is defined here)](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x-linux.c#L263-L266)

[(__SX126xReadRegister__ is defined here)](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x-linux.c#L276-L281)

Then we copy the __Received Message Payload__ and get the __Packet Status__...

```c
        //  Copy the Received Message Payload (max 255 bytes)
        SX126xGetPayload( RadioRxPayload, &size , 255 );
        
        //  Get the Packet Status:
        //  Packet Signal Strength (RSSI), Signal-to-Noise Ratio (SNR),
        //  Signal RSSI, Frequency Error
        SX126xGetPacketStatus( &RadioPktStatus );
```

[(__SX126xGetPacketStatus__ is defined here)](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x.c#L746-L778)

And we call the __Callback Function for Receive Done__...

```c
        //  Call the Callback Function for Receive Done
        if( ( RadioEvents.RxDone != NULL ) ) {
          RadioEvents.RxDone( 
            RadioRxPayload, 
            size, 
            RadioPktStatus.Params.LoRa.RssiPkt, 
            RadioPktStatus.Params.LoRa.SnrPkt 
          );
        }
      }
    }
```

__RxDone__ points to the __on_rx_done__ Callback Function that we've seen earlier.

__SX126xGetPayload__ copies the received message payload from the SX1262 __Receive Buffer__: [sx126x.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x.c#L149-L160)

```c
/// Copy message payload from Receive Buffer
uint8_t SX126xGetPayload( uint8_t *buffer, uint8_t *size,  uint8_t maxSize ) {
  uint8_t offset = 0;

  //  Get the size and offset of the received message
  //  in the Receive Buffer
  SX126xGetRxBufferStatus( size, &offset );
  if( *size > maxSize ) {
    return 1;
  }

  //  Copy message payload from Receive Buffer
  SX126xReadBuffer( offset, buffer, *size );
  return 0;
}
```

[(__SX126xGetRxBufferStatus__ is defined here)](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x.c#L727-L744)

__SX126xReadBuffer__ wakes up the LoRa Module, reads from the Receive Buffer and waits for the operation to be completed: [sx126x-linux.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x-linux.c#L291-L297)

```c
/// Copy message payload from Receive Buffer
void SX126xReadBuffer( uint8_t offset, uint8_t *buffer, uint8_t size ) {
  //  Wake up SX1262 if sleeping
  SX126xCheckDeviceReady( );

  //  Copy message payload from Receive Buffer
  int rc = sx126x_read_buffer(NULL, offset, buffer, size);
  assert(rc == 0);

  //  Wait for SX1262 to be ready
  SX126xWaitOnBusy( );
}
```

[(__SX126xCheckDeviceReady__ is defined here)](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x.c#L133-L142)

[(__sx126x_read_buffer__ is explained here)](https://lupyuen.github.io/articles/usb#receive-long-message)

[(__SX126xWaitOnBusy__ is defined here)](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x-linux.c#L171-L182)

### CAD Done

__Channel Activity Detection__ lets us __detect whether there's any ongoing transmission__ in a LoRa Radio Channel, in a power-efficient way.

We won't be doing Channel Activity Detection in our driver: [radio.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/radio.c#L1391-L1400)

```c
    //  If Channel Activity Detection is complete...
    if( ( irqRegs & IRQ_CAD_DONE ) == IRQ_CAD_DONE ) {

      //!< Update operating mode state to a value lower than \ref MODE_STDBY_XOSC
      SX126xSetOperatingMode( MODE_STDBY_RC );

      //  Call Callback Function for CAD Done
      if( ( RadioEvents.CadDone != NULL ) ) {
        RadioEvents.CadDone( ( 
            ( irqRegs & IRQ_CAD_ACTIVITY_DETECTED ) 
            == IRQ_CAD_ACTIVITY_DETECTED 
        ) );
      }
    }
```

### Transmit / Receive Timeout

When the LoRa Module __fails to transmit__ a LoRa Message due to Timeout, we stop the Transmit Timer and call the __Callback Function for Transmit Timeout__: [radio.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/radio.c#L1402-L1425)

```c
    //  If a LoRa Message failed to Transmit or Receive due to Timeout...
    if( ( irqRegs & IRQ_RX_TX_TIMEOUT ) == IRQ_RX_TX_TIMEOUT ) {

      //  If the message failed to Transmit due to Timeout...
      if( SX126xGetOperatingMode( ) == MODE_TX ) {

        //  Stop the Transmit Timer
        TimerStop( &TxTimeoutTimer );

        //!< Update operating mode state to a value lower than \ref MODE_STDBY_XOSC
        SX126xSetOperatingMode( MODE_STDBY_RC );

        //  Call the Callback Function for Transmit Timeout
        if( ( RadioEvents.TxTimeout != NULL ) ) {
          RadioEvents.TxTimeout( );
        }
      }
```

[(__SX126xGetOperatingMode__ is defined here)](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x-linux.c#L121-L124)

__TxTimeout__ points to the __on_tx_timeout__ Callback Function that we've seen earlier.

When the LoRa Module __fails to receive__ a LoRa Message due to Timeout, we stop the Receive Timer and call the __Callback Function for Receive Timeout__...

```c
      //  If the message failed to Receive due to Timeout...
      else if( SX126xGetOperatingMode( ) == MODE_RX ) {

        //  Stop the Receive Timer
        TimerStop( &RxTimeoutTimer );

        //!< Update operating mode state to a value lower than \ref MODE_STDBY_XOSC
        SX126xSetOperatingMode( MODE_STDBY_RC );

        //  Call the Callback Function for Receive Timeout
        if( ( RadioEvents.RxTimeout != NULL ) ) {
          RadioEvents.RxTimeout( );
        }
      }
    }
```

__RxTimeout__ points to the __on_rx_timeout__ Callback Function that we've seen earlier.

### Preamble Detected

Preamble is the Radio Signal that __precedes the LoRa Message__. When the LoRa Module detects the Preamble Signal, it knows that it's about to receive a LoRa Message.

We don't need to handle the Preamble Signal, the LoRa Module does it for us: [radio.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/radio.c#L1427-L1431)

```c
    //  If LoRa Preamble was detected...
    if( ( irqRegs & IRQ_PREAMBLE_DETECTED ) == IRQ_PREAMBLE_DETECTED ) {
      //__NOP( );
    }
```

Our [__Receive Message Log__](https://github.com/lupyuen/lora-sx1262#receive-message) shows that the Preamble Signal (__IRQ_PREAMBLE_DETECTED__) is always detected before receiving a LoRa Message.

(__IRQ_PREAMBLE_DETECTED__ appears just before the LoRa Header: __IRQ_HEADER_VALID__)

[(More about LoRa Preamble)](https://www.link-labs.com/blog/what-is-lora)

### Sync Word Valid

__Sync Words__ are 16-bit values that differentiate the types of LoRa Networks.

The LoRa Module detects the Sync Words when it receive a LoRa Message: [radio.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/radio.c#L1433-L1437)

```c
    //  If a valid Sync Word was detected...
    if( ( irqRegs & IRQ_SYNCWORD_VALID ) == IRQ_SYNCWORD_VALID ) {
      //__NOP( );
    }
```

Note that the __Sync Word differs for LoRaWAN__ vs Private LoRa Networks...

```c
//  Syncword for Private LoRa networks
#define LORA_MAC_PRIVATE_SYNCWORD                   0x1424

//  Syncword for Public LoRa networks (LoRaWAN)
#define LORA_MAC_PUBLIC_SYNCWORD                    0x3444
```

[(More about Sync Words)](https://lupyuen.github.io/articles/lorawan#appendix-lora-sync-word)

### Header Valid

The LoRa Module checks for a __valid LoRa Header__ when receiving a LoRa Message: [radio.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/radio.c#L1439-L1443)

```c
    //  If a valid Header was received...
    if( ( irqRegs & IRQ_HEADER_VALID ) == IRQ_HEADER_VALID ) {
      //__NOP( );
    }
```

Our [__Receive Message Log__](https://github.com/lupyuen/lora-sx1262#receive-message) shows that the LoRa Header (__IRQ_HEADER_VALID__) is always detected before receiving a LoRa Message.

(__IRQ_HEADER_VALID__ appears right after the Preamble Signal: __IRQ_PREAMBLE_DETECTED__)

### Header Error

When the LoRa Module detects a __LoRa Header with CRC Error__, we stop the Receive Timer and call the Callback Function for Receive Timeout: [radio.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/radio.c#L1445-L1458)

```c
    //  If a Header with CRC Error was received...
    if( ( irqRegs & IRQ_HEADER_ERROR ) == IRQ_HEADER_ERROR ) {

      //  Stop the Receive Timer
      TimerStop( &RxTimeoutTimer );

      if( RxContinuous == false ) {
        //!< Update operating mode state to a value lower than \ref MODE_STDBY_XOSC
        SX126xSetOperatingMode( MODE_STDBY_RC );
      }

      //  Call the Callback Function for Receive Timeout
      if( ( RadioEvents.RxTimeout != NULL ) ) {
        RadioEvents.RxTimeout( );
      }
    }
  }
}
```

__RxTimeout__ points to the __on_rx_timeout__ Callback Function that we've seen earlier.

### RadioOnDioIrq

__RadioIrqProcess__ (as defined above) is called by __RadioOnDioIrq__ to handle LoRa Transmit and Receive Events: [radio.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/radio.c#L1300-L1312)

```c
/// Callback Function for Transmit and Receive Interrupts.
/// For BL602: This function runs in the context of the 
/// Background Application Task. So we are safe to call 
/// printf and SPI Functions now.
void RadioOnDioIrq( struct ble_npl_event *ev ) {
  //  Set the Interrupt Flag
  IrqFired = true;

  //  BL602 Note: It's OK to process the interrupt here because we are in
  //  Application Task Context, not Interrupt Context.
  //  The Reference Implementation processes the interrupt in the main loop.
  RadioIrqProcess();
}
```

## RadioSleep: Switch to Sleep Mode

__RadioSleep__ switches SX1262 to low-power sleep mode: [radio.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/radio.c#L1100-L1109)

```c
/// Switch to Sleep Mode
void RadioSleep( void ) {
  SleepParams_t params = { 0 };
  params.Fields.WarmStart = 1;

  //  Switch to Sleep Mode and wait 2 milliseconds
  SX126xSetSleep( params );
  DelayMs( 2 );
}
```

__SX126xSetSleep__ executes the Sleep Command on the LoRa Module: [sx126x.c](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x.c#L253-L268)

```c
/// Switch to Sleep Mode
void SX126xSetSleep( SleepParams_t sleepConfig ) {
  //  Switch off antenna (not used)
  SX126xAntSwOff( );

  //  Compute Sleep Parameter
  uint8_t value = ( 
      ( ( uint8_t )sleepConfig.Fields.WarmStart << 2 ) |
      ( ( uint8_t )sleepConfig.Fields.Reset << 1 ) |
      ( ( uint8_t )sleepConfig.Fields.WakeUpRTC ) 
  );

  if( sleepConfig.Fields.WarmStart == 0 ) {
    // Force image calibration
    ImageCalibrated = false;
  }

  //  Run Sleep Command
  SX126xWriteCommand( RADIO_SET_SLEEP, &value, 1 );
  SX126xSetOperatingMode( MODE_SLEEP );
}
```

[(__SX126xAntSwOff__ is defined here)](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x-linux.c#L324-L329)

[(__SX126xWriteCommand__ is defined here)](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x-linux.c#L204-L217)

[(__SX126xSetOperatingMode__ is defined here)](https://github.com/lupyuen/lora-sx1262/blob/master/src/sx126x-linux.c#L126-L147)

![Pinebook Pro with PineDio USB Adapter](https://lupyuen.github.io/images/usb-pinedio2.jpg)
