# esphome_mspacontroller

ESPHome controller for M-Spa spabaths.

__USE AT YOUR OWN RISK, I TAKE NO RESPONSIBILITY FOR BRICKED SPABATHS THROUGH USING THIS MODULE!__

This controller will replace the wire connected controller for M-Spa spabaths.

This is heavily based on the work done by Marc Horst, all credit for the decoding of the M-Spa UART logic goes to him.
I used this article as base for this module, please read it for a more thorough introduction: 
https://www.worldhack.de/your-mspa-goes-smart-step-by-step-to-a-diy-smart-home-hot-tub-wi-fi-upgrade/

The ESP32 will replace the wired controllers as connecting both of them would send conflicting signals to the bath.
But for debugging purposes you could connect the ESP32 RX pin to either direction with the controlelr still attached
to read the communication and further improve this module.

Note: Different baths use different UART logic levels. Some baths uses 5V and could damage an ESP32 if connected directly.
I strongly recommend using a logic level shifter to 3V3 to avoid damaging your ESP32 if your bath uses 5V logic levels.
To check the logic level, simply measure DC voltage between any of the center pins of the connector and ground.

Here is an article on a DIY level shifter I first used to read UART signals (though I never tried to control the bath).
I recommend just getting a real one though, theyre quite cheap:
https://hackaday.com/2016/12/05/taking-it-to-another-level-making-3-3v-and-5v-logic-communicate-with-level-shifters/

## Hardware
M-Spa Super Camaro P-CA069 (5V logic levels)
AZDelivery ESP32 Dev Kit C V4 NodeMCU WLAN WiFi Development Board (Amazon)
Gebildet 3.3V-5V 4 Channel Logic Level Converter Bi-Directional Shifter Module CYT1076 (Amazon)
Male JST XH 2.54mm 4 Pin connector (the one with the housing)

I tried to mimic the remote as much as I could, but some functionality might not work exactly as the original remote. 
I did not bother to figure out the timer for example as that can be handled from Home Assistant instead.

### Wiring
The connector from the bath should have +5V and GND as the outer two pins, and the middle two for logic. If you inspect
the wired remote, you should see GND directly connected to the big copper areas on the circuit board, and the +5V pin to
the plus side of a large capacitor next to the JST connector. Different baths may have different pinout, you need to find
out that yourself. The 5V pin should have a fairly stable 5V level, and the logic pins should fluctuate a little below 5V. 

Connect 5V and GND from the bath to 5V and GND respectively on the ESP32, use a multimeter to be certain. 
Connect The middle pins to pin 16 and 17 of the ESP32. You need to figure out which one is which, you should get a 
temperature reading from the bath (check logs or the current_temperature entity) when you connected them correctly.
If your bath uses 5V logic and you use a level shifter, you also need to connect 3V3 from the ESP32 to the low side of 
the level shifter.

If all is working correctly, when you power the bath, you should get a temperature reading in a second or two. If you check
the logs of the module, you should aso see the commands it sends.

## Module

I added a selector for type of bath as some baths expect [target temperature] * 2 when setting it. Mine did not.
If your bath expects double value and is not listed, use the type 'Other (double temp)'. 
I am also not entirely sure what the target temp setting does, because the bath itself will keep heating even if you reach 
that value - more research needed.

The module is rigged to turn on the filter as well if the heater is turned on, and also stop heating if the target temp is 
met. It should stop heating once it reaches [taget].5°C. After it reaches the target, the temp needs to dip below the
target before it starts again. This is done to avoid the heater continuously going on/off.
Example: If the target is 35°C, the module will heat until 35.5°C, then wait until the temp is 34.5°C before starting again.

As only some baths have some functions you should disable the switches for the ones you dont have. I disabled Jet and UV 
by default as I dont have those in my bath.

There is also an external energy meter configured (but disabled) named "sensor.energimatare_spabad_power", change this
to your own if you have a sensor in Home Assistant that measures its power. Otherwise just remove it entirely.

If the controller is restarted, the bath stops, so be sure to start it again should that happen.

Also, configure the default ESPHome settings like encryption, ota, wifi and backup ap.
If you want to debug something, enable more logging the INFO in the logger section.

### Logic codes

All commands are in 4 byte sequences, starting with 0xA5, both from and to the bath. The checksum is always
written as 0xXX here. Check the module logic to see how to calculate it from the values.
0xA5 [command] [value] [checksum]

#### Commands sent from the remote to the bath

0xA5 0x01 0x0[0][1] 0xXX: 
  Heater on(1)/off(0)

0xA5 0x02 0x0[0][1] 0xXX: 
  Filter on(1)/off(0)

0xA5 0x03 0x0[0 -3] 0xXX: 
  Bubble level 0(off)-3

0xA5 0x0D 0x0[0][1] 0xXX: 
  Jet on(1)/off(0)

0xA5 0x0E 0x0[0][1] 0xXX: 
  Ozone on(1)/off(0)

0xA5 0x15 0x0[0][1] 0xXX: 
  UVC on(1)/off(0)

0xA5 0x04 0x0[xx] 0xXX: 
  Set target temperature to xx (20-40 on my bath) (some baths expect value x 2).
  Note: I dont know if this actually does anything, when I set it to a temperature already reached, 
        the heater still runs, checked with a power meter. It could be depending on bath type.

0xA5 0x0B 0x00 0xXX: 
  Sent by the controller when resetting the bath after an error, like F-1 (flow error). This is
  done by long pressing the "Filter" button on the original remote.

#### Commands/statuses sent from the bath to the remote
0xA5 0x06 0x[xx] 0xXX:
  Current temperature value * 2. Divide by 2 to get the value in Celsius. Note that the division
  should be done even if the remote dont need to send the target temperature * 2.
  Value is down to .5°C accuracy.

0xA5 0x08 0x0[x] 0xXX:
  Bath status. The values I have seen is 0x00 and 0x03. 0 is when the bath is not running, and 3 when its running.
  When sending the 0x0B reset command, this will temporarily respond 3 and then 0 again. The module uses this for
  the reset switch that will both turn off all other switches and send the command.

#### Unknown commands sent by the remote, further investigation needed
0xA5 0x09 
0xA5 0x10 
0xA5 0x11 
0xA5 0x0A 
One of these could be related to the timer function

#### Unknown commands sent by the bath, further investigation needed
0xA5 0x12 0x00 0xXX
  Continuously sends 0 as value, have not seen any other value sent

0xA5 0x07 0x0C 0xXX
  Continuously sends 0x0C (12) as value, have not seen any other value sent
