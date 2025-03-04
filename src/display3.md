# RISC-V Star64 JH7110: Power Up the Display Controller with U-Boot Bootloader

📝 _2 Sep 2023_

![Star64 JH7110 Display Controller is alive!](https://lupyuen.github.io/images/display3-title.png)

In olden days we'd [__`peek`__](https://en.wikipedia.org/wiki/PEEK_and_POKE) and [__`poke`__](https://en.wikipedia.org/wiki/PEEK_and_POKE) the [__Display Controller__](https://en.wikipedia.org/wiki/Apple_II_graphics), to see weird and wonderful displays.

Today (46 years later), we poke around the Display Controller of [__Star64 JH7110 RISC-V SBC__](https://wiki.pine64.org/wiki/STAR64) with a modern tool (not BASIC): [__U-Boot Bootloader__](https://lupyuen.github.io/articles/linux#u-boot-bootloader-for-star64)!

(Spoiler: No weird and wonderful displays for today!)

In this article we discover...

- U-Boot Commands __`md`__ and __`mw`__  for Dumping and Writing Memory (Pic above)

- That we use to power up the __Video Output__ and __Display Controller__ on the [__StarFive JH7110 SoC__](https://doc-en.rvspace.org/Doc_Center/jh7110.html)

- By modding the JH7110 Registers for __Power Management Unit__, __Clock and Reset__

- Also how we'll create our own __Display Driver__ for JH7110

- In spite of the __Missing and Incorrect Docs__

_Why are we doing this?_

We're building a __HDMI Display Driver__ for [__Apache NuttX Real-Time Operating System__](https://lupyuen.github.io/articles/release) (RTOS) on the Star64 SBC. (And probably for VisionFive2 too)

The analysis in this article will be super useful for creating our __HDMI Driver for NuttX__ on Star64. (Pic below)

And hopefully this article will be helpful for __porting other Operating Systems__ to JH7110! [__FreeBSD__](https://adventurist.me/posts/00315), [__Haiku__](https://discuss.haiku-os.org/t/progress-on-running-haiku-on-visionfive-2/13369), [__Multiplix__](https://github.com/zyedidia/multiplix), ...

![Pine64 Star64 RISC-V SBC](https://lupyuen.github.io/images/nuttx2-title.jpg)

# Dump and Write Memory with U-Boot Bootloader

_Our Bootloader will Read AND Write any Memory?_

Yep! Inside our Star64 SBC is the powerful (maybe risky) [__U-Boot Bootloader__](https://lupyuen.github.io/articles/linux#u-boot-bootloader-for-star64) that will read and write JH7110 SoC Memory: __RAM, ROM, even I/O Registers__!

(ROM is read-only of course)

Boot Star64 __without a microSD Card__ and shut down our [__TFTP Server__](https://lupyuen.github.io/articles/tftp). We should see the [__U-Boot Prompt__](https://lupyuen.github.io/articles/linux#u-boot-bootloader-for-star64).

The __`md`__ command will dump JH7110 Memory (RAM, ROM, I/O Registers)...

```text
# md
md - memory display
Usage: md [.b, .w, .l, .q] address [# of objects]
```

This is how we dump the __JH7110 Boot ROM__ at [__`0x2A00` `0000`__](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/system_memory_map.html)...

```text
# md 2A000000 0x20
2a000000: 00000297 12628293 30529073 30005073  ......b.s.R0sP.0
2a000010: 30405073 41014081 42014181 43014281  sP@0.@.A.A.B.B.C
2a000020: 44014381 45014481 46014581 47014681  .C.D.D.E.E.F.F.G
2a000030: 48014781 49014881 4a014981 4b014a81  .G.H.H.I.I.J.J.K
2a000040: 4c014b81 4d014c81 4e014d81 4f014e81  .K.L.L.M.M.N.N.O
2a000050: 01974f81 8193d710 02970ae1 82930000  .O..............
```

(Cute Alphabet Soup. Where's the Source, Luke?)

_This works for I/O Registers?_

Indeed! We can dump the __JH7110 UART Registers__ at [__`0x1000` `0000`__](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/system_memory_map.html)...

```text
# md 10000000 0x20
10000000: 0000006d 00000000 000000c1 00000003  m...............
10000010: 00000003 00000000 00000000 00000000  ................
10000020: 00000000 00000000 00000000 00000000  ................
10000030: 00000064 00000064 00000064 00000064  d...d...d...d...
```

_What about writing to I/O Registers?_

Yep we can write to a __UART Register__. The __`mw`__ command writes to JH7110 Memory...

```text
# mw
mw - memory write (fill)
Usage: mw [.b, .w, .l, .q] address value [count]
```

(Hmmm sounds like a scary Security Risk)

To print something on UART, we poke `0x2A` into the __UART Transmit Register__ at [__`0x1000` `0000`__](https://lupyuen.github.io/articles/nuttx2#uart-controller-on-star64)...

```text
# mw 10000000 2a 1
*
```

It prints "__`*`__", equivalent to ASCII Code __`0x2A`__!

Moving to something more sophisticated: JH7110 Display Controller...

![JH7110 System Memory Map](https://lupyuen.github.io/images/display3-memory.png)

[_JH7110 System Memory Map_](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/system_memory_map.html)

# Dump the JH7110 Display Controller

_Dumping the JH7110 Display Controller should work right?_

We'll find out! Based on...

- [__JH7110 Display Subsystem Memory Map__](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/memory_map_display.html)

- [__JH7110 System Memory Map__](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/system_memory_map.html)

The registers for __JH7110 Display Subsystem__ (pic above) are at...

| Address | Display Registers |
|:-------:|:------------------|
| __`0x2940 0000`__ | __DC8200 AHB0__ <br>_(Display Bus 0)_
| __`0x2948 0000`__ | __DC8200 AHB1__ <br>_(Display Bus 1)_
| __`0x2959 0000`__ | __U0_HDMITX__ <br>_(HDMI)_
| __`0x295B 0000`__ | __VOUT_SYSCON__ <br>_(System Config)_
| __`0x295C 0000`__ | __VOUT_CRG__ <br>_(Clock and Reset)_
| __`0x295D 0000`__ | __DSI TX__ <br>_(MIPI Display Serial Interface)_
| __`0x295E 0000`__ | __MIPITX DPHY__ <br>_(MIPI Display Physical Layer)_

[(__DC8200__ is the __VeriSilicon Dual Display Controller__)](https://lupyuen.github.io/articles/display2#dc8200-display-controller)

We dump the above __Display Subsystem Registers__...

```text
# md 29400000 0x20
29400000: 00000000 00000000 00000000 00000000  ................
29400010: 00000000 00000000 00000000 00000000  ................

# md 29480000 0x20
29480000: 00000000 00000000 00000000 00000000  ................
29480010: 00000000 00000000 00000000 00000000  ................

# md 29590000 0x20
29590000: 00000000 00000000 00000000 00000000  ................
29590010: 00000000 00000000 00000000 00000000  ................

# md 295B0000 0x20
295b0000: 00000000 00000000 00000000 00000000  ................
295b0010: 00000000 00000000 00000000 00000000  ................

# md 295C0000 0x20
295c0000: 00000000 00000000 00000000 00000000  ................
295c0010: 00000000 00000000 00000000 00000000  ................
```

_But the values are all zero!_

That's because the Display Subsystem is __not powered up__.

To fix this, we tweak some registers and power up the Display Subsystem...

![JH7110 Display Subsystem Block Diagram](https://lupyuen.github.io/images/display2-vout_block_diagram18.png)

[_JH7110 Display Subsystem Block Diagram_](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/block_diagram_display.html)

# JH7110 Power Management Unit

_How to power up the Display Subsystem?_

From the pic above, the Display Subsystem is powered in the __Video Output Power Domain (DOM_VOUT)__.

Which is powered by the JH7110 [__Power Management Unit (PMU)__](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/overview_pm.html)...

![Power Management Unit](https://lupyuen.github.io/images/display3-power.png)

[(Source)](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/overview_pm.html)

[(Not to be confused with __External PMIC__: X-Powers AXP15060)](https://lupyuen.github.io/articles/display3#appendix-jh7110-display-driver)

_Are the lights on for Video Output DOM_VOUT?_

To find out, we dump the status of the Power Domains...

- From [__System Memory Map__](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/system_memory_map.html): Base Address of PMU is __`0x1703` `0000`__

- From [__PMU Registers__](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/register_info_pmu.html#register_info_pmu__section_rcx_pqz_msb): Current Power Mode is at Offset __`0x80`__

Hence the __Current Power Mode__ is at __`0x1703` `0080`__. When we dump the register...

```text
# md 17030080 1
17030080: 00000003
```

[__Current Power Mode__](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/register_info_pmu.html#register_info_pmu__section_rcx_pqz_msb) is 3, saying that...

- __System Power__ is On (Bit 0)
- __CPU Power__ is On (Bit 1)
- But __Video Output Power__ is Off! (Bit 4)

_How to power up VOUT?_

We read the [__PMU Function Description__](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/function_descript_pmu.html)...

> __SW Encourage Turn-on Sequence__

> (1) Configure the register __SW Turn-On Power Mode__ (Offset __`0x0C`__), write the bit 1 which Power Domain will be turn-on, write the others 0;

> (2) Write the __SW Turn-On Command Sequence__. Write the register Software Encourage (Offset __`0x44`__) __`0xFF`__ → __`0x05`__ → __`0x50`__

_What's a "Software Encourage"?_

Something got Lost in Translation. We assume it means [__"Software Triggered"__](https://lupyuen.github.io/articles/display3#pmu-software-encourage).

Thus we set the [__Power Mode__](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/register_info_pmu.html#register_info_pmu__section_nhb_slz_msb) (Offset __`0x0C`__) to __`0x10`__ (Bit 4 for VOUT)...

```text
# mw 1703000c 0x10 1
```

Then we write the [__Command Sequence__](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/register_info_pmu.html#register_info_pmu__section_jdh_x4z_msb) at Offset __`0x44`__ (no delays needed)...

```text
# mw 17030044 0xff 1
# mw 17030044 0x05 1
# mw 17030044 0x50 1
```

Finally we dump the [__Current Power Mode__](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/register_info_pmu.html#register_info_pmu__section_rcx_pqz_msb)...

```text
# md 17030080 1
17030080: 00000013
```

Lights on folks, __Video Output Power VOUT__ (Bit 4) is On!

(Actually we should wait [__50 milliseconds__](https://lupyuen.github.io/articles/display3#appendix-jh7110-display-driver) to Power Up)

_We're ready to dump the Display Subsystem?_

Sadly nope, the __Display Subsystem Registers__ are still empty...

```text
# md 295C0000 0x20
295c0000: 00000000 00000000 00000000 00000000  ................
295c0010: 00000000 00000000 00000000 00000000  ................
```

We need more mods, for the Clock and Reset Signals...

![JH7110 Clock Structure](https://lupyuen.github.io/images/display3-clock.png)

[_JH7110 Clock Structure_](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/clock_structure.html)

# Clocks and Resets for Display Subsystem

_Display Subsystem (VOUT) is already powered up via the Power Management Unit (PMU)..._

_Anything else we need to make it work?_

Always remember to enable the __VOUT Clocks__ and deassert the __VOUT Resets__!

According to the [__JH7110 Clock Structure__](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/clock_structure.html) (pic above), the __Display Subsystem (VOUT)__ is Clocked by...

- __clk_vout_root__
- __mipiphy ref clk__
- __hdmiphy ref clk__
- __clk_vout_src__ (1,228.8 MHz)
- __clk_vout_axi__ (614.4 MHz)
- __clk_vout_ahb__ (204.8 MHz) / __clk_ahb1__
- __clk_mclk__ (51.2 MHz)

Plus one Reset...

- __rstn_vout__

_How to enable the Clocks and deassert the Resets?_

We set the [__System Control Registers (SYS CRG)__](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/sys_crg.html) at Base Address [__`0x1302` `0000`__](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/system_memory_map.html)

When we match the above Clocks to the [__System Control Registers (SYS CRG)__](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/sys_crg.html), we get...

| SYS CRG<br>Offset | Clock |
|:--------------:|:------|
| [__`0x28`__](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/sys_crg.html#sys_crg__section_cvr_2qm_wsb) | Clock AHB 1
| [__`0x4C`__](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/sys_crg.html#sys_crg__section_wmd_xrm_wsb) | MCLK Out  
| [__`0x98`__](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/sys_crg.html#sys_crg__section_jbf_mtm_wsb) | clk_u0_sft7110_noc_bus _clk_cpu_axi  
| [__`0x9C`__](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/sys_crg.html#sys_crg__section_ctf_mtm_wsb) | clk_u0_sft7110_noc_bus _clk_axicfg0_axi  
| [__`0xE8`__](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/sys_crg.html#sys_crg__section_pd3_dvm_wsb) | clk_u0_dom_vout_top _clk_dom_vout_top_clk_vout_src  
| [__`0xF0`__](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/sys_crg.html#sys_crg__section_vkj_dvm_wsb) | Clock NOC Display AXI  
| [__`0xF4`__](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/sys_crg.html#sys_crg__section_htj_dvm_wsb) | Clock Video Output AHB  
| [__`0xF8`__](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/sys_crg.html#sys_crg__section_zdk_dvm_wsb) | Clock Video Output AXI  
| [__`0xFC`__](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/sys_crg.html#sys_crg__section_hlk_dvm_wsb) | Clock Video Output HDMI TX0 MCLK  

(Looks excessive, but better to enable too many Clocks than too few!)

[(NoC means __Network-on-Chip__)](https://en.wikipedia.org/wiki/Network_on_a_chip)

[(AXI is the __Advanced eXtensible Interface__)](https://en.wikipedia.org/wiki/Advanced_eXtensible_Interface)

To enable the Clocks above, we set [__Bit 31 (clk_icg)__](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/sys_crg.html#sys_crg__section_cvr_2qm_wsb) to 1.

Here are the U-Boot Commands to __enable the VOUT Clocks__...

```text
## Enable the Clocks for Video Output (Display Subsystem)
mw 13020028 0x80000000 1
mw 1302004c 0x80000000 1
mw 13020098 0x80000000 1
mw 1302009c 0x80000000 1
mw 130200e8 0x80000000 1
mw 130200f0 0x80000000 1
mw 130200f4 0x80000000 1
mw 130200f8 0x80000000 1
mw 130200fc 0x80000000 1
```

_What about the Resets?_

Digging through the [__System Control Registers (SYS CRG)__](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/sys_crg.html), we need to __deassert these Resets__...

- [__Software RESET 1 Address Selector__](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/sys_crg.html#sys_crg__section_ph5_pfn_wsb) (SYS CRG Offset __`0x2FC`__)

  __Bit 11:__ rstn_u0_dom_vout_top _rstn_dom_vout_top_rstn_vout_src

- [__SYSCRG RESET Status 0__](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/sys_crg.html#sys_crg__section_psk_ks3_2tb) (SYS CRG Offset __`0x308`__)

  __Bit 26:__ rstn_u0_sft7110_noc_bus _reset_disp_axi_n

- [__SYSCRG RESET Status 1__](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/sys_crg.html#sys_crg__section_ffb_xt3_2tb) (SYS CRG Offset __`0x30C`__)

  __Bit 11:__ rstn_u0_dom_vout_top _rstn_dom_vout_top_rstn_vout_src

We set the above bits to 0 to deassert the Resets.

First we dump the above __VOUT Reset Registers__...

```text
# md 130202fc 1
130202fc: 07e7fe00

# md 13020308 1
13020308: ff9fffff

# md 1302030c 1
1302030c: f80001ff
```

Then we flip the __VOUT Reset Bits__ to 0...

```text
# mw 130202fc 0x07e7f600 1
# mw 13020308 0xfb9fffff 1
```

(The last Reset is already deasserted, no need to flip it)

_What happens when we enable the Clocks and deassert the Resets?_

We run the above commands to set the Clocks and Resets.

And again we dump the __Display Subsystem (VOUT) Registers__...

```text
# md 295C0000 0x20
295c0000: 00000004 00000004 00000004 0000000c  ................
295c0010: 00000000 00000000 00000000 00000000  ................
295c0020: 00000000 00000000 00000000 00000000  ................
295c0030: 00000000 00000000 00000000 00000000  ................
295c0040: 00000000 00000000 00000fff 00000000  ................
```

The Display Subsystem Registers are finally visible at VOUT CRG __`0x295C` `0000`__

Which means the Display Subsystem is alive yay!

[(__`0x295C` `000C`__ defaults to __`0x0C`__, there's a typo in the doc)](https://lupyuen.github.io/articles/display3#clock-default)

![Star64 JH7110 Display Subsystem is alive!](https://lupyuen.github.io/images/display3-title.png)

# U-Boot Script to Power Up the Display Subsystem

_Phew that's a long list of U-Boot Commands. Can we automate this?_

```text
## Power Up the PMU for Video Output (Display Subsystem)
mw 1703000c 0x10 1
mw 17030044 0xff 1
mw 17030044 0x05 1
mw 17030044 0x50 1

## Enable the Clocks for Video Output (Display Subsystem)
mw 13020028 0x80000000 1
mw 1302004c 0x80000000 1
mw 13020098 0x80000000 1
mw 1302009c 0x80000000 1
mw 130200e8 0x80000000 1
mw 130200f0 0x80000000 1
mw 130200f4 0x80000000 1
mw 130200f8 0x80000000 1
mw 130200fc 0x80000000 1

## Deassert the Resets for Video Output (Display Subsystem)
mw 130202fc 0x07e7f600 1
mw 13020308 0xfb9fffff 1
md 295C0000 0x20
```

Sure can!

Run this to create a __U-Boot Script__ that powers up the Display Subsystem...

```text
## Create the U-Boot Script to power up the Video Output
setenv video_on 'mw 1703000c 0x10 1 ; mw 17030044 0xff 1 ; mw 17030044 0x05 1 ; mw 17030044 0x50 1 ; mw 13020028 0x80000000 1 ; mw 1302004c 0x80000000 1 ; mw 13020098 0x80000000 1 ; mw 1302009c 0x80000000 1 ; mw 130200e8 0x80000000 1 ; mw 130200f0 0x80000000 1 ; mw 130200f4 0x80000000 1 ; mw 130200f8 0x80000000 1 ; mw 130200fc 0x80000000 1 ; mw 130202fc 0x07e7f600 1 ; mw 13020308 0xfb9fffff 1 ; md 295C0000 0x20 ; '

## Check that it's correct
printenv video_on

## Save it for future reboots
saveenv

## Run the U-Boot Script to power up the Video Output
run video_on
```

The U-Boot Script __`video_on`__ is saved into our SBC's __Internal Flash Memory__. 

Which means we can switch on our SBC and run the script anytime...

```text
# run video_on
295c0000: 00000004 00000004 00000004 0000000c  ................
295c0010: 00000000 00000000 00000000 00000000  ................
295c0020: 00000000 00000000 00000000 00000000  ................
295c0030: 00000000 00000000 00000000 00000000  ................
295c0040: 00000000 00000000 00000fff 00000000  ................
```

So much easier to power up the Display Subsystem!

Next we do the Display Controller...

![JH7110 Display Subsystem Clock and Reset](https://lupyuen.github.io/images/display2-vout_clkrst18.png)

[_JH7110 Display Subsystem Clock and Reset_](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/clock_n_reset_display.html)

# Clocks and Resets for Display Controller

_JH7110 Display Subsystem is already powered up..._

_What about the Display Controller?_

Nope our __DC8200 Display Controller__ (DC8200 AHB0) is stil invisible...

```text
# md 29400000 0x20
29400000: 00000000 00000000 00000000 00000000  ................
29400010: 00000000 00000000 00000000 00000000  ................
```

_What about the Clocks and Resets for the Display Controller?_

The pic above shows the [__Display Controller Clocks and Resets__](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/clock_n_reset_display.html).

According to the [__VOUT CRG__](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/dom_vout_crg.html) (Video Output Clock and Reset Registers), these are the __DC8200 Clock Registers__ for HDMI...

| VOUT CRG<br>Offset | Clock |
|:------------------:|:------|
| [__`0x10`__](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/dom_vout_crg.html#dom_vout_crg__section_jtl_fht_3sb) | clk_u0_dc8200_clk_axi   
| [__`0x14`__](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/dom_vout_crg.html#dom_vout_crg__section_w1m_x1p_jsb) | clk_u0_dc8200_clk_core  
| [__`0x18`__](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/dom_vout_crg.html#dom_vout_crg__section_lyy_z1p_jsb) | clk_u0_dc8200_clk_ahb   
| [__`0x1C`__](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/dom_vout_crg.html#dom_vout_crg__section_xtz_dbp_jsb) | clk_u0_dc8200_clk_pix0  
| [__`0x20`__](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/dom_vout_crg.html#dom_vout_crg__section_c2y_jbp_jsb) | clk_u0_dc8200_clk_pix1  
| [__`0x3C`__](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/dom_vout_crg.html#dom_vout_crg__section_hpl_ncp_jsb) | clk_u0_hdmi_tx_clk_mclk 
| [__`0x40`__](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/dom_vout_crg.html#dom_vout_crg__section_edw_qcp_jsb) | clk_u0_hdmi_tx_clk_bclk 
| [__`0x44`__](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/dom_vout_crg.html#dom_vout_crg__section_olm_5cp_jsb) | clk_u0_hdmi_tx_clk_sys  

To enable the Clocks above, we set [__Bit 31 (clk_icg)__](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/dom_vout_crg.html#dom_vout_crg__section_jtl_fht_3sb) to 1.

[(__VOUT CRG__ Base Address is __`0x295C` `0000`__)](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/memory_map_display.html)

__DC8200 Resets__ for HDMI are at _Software_RESET_assert0_addr_assert_sel_ [(VOUT CRG Offset __`0x48`__)](https://lupyuen.github.io/articles/display3#vout-reset)...

- __Bit 0:__ rstn_u0_dc8200_rstn_axi

- __Bit 1:__ rstn_u0_dc8200_rstn_ahb

- __Bit 2:__ rstn_u0_dc8200_rstn_core

- __Bit 9:__ rstn_u0_hdmi_tx_rstn_hdmi

We set the above bits to 0 to deassert the Resets.

[(__DC8200 Reset Register__ is mislocated in the docs)](https://lupyuen.github.io/articles/display3#vout-reset)

These are the U-Boot Commands to set the __DC8200 Clocks and Resets__ for HDMI...

```text
## Power up the Video Output (Display Subsystem)
run video_on

## Enable the DC8200 HDMI Clocks
mw 295C0010 0x80000000 1
mw 295C0014 0x80000000 1
mw 295C0018 0x80000000 1
mw 295C001c 0x80000000 1
mw 295C0020 0x80000000 1
mw 295C003c 0x80000000 1
mw 295C0040 0x80000000 1
mw 295C0044 0x80000000 1

## Deassert the DC8200 HDMI Resets.
## We deassert all Resets for today.
mw 295C0048 0 1
```

_Does it work?_

We run the above U-Boot Commands for VOUT HDMI. Then we dump our __DC8200 Display Controller__ (DC8200 AHB0)...

```text
# md 29400000 0x20
29400000: 00000900 80010000 00222200 00000000  ........."".....
29400010: 00000000 00000004 14010000 000b4b41  ............AK..
29400020: 00008200 00005720 20210316 16015600  .... W....! .V..
29400030: 0000030e a0600084 00000000 00000000  ......`.........
```

Yep our DC8200 Display Controller is finally alive yay!

Based on the above commands, we write the __U-Boot Script__ to start the Display Controller (pic below)...

```text
## Create the U-Boot Script to power up the Display Controller
setenv display_on 'mw 295C0010 0x80000000 1 ; mw 295C0014 0x80000000 1 ; mw 295C0018 0x80000000 1 ; mw 295C001c 0x80000000 1 ; mw 295C0020 0x80000000 1 ; mw 295C003c 0x80000000 1 ; mw 295C0040 0x80000000 1 ; mw 295C0044 0x80000000 1 ; mw 295C0048 0 1 ; md 29400000 0x20 ; '

## Check that it's correct
printenv display_on

## Save it for future reboots
saveenv

## Run the U-Boot Script to power up the Video Output
run video_on

## Run the U-Boot Script to power up the Display Controller
run display_on
```

[(See the __Output Log__)](https://github.com/lupyuen/nuttx-star64#read-the-star64-jh7110-display-controller-registers-with-u-boot-bootloader)

One last thing: We verify the DC8200 Register Values...

![DC8200 Display Controller is finally alive yay!](https://lupyuen.github.io/images/display3-run.png)

# Read the Hardware Revision and Chip ID

_Are the DC8200 Register Values correct?_

According to the [__DC8200 Display Driver__](https://lupyuen.github.io/articles/display2#dc8200-display-hardware-driver), we can sniff these Register Values from the DC8200 Display Controller...

- __Hardware Revision__: DC8200 AHB0 Offset __`0x24`__

- __Chip ID__: DC8200 AHB0 Offset __`0x30`__

  [(__DC8200 AHB0__ Base Address is __`0x2940` `0000`__)](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/memory_map_display.html)

  [(Source)](https://github.com/starfive-tech/linux/blob/JH7110_VisionFive2_devel/drivers/gpu/drm/verisilicon/vs_dc_hw.c#L1301-L1361) 

This is how we dump the __Hardware Revision__ and __Chip ID__ in U-Boot...

```text
## Power up the Video Output
run video_on

## Power up the Display Controller
run display_on

## TODO: Wait 500 milliseconds to Power Up

## Dump the Hardware Revision
md 29400024 1

## Dump the Chip ID
md 29400030 1
```

We should see...

```text
# md 29400024 1
29400024: 00005720

# md 29400030 1
29400030: 0000030e
```

That means...

- __Hardware Revision__ is __`0x5720`__ (DC_REV_0)
	
- __Chip ID__ is __`0x30E`__

  [(Source)](https://github.com/starfive-tech/linux/blob/JH7110_VisionFive2_devel/drivers/gpu/drm/verisilicon/vs_dc_hw.c#L1301-L1361)

Our DC8200 Display Controller returns the correct Register Values yay!

(Do we have a Date at __`0x2940` `0028`__? It reads "20210316")

# NuttX Display Driver for JH7110

_How will we create the NuttX Display Driver for JH7110?_

Earlier we talked about the steps to power up the __Display Subsystem__ and __Display Controller__...

- [__"Power Up the Display Subsystem"__](https://lupyuen.github.io/articles/display3#u-boot-script-to-power-up-the-display-subsystem)

- [__"Clocks and Resets for Display Controller"__](https://lupyuen.github.io/articles/display3#clocks-and-resets-for-display-controller)

This is how we implement the steps in our __NuttX Display Driver__: [jh7110_appinit.c](https://github.com/lupyuen2/wip-nuttx/blob/hdmi/boards/risc-v/jh7110/star64/src/jh7110_appinit.c#L136-L290)

```c
// Power Up the Power Management Unit
// for Video Output / Display Subsystem
// TODO: Switch to constants
putreg32(0x10, 0x1703000c);
putreg32(0xff, 0x17030044);
putreg32(0x05, 0x17030044);
putreg32(0x50, 0x17030044);
```

This code will power up the Display Subsystem via the __Power Management Unit__.

(Yeah needs cleanup)

Next we enable the VOUT Clocks and deassert the VOUT Resets for the __Display Subsystem__...

```c
// Enable the Clocks for Video Output / Display Subsystem
modifyreg32(0x13020028, 0, 1 << 31);  // Addr, Clear Bits, Set Bits
modifyreg32(0x1302004c, 0, 1 << 31);
modifyreg32(0x13020098, 0, 1 << 31);
modifyreg32(0x1302009c, 0, 1 << 31);
modifyreg32(0x130200e8, 0, 1 << 31);
modifyreg32(0x130200f0, 0, 1 << 31);
modifyreg32(0x130200f4, 0, 1 << 31);
modifyreg32(0x130200f8, 0, 1 << 31);
modifyreg32(0x130200fc, 0, 1 << 31);

// Deassert the Resets for Video Output / Display Subsystem
modifyreg32(0x130202fc, 1 << 11, 0);  // Addr, Clear Bits, Set Bits
modifyreg32(0x13020308, 1 << 26, 0);

// Verify that Video Output / Display Subsystem is up
val = getreg32(0x295C0000);
DEBUGASSERT(val == 4);
```

Then we do the same for the __Display Controller__...

```c
// Enable the Clocks for DC8200 Display Controller (HDMI)
modifyreg32(0x295C0010, 0, 1 << 31);  // Addr, Clear Bits, Set Bits
modifyreg32(0x295C0014, 0, 1 << 31);
modifyreg32(0x295C0018, 0, 1 << 31);
modifyreg32(0x295C001c, 0, 1 << 31);
modifyreg32(0x295C0020, 0, 1 << 31);
modifyreg32(0x295C003c, 0, 1 << 31);
modifyreg32(0x295C0040, 0, 1 << 31);
modifyreg32(0x295C0044, 0, 1 << 31);

// Deassert the Resets for DC8200 Display Controller (HDMI)
modifyreg32(
  0x295C0048,  // Addr
  (1 << 0) | (1 << 1) | (1 << 2) | (1 << 9),  // Clear Bits
  0  // Set Bits
);
```

Finally we check the __Hardware Revision and Chip ID__...

```c
// TODO: Power up ALDO3 and ALDO5 on the External Power Management IC
// (X-Powers AXP15060 PMIC over I2C)
// (I2C SCL on GPIO 19, SDA on GPIO 20)

// Wait 500 milliseconds to Power Up PMIC
up_mdelay(500);

// Verify that Hardware Revision and Chip ID are non-zero
uint32_t revision = getreg32(0x29400024);
uint32_t chip_id  = getreg32(0x29400030);
_info("revision=0x%x, chip_id=0x%x", revision, chip_id);
DEBUGASSERT(revision != 0 && chip_id != 0);
```

_Does it work with NuttX?_

Yep NuttX will start our Display Driver and print the __Hardware Revision and Chip ID__...

```text
Starting kernel...
board_late_initialize:
  revision=0x5720
  chip_id=0x30e
  hdmi_status=0x30

NuttShell (NSH) NuttX-12.0.3
nsh> 
```

But our [__HDMI Status__](https://lupyuen.github.io/articles/display3#appendix-jh7110-display-driver) says that [__Hot Plug is Disconnected__](https://lupyuen.github.io/articles/display3#appendix-jh7110-display-driver).

[(Probably because our __ALDOs__ are missing)](https://lupyuen.github.io/articles/display3#appendix-jh7110-display-driver)

_What about the rest of the Display Driver?_

The remaining steps are described here...

- [__"JH7110 Display Driver"__](https://lupyuen.github.io/articles/display3#appendix-jh7110-display-driver)

But we have a problem with __Incomplete and Incorrect Docs__, see the next section.

_Who starts our Display Driver?_

At Startup, our NuttX Driver will be called from [__board_late_initialize__](https://github.com/lupyuen2/wip-nuttx/blob/hdmi/boards/risc-v/jh7110/star64/src/jh7110_appinit.c#L136-L290).

[(See the __NuttX Boot Sequence__)](https://lupyuen.github.io/articles/release#nuttx-startup-explained)

![Official Display Driver for JH7110](https://lupyuen.github.io/images/display2-title.jpg)

[_Official Display Driver for JH7110_](https://lupyuen.github.io/articles/display2)

# Unsolved Mysteries

_Are we done with our NuttX Display Driver?_

Not quite! We have a couple of challenges with __Incomplete and Incorrect Docs__...

1.  What are these __Mystery Writes to Undocumented Registers__ in the DC8200 Display Controller?

    [__"HDMI Output"__](https://lupyuen.github.io/articles/display3#hdmi-output)

1.  __Reset Register for Video Output (VOUT)__ seems to be misplaced...

    [__"VOUT Reset"__](https://lupyuen.github.io/articles/display3#vout-reset)

1.  How to set the __VOUT Clock Source__?

    [__"Clock Multiplexing"__](https://lupyuen.github.io/articles/display3#clock-multiplexing)

1.  How to set the __VOUT Clock Rate__?

    [__"Clock Rate"__](https://lupyuen.github.io/articles/display3#clock-rate)

1.  Typo in the __VOUT Clock Default Rate__...

    [__"Clock Default"__](https://lupyuen.github.io/articles/display3#clock-default)

1.  Indeed __Software is Encouraging__, but we'll make our own interpretation...

    [__"PMU Software Encourage"__](https://lupyuen.github.io/articles/display3#pmu-software-encourage)

Hopefully we'll overcome these issues and complete our NuttX Display Driver!

# What's Next

Today we did some cool things with U-Boot Bootloader on JH7110 SoC...

- We ran the U-Boot Commands __`md`__ and __`mw`__  for reading and writing JH7110 Memory

- We powered up the __Video Output__ and __Display Controller__ on the JH7110 SoC

- By modding the JH7110 Registers for __Power Management Unit__, __Clock and Reset__

- We're one step closer to creating our __NuttX Display Driver__ for JH7110

- Hopefully we'll see many other Display Drivers for JH7110: [__FreeBSD__](https://adventurist.me/posts/00315), [__Haiku__](https://discuss.haiku-os.org/t/progress-on-running-haiku-on-visionfive-2/13369), [__Multiplix__](https://github.com/zyedidia/multiplix), ...

Now we need an __I2C Driver for JH7110__, to power up the External Power Management IC. Which we'll cover in the next article!

Many Thanks to my [__GitHub Sponsors__](https://lupyuen.github.io/articles/sponsor) (and the awesome NuttX Community) for supporting my work! This article wouldn't have been possible without your support.

-   [__Sponsor me a coffee__](https://lupyuen.github.io/articles/sponsor)

-   [__Discuss this article on Hacker News__](https://news.ycombinator.com/item?id=37362736)

-   [__Discuss this article on RVspace Forum__](https://forum.rvspace.org/t/article-power-up-the-jh7110-display-controller-with-u-boot-bootloader/3478)

-   [__Discuss this article on Pine64 Forum__](https://forum.pine64.org/showthread.php?tid=18680)

-   [__My Current Project: "Apache NuttX RTOS for Ox64 BL808"__](https://github.com/lupyuen/nuttx-ox64)

-   [__My Other Project: "NuttX for Star64 JH7110"__](https://github.com/lupyuen/nuttx-star64)

-   [__Older Project: "NuttX for PinePhone"__](https://github.com/lupyuen/pinephone-nuttx)

-   [__Check out my articles__](https://lupyuen.github.io)

-   [__RSS Feed__](https://lupyuen.github.io/rss.xml)

_Got a question, comment or suggestion? Create an Issue or submit a Pull Request here..._

[__lupyuen.github.io/src/display3.md__](https://github.com/lupyuen/lupyuen.github.io/blob/master/src/display3.md)

# Appendix: JH7110 Display Driver

Based on everything we deciphered in this article and the previous one...

- [__"RISC-V Star64 JH7110: Inside the Display Controller"__](https://lupyuen.github.io/articles/display2)

The __JH7110 Display Driver (HDMI)__ that we create for NuttX (and other Operating Systems) shall do the following...

1.  Power Up the [__Power Management Unit__](https://lupyuen.github.io/articles/display3#u-boot-script-to-power-up-the-display-subsystem) for Video Output (Display Subsystem)

1.  Wait [__50 milliseconds__](https://github.com/starfive-tech/u-boot/blob/JH7110_VisionFive2_devel/drivers/video/starfive/sf_vop.c#L656-L667) to Power Up

1.  [__Enable the Clocks__](https://lupyuen.github.io/articles/display3#u-boot-script-to-power-up-the-display-subsystem) for Video Output (Display Subsystem)

1.  [__Deassert the Resets__](https://lupyuen.github.io/articles/display3#u-boot-script-to-power-up-the-display-subsystem) for Video Output (Display Subsystem)

1.  Verify that [__Video Output (Display Subsystem) is up__](https://github.com/lupyuen2/wip-nuttx/blob/hdmi/boards/risc-v/jh7110/star64/src/jh7110_appinit.c#L203-L207)

1.  [__Enable the Clocks__](https://lupyuen.github.io/articles/display3#clocks-and-resets-for-display-controller) for DC8200 Display Controller (HDMI)

1.  [__Deassert the Resets__](https://lupyuen.github.io/articles/display3#clocks-and-resets-for-display-controller) for DC8200 Display Controller (HDMI)

1.  Power up [__ALDO3 (1.8V) and ALDO5 (0.9V)__](https://github.com/lupyuen/nuttx-star64/blob/main/jh7110-visionfive-v2.dts#L1487-L1502) on the External Power Management IC

    [(__X-Powers AXP15060__ PMIC over I2C)](https://files.pine64.org/doc/datasheet/star64/AXP15060%20datasheet%20V0.1.pdf)

    [(I2C SCL on __GPIO 19__, SDA on __GPIO 20__, Page 17)](https://files.pine64.org/doc/star64/Star64_Schematic_V1.1_20230504.pdf)

1.  Wait [__500 milliseconds__](https://github.com/starfive-tech/linux/blob/JH7110_VisionFive2_devel/drivers/gpu/drm/verisilicon/inno_hdmi.c#L658-L676) to Power Up PMIC

1.  Verify that [__Hardware Revision and Chip ID__](https://lupyuen.github.io/articles/display3#read-the-hardware-revision-and-chip-id) are non-zero

1.  Read the [__HDMI Status__](https://github.com/starfive-tech/u-boot/blob/JH7110_VisionFive2_devel/drivers/video/starfive/sf_hdmi.c#L527-L531), check for Hot Plug

    [(HDMI Base Address is __`0x2959` `0000`__)](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/memory_map_display.html)

1.  [__Enable HDMI__](https://github.com/starfive-tech/u-boot/blob/JH7110_VisionFive2_devel/drivers/video/starfive/sf_hdmi.c#L444-L457)

    - Detect HDMI
    - Disable HDMI Transmit PHY
    - Configure HDMI Transmit PHY for 1080p 60 Hz
    - Enable HDMI Transmit PHY
    - Enable HDMI TMDS
    - Enable HDMI Data Sync

      [(See __inno_hdmi_enable__)](https://github.com/starfive-tech/u-boot/blob/JH7110_VisionFive2_devel/drivers/video/starfive/sf_hdmi.c#L444-L457)

      [(HDMI Base Address is __`0x2959` `0000`__)](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/memory_map_display.html)

1.  Set the Clock Source of __u0_dc8200.clk_pix0__ to __clk_dc8200_pix0__

    [(See this)](https://lupyuen.github.io/articles/display3#clock-multiplexing)

1.  Set the Clock Rate of __dc8200_pix0__ to __148.5 MHz__ (HDMI Clock)

    [(See this)](https://lupyuen.github.io/articles/display3#clock-rate)

1.  Bunch of __Mystery Writes__ to DC8200 AHB0

    [(See this)](https://lupyuen.github.io/articles/display3#hdmi-output)

1.  __TODO:__ How to write to Framebuffer?

    Shall we check the [__Official Linux Driver__](https://lupyuen.github.io/articles/display2)? (Pic below)

    Why is the [__HDMI Display Unstable__](https://github.com/starfive-tech/u-boot/commit/26a31fa6bc9debeb71df0a0f9a715ac8e46cff1c)?

Our JH7110 Display Driver is partially implemented here: [jh7110_appinit.c](https://github.com/lupyuen2/wip-nuttx/blob/hdmi/boards/risc-v/jh7110/star64/src/jh7110_appinit.c#L136-L270)

__TODO__: Implement the [__JH7110 I2C Driver__](https://doc-en.rvspace.org/VisionFive2/DG_I2C/JH7110_SDK/source_code_structure_i2c.html) (based on [__DesignWare I2C__](https://github.com/torvalds/linux/blob/master/drivers/i2c/busses/i2c-designware-core.h)) so we can control the PMIC. This NuttX I2C Driver might work: [__cxd56_i2c.c__](https://github.com/apache/nuttx/blob/master/arch/arm/src/cxd56xx/cxd56_i2c.c)

- [__"Power Up the I2C Controller for Star64 JH7110"__](https://github.com/lupyuen/nuttx-star64#power-up-the-i2c-controller-for-star64-jh7110)

- [__"Explore the I2C Controller for Star64 JH7110"__](https://github.com/lupyuen/nuttx-star64#explore-the-i2c-controller-for-star64-jh7110)

- [Search for __"DesignWare DW_apb_i2c Databook"__](https://www.google.com/search?q=%22DesignWare+DW_apb_i2c+Databook%22)

(Many Thanks to [__Justin / Fishwaldo__](https://fosstodon.org/@Fishwaldo/110902984442385966) for recommending the HDMI Driver from U-Boot Bootloader)

![Official Display Driver for JH7110](https://lupyuen.github.io/images/display2-title.jpg)

[_Official Display Driver for JH7110_](https://lupyuen.github.io/articles/display2)

# Appendix: JH7110 Display Controller Mysteries

In this section we talk about the __Missing, Mistaken and Mysterious__ things in the JH7110 Display Controller.

## HDMI Output

We spoke about our JH7110 Display Driver and a Bunch of Mystery Writes...

- [__"JH7110 Display Driver"__](https://lupyuen.github.io/articles/display3#appendix-jh7110-display-driver)

Inside the __U-Boot Display Driver__ (officially contributed by StarFive), there's a long list of __Mystery Writes to Undocumented Registers__ in the DC8200 Display Controller: [sf_vop.c](https://github.com/starfive-tech/u-boot/blob/JH7110_VisionFive2_devel/drivers/video/starfive/sf_vop.c#L493-L540)

```c
writel(0xc0001fff, priv->regs_hi+0x00000014);
writel(0x00002000, priv->regs_hi+0x00001cc0);
//writel(uc_plat->base+0x1fa400, priv->regs_hi+0x00001530);
writel(0x00000000, priv->regs_hi+0x00001800);
writel(0x00000000, priv->regs_hi+0x000024d8);
writel(0x021c0780, priv->regs_hi+0x000024e0);
writel(0x021c0780, priv->regs_hi+0x00001810);
...
```

(__regs_hi__ is DC8200 AHB0 Base Address __`0x2940` `0000`__)

Plus this super baffling one...

```c
writel(uc_plat->base, priv->regs_hi+0x00001400);
```

What's happening here?

Shall we match the Mystery Writes with the [__Official Linux Driver__](https://lupyuen.github.io/articles/display2)? (Pic above)

![Software_RESET_assert0_addr_assert_sel](https://lupyuen.github.io/images/display3-reset.png)

[_Software_RESET_assert0_addr_assert_sel_](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/dom_vout_crg.html#dom_vout_crg__section_bkl_jjt_3sb)

## VOUT Reset

DC8200 Reset Register [_Software_RESET_assert0_addr_assert_sel_](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/dom_vout_crg.html#dom_vout_crg__section_bkl_jjt_3sb) is actually at __Offset `0x48`__, Address __`0x295C` `0048`__.

(Instead of Offset `0x38`, pic above)

_How did we figure out it's at Offset `0x48`?_

Clearing the Reset Bits at Offset `0x38` [(and Offset `0x4C`)](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/dom_vout_crg.html#dom_vout_crg__section_xgs_jd4_2tb) has no effect, [__according to our testing__](https://github.com/lupyuen/nuttx-star64#read-the-star64-jh7110-display-controller-registers-with-u-boot-bootloader).

Thanks to U-Boot, we have this dump of the [__VOUT Clock and Reset Registers (CRG)__](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/dom_vout_crg.html)...

```text
# md 295C0000 0x20
295c0000: 00000004 00000004 00000004 0000000c  ................
295c0010: 00000000 00000000 00000000 00000000  ................
295c0020: 00000000 00000000 00000000 00000000  ................
295c0030: 00000000 00000000 00000000 00000000  ................
295c0040: 00000000 00000000 00000fff 00000000  ................
```

Offset `0x38` is 0, which doesn't look right for a Reset Register that should be filled with "1" Bits (by default).

Then we noticed that __Offset `0x48`__ is filled with "1" Bits: __`0xFFF`__. Thus we cleared the Reset Bits at Offset `0x48`, and it works!

![Clock clk_u0_dc8200_clk_pix0](https://lupyuen.github.io/images/display3-clock2.png)

[_Clock clk_u0_dc8200_clk_pix0_](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/dom_vout_crg.html#dom_vout_crg__section_xtz_dbp_jsb)

## Clock Multiplexing

How to set the Clock Source of __u0_dc8200.clk_pix0__ to __clk_dc8200_pix0__?

[__clk_u0_dc8200_clk_pix0__](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/dom_vout_crg.html#dom_vout_crg__section_xtz_dbp_jsb) is at __Offset `0x1C`__ (Pic above)

- Bits 24 to 29:	__clk_mux_sel__	(Default 0)

  Clock multiplexing selector:
  - clk_dc8200_pix0
  - clk_hdmitx0_pixelclk

But what are the valid values for __clk_mux_sel__?

Can we read another Multiplexed Clock to figure out the valid values?

![Clock clk_dc8200_pix0](https://lupyuen.github.io/images/display3-clock3.png)

[_Clock clk_dc8200_pix0_](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/dom_vout_crg.html#dom_vout_crg__section_iby_5z4_jsb)

## Clock Rate

How to set the Clock Rate of __dc8200_pix0__ to __148.5 MHz__? (HDMI Clock)

[__clk_dc8200_pix0__](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/dom_vout_crg.html#dom_vout_crg__section_iby_5z4_jsb) is at __Offset `0x04`__ (Pic above)

- Bits 0 to 23: __clk_divcfg__ (Default 4)

  Clock divider coefficient:

  Max: 63 (307.2 MHz?), Default: 4, Min: 4, Typical: 4

What is the Clock Divider value we should set?

What is the Input Frequency to the Clock? 307.2 MHz?

![Clock clk_tx_esc](https://lupyuen.github.io/images/display3-typo.png)

[_Clock clk_tx_esc_](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/dom_vout_crg.html#dom_vout_crg__section_gwx_k1p_jsb)

## Clock Default

[__clk_tx_esc__](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/dom_vout_crg.html#dom_vout_crg__section_gwx_k1p_jsb) at __Offset `0x0C`__ (Address __`0x295C` `000C`__) should have default __`24'hc`__. (Equivalent to __`0x0C`__)

There's a typo in the doc, it shows default __`24'h12`__ (Pic above)

![PMU Function Description](https://lupyuen.github.io/images/display3-pmu.png)

[_PMU Function Description_](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/function_descript_pmu.html)

## PMU Software Encourage

From the [__Power Management Unit (PMU) Function Description__](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/function_descript_pmu.html) (pic above)...

> __SW Encourage Turn-on Sequence__

> (1) Configure the register __SW Turn-On Power Mode__ (Offset __`0x0C`__), write the bit 1 which Power Domain will be turn-on, write the others 0;

> (2) Write the __SW Turn-On Command Sequence__. Write the register Software Encourage (Offset __`0x44`__) __`0xFF`__ → __`0x05`__ → __`0x50`__

_What's a "Software Encourage"?_

Something got Lost in Translation. We assume it means __"Software Triggered"__.
