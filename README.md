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

## Initial Thoughts

### What we search?

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

### What bit range?

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
 
## Analytics

### Hardware

Just teardown the hardware (sender and receiver) and look for the used IC to send the signals. The case can be easily open by a normal screwdriver.
On the pcb a `PT4302` can be found which is a `Ultra-Low Power OOK/ASK Receiver` for `315 MHz/433.92 MHz`.
In Europe only `433.92 MHz` can be used, so we look in this range for signals.

### Signals

Every 61 seconds are new signals received.

The modulation is `OOK/ASK` and the signal is coded as `PWM` like a morse code.

The information is sent 6 times with following structure:
```
[begin]
[5x]{
  [pause]
  [package]
  [pause package]
}
[pause]
[package]
```

Raw data example:
```
[begin]: 001001001001001001001

[package]+[pause package]: 1101001101001101001101001101001101001001101001101101001001101101001001001001001101101101101001101001001001001001001001001101101001101001101101101001001101001101001001101001001001001001001001101001001001001001001001001001001001001101001001001101001001001101101001101001001001001001001001
```

To get the data as `bit` representation we have to decode the raw data as morse code. A raw `1` is a `low bit` and a raw `11` represents a `high bit`. All raw `0` can be ignored or means a pause if there are more than two.

Example:
```
Raw data: 11010011010011010011010011010011010010011010011011010010011
Bit data:  1 0   1 0   1 0   1 0   1 0   1 0  0   1 0   1  1 0  0   1
```

Bit coding to signal duration:
 * 0 bit: 490 us
 * 1 bit: 966 us

A package contains 88 bit plus pause package 8 bit.

### Package Format

A package has 22 nibbles (4 bit). The pause package has 2 nibbles which are always `0`.

```
Nibble:      1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24
Value(hex):  a a a 5 9 8 3 d 0  0  d  8  2  9  0  1  0  0  0  0  8  a  0  0
```

#### Preamble

Nibble 1, 2, 3 are always `a`.

#### Version (?)

Nibble 4, 5, 6 are always `598` maybe it is the protocol version or it is part of the preamble.

#### Random Id

Nibble 7, 8 changes very poweron and is constant until battery is changed.

#### Low Voltage

Nibble 9 indicates the battery state. `0` battery is ok. `f` if voltage is lower than 2.7V. Only two values `0` and `f` observed.

#### Temperature

Nibble 10, 11, 12 are the temperature in °C. The decimal representation divided by 10 is the current degree. Negative Temperature is indicated by nibble 10 if is `f`.

#### Humidity

Nibble 13, 14 are the humidity in %. The decimal representation is the humidity.

#### Rain Volum

Nibble 15, 16, 17 are the steps of the rain sensor. One step is about 0,75mm rain. The number is imcremented and not reset between transmissions.

#### Wind Speed

Nibble 18, 19 are the wind speed in km/h.

#### Wind Direction

Nibble 20 is the wind direction. Values between are the according wind directions.
```
0 => N
4 => O
8 => S
c => W
```

#### Checksum

Nibble 21, 22 seems to be a checksum.

The checksum is not completly clear. It seems to be a XOR of all bytes, but it is a bit different. Pseudo code formula (names does not exactly match the package format):
```php
0x47 ^ $id ^ $bat ^ $temp ^ $hum ^ $rain ^ $wind ^ $dir ^ $check
```

Exact formula:
```
0x47 ^ byte1 ^ byte2 ^ byte3 ^ byte4 ^ byte5 ^ byte6 ^ byte7 ^ check_byte == 0x00
```

But with the test set of messages there is a difference. According to the `byte4` (humidity) the checksum fails if `byte4 & 0x0f` is different.

After analysing the test messages is seems to be following assosiaction between `byte4 & 0x0f` and offset `calculated_xor & 0xf0` after calcutation.
```
    [0] => b
    [1] => 8
    [2] => d
    [3] => e
    [4] => 7
    [5] => 7
    [6] => 1
    [7] => 2
    [8] => 0
    [9] => 3
    [a] => 5
    [b] => 6
    [c] => f
    [d] => c
    [e] => 9
    [f] => a
```

In the list you can see, that there is no bidirectional relation. There are `4` and `5` which result in `7`. I did not figure out how they calculate the checksum. Maybe they have a bug in the calculation of the checksum.

## Data Samples

Used data for urh can be found in folder [weatherstation_recorded_signals/](weatherstation_recorded_signals/). First value is temperature followed by humidiy and wind direction. All recorded signals are already truncated to one signal of a transmission to save space and make is easier to interpret.

Some additional recorded data without interpretation can be found in [datasamples.txt](datasamples.txt).

## Useful References

Other protocols

 * e.g. https://github.com/merbanan/rtl_433/blob/master/src/devices/alecto.c

Mr Spiess

 * https://www.youtube.com/watch?v=L0fSEbGEY-Q
 * https://www.youtube.com/watch?v=stQPjNI7_DA

