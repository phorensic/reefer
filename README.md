# reefer

This started out as a project to control my reef aquarium, hence the name. It then morphed into a simple Python script to check the indoor temp/humidity and compare it with the outdoors using OpenWeather API. Update: It now monitors the temperature of my aquarium!

Note: This project is now split into a FT232H version and a Raspberry Pi version. Updated 11/11/2021

## FT232H

The hardware is a FT232H from Adafruit and a Si7021 sensor. The FT232H is a pain in the butt to set up from scratch, espcially on Windows. I use Zadig to change the driver to libusbK, then the environment variable must be set "BLINKA_FT232H=1". Use the hack to refresh the environment variables without rebooting (now included, see ```RefreshEnv.cmd```), then the Python script should work. If you ever switch USB ports you must use Zadig to change the driver again.

Adafruit has written a guide for the FT232H, but it is confusing and becoming out of date: https://learn.adafruit.com/circuitpython-on-any-computer-with-ft232h

### Dependencies
- Python 3
- pip3
- pyftdi
- pyserial
- pyusb
- Adafruit-Blinka
- adafruit-circuitpython-busdevice
- adafruit-circuitpython-si7021
- Adafruit-PlatformDetect
- Adafruit-PureIO
- requests
- charset_normalizer
- certifi
- urllib3
- idna
- rrdtool (added to system $PATH/%PATH%/Path)

### RRD
#### Create RRD databases:

```
$ rrdtool create temperatures-c.rrd --step 60 DS:outdoor:GAUGE:120:0:55 DS:indoor:GAUGE:120:0:55 RRA:MAX:0.5:1:1440
$ rrdtool create humidities.rrd --step 60 DS:outdoor:GAUGE:120:0:100 DS:indoor:GAUGE:120:0:100 RRA:MAX:0.5:1:1440
```
This will create databases with a 60 second interval, 120 second heartbeat timeout, between 0 and 55 degrees Celsius, and 0-100% relative humidity, with 24 hours of data before rolling over. You may need to configure for lower than 0 degrees Celsius, but I live in the second hottest place on the planet so these are my settings.

More information here: https://michael.bouvy.net/post/graph-data-rrdtool-sensors-arduino

### HTML

```index.html```

Very simple HTML to display both graphs. This does not need a webserver. It can be opened locally and will refresh the entire page every 60 seconds.

Note: This version is no longer under active development. See Pi version below.

## Raspberry Pi

I moved the project from a FT232H to a Raspberry Pi Zero W and added a DS18B20 waterproof temperature sensor for an aquarium. With plenty of GPIO on the Pi and relays in hand, I plan on controlling pumps/lights/etc in the future.

### Hardware
#### Si7021
https://pinout.xyz/pinout/i2c
- VIN to 3v3 (pin 1)
- GND to ground (pin 9)
- SCL to GPIO 3 [I2C1 SCL Clock] (pin 5)
- SDA to GPIO 2 [I2C1 SDA Data] (pin 3)

#### DS18B20
https://pinout.xyz/pinout/1_wire
- Red Wire to 5v (pin 2)
- Blue Wire to ground (pin 6 or 9)
- Yellow Wire to GPIO 4 [Data] (pin 7)
- 4k7 pullup resistor between 5V and Data (yellow wire)

According to my testing, Google searches, and the DS18B20 datasheet 3v3 is not enough voltage to pull up the data line with the commonly included 4k7 resistor. I measured ~1.6 volts with the 4k7 resistor in place and the datasheet says it wants a minimum of 3.0 volts. I was having massive dropouts of the sensor until I switched everything from the 3v3 rail to the 5v rail. I don't know how anybody else got it working on 3v3 and then wrote the guides floating around the web (I'm looking at you too Adafruit!). You could, of course, substitute an appropriate resistor and still use 3v3 based on some multimeter measuring, calculating, etc. But why not give that sucker some juice! It's been rock solid since I changed to the 5v rail + 4k7 pullup.

### Dependencies

You can satisfy pretty much all dependencies with these commands on a fresh Pi:
```
$ sudo apt install rrdtool
$ sudo apt install python3-pip
$ sudo pip3 install adafruit-circuitpython-si7021
$ sudo pip3 install git+https://github.com/nicmcd/vcgencmd.git
```

I2C and 1-Wire interfaces must be turned on in ```$ sudo raspi-config```

### RRD
#### Create RRD databases:

```
$ rrdtool create temperatures.rrd --step 60 DS:outdoor:GAUGE:120:0:55 DS:indoor:GAUGE:120:0:55 DS:tank:GAUGE:120:0:55 DS:pi:GAUGE:120:0:100 RA:MAX:0.5:1:1440
$ rrdtool create humidities.rrd --step 60 DS:outdoor:GAUGE:120:0:100 DS:indoor:GAUGE:120:0:100 RRA:MAX:0.5:1:1440
```
This will create databases with a 60 second interval, 120 second heartbeat timeout, between 0 and 55 degrees Celsius for the air/water sensors, 0-100 degrees Celsius for the pi CPU sensor, and 0-100% relative humidity, with 24 hours of data before rolling over. You may need to configure for lower than 0 degrees Celsius, but I live in the second hottest place on the planet so these are my settings.

More information here: https://michael.bouvy.net/post/graph-data-rrdtool-sensors-arduino

### HTML

```index.html```

Very simple HTML to display the graphs. You will likely want a webserver for this. I use nginx. It refreshes the page every 60 seconds. Sometimes this generates a miss while the PNG files are being generated.
