# NuttX RTOS for PinePhone: 4G LTE Modem

📝 _12 Apr 2023_

![Quectel EG25-G LTE Modem inside PinePhone](https://lupyuen.github.io/images/lte-title.jpg)

[_Quectel EG25-G LTE Modem inside PinePhone_](https://wiki.pine64.org/images/2/20/Quectel_EG25-G_Hardware_Design_V1.4.pdf)

What makes [__Pine64 PinePhone__](https://wiki.pine64.org/index.php/PinePhone) a phone? It's the [__4G LTE Modem__](https://en.wikipedia.org/wiki/LTE_(telecommunication)) inside that makes Phone Calls and sends Text Messages!

Now we're building a [__Feature Phone__](https://lupyuen.github.io/articles/usb2#pinephone--nuttx--feature-phone) with [__Apache NuttX RTOS__](https://lupyuen.github.io/articles/what) (Real-Time Operating System). To make things simpler, we're writing down __everything we know__ about the 4G LTE Modem, and how it works inside PinePhone...

-   What's the __Quectel EG25-G LTE Modem__

-   How it's __connected inside PinePhone__

-   How we make __Phone Calls__ and send __Text Messages__

-   How we __power up__ the LTE Modem

-   __Programming the LTE Modem__ with UART, USB and Apache NuttX RTOS

Read on to learn all about PinePhone's 4G LTE Modem...

![Quectel EG25-G LTE Modem](https://lupyuen.github.io/images/usb2-modem.jpg)

[_Quectel EG25-G LTE Modem_](https://wiki.pine64.org/wiki/File:Quectel_EG25-G_LTE_Standard_Specification_V1.3.pdf)

# Quectel EG25-G LTE Modem

_What's this LTE Modem?_

Inside PinePhone is the [__Quectel EG25-G LTE Modem__](https://wiki.pine64.org/index.php/PinePhone#Modem) for [__4G LTE__](https://en.wikipedia.org/wiki/LTE_(telecommunication)) Voice Calls, SMS and Mobile Data, plus GPS (pic above)...

-   [__Quectel EG25-G Datasheet__](https://wiki.pine64.org/wiki/File:Quectel_EG25-G_LTE_Standard_Specification_V1.3.pdf)

-   [__EG25-G Hardware Design__](https://wiki.pine64.org/wiki/File:Quectel_EG25-G_Hardware_Design_V1.4.pdf)

To control the LTE Modem, we send __AT Commands__...

-   [__EG25-G AT Commands__](https://wiki.pine64.org/images/1/1b/Quectel_EC2x%26EG9x%26EG2x-G%26EM05_Series_AT_Commands_Manual_V2.0.pdf)

-   [__EG25-G AT QCFG Commands__](https://github.com/lupyuen/lupyuen.github.io/blob/master/images/Quectel_EC2x&EG2x&EG9x&EM05_Series_QCFG_AT_Commands_Manual_V1.0.pdf)

-   [__EG25-G GNSS__](https://wiki.pine64.org/wiki/File:Quectel_EC2x%26EG9x%26EG2x-G%26EM05_Series_GNSS_Application_Note_V1.3.pdf)

-   [__Get Started with AT Commands__](https://www.twilio.com/docs/iot/supersim/works-with-super-sim/quectel-eg25-g)

So to dial the number __`1711`__, we send this AT Command...

```text
ATD1711;
```

The LTE Modem has similar AT Commands for SMS and Mobile Data.

[(EG25-G runs on __Qualcomm MDM 9607__ with a Cortex-A7 CPU inside)](https://xnux.eu/devices/feature/modem-pp.html#toc-modem-on-pinephone)

_How to send the AT Commands to LTE Modem?_

The LTE Modem accepts __AT Commands__ in two ways (pic below)...

-   Via the __UART Port (Serial)__

    Which is Slower: Up to 921.6 kbps

-   Via the __USB Port (USB Serial)__

    Which is Faster: Up to 480 Mbps

So if we're sending and receiving __lots of 4G Mobile Data__, USB is the better way.

(UART Interface is probably sufficient for a Feature Phone)

Let's talk about the UART and USB Interfaces...

![Data Interfaces for LTE Modem](https://lupyuen.github.io/images/lte-title4.jpg)

# Data Interfaces for LTE Modem

_There's a band of bass players in my PinePhone?_

Ahem the [__Baseband Processor__](https://en.wikipedia.org/wiki/Baseband_processor) inside the LTE Modem (pic above) is the hardware that handles the __Radio Functions__ for 4G LTE and GPS.

According to the [__PinePhone Schematic__](https://files.pine64.org/doc/PinePhone/PinePhone%20v1.2b%20Released%20Schematic.pdf) (Page 15), the Baseband Processor talks to PinePhone (Allwinner A64) over the following __Data Interfaces__ (pic above)...

-   __USB__ ⇆ A64 Port __USB1__ _(USB Serial)_

    For AT Commands and GPS Output. (Up to 480 Mbps)

-   __SIM__ ⇆ PinePhone 4G SIM Card

    For connecting to the 4G LTE Mobile Network.

-   __PCM__ ⇆ A64 Port __PCM0__

    [__PCM Digital Audio Stream__](https://en.wikipedia.org/wiki/Pulse-code_modulation) for 4G Voice Calls.

-   __UART__ ⇆ A64 Port __UART3__ _(RX / TX)_, __UART4__ _(CTS / RTS)_, __PB2__ _(DTR)_

    Simpler, alternative interface for AT Commands. Default 115.2 kbps, up to 921.6 kbps.

    [(CTS / RTS might not work correctly)](https://wiki.pine64.org/wiki/PinePhone_v1.1_-_Braveheart#Modem_UART_flow_control_is_broken)

    [(More about the UART pins)](https://lupyuen.github.io/articles/lte#main-uart-interface)

UART is slower than USB, so we should probably use USB instead of UART.

(Unless we're building a simple Feature Phone without GPS)

PinePhone also controls the LTE Modem with a bunch of GPIO Pins...

![Control Pins for LTE Modem](https://lupyuen.github.io/images/lte-title3.jpg)

# Control Pins for LTE Modem

_PinePhone's LTE Modem is controlled only by AT Commands?_

There's more! According to the [__PinePhone Schematic__](https://files.pine64.org/doc/PinePhone/PinePhone%20v1.2b%20Released%20Schematic.pdf) (Page 15), the LTE Modem is controlled by the following GPIO Pins (pic above)...

-   __Baseband Power__ ← A64 Port __PL7__

    Supplies power to LTE Modem.

    (Also connected to Battery Power VBAT and Power Management DCDC1)

-   __Power Key__ ← A64 Port __PB3__

    Power up the LTE Modem.

    [(See this)](https://lupyuen.github.io/articles/lte#power-on--off)

-   __Reset__ ← A64 Port __PC4__

    Reset the LTE Modem.

    [(See this)](https://lupyuen.github.io/articles/lte#power-on--off)

We'll control the above GPIO Pins to __power up the LTE Modem__ at startup. (More in the next section)

Also at startup, we'll read this GPIO Pin to check if the __LTE Modem is hunky dory__...

-   __Status__ → A64 Port __PH9__

    Read the Modem Status.

    [(See this)](https://lupyuen.github.io/articles/lte#status-indication)

These GPIO Pins control the __Airplane Mode__ and __Sleep State__...

-   __Disable__ ← A64 Port __PH8__

    Enable or Disable Airplane Mode.

    [(See this)](https://lupyuen.github.io/articles/lte#other-interface-pins)

-   __AP Ready__ ← A64 Port __PH7__

    Set the Modem Sleep State.

    [(See this)](https://lupyuen.github.io/articles/lte#other-interface-pins)

And the LTE Modem signals PinePhone on this GPIO Pin for __Incoming Calls and SMS Messages__...

-   __Ring Indicator__ → A64 Port __PL6__

    Indicates Incoming Calls and SMS Messages.

    [(See this)](https://lupyuen.github.io/articles/lte#main-uart-interface)

Let's power up the LTE Modem...

![LTE Modem Power](https://lupyuen.github.io/images/lte-title1.jpg)

# LTE Modem Power

_How will we power up the LTE Modem?_

[__PinePhone Schematic__](https://files.pine64.org/doc/PinePhone/PinePhone%20v1.2b%20Released%20Schematic.pdf) (Page 15) says that PinePhone controls the power via __GPIO Pin PL7__ (pic above)...

-   __RF Power__ ← A64 Port __PL7__

    Supplies power to the Radio Frequency Transmitter and Receiver.
    
    (4G LTE and GPS Transceiver)

-   __Baseband Power__ ← A64 Port __PL7__

    Supplies power to the Baseband Processor.

    (4G LTE and GPS Radio Functions)

__GPIO Pin PL7__ (bottom left) switches on the __Battery Power 4G-BAT__ (top left)...

![LTE Modem Power](https://lupyuen.github.io/images/lte-power.png)

[_PinePhone Schematic (Page 15)_](https://files.pine64.org/doc/PinePhone/PinePhone%20v1.2b%20Released%20Schematic.pdf)

Which powers the __LTE Modem via VBAT_BB__ (top right).

Thus LTE Modem won't power up without the Lithium-ion Battery.

[(__WPM1481__ is a Power Controller)](https://datasheet.lcsc.com/lcsc/1811131731_WILLSEMI-Will-Semicon-WPM1481-6-TR_C239798.pdf)

_What's Switch SW1-A? (Bottom left)_

[__Hardware Privacy Switch 1 (Modem)__](https://wiki.pine64.org/index.php/PinePhone#Privacy_switch_configuration) should be set to "On".

Or the LTE Modem won't power on!

(Indeed that's a Hardware Switch, not a Soft Switch)

_So we set GPIO Pin PL7 and the modem powers on?_

There's a __Soft Switch__ inside the LTE Modem that we need to toggle...

-   __Power Key__ ← A64 Port __PB3__

    Power up the LTE Modem.
    
    [(See this)](https://lupyuen.github.io/articles/lte#power-on--off)

Sounds complicated, but we'll explain the complete Power Up Sequence in a while.

(Power Key works like the press-and-hold Power Button on vintage Nokia Phones)

_Anything else to power up the LTE Modem?_

In a while we'll set the __Reset Pin__ and check the __Status Pin__...

-   __Reset__ ← A64 Port __PC4__

    Reset the LTE Modem.

    [(See this)](https://lupyuen.github.io/articles/lte#power-on--off)

-   __Status__ → A64 Port __PH9__

    Read the Modem Status.

    [(See this)](https://lupyuen.github.io/articles/lte#status-indication)

We need to program PinePhone's __Power Management Integrated Circuit (PMIC)__ to supply __3.3 V on DCDC1__.  Here's why...

![LTE Modem Power Output](https://lupyuen.github.io/images/lte-title2.jpg)

# Power Output

_Wait there's a Power Output for the LTE Modem?_

Yeah it gets confusing. The LTE Modem __outputs 1.8 Volts__ to PinePhone (pic above)...

-   __Power Output (1.8 V)__ → PinePhone __VDD_EXT__

    Power Output from LTE Modem to PinePhone. (1.8 V)

    [(See this)](https://lupyuen.github.io/articles/lte#power-supply)

Which goes into PinePhone's __Voltage Translators__ as __VDD_EXT__ (top left)...

![LTE Modem Power Output](https://lupyuen.github.io/images/lte-vddext.png)

[_PinePhone Schematic (Page 15)_](https://files.pine64.org/doc/PinePhone/PinePhone%20v1.2b%20Released%20Schematic.pdf)

[(__TXB0104__ is a Voltage Translator)](https://www.ti.com/lit/ds/symlink/txb0104.pdf)

The circuit above converts the __UART Signals__ (TX / RX / CTS / RTS)...

-   From __1.8 V (LTE Modem, left)__

-   To __3.3 V (PinePhone, right)__

Voltage Translators are also used for the [__LTE Modem Control Pins__](https://lupyuen.github.io/articles/lte#control-pins-for-lte-modem).

_What's DCDC1? (Top right)_

We need to program PinePhone's [__Power Management Integrated Circuit (PMIC)__](https://lupyuen.github.io/articles/de#appendix-power-management-integrated-circuit) to supply __3.3 V on DCDC1__.

Otherwise the UART Port (and the Control Pins) will get blocked by the Voltage Translators.

_Why 1.8 V for the LTE Modem?_

Most parts of the LTE Modem run on 3.3 V... Just that it needs to power up the __SIM Card at 1.8 V__. [(See this)](https://en.wikipedia.org/wiki/SIM_card#Design)

(Remember that the SIM Card is actually a microcontroller)

This [__Low Voltage Signaling__](https://www.sdcard.org/developers/sd-standard-overview/low-voltage-signaling/) is probably meant for newer, power-efficient gadgets. [(Like this)](https://www.sdcard.org/developers/sd-standard-overview/low-voltage-signaling/)

![LTE Modem inside PinePhone](https://lupyuen.github.io/images/lte-title.jpg)

# Power On LTE Modem

_Whoa LTE Modem has more pins than a Bowling Alley! (Pic above)_

_How exactly do we power up the LTE Modem?_

Earlier we spoke about PinePhone's __GPIO Pins__ that control the LTE Modem...

| LTE Modem Pin | A64 GPIO Pin |
|:--------------|:--------:|
| [__RF Power__](https://lupyuen.github.io/articles/lte#lte-modem-power) | ← __PL7__
| [__Baseband Power__](https://lupyuen.github.io/articles/lte#lte-modem-power) | ← __PL7__
| [__Reset__](https://lupyuen.github.io/articles/lte#lte-modem-power) | ←  __PC4__
| [__Power Key__](https://lupyuen.github.io/articles/lte#lte-modem-power) | ← __PB3__
| [__Disable__](https://lupyuen.github.io/articles/lte#control-pins-for-lte-modem) | ← __PH8__
| [__Status__](https://lupyuen.github.io/articles/lte#lte-modem-power) | → __PH9__

This is how we control the GPIO Pins to __power up the LTE Modem__...

1.  Program PinePhone's [__Power Management Integrated Circuit (PMIC)__](https://lupyuen.github.io/articles/de#appendix-power-management-integrated-circuit) to supply __3.3 V on DCDC1__ [(Like this)](https://github.com/lupyuen2/wip-nuttx/blob/0216f6968a82a73b67fb48a276b3c0550c47008a/boards/arm64/a64/pinephone/src/pinephone_pmic.c#L294-L340)

    We skip this because __DCDC1 is already powered on__.

1.  Set __PL7 to High__ to power on the RF Transceiver and Baseband Processor

1.  Set __PC4 to Low__ to deassert LTE Modem Reset

1.  Set __PH7 (AP-READY) to Low__ to wake up the modem

1.  Set __PB2 (DTR) to Low__ to wake up the modem

1.  __Wait 30 milliseconds__ for VBAT Power Supply to be stable

1.  Toggle __PB3 (Power Key)__ to start the LTE Modem, like this:

    Set __PB3 to High__...

    And wait __600 milliseconds__...
    
    Then set __PB3 to Low__.

1.  Set __PH8 to High__ to disable Airplane Mode

1.  __Read PH9__ to check the LTE Modem Status:

    PH9 goes from __High to Low__ when the LTE Modem is ready, in 2.5 seconds.

1.  __UART and USB Interfaces__ will be operational in 13 seconds

[__EG25-G Hardware Design__](https://wiki.pine64.org/images/2/20/Quectel_EG25-G_Hardware_Design_V1.4.pdf) (Page 41) beautifully illustrates the __Power On Sequence__...

![LTE Modem Power](https://lupyuen.github.io/images/lte-power2.png)

Note that Power Key and Reset are [__High-Low Inverted__](https://lupyuen.github.io/articles/lte#power-on--off) when accessed via GPIO Pins PC4 and PB3.

[(High becomes Low and vice versa)](https://lupyuen.github.io/articles/lte#power-on--off)

_Power Key looks funky: High → Low → High..._

Yeah the Power Key is probably inspired by the press-and-hold Power Button on vintage Nokia Phones.

[(Power Key works differently on pre-production PinePhones)](https://wiki.pine64.org/wiki/PinePhone_v1.1_-_Braveheart#Modem_PWR_KEY_signal_resistor_population)

Let's implement the steps with Apache NuttX RTOS...

[(Kudos to __Genode OS__ for the Power On Sequence)](https://github.com/genodelabs/genode-allwinner/blob/master/src/drivers/modem/pinephone/power.h#L52-L130)

# Power Up wth NuttX

_We've seen the Power On Sequence for LTE Modem..._

_How will we implement it in Apache NuttX RTOS?_

This is how we implement the LTE Modem's __Power On Sequence__ in NuttX: [pinephone_modem.c](https://github.com/lupyuen2/wip-nuttx/blob/modem/boards/arm64/a64/pinephone/src/pinephone_modem.c#L89-L132)

```c
// Read PH9 to check LTE Modem Status
#define STATUS (PIO_INPUT | PIO_PORT_PIOH | PIO_PIN9)
ret = a64_pio_config(STATUS);
DEBUGASSERT(ret == OK);
_info("Status=%d\n", a64_pio_read(STATUS));
```

[(__a64_pio_config__ comes from A64 PIO Driver)](https://github.com/apache/nuttx/blob/master/arch/arm64/src/a64/a64_pio.c#L175-L343)

[(__a64_pio_read__ too)](https://github.com/apache/nuttx/blob/master/arch/arm64/src/a64/a64_pio.c#L391-L419)

We begin by reading PH9 for the __LTE Modem Status__.

Then we set PL7 to High to __power up the RF Transceiver and Baseband Processor__: [pinephone_modem.c](https://github.com/lupyuen2/wip-nuttx/blob/modem/boards/arm64/a64/pinephone/src/pinephone_modem.c#L138-L245)

```c
// Set PL7 to High to Power On LTE Modem (4G-PWR-BAT)
// Configure PWR_BAT (PL7) for Output
#define P_OUTPUT (PIO_OUTPUT | PIO_PULL_NONE | PIO_DRIVE_MEDLOW | \
                  PIO_INT_NONE | PIO_OUTPUT_SET)
#define PWR_BAT (P_OUTPUT | PIO_PORT_PIOL | PIO_PIN7)
a64_pio_config(PWR_BAT);  // TODO: Check result

// Set PWR_BAT (PL7) to High
a64_pio_write(PWR_BAT, true);
// Omitted: Print the status
```

[(__a64_pio_write__ comes from A64 PIO Driver)](https://github.com/apache/nuttx/blob/master/arch/arm64/src/a64/a64_pio.c#L345-L389)

[(__pinephone_modem_init__ is called by __pinephone_bringup__)](https://github.com/lupyuen2/wip-nuttx/blob/modem/boards/arm64/a64/pinephone/src/pinephone_bringup.c#L211-L219)

We set PC4 to Low to __deassert the LTE Modem Reset__...

```c
// Set PC4 to Low to Deassert LTE Modem Reset (BB-RESET / RESET_N)
// Configure RESET_N (PC4) for Output
#define RESET_N (P_OUTPUT | PIO_PORT_PIOC | PIO_PIN4)
a64_pio_config(RESET_N);  // TODO: Check result

// Set RESET_N (PC4) to Low
a64_pio_write(RESET_N, false);
// Omitted: Print the status
```

Set PH7 (AP-READY) and PB2 (DTR) to Low to __wake up the modem__...

```c
// Set AP-READY (PH7) to Low to wake up modem
// Configure AP-READY (PH7) for Output and set to Low
#define AP_READY (P_OUTPUT | PIO_PORT_PIOH | PIO_PIN7)
a64_pio_config(AP_READY);  // TODO: Check result
a64_pio_write(AP_READY, false);

// Set DTR (PB2) to Low to wake up modem
// Configure DTR (PB2) for Output and set to Low
#define DTR (P_OUTPUT | PIO_PORT_PIOB | PIO_PIN2)
a64_pio_config(DTR);  // TODO: Check result
a64_pio_write(DTR, false);
```

Wait 30 milliseconds for the __power to be stable__...

```c
// Wait 30 ms
up_mdelay(30);
```

Now we __toggle PB3 for the Power Key__: High → 600 milliseconds → Low...

```c
// Set PB3 to Power On LTE Modem (BB-PWRKEY / PWRKEY).
// PWRKEY should be pulled down at least 500 ms, then pulled up.
// Configure PWRKEY (PB3) for Output
#define PWRKEY (P_OUTPUT | PIO_PORT_PIOB | PIO_PIN3)
a64_pio_config(PWRKEY);  // TODO: Check result

// Set PWRKEY (PB3) to High
a64_pio_write(PWRKEY, true);
// Omitted: Print the status

// Wait 600 ms for PWRKEY
up_mdelay(600);
// Omitted: Print the status

// Set PWRKEY (PB3) to Low
a64_pio_write(PWRKEY, false);
// Omitted: Print the status
```

Finally we set PH8 to High to __disable Airplane Mode__...

```c
// Set PH8 to High to Disable Airplane Mode (BB-DISABLE / W_DISABLE#)
// Configure W_DISABLE (PH8) for Output
#define W_DISABLE (P_OUTPUT | PIO_PORT_PIOH | PIO_PIN8)
a64_pio_config(W_DISABLE);  // TODO: Check result

// Set W_DISABLE (PH8) to High
a64_pio_write(W_DISABLE, true);
// Omitted: Print the status
```

To wrap up, we wait for __LTE Modem Status to turn Low__: [pinephone_modem.c](https://github.com/lupyuen2/wip-nuttx/blob/modem/boards/arm64/a64/pinephone/src/pinephone_modem.c#L89-L132)

```c
// Poll for Modem Status until it becomes Low
for (int i = 0; i < 30; i++) {  // Max 1 minute

  // Read the Modem Status
  uint32_t status = a64_pio_read(STATUS);
  // Omitted: Print the status

  // Stop if Modem Status is Low
  if (status == 0) { break; }

  // Wait 2 seconds
  up_mdelay(2000);
}
```

Let's run this!

![NuttX starts LTE Modem](https://lupyuen.github.io/images/lte-run2.png)

[_NuttX starts LTE Modem_](https://github.com/lupyuen2/wip-nuttx-apps/blob/86899c1350e2870ce686f6c1a01ba5541f243b77/examples/hello/hello_main.c#L168-L513)

# Is LTE Modem Up?

_We've implemented the Power On Sequence for LTE Modem..._

_Does it work on Apache NuttX RTOS?_

Yes it works! NuttX reads the Modem Status and __switches on the power__ (PL7)...

```text
Configure STATUS (PH9) for Input
Status=0

Set PWR_BAT (PL7) to High
Status=1
```

[(See the Complete Log)](https://github.com/lupyuen2/wip-nuttx-apps/blob/86899c1350e2870ce686f6c1a01ba5541f243b77/examples/hello/hello_main.c#L168-L513)

Then it __deasserts the reset__ (PC4) and __wakes up the modem__ (PH7 and PB2)...

```text
Set RESET_N (PC4) to Low
Set AP-READY (PH7) to Low to wake up modem
Set DTR (PB2) to Low to wake up modem

Wait 30 ms
Status=1
```

NuttX toggles the __Power Key (PB3)__: High → 600 milliseconds → Low...

```text
Set PWRKEY (PB3) to High
Wait 600 ms

Set PWRKEY (PB3) to Low
Status=1
```

And it __disables Airplane Mode__ (PH8)...

```text
Set W_DISABLE (PH8) to High
Status=1
```

Finally it waits for the __LTE Modem Status (PH9)__ to turn Low...

```text
pinephone_modem_init: Status=1
pinephone_modem_init: Status=1
pinephone_modem_init: Status=1
pinephone_modem_init: Status=0
```

And the LTE Modem is up! (Roughly 6 seconds)

[(See the Complete Log)](https://github.com/lupyuen2/wip-nuttx-apps/blob/86899c1350e2870ce686f6c1a01ba5541f243b77/examples/hello/hello_main.c#L168-L513)

This matches the __Power Up Sequence__ that we saw earlier...

![LTE Modem Power](https://lupyuen.github.io/images/lte-power2.png)

[(EG25-G Hardware Design, Page 41)](https://wiki.pine64.org/images/2/20/Quectel_EG25-G_Hardware_Design_V1.4.pdf)

[(Power Key and Reset are __High-Low Inverted__)](https://lupyuen.github.io/articles/lte#power-on--off)

_Will the LTE Modem accept AT Commands now?_

Not yet. The LTE Modem might take __30 seconds__ to be fully operational!

Let's observe the UART Port...

![PinePhone Schematic (Page 15)](https://lupyuen.github.io/images/lte-vddext.png)

[_PinePhone Schematic (Page 15)_](https://files.pine64.org/doc/PinePhone/PinePhone%20v1.2b%20Released%20Schematic.pdf)

[(__TXB0104__ is a Voltage Translator)](https://www.ti.com/lit/ds/symlink/txb0104.pdf)

# Test UART with NuttX

_LTE Modem has started successfully on NuttX..._

_How will we send AT Commands to the modem?_

The LTE Modem is connected to PinePhone (Allwinner A64) at these UART Ports (pic above)...

-   __A64 Port UART3__: RX and TX

    (GPIO PD1 and PD0)

    (Default 115.2 kbps, up to 921.6 kbps)

-   __A64 Port UART4__: CTS and RTS

    (GPIO PD5 and PD4)

    [(CTS and RTS might not work)](https://wiki.pine64.org/wiki/PinePhone_v1.1_-_Braveheart#Modem_UART_flow_control_is_broken)

-   __A64 Port PB2__: DTR

    [(More about the UART Pins)](https://lupyuen.github.io/articles/lte#main-uart-interface)

Thus we may __check UART3__ to see if the LTE Modem responds to [__AT Commands__](https://lupyuen.github.io/articles/lte#quectel-eg25-g-lte-modem): [hello_main.c](https://github.com/lupyuen2/wip-nuttx-apps/blob/modem/examples/hello/hello_main.c#L69-L221)

```c
// Open /dev/ttyS1 (UART3)
int fd = open("/dev/ttyS1", O_RDWR);
printf("Open /dev/ttyS1: fd=%d\n", fd);
assert(fd > 0);

// Repeat 5 times: Write command and read response
for (int i = 0; i < 5; i++) {

  // Write command
  const char cmd[] = "AT\r";
  ssize_t nbytes = write(fd, cmd, strlen(cmd));
  printf("Write command: nbytes=%ld\n%s\n", nbytes, cmd);
  assert(nbytes == strlen(cmd));

  // Read response
  static char buf[1024];
  nbytes = read(fd, buf, sizeof(buf) - 1);
  if (nbytes >= 0) { buf[nbytes] = 0; }
  else { buf[0] = 0; }
  printf("Response: nbytes=%ld\n%s\n", nbytes, buf);

  // Wait a while
  sleep(2);
}

// Close the device
close(fd);
```

The NuttX App above sends the command "__`AT`__" to the LTE Modem over UART3. (5 times)

Watch what happens when we run it...

![Testing LTE Modem over UART](https://lupyuen.github.io/images/lte-run3a.png)

[(See the Complete Log)](https://github.com/lupyuen2/wip-nuttx-apps/blob/86899c1350e2870ce686f6c1a01ba5541f243b77/examples/hello/hello_main.c#L168-L513)

Our NuttX App sends command "__`AT`__" to the LTE Modem over UART3...

```text
Open /dev/ttyS1
Write command: AT\r
```

But it hangs there. No response!

Remember that the LTE Modem might take __30 seconds__ to become operational. Be patient, the response appears in a while...

```text
Response:
RDY
```

"__`RDY`__" means that the LTE Modem is ready for AT Commands!

[(EG25-G AT Commands, Page 297)](https://wiki.pine64.org/images/1/1b/Quectel_EC2x%26EG9x%26EG2x-G%26EM05_Series_AT_Commands_Manual_V2.0.pdf)

Our NuttX App sends command "__`AT`__" again...

```text
Write command: AT\r
Response:
+CFUN: 1
+CPIN: NOT INSERTED
```

LTE Modem replies...

- "__`+CFUN: 1`__"

  This says that the LTE Modem is fully operational

  [(EG25-G AT Commands, Page 33)](https://wiki.pine64.org/images/1/1b/Quectel_EC2x%26EG9x%26EG2x-G%26EM05_Series_AT_Commands_Manual_V2.0.pdf)

- "__`+CPIN: NOT INSERTED`__"

  This says that we haven't inserted a SIM Card into the LTE Modem

  [(EG25-G AT Commands, Page 60)](https://wiki.pine64.org/images/1/1b/Quectel_EC2x%26EG9x%26EG2x-G%26EM05_Series_AT_Commands_Manual_V2.0.pdf)

Our NuttX App sends command "__`AT`__" once more...

```text
Write command: AT\r
Response:
AT
OK
```

LTE Modem echoes our command "__`AT`__"...

And responds to our command with  "__`OK`__".

Which means that our LTE Modem is running AT Commands all OK!

[(See the Complete Log)](https://github.com/lupyuen2/wip-nuttx-apps/blob/86899c1350e2870ce686f6c1a01ba5541f243b77/examples/hello/hello_main.c#L168-L513)

_UART3 works with NuttX?_

We modified the __Allwinner A64 UART Driver__ to support UART3. The changes will be upstreamed to NuttX Mainline later...

-   [__"Configure UART Port"__](https://github.com/lupyuen/pinephone-nuttx#configure-uart-port)

-   [__"Test UART3 Port"__](https://github.com/lupyuen/pinephone-nuttx#test-uart3-port)

-   [__"NuttX RTOS for PinePhone: Phone Calls and Text Messages"__](https://lupyuen.github.io/articles/lte2)

    [(See the __Incomplete Pull Request__)](https://github.com/apache/nuttx/pull/9304)

TODO: Disable UART2 and /dev/ttyS2 so that UART3 maps neatly to /dev/ttyS3. [(See this)](https://github.com/apache/nuttx/pull/9304#discussion_r1195862416)

There's another way to test the LTE Modem: Via USB...

![USB Controller Block Diagram from Allwinner A64 User Manual](https://lupyuen.github.io/images/usb3-title.jpg)

[_USB Controller Block Diagram from Allwinner A64 User Manual_](https://github.com/lupyuen/pinephone-nuttx/releases/download/doc/Allwinner_A64_User_Manual_V1.1.pdf)

# Test USB with NuttX

_We talked about testing the LTE Modem the UART way..._

_What about the USB way?_

Yep the __USB Interface__ should work for testing the LTE Modem...

-   __USB__ ⇆ A64 Port __USB1__ _(USB Serial)_

    (Up to 480 Mbps)

_How's that coming along?_

We fixed the [__NuttX USB EHCI Driver__](https://lupyuen.github.io/articles/usb3) (pic above) to handle USB Interrupts...

-   [__"Enumerate USB Devices on PinePhone"__](https://github.com/lupyuen/pinephone-nuttx-usb#enumerate-usb-devices-on-pinephone)

-   [__"Handle USB Interrupt"__](https://github.com/lupyuen/pinephone-nuttx-usb#handle-usb-interrupt)

But somehow the LTE Modem __isn't triggering any USB Interrupts__ (13 seconds after startup)...

Which __fails the enumeration__ of USB Devices (like the LTE Modem). And we can't connect to the USB Interface of the LTE Modem.

But then we discovered that the GPIO Pins for Power Key and Reset are [__High-Low Inverted__](https://lupyuen.github.io/articles/lte#power-on--off). So we need to retest.

Stay tuned for updates on the USB Testing!

[(This crash needs to be fixed when __USB Hub Support__ is enabled)](https://github.com/lupyuen/pinephone-nuttx-usb#ls-crashes-when-usb-hub-support-is-enabled)

![Quectel EG25-G LTE Modem inside PinePhone](https://lupyuen.github.io/images/wayland-sd.jpg)

[_Quectel EG25-G LTE Modem inside PinePhone_](https://wiki.pine64.org/index.php/PinePhone#Modem)

# What's Next

I hope this article was helpful for learning about PinePhone's 4G LTE Modem...

-   What's the __Quectel EG25-G LTE Modem__

-   How it's __connected inside PinePhone__

-   How we make __Phone Calls__ and send __Text Messages__

-   How we __power up__ the LTE Modem

-   __Programming the LTE Modem__ with UART, USB and Apache NuttX RTOS

Up Next: Find out how we make phone calls and send text messages with PinePhone's 4G LTE Modem...

-   [__"NuttX RTOS for PinePhone: Phone Calls and Text Messages"__](https://lupyuen.github.io/articles/lte2)

Also check out the other articles on NuttX for PinePhone...

-   [__"Apache NuttX RTOS for PinePhone"__](https://github.com/lupyuen/pinephone-nuttx)

Many Thanks to my [__GitHub Sponsors__](https://lupyuen.github.io/articles/sponsor) for supporting my work! This article wouldn't have been possible without your support.

-   [__Sponsor me a coffee__](https://lupyuen.github.io/articles/sponsor)

-   [__Discuss this article on Reddit__](https://www.reddit.com/r/PINE64official/comments/12i3qzi/nuttx_rtos_for_pinephone_4g_lte_modem/)

-   [__Discuss this article on Hacker News__](https://news.ycombinator.com/item?id=35519549)

-   [__My Current Project: "Apache NuttX RTOS for Ox64 BL808"__](https://github.com/lupyuen/nuttx-ox64)

-   [__My Other Project: "NuttX for Star64 JH7110"__](https://github.com/lupyuen/nuttx-star64)

-   [__Older Project: "NuttX for PinePhone"__](https://github.com/lupyuen/pinephone-nuttx)

-   [__Check out my articles__](https://lupyuen.github.io)

-   [__RSS Feed__](https://lupyuen.github.io/rss.xml)

_Got a question, comment or suggestion? Create an Issue or submit a Pull Request here..._

[__lupyuen.github.io/src/lte.md__](https://github.com/lupyuen/lupyuen.github.io/blob/master/src/lte.md)

# Notes

1.  There's plenty more inside PinePhone's LTE Modem. Check out these articles...

    [__"Genode: PinePhone Telephony"__](https://genodians.org/ssumpf/2022-05-09-telephony)

    [__"PinePhone Power Management"__](https://wiki.pine64.org/wiki/PinePhone_Power_Management)

    [__"Modem on PinePhone"__](https://xnux.eu/devices/feature/modem-pp.html)

    [__"Audio on PinePhone"__](https://xnux.eu/devices/feature/audio-pp.html)

    [__"EG25-G Reverse Engineering"__](https://xnux.eu/devices/feature/modem-pp-reveng.html)

    [__"OSDev: PinePhone"__](https://wiki.osdev.org/PinePhone)

1.  Take note of these __Limitations and Design Decisions__...

    [__"Modem UART flow control is broken"__](https://wiki.pine64.org/wiki/PinePhone_v1.1_-_Braveheart#Modem_UART_flow_control_is_broken)

    [__"Modem PWR_KEY signal resistor population"__](https://wiki.pine64.org/wiki/PinePhone_v1.1_-_Braveheart#Modem_PWR_KEY_signal_resistor_population)

1.  __LTE Modem Pinout__ is different for pre-production editions of PinePhone! (Like Braveheart 1.1)

    I'm using PinePhone Hardware Revision 1.2 (Ubports Edition)...

    [__"PinePhone Hardware Revisions"__](https://wiki.pine64.org/wiki/PinePhone#Hardware_revisions)

1.  Why are we experimenting with __BOTH UART and USB__ for the LTE Modem?

    [__Here's the long story__](https://qoto.org/@lupyuen/110241405945668088)

![LTE Modem inside PinePhone](https://lupyuen.github.io/images/lte-title.jpg)

# Appendix: LTE Modem Pins

_What's the purpose of the above LTE Modem pins?_

This section describes the purpose of every LTE Modem pin connected to PinePhone...

## Power Supply

From [__EG25-G Hardware Design__](https://wiki.pine64.org/images/2/20/Quectel_EG25-G_Hardware_Design_V1.4.pdf) (Page 22)...

| Pin Name | Pin No. | I/O | Description
|:---------|:-------:|:---:|:-----------
| VDD_EXT | 7 | PO | Provide 1.8 V for external circuit

[(__PO__ is Power Output)](https://lupyuen.github.io/articles/lte#io-parameters-definition)

## Power On / Off

From [__EG25-G Hardware Design__](https://wiki.pine64.org/images/2/20/Quectel_EG25-G_Hardware_Design_V1.4.pdf) (Page 22)...

| Pin Name | Pin No. | I/O | Description
|:---------|:-------:|:---:|:-----------
| PWRKEY | 21 | DI | Turn on / off the module
| RESET_N | 20 | DI | Reset signal of the module

[(__DI__ is Digital Input)](https://lupyuen.github.io/articles/lte#io-parameters-definition)

-   __PWRKEY__ should be pulled down at least 500 ms, then pulled up
    
    [(EG25-G Hardware Design, Page 41)](https://wiki.pine64.org/images/2/20/Quectel_EG25-G_Hardware_Design_V1.4.pdf)

-   "Make sure that __VBAT is stable__ before pulling down PWRKEY pin. It is recommended that the time between powering up VBAT and pulling down PWRKEY pin is no less than 30 ms."
    
    [(EG25-G Hardware Design, Page 41)](https://wiki.pine64.org/images/2/20/Quectel_EG25-G_Hardware_Design_V1.4.pdf)

-   "__RESET_N__ pin can be used to reset the module. The module can be reset by driving RESET_N to a low level voltage for 150–460 ms"

    [(EG25-G Hardware Design, Page 42)](https://wiki.pine64.org/images/2/20/Quectel_EG25-G_Hardware_Design_V1.4.pdf)

-   Note: __PWRKEY and RESET_N__ are __High-Low Inverted__ when accessed through PinePhone's GPIO Pins

    (High becomes Low and vice versa)

    ![PWRKEY is High-Low Inverted](https://lupyuen.github.io/images/lte-powerkey.png)

    ![RESET_N is High-Low Inverted](https://lupyuen.github.io/images/lte-reset.png)

    [(EG25-G Hardware Design, Pages 40 and 43)](https://wiki.pine64.org/images/2/20/Quectel_EG25-G_Hardware_Design_V1.4.pdf)

## Status Indication

From [__EG25-G Hardware Design__](https://wiki.pine64.org/images/2/20/Quectel_EG25-G_Hardware_Design_V1.4.pdf) (Page 22)...

| Pin Name | Pin No. | I/O | Description
|:---------|:-------:|:---:|:-----------
| STATUS | 61 | OD | Indicate the module operating status

[(__OD__ is Open Drain)](https://lupyuen.github.io/articles/lte#io-parameters-definition)

-   When PWRKEY is pulled Low, __STATUS goes High__ for ≥2.5 s, then STATUS goes Low

    [(EG25-G Hardware Design, Page 41)](https://wiki.pine64.org/images/2/20/Quectel_EG25-G_Hardware_Design_V1.4.pdf)

## USB Interface

From [__EG25-G Hardware Design__](https://wiki.pine64.org/images/2/20/Quectel_EG25-G_Hardware_Design_V1.4.pdf) (Page 22)...

| Pin Name | Pin No. | I/O | Description
|:---------|:-------:|:---:|:-----------
| USB_VBUS | 71 | PI | USB connection detection

[(__PI__ is Power Input)](https://lupyuen.github.io/articles/lte#io-parameters-definition)

## Main UART Interface

From [__EG25-G Hardware Design__](https://wiki.pine64.org/images/2/20/Quectel_EG25-G_Hardware_Design_V1.4.pdf) (Page 24)...

| Pin Name | Pin No. | I/O | Description
|:---------|:-------:|:---:|:-----------
| RI | 62 | DO | Ring indicator
| CTS | 64 | DO | Clear to send
| RTS | 65 | DI | Request to send
| DTR | 66 | DI | Data terminal ready, sleep mode control
| TXD | 67 | DO | Transmit data
| RXD | 68 | DI | Receive data

[(__DO__ is Digital Output)](https://lupyuen.github.io/articles/lte#io-parameters-definition)

[(__DI__ is Digital Input)](https://lupyuen.github.io/articles/lte#io-parameters-definition)

-   __Voltage Level__ is 1.8 V

-   __Ring Indicator__ is configured with AT Command "__`AT+QCFG`__"  [(See this)](https://lupyuen.github.io/articles/lte2#appendix-receive-phone-call-and-sms)

-   __CTS and RTS__ might not work correctly on PinePhone [(See this)](https://wiki.pine64.org/wiki/PinePhone_v1.1_-_Braveheart#Modem_UART_flow_control_is_broken)

-   __DTR__ is pulled up by default. Low level wakes up the module.

-   __TXD and RXD__ (UART) default to 115.2 kbps, support up to 921.6 kbps

## Other Interface Pins

From [__EG25-G Hardware Design__](https://wiki.pine64.org/images/2/20/Quectel_EG25-G_Hardware_Design_V1.4.pdf) (Page 32)...

| Pin Name | Pin No. | I/O | Description
|:---------|:-------:|:---:|:-----------
| W_DISABLE# | 4 | DI | Airplane mode control
| AP_READY | 2 | DI | Application processor sleep state detection

[(__DI__ is Digital Input)](https://lupyuen.github.io/articles/lte#io-parameters-definition)

-   __Voltage Level__ is 1.8 V

-   "__W_DISABLE#__ pin is pulled up by default. Driving it to low level will let the module enter airplane mode"

    [(EG25-G Hardware Design, Page 37)](https://wiki.pine64.org/images/2/20/Quectel_EG25-G_Hardware_Design_V1.4.pdf)

## I/O Parameters Definition

From [__EG25-G Hardware Design__](https://wiki.pine64.org/images/2/20/Quectel_EG25-G_Hardware_Design_V1.4.pdf) (Page 21)...

| Type | Description
|:-----|:-----------
| AI | Analog Input
| AO | Analog Output
| DI | Digital Input
| DO | Digital Output
| IO | Bidirectional
| OD | Open Drain
| PI | Power Input
| PO | Power Output
