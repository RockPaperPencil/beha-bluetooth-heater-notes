# BEHA Bluetooth heater notes

These are notes from having a go at a BEHA PGB10 Bluetooth heater. It seems like
all Beha bluetooth heaters are using the same control module, so this should at
least in theory be useful for other models as well.
It is quite possible that newer firmware could change how things work thus invalidating these findings without warning, but then again, that will always be the case with closed proprietary stuff.
The firmware version reported by the heater at the time of writing is **1.0**.

## Power loss
There is no backup power to keep the internal clock running when power is lost.
This also apply in cases where the unit is power cycled using the switch on the side of the unit when connected to a powered outlet. When power is restored, the internal clockis reset to june 2020.

## Connecting/pairing
Tested on Raspberry Pi running Raspbian.

### The pairing procedure I used
- Open the bluetoothctl program
- \[bluetoothctl\] menu scan
- \[bluetoothctl\] transport le
- \[bluetoothctl\] back
- \[bluetoothctl\] scan on
- \*Push the pairing button on the heater\*
- \[bluetoothctl\] pair [MAC address]

## Modes of operation
The heater have three modes of operation:
- Dial-set temperature target
- App-set temperature target
- Weekly schedule (four time slots per day)

Turning the dial on the heater automatically puts the heater in "dial-mode".

## GATT Services and characteristics
The heater is using 16-bit UUIDs (reserved address space) for
pretty much everything, even though I could not find the UUIDs listed at the
bluetooth.org website.

The following list of services and characteristics is not a complete list of
absolutely everything the heater exposes.

### Service UUID 1800 (Generic access)

#### Characteristic 2A00 (read/write)
Device name. My device returns "BEHA heater". I have not tried writing to this
characteristic yet and thereby possibly change the broadcasted name of the device.
Could be that the official apps are looking for this specific name and that they
will stop discovering the heaters, if the heater even allows me to write a new value.
Who knows?

### Service UUID EE0D
This service exposes device information and a couple of characteristics of
unknown purpose.

#### Characteristic UUID 2A29 (read)
Manufacturer name string. Mine returns "BEHA".

#### Characteristic UUID C3F9 (read)
Unknown purpose, my heater does not return anything.

#### Characteristic UUID 2A25 (read)
Serial number string. Seems to work.

#### Characteristic UUID 2A27 (read)
Hardware revision string. My heater returns hardware version "0.1".

#### Characteristic UUID 2A26 (read)
Firmware revision string. My heater returns "1.0".

#### Characteristic UUID 3263 (read)
Not sure what this characteristic is/does. My unit constantly returns two bytes containing zeroes.

### Service UUID 82CD
This service exposes the internal clock.

#### Characteristic UUID CB70 (read/write)
This characteristic is used to read what the heater thinks the time is and to
set the clock of the unit.

This characteristic is written to every time the official app connects.

* The two first bytes represents the year.
* The third byte is the month of the year number (1-indexed)
* The fourth byte is the day of the month (1-indexed)
* The fifth byte is the hour of the day
* The sixth byte is the minute of the hour.
* The seventh byte is the second of the minute
* The eight byte is the day of the week. Monday is day 01, sunday is day 07.
* The ninth byte seem to always be zero.
* The tenth byte seem to always be zero.

#### Characteristic UUID 2A14
Does not seem to be in use, always returns four bytes of zero for me.

### Service UUID FA3D
This service can best be described as "the heating service".

#### Characteristic UUID D749 (read/write)
Get and set heating mode.

0: Use temperature dial on the top of the heater as temperature target

1: Perform heating according to the heating schedule

2: Use single/constant temperature target defined through bluetooth

#### Characteristic UUID DEFC (read)
Not sure what first and second bytes are

Third byte is a readout of temperature as sensed by the heater in celcius when divided by 8.

Not sure what the fourth and last byte is/does.

#### Characteristic UUID 2622 (read/write)
Current temperature target in bluetooth setpoint heating mode (UUID D749, value 2)

First and second bytes seem to always be zeroes.

Third byte is current temperature target in celcius when multiplying by 8.

Not sure what the fourth and last byte is, but it seem to always have 65 as its value.

#### Characteristic UUID F53C (read, write, notify)
Name of the heater, as visible in the app.
Hex-coded string.

### Service UUID AF54
This service exposes the schedule for the scheduled heating mode.
Seven characteristics are exposed under this service, one for each day of the week.

Each day of the week has four time slots. In the manufacturers app, the first time
slot of the day appears to be restricted to always kick in at midnight (00:00).
The UI does not allow that time slot to be changed. Not sure what happens if
a non-zero time was to be written to the heater. Uncharted territory, possibly
some danger ahead. Another untested scenario is if the time slots are NOT sorted
from earliest to latest when writing values to the heater.

A time slot is in the format of "<HBBBB".
* The first field represents the time of which the timeslot starts, given in number of minutes since midnight.
* The second field seem to always be zero.
* The third field seem to always be zero.
* The fourth field represents the desired temperature for the time slot, calculated
after the formula (degreesCelcius * 8).
* The fifth and last field seem to always be the number 65.

When reading and writing characteristics for a given day, the four time slots
are just concatenated like this: "<HBBBBHBBBBHBBBBHBBBB".

#### Characteristic UUIDs for days of the week
- Monday: 7B1F (read/write)
- Tuesday: F17A (read/write)
- Wednesday: 6057 (read/write)
- Thursday: 9B38 (read/write)
- Friday: A216 (read/write)
- Saturday: 323C (read/write)
- Sunday: E4C0 (read/write)