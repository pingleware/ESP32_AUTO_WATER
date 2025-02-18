# ESP32C3_AUTO_WATER

An ESP32-C3 module together with a capacitive soil moisture sensor and DS3231 Real-time Clock (RTC) is used to automatically water a plant bed using a pump placed in a water storage container. 

Power for the pump comes from a bank of super-capacitors charged by a solar panel. 

Power for the electronics comes from a Li-Poly battery (or 4 x 1.5V AA cells).

The system is self-contained. No mains power supply or connection to a water faucet is required.

<o>

<img src="docs/hardware.jpg" />

<o>
System control parameters are configured via a WiFi access point and web server page.

In normal watering mode, the ESP32-C3 is woken up from deep-sleep once a day at a scheduled time by the DS3231 RTC. It checks the soil moisture level and if required, turns on the water pump for a fixed duration (e.g. 20 seconds). 

It then (optionally) logs the calendar date and time, moisture sensor reading, power supply voltages, watering duration, RTC clock drift (compared to Network time) as a row of entries in a Google Docs spreadsheet document. 

If the Google Sheet access option is enabled and internet access is not available, this data record is queued to a buffer in ESP32-C3 flash memory. 

If Google Sheet access is enabled and internet access is available, any queued records in flash are removed from the queue and uploaded to the spreadsheet, before the current days data record. 

Up to 30 data records can be queued in ESP32 flash. After 30 days, the oldest queued record will be over-written by the new data record.

The schedule parameters (wake-up hour and minute, sensor threshold, watering time) configured via WiFi can be over-ridden by contents of a control tab in the Google Sheets document.

<p>

The spreadsheet tab "AutoWater" is used for logging data. We can see that :
1. For the first entry, the battery was connected, the DS3231 RTC local date and time is 00:00, Jan 1, 2000. So there is a large initial error between NTP time and RTC time.
2. On Dec 29, Jan 12 and Jan 16 the system was not able to connect to the Internet. I use my mobile phone as a hot-spot for Internet access, so it may not always be located within range.
 The queued data records for these days were successfully uploaded on the next day.
3. The accompanying charts  are automatically updated whenever a new row entry is added to the spreadsheet.
4. This is using a small 200mAH lipoly battery (seen in the prototype picture) as an experiment in power consumption.
<p>
<img src="docs/autowater_gs_update.png" />


<p>
The spreadsheet tab "Control" is used to over-ride the wifi-configured schedule parameters. They will take effect on the next scheduled reset and boot. If the spreadsheet entry is marked with an 'x', the currently configured parameters stored in the flash partition will be used.

In the example below, we only want to over-ride the sensor threshold for watering.
<img src="docs/autowater_gs_control.png" />

## Configuration
To configure the watering system, press the reset button for the ESP32-C3 module and then immediately press the configuration button (GPIO9) when you hear a pulsing tone. Keep it pressed until you hear a long confirmation tone, and then release. The system is now configured as a stand-alone WiFi Access Point with SSID `ESPC3WaterTimer` and password `123456789`.

A web server running on this Access Point at the url `http://192.168.4.1` can then be used to configure the following :
* System Options
  * Enable / disable data uploads to the Google Sheet
  * Google Sheet URL ID
  * Google Sheet Tab name
  * Internet Access Point SSID 
  * Internet Access Point password
  * Local Time Zone UTC offset in minutes
  * Local Daylight Savings offset in minutes
  * Daily wake-up time
  * Soil moisture threshold for watering
  * Pump on-time in seconds
* Real-Time Clock
  * Date
  * Time

If data upload is enabled and internet access is available, the system will get the local time from a Network Time Protocol (NTP) server and correct the RTC if required. So if you disconnect and reconnect the battery, you don't have to configure the RTC calendar/time, it will be done automatically. The error in seconds (NTP time minus RTC time) is also logged to the spreadsheet.

If data upload is disabled, the system will not attempt to connect to the Internet.Manual RTC calendar/time configuration via the web server page is then required.

Click on the `Submit` button after making changes to the configuration options or RTC settings.

When you are done with configuration, press the hardware reset button on the ESP32-C3 module. The system will reboot in normal watering mode.

<p>

<img src="docs/ap_config_homepage.png" />

<p>

## Google Sheet update
[Tutorial](https://www.youtube.com/watch?v=RRQvySxaCW0)

This is the google script code used for logging data to the "AutoWater" sheet and retrieving the schedule parameters from the "Control" sheet in our Google Docs sheet.

<img src="docs/gs_script.png" />

## OTA Firmware Updates
You can update the firmware via the WiFi server webpage url `http://192.168.4.1/update`. Choose the new firmware binary file.  After the file is uploaded, the ESP32-C3 module will automatically re-boot with the updated firmware. Check the new firmware revison string in the configuration server home page (assuming the revision string has been updated along with code changes).

<br>

<img src="docs/ap_firmware_update.png" />

<br>

# Execution Log and Current Drain in Active Mode

<img src="docs/ontime_current_draw.png" />

The sensor was disconnected for this run. This is from an  earlier version of the code. Google Sheets returns a redirect HTTP code (302). In the current
version of the code, we then connect to the redirected url to retrieve the control parameters from the html response. This does return the expected 200 success code.

When
* Google Sheet update is enabled
* Internet access is available
* Watering is not required
* No queued unsent records

the total time each day in active mode (i.e. not in deep-sleep) is < 15 seconds. This includes time spent to get local time from an NTP server and correct the RTC.

If watering is required, an additional 20 seconds assuming the default pump on-time of 20 seconds. 

The current drawn from a Li-Ion battery was monitored by an external INA219 current meter, gated by a gpio signal (pin 18) from the ESP32. This gpio pin is set to 1 on entering `setup()` and reset to 0 just before entering deep-sleep. The meter samples the current at ~1.1kHz while the gate pin is high. 

The current meter serial monitor shows that the average circuit current drain from the battery in active mode (9.875 seconds total) is ~46mA, with peaks of ~300mA (during wifi transmission bursts). 

In deep-sleep mode, the total circuit current drain is ~15uA. 

# Build Environment
* Ubuntu 20.04 LTS AMDx64
* Visual Studio Code with PlatformIO plugin using Arduino framework targeting `esp32dev` board. The file `platformio.ini` specifies the framework packages and toolchain required for the ESP32-C3, and the libraries used by the project. The ESP32-C3 Arduino framework support is new and not as solid as for the ESP32. A minor type-cast compile error for the AsyncTCP library had to be fixed by editing the local version of the library source code in the project `.pio` subdirectory.
* Custom `partition.csv` file with two 1.9MB code partitions supporting OTA firmware updates.
* ~160kByte LittleFS partition for hosting static HTML and CSS content.

# Hardware 

## [Circuit Schematic](docs/espc3_autowater_schematic.pdf)

## Power supplies

* 20V Solar Panel<br>
It may not be sunny enough to run the water pump directly off the solar panel when the ESP32-C3 wakes up at the scheduled time.
The solar panel is used to charge up a bank of super-capacitors. This capacitor bank is used to power the 12V water pump.

* Super-capacitor bank<br>
I used eight 10F 2.7V super-capacitors in series to provide an energy storage bank to power the 12V water pump.<br>
Even on a moderately cloudy day, the capacitor bank charges up to ~18V.<br>
A series 1N4007 diode from the solar panel ensures that the capacitor bank does not discharge back through the panel when it's cloudy or dark. Once charged to 17V+, the super-capacitor bank can run the 12V water pump for at least 30 seconds.

* Li-ion Battery<br>
This is used to provide circuit power via an HT7333 LDO 3.3V regulator.<br>

## DS3231 Real-Time Clock 
This provides a daily alarm at the scheduled time to wake up the ESP32-C3 from deep sleep.

### Minimizing DS3231 Current Drain

#### Option 1
The DS3231 VCC pin is connected to the 3.3V circuit supply, and an external CR2032 coin cell connected to the VBAT pin provides backup when you disconnect the system battery. The RTC drains < 100uA as per the datasheet. I measured ~110uA total circuit current drain in deep-sleep mode. The bulk of the deep-sleep circuit current drain is from the RTC.
    
#### Option 2
The DS3231 datsheet gives us a useful option : leave the DS3231 VCC pin un-connected and power the VBAT pin with the circuit 3.3V supply. The RTC SCL, SDA and INT_ pins are now pulled up via resistors to the VBAT pin. In this case, the RTC time-keeping operation works as before but with much less current drain. I measured ~15uA total circuit drain in deep-sleep mode. 
    
There are a couple of caveats : 
1. It takes a couple of seconds for the oscillator to start up the first time power is applied to VBAT.
2. If you disconnect the battery, there is no backup so the RTC loses all date/time information. This is not an issue if Google Sheet updates are enabled. When the ESP32-C3 connects to the internet, it updates the RTC with date & time from an Network Time Protocol (NTP) server. If Google Sheet updates are disabled, manually set the RTC date/time via the WiFi configuration web page. 

### ESP32-C3 Wake-up Reset Pulse
A 4.7uF capacitor in series between the DS3231 INT_ output pin and the ESP32-C3 EN pin generates a reset pulse for the ESP32-C3 at the daily scheduled time.

For the ESP32-C3 EN pin, I used a 2.2K resistor pullup to VCC and a 1uF ceramic cap to ground. 

## Capacitive Soil Moisture Sensor

<br>

<img src="docs/capacitive_sensor.png" />

<br>

[Ensure you have a capacitive sensor module that actually works!](https://www.youtube.com/watch?v=IGP38bz-K48) The version I have uses a 555 timer IC marked "NE555 20M". 

I sealed the electronics back-end of the sensor board with silicone caulk and a heatshrink tube to prevent any corrosion of the electronics. 

It is possible for the top soil layer to dry out while the roots are still in damp soil. So the sensor is placed horizontally, half-way down the side of the plant pot. 

A 1-meter shielded cable provides ground, power supply and analog sensor output interface. The analog sensor output voltage increases as the soil gets drier.

This particular sensor module has a current draw of ~6mA.  It has an on-board voltage regulator and is powered from the battery via an nmos-pmos switch controlled by an ESP32-C3 GPIO pin. After reading the sensor, the sensor power supply is switched off. This is necessary to minimize total circuit current draw during ESP32-C3 deep-sleep mode.

## Power Mosfet Switch Module for Water Pump

<br>

<img src="docs/mosfet_control_module.png" />

<br>

I used this module to control the water pump. The PWM input is connected to an ESP32-C3 GPIO pin for simple on-off control. 

<br>

<img src="docs/LR7843-MOSFET-Control-Module-Schematic.jpg" />

<br>

The super-capacitor power bank provides the DC power supply for the water pump. 

I used an FR303 diode as flyback protection for the inductive pump motor load.

A ferrite bead isolates the high current spikes in the water pump circuit ground from the electronics ground. Actually the solar panel + capacitor bank + water pump circuit could have been completely isolated from the control electronics if we were not interested in monitoring the super-capacitor bank voltage via the ESP32-C3 ADC (this requires a common ground).


## Minimizing Circuit Current Drain

The eventual project goal is to use a primary battery (or a rechargeable battery with low self-discharge rate), without needing to replace or recharge it for a few months at least. 

I used a Holtek HT7333 voltage regulator because it draws very little quiescent current (~4uA) and has a low dropout voltage of 100mV-150mV.

The ESP32-C3 is clocked at the minimum clock frequency that gives us WiFi capability (80MHz).

The moisture sensor module draws ~6mA but the power supply to the sensor is switched off after the reading is taken. 

The ADC resistor drop for measuring battery voltage is taken from the switched power supply for the moisture sensor, so there is no resistive drain path in deep-sleep mode.

In deep-sleep mode, the components drawing current from the battery are :
* ESP32-C3 deep-sleep current
* HT7333 regulator quiescent current
* DS3231 RTC (minimized current drain by using VBAT as power supply with VCC pin unconnected - see above)

We can estimate battery capacity drain over the course of a day : 
1. Active mode : ~50mA x ~20 seconds = 50mA x (20/3600)Hr = 0.28mAHr
2. Deep-sleep mode : ~15uA x ~24 hours = 0.015mA x 24Hr = 0.36mAHr

So the total capacity drain in 24 hours = 0.64mAHr.

We could use this to estimate the number of days we can run with a fully charged battery. 

E.g. A 500mAHr Li-poly battery gives us 500/0.64 = 781 days. Impressive, but we can't use the full battery capacity for two reasons :
1. VCC is 3.3V and the LDO regulator has a minimum drop-out voltage of 100-150mV. So realistically we should only allow for a discharge to 3.5V from the fully charged 4.2V.
2. As the battery discharges, its internal series resistance increases. There is an I.R drop across the battery internal series resistance. The ESP32-C3 peak current requirements are ~300mA during wifi transmission bursts. If the internal series resistance is high, the additional I.R drop will cause a brown-out reset.

<b>I am currently experimenting with a tiny 200mAHr Li-Poly battery. </b>

To compensate for the increased internal resistance on discharge, I soldered a 1500uF 6.3V solid polymer capacitor (low ESR, low leakage) in parallel with the battery terminals, BEFORE the overcharge/overdischarge protection circuit board that is part of the battery module. 

Prior to this I tried to measure the self-discharge leakage current of the capacitor by charging it with a battery. When it had fully charged, the residual current was below the resolution of my current meter. So I'm assuming the capacitor adds insignificant self-discharge leakage to the combined (li-poly + capacitor) battery. 

Even assuming just 100mAHr capacity, that gives us 100/0.64 = 156 days! Hard to believe. Let's see if the capacitor in parallel will act as a low ESR storage bank for high current pulses and prevent ESP32-C3 brown-outs when the li-poly battery has discharged 50% to 3.6V.

# Credits
* [Updating Google Sheet via HTTPS](https://stackoverflow.com/questions/69685813/problem-esp32-send-data-to-google-sheet-through-google-app-script)
* [ESP32 Async Web Server using SPIFFS]( https://randomnerdtutorials.com/esp32-web-server-spiffs-spi-flash-file-system/)

# Additional References
[![Capacitive Soil Moisture Sensors don't work correctly + Fix for v2.0 v1.2 Arduino ESP32 Raspberry Pi](https://i3.ytimg.com/vi/IGP38bz-K48/hqdefault.jpg)](https://www.youtube.com/watch?v=IGP38bz-K48)

Search https://www.bing.com/search?q=automatic+plant+watering+system+using+esp32

wESP32: Wired ESP32 with Ethernet and PoE - https://hackaday.io/project/85389/files