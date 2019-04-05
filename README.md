# Reverse Engineering of Weatherstation from Conrad

## License

[Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International Public License](https://creativecommons.org/licenses/by-nc-sa/4.0/)

## Motivation

The aim of the analysis is to get the package format of the [Renkforce AOK-5055](https://www.conrad.de/de/renkforce-aok-5055-funk-wetterstation-vorhersage-fuer-12-bis-24-stunden-1267773.html).

## Used Tools

Hardware

1. DVB-T USB stick (rtl chipset), as cheap software defined radio.

Software

1. Kali Linux
1. urh 2.5.6 - [Universal Radio Hacker](https://github.com/jopohl/urh), for receiving, analysing and decoding signals.

## Setup

### Setup Tools

1. Install dependencies for `urh`, also `librtlsdr-dev`.
1. Install `urh` by `pip`.
1. Plugin DVB-T stick.

### Setup Weather Station

Assamble the station as described in the manual, but do not cover the sender module. Access to the batteries is useful to do a quick reset.

## Analytics

### Hardware

Just teardown the hardware (sender and receiver) and look for the used IC to send the signals. The case can be easily open by a normal screwdriver.
On the pcb a `PT4302` can be found which is a `Ultra-Low Power OOK/ASK Receiver` for `315 MHz/433.92 MHz`.
In Europe only `433.92 MHz` can be used, so we look in this range for signals.

### Initial Thoughts

#### What we search?

Values sent from outdoor to indoor station. The indoor station shows following values according to the manual:

 * temperature
 * humidiy
 * wind direction
 * wind speed
 * rain volum
 * low battery

Maybe some additional basic values of protocol design:
 
 * preamble
 * protocol version
 * device identification
 * checksum

#### What bit range?

 * temperature
   * -20-70°C = diff 90 => 7 bit
 * humidiy
   * 20-95% = diff 75 => 7 bit or 6 bit
 * wind direction
   * 16 directions => 4 bit
 * wind speed
   * 0~256kmh => 8 bit
 * rain volum
   * 0~999.9MM => 14 bit
   * smalles step 0,7mm next 1,5mm suggested step about 0,75mm = 1333 steps => 11 bit
 * low battery
   * 1 bit

### Signals

TODO

## Useful References

Other protocols

 * e.g. https://github.com/merbanan/rtl_433/blob/master/src/devices/alecto.c

Mr Spiess

 * https://www.youtube.com/watch?v=L0fSEbGEY-Q
 * https://www.youtube.com/watch?v=stQPjNI7_DA

