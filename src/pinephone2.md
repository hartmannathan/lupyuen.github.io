# NuttX RTOS for PinePhone: The First Year

📝 _21 Jun 2023_

![Apache NuttX RTOS for Pine64 PinePhone](https://lupyuen.github.io/images/pinephone2-title.jpg)

[(Watch the Presentation on Google Drive)](https://drive.google.com/file/d/1WL-6HVjhtqktHRmZiDbPCOs6934fQlEQ/view?usp=drive_link)

One year ago we started porting [__Apache NuttX RTOS__](https://lupyuen.github.io/articles/what) (Real-Time Operating System) to [__Pine64 PinePhone__](https://wiki.pine64.org/index.php/PinePhone)...

-   [__"NuttX RTOS for PinePhone: What is it?"__](https://lupyuen.github.io/articles/what)

Let's look back and talk about...

-   __The Features__ that we've implemented

-   __Our Plans__ for the future

-   Why we might move to a __RISC-V Tablet__!

![Roadmap for PinePhone NuttX](https://lupyuen.github.io/images/pinephone2-roadmap.jpg)

# Roadmap for PinePhone NuttX

The pic above shows the __features we planned__ for NuttX on PinePhone...

|       |     |
|:-----:|:----|
|__`✓`__ | Implemented
|__`?`__ | Not Yet
|__`✓ ?`__ | Partially Implemented

Over the past year we've carefully documented the __entire porting process__ in a series of 24 articles (enough to fill a book)...

-   [__"Apache NuttX RTOS for PinePhone"__](https://github.com/lupyuen/pinephone-nuttx#apache-nuttx-rtos-for-pinephone)

(Yep we targeted 1 article every 2 weeks!)

_Why did we do all this?_

1.  We ported NuttX RTOS to PinePhone as an __Educational Experiment__.

    NuttX on PinePhone could become a Teaching Tool for learning the inner workings of a Smartphone.

1.  Some folks think that NuttX might become a __Benchmarking Tool__.

    How fast can PinePhone actually run? Just run barebones real-time NuttX on PinePhone and measure the performance... Without any unnecessary overheads!

_Is NuttX on PinePhone a Daily Driver?_

We're not quite ready for that, we'll see why in a while.

Let's begin with the Touchscreen Features...

> ![PinePhone Touchscreen](https://lupyuen.github.io/images/pinephone2-roadmap1.jpg)

# PinePhone Touchscreen

We're incredibly fortunate that __PinePhone's Touchscreen__ runs OK with NuttX, after we built these features (pic above)...

-   [__MIPI Display Serial Interface (DSI)__](https://lupyuen.github.io/articles/dsi3) transmits pixel data to the [__LCD Panel__](https://lupyuen.github.io/articles/lcd)

-   [__Allwinner Display Engine__](https://lupyuen.github.io/articles/de3) renders bitmap graphics and pushes the pixels over MIPI DSI

-   [__NuttX Framebuffer__](https://lupyuen.github.io/articles/fb) exposes the rendering API to NuttX Apps

-   [__I2C Touch Panel__](https://lupyuen.github.io/articles/touch2) detects Touch Input from the LCD Panel

-   [__LVGL Graphics Library__](https://lupyuen.github.io/articles/lvgl2) renders User Interfaces and handles Touch Input

-   [__LVGL Terminal__](https://lupyuen.github.io/articles/terminal) is a Touchscreen App that we created with LVGL

-   [__WebAssembly Simulator__](https://lupyuen.github.io/articles/lvgl4) previews Touchscreen Apps in the Web Browser

Today with NuttX for PinePhone, we can create __Touchscreen Apps__ that will work like a regular Smartphone App!

(But we're not yet a Complete Smartphone, we'll come back to this)

Let's talk about the Sensors inside PinePhone...

> ![PinePhone Sensors](https://lupyuen.github.io/images/pinephone2-roadmap3.jpg)

# PinePhone Sensors

Our support for __PinePhone's Sensors__ is a little spotty...

-   [__Accelerometer and Gyroscope__](https://www.hackster.io/lupyuen/inside-a-smartphone-accelerometer-pinephone-with-nuttx-rtos-b92b58) will detect PinePhone motion and orientation

-   __Magnetometer, Light and Proximity Sensors__ are not yet supported

-   __Front and Rear Cameras__ are not supported

-   [__Power Management__](https://lupyuen.github.io/articles/lcd#power-on-lcd-panel) is partially implemented.

    PinePhone's LCD Display and Sensors will power on correctly, but...

-   __Battery Charging__ and __Sleep Mode__ are not done yet

_Can we build the missing drivers with NuttX?_

Most certainly! Though to me it's starting feel a bit like "grinding". (Like the pic at the top of the article)

But it would be a highly educational experience for Embedded Learners!

Let's talk about the key component inside PinePhone...

![LTE Modem](https://lupyuen.github.io/images/pinephone2-roadmap5.jpg)

# LTE Modem

_What makes PinePhone a Phone?_

It's the __4G LTE Modem__ inside PinePhone! Let's walk through the features for Phone Calls, SMS and GPS (pic above)...

-   [__Outgoing Calls__](https://lupyuen.github.io/articles/lte2#outgoing-phone-call) and [__Outgoing SMS__](https://lupyuen.github.io/articles/lte2#send-sms-in-pdu-mode) are OK, but...

-   __PCM Audio__ is [not implemented](https://lupyuen.github.io/articles/lte2#appendix-pcm-digital-audio), so we won't have audio

-   __Incoming Calls__ and __Incoming SMS__: [Not yet](https://lupyuen.github.io/articles/lte2#appendix-receive-phone-call-and-sms)

-   [__UART Interface__](https://lupyuen.github.io/articles/lte2#send-at-commands) is ready for Voice Call and SMS Commands

-   __USB Interface__ is [not ready](https://lupyuen.github.io/articles/lte#test-usb-with-nuttx), so we won't have [__GPS__](https://lupyuen.github.io/articles/lte#data-interfaces-for-lte-modem)

-   [__USB EHCI Controller__](https://lupyuen.github.io/articles/usb3) is partially done

-   __USB OTG Controller__: [Not started](https://lupyuen.github.io/articles/usb3#ehci-is-simpler-than-usb-on-the-go)

With the LTE Modem partially supported, we could build a Feature Phone...

![Feature Phone](https://lupyuen.github.io/images/pinephone2-roadmap4.jpg)

# NuttX Feature Phone

_We've done quite a bit with the LTE Modem..._

_Are we a Feature Phone yet?_

Almost! Let's talk about the User Interface, Phone Calls and SMS needed for a __Feature Phone__ (pic above)...

-   We've created a [__Feature Phone UI__](https://lupyuen.github.io/articles/lvgl4) as an LVGL Touchscreen App

-   That also runs in the [__Web Browser with WebAssembly__](https://lupyuen.github.io/articles/lvgl4#run-lvgl-app-in-web-browser)

-   We need to integrate [__Outgoing Calls__](https://lupyuen.github.io/articles/lte2#outgoing-phone-call) and [__Outgoing SMS__](https://lupyuen.github.io/articles/lte2#send-sms-in-pdu-mode) into our Feature Phone App

-   Though __PCM Audio__, __Incoming Calls__ and __Incoming SMS__ are [still missing](https://lupyuen.github.io/articles/pinephone2#lte-modem)

It's sad that we haven't done PCM Audio. It would've been a terrific educational exercise. And we'd have a working Feature Phone!

Let's head back to our question about NuttX as a Daily Driver...

![Smartphone](https://lupyuen.github.io/images/pinephone2-roadmap6.jpg)

# NuttX Smartphone

_OK we're almost a Feature Phone..._

_But are we a Smartphone yet?_

Sorry we're not quite ready to be a __Smartphone__ (pic above), because...

-   __Wireless Networking__ is completely missing: __Bluetooth LE__, __WiFi__ and __Mobile Data__

    (Which will require plenty of coding)

-   __LoRa Networking__ with the [__LoRa Add-On Case__](https://lupyuen.github.io/articles/usb2#appendix-lora-communicator-for-pinephone-on-nuttx) will be really interesting, but sadly missing today

    (Mesh Networking with __Meshtastic__ would be awesome)

-   __USB EHCI and OTG__ [won't work either](https://lupyuen.github.io/articles/pinephone2#lte-modem)

If we had the energy (and patience), we should definitely do LoRa with Meshtastic on PinePhone!

Our Daily Driver also needs these features...

> ![Core Features](https://lupyuen.github.io/images/pinephone2-roadmap2.jpg)

# Core Features

_What else do we need for a Smartphone..._

_Have we missed any Core Features?_

Yeah these are the __Core Features__ needed to complete our Smartphone OS (pic above)...

-   __Multiple CPUs__ are not working yet, we're running on a Single Core today

-   __Memory Management__ will be needed for Virtual Memory and to protect the NuttX Kernel

-   __App Security__ needs to be implemented (similar to SELinux and AppArmor)

-   __eMMC and microSD Storage__ won't work (because we're running in RAM)

-   __GPU__ will be needed for serious graphics

-   [__PinePhone Emulator__](https://lupyuen.github.io/articles/unicorn.html) will be super helpful for testing the above features

Some of these features are probably supported by NuttX already. But we need to test thoroughly on PinePhone. (Hence the PinePhone Emulator)

Unfortunately we're running out of time...

![Rolling to RISC-V](https://lupyuen.github.io/images/pinephone2-roadmap6.jpg)

# Rolling to RISC-V

1.  _Fixing up NuttX for PinePhone..._

    _Surely we can do that for the next couple of months?_

    Allwinner A64 SoC was released in 2015... That's __8 years ago__!

    Before Allwinner A64 becomes obsolete, maybe we should consider a newer device?

    [(NuttX might still run on other Allwinner A64 handhelds)](https://retrododo.com/funnyplaying-retro-pixel-pocket/)

1.  _Like PinePhone Pro? Or PineTab 2?_

    Well that's more of the same same Arm64, innit?

    Just follow the exact same steps we've meticulously documented for NuttX on PinePhone...
    
    And NuttX will (probably) run on __any Arm64 Device__: iPhone, Samsung Phones, Tablets, Gaming Handhelds, ...

    [(Like the super-impressive Mobile Linux ecosystem)](https://postmarketos.org/)

1.  _So we're moving from Arm64 to RISC-V?_

    Yep! We have a fresh new opportunity to teach the __RISC-V 64-bit Architecture__ from scratch.

    And hopefully RISC-V Devices will still be around after 8 years!

1.  _We're porting NuttX to a RISC-V Phone?_

    Sadly there isn't a __RISC-V Phone__ yet.
    
    Thus we'll port NuttX to a RISC-V Tablet instead: [__PineTab-V__](https://wiki.pine64.org/wiki/PineTab-V)

1.  _But PineTab-V isn't shipping yet!_

    That's OK, we'll begin by porting NuttX to the [__Star64 SBC__](https://wiki.pine64.org/wiki/STAR64)

    Which runs on the same RISC-V SoC as PineTab-V: [__StarFive JH7110__](https://doc-en.rvspace.org/Doc_Center/jh7110.html)

    (Hopefully we have better docs and tidier code than the older Arm64 SoCs)

1.  _Hopping from Arm64 to RISC-V sounds like a major migration..._

    Actually we planned for this [__one year ago__](https://www.mail-archive.com/dev@nuttx.apache.org/msg08395.html).

    NuttX already runs OK on the (64-bit) [__QEMU RISC-V Emulator__](https://lupyuen.github.io/articles/riscv). (Pic below)
    
    So the migration might not be so challenging after all!

1.  _Why not FreeRTOS? Or Zephyr OS?_

    Our objective is to teach the internals of PinePhone with a very simple Operating System. NuttX is super tiny, so it works just fine!

    __FreeRTOS__ is too bare-bones though. We'd need to build a bunch of drivers from scratch: Display, Touch Input, USB, LVGL, Accelerometer, ... NuttX has many drivers that we need.

    __Zephyr OS__ has plenty of code contributed by large companies, it's great for writing commercial, industrial-grade firmware. But it might be too complex for learning about the internals of a smartphone.

1.  _Why Pine64 gadgets? Are they sponsored?_

    I bought my own PinePhone for porting NuttX. And I'll do the same for the [__RISC-V Gadgets__](https://qoto.org/@lupyuen/110607965321072519).

    Pine64 sells affordable phones and tablets for devs and learners. If you know of similar companies, please lemme know! 🙏

1.  _Why not collaborate with the Pine64 Community on Matrix or Discord?_

    I tried... But my sleeping hours got out of whack.

    (I'm in Singapore, time zone is GMT+8 hours)

    [__Pine64 Forum__](https://forum.pine64.org/index.php) is probably the best place to catch me for a discussion.

1.  _Why not spend a bit more time on PinePhone or PinePhone Pro?_

    I'm already in my fifties and I have severe hypertension (when I get stressed)...

    I'm carefully planning my remaining days as IoT Techie and Educator :-)

    [(And possibly __Sourdough Hacker__)](https://lupyuen.github.io/articles/sourdough)

1.  _What will happen to NuttX for PinePhone?_

    I'm still keen to promote NuttX as a teaching tool for learning the internals of PinePhone!

    If you know of any schools that might be interested, please lemme know! 🙏

![Apache NuttX RTOS on 64-bit QEMU RISC-V Emulator](https://lupyuen.github.io/images/riscv-title.png)

[_Apache NuttX RTOS on 64-bit QEMU RISC-V Emulator_](https://lupyuen.github.io/articles/riscv)

# What's Next

NuttX on PinePhone has been an incredibly rewarding journey, thanks to the awesome NuttX and Pine64 Communities!

Please join me in the next article, as we begin our exploration of __Apache NuttX RTOS on 64-bit RISC-V__...

-   [__"Inspecting the RISC-V Linux Images for Star64 SBC"__](https://lupyuen.github.io/articles/star64)

-   [__"64-bit RISC-V with Apache NuttX Real-Time Operating System"__](https://lupyuen.github.io/articles/riscv)

-   [__"Apache NuttX RTOS for Pine64 Star64 64-bit RISC-V SBC (StarFive JH7110)"__](https://github.com/lupyuen/nuttx-star64)

Many Thanks to my [__GitHub Sponsors__](https://lupyuen.github.io/articles/sponsor) for supporting my work! This article wouldn't have been possible without your support.

-   [__Sponsor me a coffee__](https://lupyuen.github.io/articles/sponsor)

-   [__Discuss this article on Hacker News__](https://news.ycombinator.com/item?id=36399145)

-   [__Discuss this article on Pine64 Forum__](https://forum.pine64.org/showthread.php?tid=18390)

-   [__My Current Project: "Apache NuttX RTOS for Ox64 BL808"__](https://github.com/lupyuen/nuttx-ox64)

-   [__My Other Project: "NuttX for Star64 JH7110"__](https://github.com/lupyuen/nuttx-star64)

-   [__Older Project: "NuttX for PinePhone"__](https://github.com/lupyuen/pinephone-nuttx)

-   [__Check out my articles__](https://lupyuen.github.io)

-   [__RSS Feed__](https://lupyuen.github.io/rss.xml)

_Got a question, comment or suggestion? Create an Issue or submit a Pull Request here..._

[__lupyuen.github.io/src/pinephone2.md__](https://github.com/lupyuen/lupyuen.github.io/blob/master/src/pinephone2.md)

# Notes

1.  To protect PinePhone's __MicroSD Connector__ from wearing out after too much MicroSD Swapping, we could use the [__PinePhone MicroSD Extender__](https://pine64.com/product/pinephone-microsd-extender/).
