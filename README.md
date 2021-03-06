# esp8266_bridge_broadcast
A SPI to ESP8266 broadcast (rfmon) firmware

Dev blog: https://jeanleflambeur.wordpress.com/

This lib + firmware allows you to inject and receive packets using an esp8266 module.
It's meant for streaming data - like video - similarly to the wifibroadcast project, but instead of using of the shelf wifi dongles with patched firmwares, it uses the esp8266.

**The advantages** over standard wifi dongles (like the WN721N) are:
* Ability to control the rate (from 1Mbps to 56Mbps) and modulation (CCK with and w/o short preamble and ODFM) 
* Cheaper and readily available
* Very short stack so less points of failure. It doesn't have to go through the kernel, 802.11 stack, rate control, firmware etc
* Easy to add new features in the firmware
* Very good sensitivity and power: Up to -98 dBi @1Mbps, -91 dBi @11Mbps, and ~20.5 dBm tx power
* SPI connection so no USB issues on the Raspberry Pi

**Disadvantages:**

* More complicated connectivity. The module is connected through SPI to the host device which is a bit more complicated than just plugging a USB dongle
* Limited bandwidth. With PIGPIO, 12Mhz SPI speed and 10us delay you can get ~8Mbps throughput. Recommended settings are 10Mhz and 20us delay which results in 5-6Mbps


There are 2 helper classes in the project:
* A Phy which talks to the esp firmware. It supports:
  - Sending and receiving data packets up to 1376K. Data is sent through the SPI bus in packets of 64 bytes. When receiving you get the RSSI as well, per packet.
  - Changing the power settings, in dBm from 0 to 20.5
  - Changing the rate & modulation. These are the supported ones:
  	- 0:  802.11b 1Mbps, CCK modulation
	- 1:  802.11b 2Mbps, CCK modulation
	- 2:  802.11b 2Mbps, Short Preamble, CCK modulation
	- 3:  802.11b 5.5Mbps, CCK modulation
	- 4:  802.11b 5.5Mbps, Short Preamble, CCK modulation
	- 5:  802.11b 11Mbps, CCK modulation
	- 6:  802.11b 11Mbps, Short Preamble, CCK modulation
	- 7:  802.11g 6Mbps, ODFM modulation
	- 8:  802.11g 9Mbps, ODFM modulation
	- 9:  802.11g 12Mbps, ODFM modulation
	- 10: 802.11g 18Mbps, ODFM modulation
	- 11: 802.11g 24Mbps, ODFM modulation
	- 12: 802.11g 36Mbps, ODFM modulation
	- 13: 802.11g 48Mbps, ODFM modulation
	- 13: 802.11g 56Mbps, ODFM modulation
  - Changing the channel.*This is broken for now as the radio doesn't seem to react to this setting for some reason.
  - Getting stats from the esp module - like data transfered, packets dropped etc.

* A FEC_Encoder that does... fec encoding. It allows settings as the K & N parameters (up to 16 and 32 respectively), timeout parameters so in case of packet loss the decoder doesn't get stuck, blocking and non blocking operation.

Both classes can be used independently in other projects.

**Test app**

There is also a test app (esp8266_app) that uses them and sends whatever is presented in its stdin and outputs to stdout whatever it received. You can configure the fec params, the spi speeds and the phy rates/power/channel.

**Firmware**

The firmware is done using the Arduino IDE and can be compiled with the 2.3.0 or 2.4.0 sdk. It does require some patching of the Arduino make command to allow access to an internal function:
After downloading the board in arduino, go to *packages/esp8266/hardware/2.3.0/platform.txt* and locate the  *compiler.c.elf.flags* line:

```
compiler.c.elf.flags={compiler.warning_flags} -O3 -nostdlib -Wl,--no-check-sections -u call_user_start -u _printf_float -u _scanf_float -Wl,-static "-L{compiler.sdk.path}/lib" "-L{compiler.sdk.path}/ld" "-L{compiler.libc.path}/lib" "-T{build.flash_ld}" -Wl,--gc-sections -Wl,-wrap,system_restart_local -Wl,-wrap,spi_flash_read
```

Then add this at the end of that line: -Wl,-wrap=ppEnqueueRxq

It should read this:
```
compiler.c.elf.flags={compiler.warning_flags} -O3 -nostdlib -Wl,--no-check-sections -u call_user_start -u _printf_float -u _scanf_float -Wl,-static "-L{compiler.sdk.path}/lib" "-L{compiler.sdk.path}/ld" "-L{compiler.libc.path}/lib" "-T{build.flash_ld}" -Wl,--gc-sections -Wl,-wrap,system_restart_local -Wl,-wrap,spi_flash_read -Wl,-wrap=ppEnqueueRxq
```


**Testing**  
Run this on node A:  
`esp8266_app --mtu 400 --fec 12 20`  
And this on node B:  
`dmesg | esp8266_app --mtu 400 --fec 12 20`  
This will send your dmesg output from node B to node A.  


To test the esp8266 firmware, connect with a serial terminal (the arduino IDE one is good) at 115200 baud and reset the board. You should see the text 'Initialized'. Send a 'V' (for Verbose) and you should start to see stats on the screen, updated every second.  



References:

Raw Wifi hacking:  
https://github.com/ernacktob/esp8266_wifi_raw  

HSPI info:  
http://www.esp8266.com/viewtopic.php?f=13&t=7247#  
http://iot-bits.com/esp32/esp32-spi-register-description/  

PIGPIO:  
https://github.com/joan2937/pigpio  

FEC:  
https://github.com/tahoe-lafs/zfec  



