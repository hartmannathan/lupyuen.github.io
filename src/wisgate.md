# Build a LoRaWAN Network with RAKwireless WisGate Developer Gateway

📝 _30 Apr 2021_

While testing a new LoRaWAN gadget ([PineCone BL602](https://lupyuen.github.io/articles/lora2)), I bought a LoRaWAN Gateway from RAKwireless: [__RAK7248 WisGate Developer D4H Gateway__](https://docs.rakwireless.com/Product-Categories/WisGate/RAK7248/Datasheet/).

Here's what I learnt about __settting up a LoRaWAN Network__ with WisGate Developer D4H... And testing it with the __RAKwireless WisBlock dev kit in Arduino__.

![RAKwireless RAK7248 WisGate Developer D4H LoRaWAN Gateway](https://lupyuen.github.io/images/wisgate-title.jpg)

_RAKwireless RAK7248 WisGate Developer D4H LoRaWAN Gateway_

# WisGate D4H Hardware

WisGate D4H is essentially a __Raspberry Pi 4 + LoRa Network Concentrator (Semtech SX1302)__ in a sturdy IP30 box.

It exposes the same ports and connectors as a Raspberry Pi 4: __Ethernet port, USB 2 and 3 ports, USB-C power, microSD Card.__

But the __HDMI and GPIO ports__ are no longer accessible. (We control the box over HTTP and SSH)

![RAKwireless RAK7248 WisGate Developer D4H LoRaWAN Gateway](https://lupyuen.github.io/images/wisgate-hw.jpg)

2 new connectors have been added...

1.  __LoRa Antenna__ (left)

1.  __GPS Antenna__ (right)

(The two connectors are slightly different, so we won't connect the wrong antenna)

The GPS Antenna will be used when we connect WisGate to The Things Network (the worldwide free-access LoRaWAN network).

WisGate D4H is shipped with the open-source __ChirpStack LoRaWAN stack__, preinstalled in the microSD card. 

(Yep please don't peel off the sticky tape and insert your own microSD card)

![microSD Slot on WisGate D4H](https://lupyuen.github.io/images/wisgate-hw2.jpg)

_microSD Slot on WisGate D4H_

# ChirpStack LoRaWAN Stack

Connect the __LoRa Antenna and GPS Antenna__ to WisGate before powering on. (To prevent damage to the RF modules)

Follow the instructions here to start the WisGate box and to connect to the preinstalled ChirpStack LoRaWAN stack...

-   [__"RAK7244C Quick Start Guide"__](https://docs.rakwireless.com/Product-Categories/WisGate/RAK7244C/Quickstart/)

(RAK7244C is quite similar to our RAK7248 gateway)

I connected an Ethernet cable to WisGate and used SSH to configure the WiFi and LAN settings (via `sudo gateway-config`).

Here's the __ChirpStack web admin__ page that we will see...

![ChirpStack web admin on WisGate](https://lupyuen.github.io/images/wisgate-chirpstack.png)

(Nope I'm nowhere near Jervois Road)

In the left bar, click __`Gateways`__ to see our pre-configured LoRaWAN Gateway...

![ChirpStack web admin on WisGate](https://lupyuen.github.io/images/wisgate-chirpstack2.png)

Click __`rak-gateway`__ to see the Gateway Details...

![ChirpStack web admin on WisGate](https://lupyuen.github.io/images/wisgate-chirpstack3.png)

This shows that my WisGate gateway receives __1,200 LoRaWAN Packets a day__ from unknown LoRaWAN Devices nearby.

WisGate won't do anything with the received LoRaWAN Packets since it's not configured to process packets with mysterious origins.

(But we may click __`Live LoRaWAN Frames`__ at top right to see the encrypted contents of the received LoRaWAN Packets)

# LoRaWAN Application

When we allow a LoRaWAN Device to talk to our LoRaWAN Gateway, we need to set 2 things in the device...

1.  __Device EUI__: A 64-bit number that uniquely identifies our LoRaWAN Device

1.  __Application Key__: A 128-bit secret key that will authenticate that specific LoRaWAN Device

Here's how we get the Device EUI and Application Key from ChirpStack...

Click __`Applications`__ then __`app`__...

![ChirpStack Application](https://lupyuen.github.io/images/wisgate-app.png)

Take note of the second __Device EUI__ (for the OTAA Device Profile)...

![ChirpStack Application](https://lupyuen.github.io/images/wisgate-app2.png)

(EUI sounds like we stepped on something unpleasant... But it actually means [__Extended Unique Identifier__](https://lora-developers.semtech.com/library/tech-papers-and-guides/the-book/deveui/
))

We shall set this Device EUI in our Arduino program in a while...

```c
uint8_t nodeDeviceEUI[8] = { 0x4b, 0xc1, 0x5e, 0xe7, 0x37, 0x7b, 0xb1, 0x5b };
```

Click __`device_ptaa_class_a`__ because we'll be doing __Over-The-Air Activation (OTAA)__ for our Arduino device

![ChirpStack Application](https://lupyuen.github.io/images/wisgate-app3.png)

Click __`Keys (OTAA)`__

Under `Application Key`, click the circular arrow to generate a random key.

Take note of the generated __Application Key__, we shall set this in our Arduino program...

```c
uint8_t nodeAppKey[16] = { 0xaa, 0xff, 0xad, 0x5c, 0x7e, 0x87, 0xf6, 0x4d, 0xe3, 0xf0, 0x87, 0x32, 0xfc, 0x1d, 0xd2, 0x5d };
```

Click __`Set Device-Keys`__

We're all set to connect our Arduino LoRaWAN Device to WisGate!

# LoRaWAN Arduino Client

I tested WisGate with an Arduino LoRaWAN Device: [__RAKwireless RAK4631 WisBlock__](https://docs.rakwireless.com/Product-Categories/WisBlock/RAK4631/Overview/), based on Nordic nRF52840 with Semtech SX1262...

![RAKwireless RAK4631 WisBlock LPWAN Module mounted on WisBlock Base Board](https://lupyuen.github.io/images/wisblock-title.jpg)

_RAKwireless RAK4631 WisBlock LPWAN Module mounted on WisBlock Base Board_

The Arduino source code is here...

-   [`github.com/lupyuen/wisblock-lorawan`](https://github.com/lupyuen/wisblock-lorawan)

-   [__Watch the demo video on YouTube__](https://youtu.be/xdyi6XCo8Z8)

To download the source code, enter this at the command prompt...

```bash
git clone --recursive https://github.com/lupyuen/wisblock-lorawan
```

Note that the program requires the [__`SX126x-Arduino`__](https://github.com/beegee-tokyo/SX126x-Arduino) Library version __2.0.0 or later__. 

With PlatformIO, we set the `SX126x-Arduino` version in [`platformio.ini`](https://github.com/lupyuen/wisblock-lorawan/blob/master/platformio.ini) like so...

```text
lib_deps = beegee-tokyo/SX126x-Arduino@^2.0.0
```

The Arduino program in this article is based on the RAKwireless LoRaWAN sample [`LoRaWAN_OTAA_ABP.ino`](https://github.com/RAKWireless/WisBlock/blob/master/examples/RAK4630/communications/LoRa/LoRaWAN/LoRaWAN_OTAA_ABP/LoRaWAN_OTAA_ABP.ino)

[(More about LoRaWAN on WisBlock RAK4631)](https://github.com/RAKWireless/WisBlock/tree/master/examples/RAK4630/communications/LoRa/LoRaWAN)

## Configure LoRaWAN Client

Let's look at the Arduino code in our LoRaWAN Client...

[`wisblock-lorawan/src/main.cpp`](https://github.com/lupyuen/wisblock-lorawan/blob/master/src/main.cpp#L38-L58)

__Important:__ Set your __LoRaWAN Region__ in [`main.cpp`](https://github.com/lupyuen/wisblock-lorawan/blob/master/src/main.cpp#L38-L58)

```c
//  TODO: Set this to your LoRaWAN Region
LoRaMacRegion_t g_CurrentRegion = LORAMAC_REGION_AS923;
```

Before sending a LoRaWAN Data Packet to WisGate, we shall join our LoRaWAN Device to the LoRaWAN Network with __Over-The-Air Activation (OTAA)__...

```c
//  Set to true to select Over-The-Air Activation 
//  (OTAA), false for Activation By Personalisation (ABP)
bool doOTAA = true;
```

We won't be using Activation By Personalisation (ABP) today.

[(More about OTAA vs ABP)](https://www.thethingsnetwork.org/docs/lorawan/addressing/index.html)

Remember the __Device EUI and Application Key__ from ChirpStack?

(Did we step on something?)

We paste the Device EUI and Application Key here...

```c
//  TODO: (For OTAA Only) Set the OTAA keys. KEYS ARE MSB !!!!

//  Device EUI: Copy from ChirpStack: 
//  Applications -> app -> Device EUI
uint8_t nodeDeviceEUI[8] = { 0x4b, 0xc1, 0x5e, 0xe7, 0x37, 0x7b, 0xb1, 0x5b };

//  App EUI: Not needed for ChirpStack, 
//  set to default 0000000000000000
uint8_t nodeAppEUI[8] = { 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00 };

//  App Key: Copy from ChirpStack: 
//  Applications -> app -> Devices -> 
//  device_otaa_class_a -> Keys (OTAA) -> 
//  Application Key
uint8_t nodeAppKey[16] = { 0xaa, 0xff, 0xad, 0x5c, 0x7e, 0x87, 0xf6, 0x4d, 0xe3, 0xf0, 0x87, 0x32, 0xfc, 0x1d, 0xd2, 0x5d };
```

The __Application EUI__ is not used by ChirpStack, so we may set it to `0000000000000000`

Since we're not Activating By Personalisation, we may ignore this section...

```c
//  TODO: (For ABP Only) Set the ABP keys
uint32_t nodeDevAddr = 0x260116F8;
uint8_t nodeNwsKey[16] = { 0x7E, 0xAC, 0xE2, 0x55, 0xB8, 0xA5, 0xE2, 0x69, 0x91, 0x51, 0x96, 0x06, 0x47, 0x56, 0x9D, 0x23 };
uint8_t nodeAppsKey[16] = { 0xFB, 0xAC, 0xB6, 0x47, 0xF3, 0x58, 0x45, 0xC7, 0x50, 0x7D, 0xBF, 0x16, 0x8B, 0xA8, 0xC1, 0x7C };
```

## Initialise LoRaWAN Client

Following the Arduino convention, our __`setup`__ function shall be executed at startup: [`main.cpp`](https://github.com/lupyuen/wisblock-lorawan/blob/master/src/main.cpp#L110-L213)

```c
// At startup, we join the LoRaWAN network, in either OTAA or ABP mode
void setup() {
  ...
  //  Initialize LoRa chip
  lora_rak4630_init();

  //  Omitted: Initialize Serial for debug output
  ...
  
  //  Create a timer to send data to server every 20 seconds
  uint32_t err_code err_code = timers_init();
  if (err_code != 0) { return; }
```

Here we __initialise the LoRa Transceiver__ (SX1262) and create a timer that will send a LoRaWAN Packet every 20 seconds.

(More about `timers_init` in a while)

Next we set the __Device EUI, Application EUI and Application Key__ that will be used for joining the LoRaWAN Network...

```c
  //  Setup the EUIs and Keys
  if (doOTAA) {
    //  Set the EUIs and Keys for OTAA
    lmh_setDevEui(nodeDeviceEUI);
    lmh_setAppEui(nodeAppEUI);
    lmh_setAppKey(nodeAppKey);
  } else {
    //  Omitted: Set the EUIs and Keys for ABP
    ...
  }
```

Then we __initialise the LoRaWAN Client__...

```c
  //  Initialise LoRaWAN
  err_code = lmh_init(  //  lmh_init now takes 3 parameters instead of 5
    &g_lora_callbacks,  //  Callbacks 
    g_lora_param_init,  //  Functions
    doOTAA,             //  Set to true for OTAA
    g_CurrentClass,     //  Class 
    g_CurrentRegion     //  Region
  );
  if (err_code != 0) { return; }
```

(More about `g_lora_callbacks` in a while)

Finally we send the __Join LoRaWAN Network Request__ to WisGate...

```c
  //  Start Join procedure
  lmh_join();
}
```

_What's in `g_lora_callbacks`?_

`g_lora_callbacks` contains a list of Callback Functions that will be called by the LoRaWAN Client: [`main.cpp`](https://github.com/lupyuen/wisblock-lorawan/blob/master/src/main.cpp#L87-L97)

```c
//  Structure containing LoRaWan callback functions, 
//  needed for lmh_init()
static lmh_callback_t g_lora_callbacks = {
    BoardGetBatteryLevel, 
    BoardGetUniqueId, 
    BoardGetRandomSeed,
    lorawan_rx_handler, 
    lorawan_has_joined_handler, 
    lorawan_confirm_class_handler, 
    lorawan_join_failed_handler
};
```

We shall see the Callback Functions in action soon.

_What about `loop`, the Arduino function that will be called to do work?_

Our program uses the Callback Functions to trigger actions, so we won't be `loop`ing today.

Here's our `loop` function in [`main.cpp`](https://github.com/lupyuen/wisblock-lorawan/blob/master/src/main.cpp#L215-L220)

```c
//  Nothing here for now
void loop() {
  //  Put your application tasks here, like reading of sensors,
  //  controlling actuators and other functions.
}
```

## Join LoRaWAN Network

During startup we called `lmh_join` to send the __Join LoRaWAN Network Request__ to WisGate...

```
//  Start Join procedure
lmh_join();
```

When __WisGate accepts the Join Request__ and returns an acknowledgement packet, this Callback Function will be called: [`main.cpp`](https://github.com/lupyuen/wisblock-lorawan/blob/master/src/main.cpp#L222-L236)

```c
//  Function that is called when we have joined the LoRaWAN network
void lorawan_has_joined_handler(void) {

  //  Request for a different LoRaWAN Device Class, if necessary
  lmh_error_status ret = lmh_class_request(g_CurrentClass);
```

Here we switch to the desired __LoRaWAN Device Class__ (A, B or C) if necessary. 

__Class A__ is the most basic Device Class (for simple LoRaWAN sensors). __Classes B and C__ are more sophisticated.

We have configured our LoRaWAN Client for __Class A__...

```c
DeviceClass_t g_CurrentClass = CLASS_A;
```

So `lmh_class_request` won't switch our Device Class.

Our Callback Function then starts a __timer that expires in 20 seconds__...

```c
  //  When we have joined the LoRaWAN network, 
  //  start a 20-second timer
  if (ret == LMH_SUCCESS) {
    delay(1000);
    TimerSetValue(&appTimer, LORAWAN_APP_INTERVAL);
    TimerStart(&appTimer);
  }
}
```

We'll learn more about the timer in a while. (Hint: It triggers the sending of a LoRaWAN Data Packet)

We have another Callback Function that's called when our __Join Request is rejected__ by WisGate: [`main.cpp`](https://github.com/lupyuen/wisblock-lorawan/blob/master/src/main.cpp#L238-L246)

```c
//  LoRa function for handling OTAA join failed
static void lorawan_join_failed_handler(void) {
  //  If we can't join the LoRaWAN network, show the error
  Serial.println("OTAA join failed!");
}
```

The LoRaWAN Gateway (WisGate) rejects our Join Request if the Device EUI and/or Application Key are incorrect.

In case you're curious: WisBlock transmits this LoRaWAN Packet to WisGate when requesting to join the LoRaWAN network...

![Join LoRaWAN Network Request](https://lupyuen.github.io/images/wisgate-join.png)

We'll learn about the Nonce and the Message Integrity Code later.

## Send LoRaWAN Data Packet

Now that we've joined the LoRaWAN Network, let's __send a LoRaWAN Data Packet__ to WisGate!

Remember that when our Join Request succeeds, it starts a 20-second timer.

Here's what happens 20 seconds later: [`main.cpp`](https://github.com/lupyuen/wisblock-lorawan/blob/master/src/main.cpp#L302-L311)

```c
//  Function for handling timeout event
void tx_lora_periodic_handler(void) {
  //  When the 20-second timer has expired, 
  //  send a LoRaWAN Packet and restart the timer
  TimerSetValue(&appTimer, LORAWAN_APP_INTERVAL);
  TimerStart(&appTimer);
  send_lora_frame();
}
```

We restart the 20-second timer and call `send_lora_frame` to send a LoRaWAN Data Packet to WisGate.

`send_lora_frame` is defined in [`main.cpp`](https://github.com/lupyuen/wisblock-lorawan/blob/master/src/main.cpp#L269-L300)

```c
//  This is called when the 20-second timer expires. 
//  We send a LoRaWAN Data Packet.
void send_lora_frame(void) {
  if (lmh_join_status_get() != LMH_SET) {
    //  Not joined, try again later
    return;
  }
```

First we verify that we have joined the LoRaWAN Network. (Because WisGate won't process our data packet unless we're in the network)

Then we set the packet port and populate the packet with 6 bytes: "`Hello!`"...

```c
  //  Copy "Hello!" into the transmit buffer
  m_lora_app_data.port = gAppPort;  //  Set to LORAWAN_APP_PORT
  memset(m_lora_app_data.buffer, 0, LORAWAN_APP_DATA_BUFF_SIZE);
  memcpy(m_lora_app_data.buffer, "Hello!", 6);
  m_lora_app_data.buffsize = 6;
```

Finally we transmit the data packet to WisGate...

```c
  //  Transmit the LoRaWAN Data Packet
  lmh_error_status error = lmh_send(
    &m_lora_app_data,  //  Data Packet
    g_CurrentConfirm   //  LMH_UNCONFIRMED_MSG: Unconfirmed Message
  );
  if (error == LMH_SUCCESS) { count++; } 
  else { count_fail++; }
}
```

Here we're sending an __Unconfirmed LoRaWAN Message__ to WisGate. Which means that we don't expect an acknowledgement from WisGate.

This is the preferred way for a low-power LoRaWAN device to transmit sensor data, since it __doesn't need to wait for the acknowledgement__ (and consume additional power).

(It's OK if a LoRaWAN Data Packet gets lost due to noise or inteference... LoRaWAN sensor devices are supposed to transmit data packets periodically anyway)

_What about receiving data from our LoRaWAN Gateway (WisGate)?_

We haven't configured WisGate to transmit data to our LoRaWAN Device.

But the Callback Function to __handle received packets__ is here: [`main.cpp`](https://github.com/lupyuen/wisblock-lorawan/blob/master/src/main.cpp#L248-L257)

```c
//  Function for handling LoRaWan received data from Gateway
void lorawan_rx_handler(lmh_app_data_t *app_data) {
  //  When we receive a LoRaWAN Packet from the 
  //  LoRaWAN Gateway, display it.
  //  TODO: Ensure that app_data->buffer is null-terminated.
  Serial.printf("LoRa Packet received on port %d, size:%d, rssi:%d, snr:%d, data:%s\n",
    app_data->port, app_data->buffsize, app_data->rssi, app_data->snr, app_data->buffer);
}
```

There's one more Callback Function (that we're not using) for __switching the LoRaWAN Device Class__: [`main.cpp`](https://github.com/lupyuen/wisblock-lorawan/blob/master/src/main.cpp#L259-L267)

```c
//  Callback Function that is called when the
//  LoRaWAN Class has been changed
void lorawan_confirm_class_handler(DeviceClass_t Class) {
  //  Informs the server that switch has occurred ASAP
  m_lora_app_data.buffsize = 0;
  m_lora_app_data.port = gAppPort;
  lmh_send(&m_lora_app_data, g_CurrentConfirm);
}
```

_How did we initialise the 20-second timer?_

During startup, the `setup` function calls `timers_init` to initialise the timer: [`main.cpp`](https://github.com/lupyuen/wisblock-lorawan/blob/master/src/main.cpp#L313-L321)

```c
//  Initialise the timer
uint32_t timers_init(void) {
  TimerInit(&appTimer, tx_lora_periodic_handler);
  return 0;
}
```

# WisBlock Talks To WisGate

Let's __run the Arduino program on WisBlock__ and watch what happens in the Arduino Log!

1.  Our Arduino program starts by reminding us of the Over-The-Air Activation (OTAA) configuration: __Device EUI, Application EUI (unused) and Application Key__...

    ```text
    OTAA 
    DevEui=4B-C1-5E-E7-37-7B-B1-5B
    DevAdd=00000000
    AppEui=00-00-00-00-00-00-00-00
    AppKey=AA-FF-AD-5C-7E-87-F6-4D-E3-F0-87-32-FC-1D-D2-5D
    ```

    (Device Address will be assigned by WisGate when we join the LoRaWAN Network)

1.  We __transmit the Join Network Request__ to WisGate...

    ```text
    Selected subband 1
    Joining LoRaWAN network...
    RadioSend: 00 00 00 00 00 00 00 00 00 5b b1 7b 37 e7 5e c1 4b 55 ed 7e 9b d6 af
    OnRadioTxDone
    ```

    The Join Network Request contains these fields...

    ![Join LoRaWAN Network Request](https://lupyuen.github.io/images/wisgate-join.png)

    _But the Application Key is missing from the Join Network Request!?_

    Yep! We'll learn why in a while.

1.  To conserve power, LoRaWAN Devices (Class A) don't receive packets all the time.

    We listen for incoming packets (for a brief moment) __only after we transmit a packet__...

    ```text
    OnRadioTxDone => RX Windows #1 5002 #2 6002
    OnRadioTxDone => TX was Join Request
    ```

    We've just transmitted a packet (Join Network Request), so __we listen for an incoming packet__. (Hopefully, the Join Network Response from WisGate)

    The LoRaWAN Specification actually defines __Two Receive Windows__. Which means that __we listen twice (very briefly) for incoming packets__, after sending every packet...

    ![LoRaWAN Receive Window](https://lupyuen.github.io/images/wisgate-window.png)

1.  And indeed, we __receive the Join Network Accepted Response__ from WisGate...

    ```text
    OnRadioRxDone
    OnRadioRxDone => FRAME_TYPE_JOIN_ACCEPT
    OTAA Mode, Network Joined!
    ```

    The LoRaWAN Spec says we don't open the Second Receive Window if we receive a packet in the First Receive Window.

    So... We're good! 

1.  20 seconds later we __transmit our first data packet__ ("`Hello!`")...

    ```text
    Sending frame now...
    RadioSend: 40 3c 59 7a 00 80 00 00 02 17 77 31 fd 99 86 8f 4f cc ef 
    lmh_send ok count 1
    OnRadioTxDone
    ```

1.  Once again we open the __Two Receive Windows__...

    ```text
    OnRadioTxDone => RX Windows #1 1002 #2 2002
    RadioIrqProcess => IRQ_RX_TX_TIMEOUT
    OnRadioRxTimeout
    ```

    We haven't configured WisGate to respond to our data packet. (Remember that we're sending Unconfirmed LoRaWAN Messages... No ack needed)

    So we get a __Receive Timeout__. Which is expected.

1.  20 seconds later we __transmit our second data packet__...

    ```text
    Sending frame now...
    RadioSend: 40 3c 59 7a 00 80 01 00 02 0b 1f 7e 4d e1 94 c9 16 fa ea 
    lmh_send ok count 2
    OnRadioTxDone
    ```

    We open the __Two Receive Windows__ and we get a __Receive Timeout__, which is OK...

    ```text
    OnRadioTxDone => RX Windows #1 1002 #2 2002
    RadioIrqProcess => IRQ_RX_TX_TIMEOUT
    OnRadioRxTimeout
    ```

    [__Watch the demo video on YouTube__](https://youtu.be/xdyi6XCo8Z8)

    [__See the complete Arduino log__](https://github.com/lupyuen/wisblock-lorawan/blob/master/README.md#output-log)

# View Received LoRaWAN Packets

Let's head back to __ChirpStack on our WisGate LoRaWAN Gateway__, to observe the LoRaWAN Packets received from our Arduino Device (WisBlock)...

1.  In ChirpStack, click __`Applications → app → device_otaa_class_a → Device Data`__

1.  Restart our Arduino LoRaWAN Device (WisBlock)

1.  The __Join Network Request__ appears...

    ![LoRaWAN Device Data](https://lupyuen.github.io/images/wisgate-devicedata1.png)


1.  20 seconds later the __Data Packet__ appears...

    ![LoRaWAN Device Data](https://lupyuen.github.io/images/wisgate-devicedata2.png)

    `DecodedDataString` says "`Hello!`"

1.  We may now configure ChirpStack to do something useful with the received packets, like publish them over MQTT, HTTP, ...

    Click this link...

    -   [__ChirpStack Application Server__](https://www.chirpstack.io/application-server/)

    Then click the __Menu__ (top left) and __Integrations__

# Troubleshoot LoRaWAN

If our packets don't appear in ChirpStack, here are a couple of things to check...

## Inspect the raw packets

To troubleshoot LoRaWAN problems, we may inspect the __Raw LoRaWAN Packets__ received by our LoRaWAN Gateway (WisGate).

In ChirpStack there are two ways do this...

1.  Click __`Gateways → rak-gateway → Live LoRaWAN Frames`__

    This shows ALL LoRaWAN Packets received, even from unknown devices.

    (The packet data is encrypted, so we won't be able to read other people's messages)

1.  Or click __`Applications → app → device_otaa_class_a → LoRaWAN Frames`__

    This only shows LoRaWAN Packets from identified devices.

Here's the __Raw Packet for Join Network Request__...

![Raw LoRaWAN Join Network Request Packet](https://lupyuen.github.io/images/wisgate-frame1.png)

And here's the __Raw Data Packet__...

![Raw LoRaWAN Data Packet](https://lupyuen.github.io/images/wisgate-frame2.png)

This is the mystery LoRaWAN Packet that my LoRaWAN Gateway (WisGate) receives every minute (with encrypted contents)...

![Raw LoRaWAN Data Packet from unknown device](https://lupyuen.github.io/images/wisgate-frame3.png)

## Check the log

SSH to WisGate and look at the __Linux System Log__ while our LoRaWAN Device is transmitting packets...

```bash
tail -f /var/log/syslog
```

Sometimes it's helpful to scan the log for __Message Integrity Code__ errors ("invalid MIC")...

```bash
grep MIC /var/log/syslog

Apr 28 04:02:05 rak-gateway 
chirpstack-application-server[568]: 
time="2021-04-28T04:02:05+01:00" 
level=error 
msg="invalid MIC" 
dev_eui=4bc15ee7377bb15b 
type=DATA_UP_MIC
```

Also for __Nonce Errors__ ("validate dev-nonce error")...

```bash
grep nonce /var/log/syslog

Apr 28 04:02:41 rak-gateway chirpstack-application-server[568]:
time="2021-04-28T04:02:41+01:00" 
level=error 
msg="validate dev-nonce error" 
dev_eui=4bc15ee7377bb15b 
type=OTAA
```

Another way to check for __Nonce Errors__: Click...

__Applications__ → __app__ → __device_otaa_class_a__  → __Device Data__

Look for __"validate dev-nonce error"__...

![Duplicate LoRaWAN Nonce](https://lupyuen.github.io/images/auto-nonce.png)

We'll talk about Message Integrity Code and Nonce in a while.

## Snoop with Software Defined Radio

To verify the actual frequency that our LoRaWAN Device is transmitting on, we may sniff the airwaves with a __Software Defined Radio__.

Or use a Spectrum Analyser like __RF Explorer__. [(See this)](https://lupyuen.github.io/articles/lora#troubleshoot-lora)

Let's talk about Software Defined Radio...

# Visualise LoRaWAN with Software Defined Radio

A __Software Defined Radio (SDR)__ is a USB gadget that scans the airwaves (for a range of frequencies) and feeds the data to our computer for processing.

I use the [__Airspy R2 SDR__](https://www.itead.cc/airspy.html) to sniff the airwaves and verify that our LoRaWAN Device is transmitting at the __right frequency with sufficient power__...

![Airspy R2 SDR](https://lupyuen.github.io/images/wisgate-airspy.jpg)

Here's the __Join Network Request__ transmitted by WisBlock to WisGate, captured by Airspy SDR and visualised with [__Cubic SDR__](https://cubicsdr.com/)...

![LoRaWAN Chirp: Device to Gateway](https://lupyuen.github.io/images/wisgate-chirp1.png)

And here's the __Join Network Response__ returned by WisGate to WisBlock...

![LoRaWAN Chirp: Gateway to Device](https://lupyuen.github.io/images/wisgate-chirp2.png)

(Our SDR is located near WisBlock, hence the WisGate signal looks slightly weaker)

LoRaWAN Packets have a distinct cross-hatch pattern known as the __LoRa Chirp__. By transmitting packets in this unique chirping pattern, LoRa ensures that packets will be delivered over long distances in spite of the noise and interference.

-   [__Watch the video on YouTube__](https://youtu.be/xdyi6XCo8Z8)

-   [__More about LoRa Chirps and Software Defined Radio__](https://lupyuen.github.io/articles/lora#visualise-lora-with-software-defined-radio)

# LoRaWAN Security

I'm no Security Expert... But lemme attempt to explain the high-level bits of LoRaWAN Security to the best of my understanding 🙏

## Join Network Request

Let's start by looking at the __Join Network Request__ transmitted by our LoRaWAN Device (WisBlock) to our LoRaWAN Gateway (WisGate). And understand why it's secure...

![Join LoRaWAN Network Request](https://lupyuen.github.io/images/wisgate-join.png)

There are 5 parts in the request...

1.  __Header__ (1 byte): Identifies this LoRaWAN Packet as a Join Network Request

1.  __Application EUI__ (8 bytes): Unused

1.  __Device EUI__ (8 bytes): Uniquely identifies our LoRaWAN Device. (The bytes are flipped)

1.  __Nonce__ (2 bytes): We'll explain in a while

1.  __Message Integrity Code__ (4 bytes): We'll explain now

## Message Integrity Code

_Why isn't the Application Key inside the Join Network Request?_

Because the __Application Key is a Secret Key__. It should never be revealed to others, otherwise our gateway will accept Join Network Requests from unauthorised devices.

Instead we use the Application Key to compute a __[Message Authentication Code](https://en.wikipedia.org/wiki/Message_authentication_code) (AES-128 CMAC)__. And we transmit the first 4 bytes of the authentication code as the __Message Integrity Code__.

According to the LoRaWAN Spec, the Message Authentication Code (for the Join Network Request) is computed with these input values...

1.  __Application Key__

1.  __Header__

1.  __Application EUI__

1.  __Device EUI__

1.  __Nonce__ (We'll explain in the next section)

_What happens when we transmit an invalid Message Integrity Code?_

Our LoRaWAN Gateway (ChirpStack on WisGate) will __reject LoRaWAN Packets with invalid Message Integrity Codes__.

To see Message Integrity Code errors, SSH to our LoRaWAN Gateway and search for "invalid MIC"...

```bash
grep MIC /var/log/syslog

Apr 28 04:02:05 rak-gateway 
chirpstack-application-server[568]: 
time="2021-04-28T04:02:05+01:00" 
level=error 
msg="invalid MIC" 
dev_eui=4bc15ee7377bb15b 
type=DATA_UP_MIC

Apr 28 04:02:05 rak-gateway 
chirpstack-network-server[1378]: 
time="2021-04-28T04:02:05+01:00" 
level=error 
msg="uplink: processing uplink frame error"
ctx_id=0ccd1478-3b79-4ded-9e26-a28e4c143edc 
error="get device-session error: invalid MIC"
```

## Nonce

_There isn't a Timestamp in the request. Could someone record and replay the Join Network Request?_

That's why we have the __Nonce__: A 16-bit number that's only __used once and never reused__ by our LoRaWAN Device.

The Nonce appears in the Join Network Request, and it's also used for computing the Message Integrity Code.

_What happens when we replay a Join Network Request?_

Our LoRaWAN Gateway (ChirpStack on WisGate) will __reject Join Network Requests that have a reused Nonce__.

(Yep the gateway will remember the Nonces from previous requests)

To see Nonce errors, SSH to our LoRaWAN Gateway and search for __"validate dev-nonce error"__...

```bash
grep nonce /var/log/syslog

Apr 28 04:02:41 rak-gateway chirpstack-application-server[568]:
time="2021-04-28T04:02:41+01:00" 
level=error 
msg="validate dev-nonce error" 
dev_eui=4bc15ee7377bb15b 
type=OTAA

Apr 28 04:02:41 rak-gateway chirpstack-network-server[1378]:
time="2021-04-28T04:02:41+01:00" 
level=error 
msg="uplink: processing uplink frame error" ctx_id=01ae296e-8ce1-449a-83cc-fb0771059d89 
error="validate dev-nonce error: object already exists"
```

Nonce errors will also appear in ChirpStack. Click...

__Applications__ → __app__ → __device_otaa_class_a__  → __Device Data__

Look for __"validate dev-nonce error"__...

![Duplicate LoRaWAN Nonce](https://lupyuen.github.io/images/auto-nonce.png)

Here's the LoRaWAN Spec for the Nonce and Message Integrity Code...

![Join LoRaWAN Network Request](https://lupyuen.github.io/images/wisgate-join2.png)

For other LoRaWAN Packets (besides the Join Network Request), the Message Integrity Code is computed differently...

![LoRaWAN Message Integrity Code](https://lupyuen.github.io/images/wisgate-mic.png)

## More About Security

LoRaWAN Payloads are __encrypted with AES-128__, says the LoRaWAN Spec...

![LoRaWAN Encryption](https://lupyuen.github.io/images/wisgate-encrypt.png)

And that's the limit of my security know-how 🙁

To learn more about LoRaWAN Security, I recommend this article...

-   [__"LoRaWAN® Is Secure (but Implementation Matters)"__](https://lora-alliance.org/resource_hub/lorawan-is-secure-but-implementation-matters/)

# What's Next

Pine64's upcoming __PineDio LoRa Gateway__ works very much like WisGate, here's my hands-on experience...

-   [__"PineDio LoRa Gateway: Testing The Prototype"__](https://lupyuen.github.io/articles/gateway)

To learn about configuring WisGate for __The Things Network__, check out this article...

-   [__"The Things Network on PineDio Stack BL604 RISC-V Board"__](https://lupyuen.github.io/articles/ttn)

This has been a fun experience for me, diving deep into LoRaWAN with the RAKwireless WisGate LoRaWAN Gateway... __My first time using LoRaWAN!__

I hope you find this article useful. Now that I know some LoRaWAN, I'm heading back to [__PineCone BL602__](https://lupyuen.github.io/articles/lora2) to finish what I started...

-   [__"PineCone BL602 Talks LoRaWAN"__](https://lupyuen.github.io/articles/lorawan)

And then to __PineDio Stack BL604__...

-   [__"LoRaWAN on Apache NuttX OS"__](https://lupyuen.github.io/articles/lorawan3)

-   [__"LoRaWAN on PineDio Stack BL604 RISC-V Board"__](https://lupyuen.github.io/articles/lorawan2)

Many Thanks to my [__GitHub Sponsors__](https://lupyuen.github.io/articles/sponsor) for supporting my work! This article wouldn't have been possible without your support.

-   [Sponsor me a coffee](https://lupyuen.github.io/articles/sponsor)

-   [Discuss this article on Reddit](https://www.reddit.com/r/Lora/comments/n1qv3l/build_a_lorawan_network_with_rakwireless_wisgate/?utm_source=share&utm_medium=web2x&context=3)

-   [Read "The RISC-V BL602 Book"](https://lupyuen.github.io/articles/book)

-   [Check out my articles](https://lupyuen.github.io)

-   [RSS Feed](https://lupyuen.github.io/rss.xml)

_Got a question, comment or suggestion? Create an Issue or submit a Pull Request here..._

[`lupyuen.github.io/src/wisgate.md`](https://github.com/lupyuen/lupyuen.github.io/blob/master/src/wisgate.md)

# Notes

1.  This article is the expanded version of [this Twitter Thread](https://twitter.com/MisterTechBlog/status/1379926160377851910)

# Appendix: Log Transmitted Packets

To log transmitted packets in our Arduino program, modify this source file...

```text
.pio/libdeps/wiscore_rak4631/SX126x-Arduino/src/mac/LoRaMac.cpp
```

Insert the section marked below...

```c
LoRaMacStatus_t SendFrameOnChannel(uint8_t channel)
{
    TxConfigParams_t txConfig;
    int8_t txPower = 0;

    txConfig.Channel = channel;
    txConfig.Datarate = LoRaMacParams.ChannelsDatarate;
    txConfig.TxPower = LoRaMacParams.ChannelsTxPower;
    txConfig.MaxEirp = LoRaMacParams.MaxEirp;
    txConfig.AntennaGain = LoRaMacParams.AntennaGain;
    txConfig.PktLen = LoRaMacBufferPktLen;

    // If we are connecting to a single channel gateway we use always the same predefined channel and datarate
    if (singleChannelGateway)
    {
        txConfig.Channel = singleChannelSelected;
        txConfig.Datarate = singleChannelDatarate;
    }

    RegionTxConfig(LoRaMacRegion, &txConfig, &txPower, &TxTimeOnAir);

    MlmeConfirm.Status = LORAMAC_EVENT_INFO_STATUS_ERROR;
    McpsConfirm.Status = LORAMAC_EVENT_INFO_STATUS_ERROR;
    McpsConfirm.Datarate = LoRaMacParams.ChannelsDatarate;
    McpsConfirm.TxPower = txPower;

    // Store the time on air
    McpsConfirm.TxTimeOnAir = TxTimeOnAir;
    MlmeConfirm.TxTimeOnAir = TxTimeOnAir;

    // Starts the MAC layer status check timer
    TimerSetValue(&MacStateCheckTimer, MAC_STATE_CHECK_TIMEOUT);
    TimerStart(&MacStateCheckTimer);

    if (IsLoRaMacNetworkJoined != JOIN_OK)
    {
        JoinRequestTrials++;
    }

    //////////////// INSERT THIS CODE

    // To replay a Join Network Request...
    // if (LoRaMacBuffer[0] == 0) {
    // 	static uint8_t replay[] = {0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x5b, 0xb1, 0x7b, 0x37, 0xe7, 0x5e, 0xc1, 0x4b, 0x67, 0xaa, 0xbb, 0x07, 0x70, 0x7d};
    // 	memcpy(LoRaMacBuffer, replay, LoRaMacBufferPktLen);
    // }

    // To dump transmitted packets...
    printf("RadioSend: size=%d, channel=%d, datarate=%d, txpower=%d, maxeirp=%d, antennagain=%d\r\n", (int) LoRaMacBufferPktLen, (int) txConfig.Channel, (int) txConfig.Datarate, (int) txConfig.TxPower, (int) txConfig.MaxEirp, (int) txConfig.AntennaGain);
    for (int i = 0; i < LoRaMacBufferPktLen; i++) {
        printf("%02x ", LoRaMacBuffer[i]);
    }
    printf("\r\n");
    
    //////////////// END OF INSERTION

    // Send now
    Radio.Send(LoRaMacBuffer, LoRaMacBufferPktLen);

    LoRaMacState |= LORAMAC_TX_RUNNING;

    return LORAMAC_STATUS_OK;
}
```
