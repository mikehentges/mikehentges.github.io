---
layout: post
title: Hitachi VFD for OneFinity CNC
date: 24-08-2021
categories: woodworking
hero: https://res.cloudinary.com/dbzsk4ytb/image/upload/v1661027872/blog-images/onefinity-hitachi_a7plb8.png
---

After retiring, one of my "new technology" investments was to pick up a CNC machine. Woodworking is a hobby of mine, and a CNC machine was an opportunity to combine technology with my hobby. The Onefinity CNC is a new manufacturer in this space, and I found their product the most compelling during my search for a hobbyist-level machine.

I wanted to upgrade from Onefinity's standard option for a cutting device - the Makita 1HP router. Even though the Makita router would have likely been just fine for my needs - the opportunity to have finer control over the router was appealing.

When researching which spindle to purchase, I first looked at the Haunyang (HY) spindle/VFD options. These are by far the most common choices - they work, and they are cheap. I ran across the Hitachi WJ200-022SF for the VFD when looking to see if anything better was available. My main reason for considering this model was to take advantage of Sensorless Vector Control - a means to get greater torque at lower speeds. The cheaper HY VFDs do not have this feature.

I purchased the VFD here: [WJ200-022SF 3HP 2.2kW 230V Single Phase Input VFD - Hitachi](https://www.driveswarehouse.com/wj200-022sf). I purchased my 2.2kW spindle from a USA-based supplier: [2.2kW air-cooled spindle](https://www.automationtechnologiesinc.com/products-page/cnc-spindles/2200w-3hp-air-cooled-cnc-milling-spindle).

The Hitachi VFD manual is much better than the HY ones - but there are not as many youtube videos that walk through the setup. Fortunately, I had previously set up an HY spindle, which made the 2nd one a lot easier.

Here are the main VFD settings I had to adjust for my setup:

- B031 = 10 (to access all menus)
- A001 - 03 = ModBus network
- A002 - 03 = ModBus network run control
- A004 - 400 (max Hz)
- A003 - 400 (base freq)
- A082 - 220V (motor volts)
- B012 - Current limit (7.6a for my motor)
- B083 - 10kz Carrier Freq
- B165 - 00 (Trip on control loss)
- H003 - 2.2 (capacity)
- H004 - 2 (poles)
  To set up the SVC, the VFD will auto-tune itself if you follow these steps:
- A044 - 3 (SVC)
- H001 - enable auto-tuning
  Hit run, and it will auto-tune, showing "0" when completed
- H002 - 2 Activate SVC tuning

Setting up RS485 to the OneFinity was fairly easy; I used the following:
ModBus settings:

- RS+ - sp
- RS- - SN
- C096 - 00 (Modbus-RTU)
- C071 - 05 (9600 bps)
- C072 - 1 (address)
- C074 - parity
- C075 - stop bit
- C076 - error select (02 disable)
- C077 - error time-out (0.00)
- C078 Communication wait time (0)

Then, the OneFinity doesn't list "Hitachi" as a VFD option - but the Omron MX2 is a re-badged Hitachi, so its setup worked. Here are the 1F settings:

<img class="center" src="https://res.cloudinary.com/dbzsk4ytb/image/upload/v1631986618/blog-images/tool_config.png" width="75%" />
<img class="center" src="https://res.cloudinary.com/dbzsk4ytb/image/upload/v1631986632/blog-images/modbus_config_o.png" width="75%" />
<img class="center" src="https://res.cloudinary.com/dbzsk4ytb/image/upload/v1631986645/blog-images/active_modbus_program.png" width="75%" />
<img class="center" src="https://res.cloudinary.com/dbzsk4ytb/image/upload/v1631986674/blog-images/modbus_communication.png" width="75%" />

Here are a few pictures of the VFD wiring:
<img class="center" src="https://res.cloudinary.com/dbzsk4ytb/image/upload/v1631986412/blog-images/_MG_3559.jpg" width="75%" />
<img class="center" src="https://res.cloudinary.com/dbzsk4ytb/image/upload/v1631986412/blog-images/_MG_3560.jpg" width="75%" />
