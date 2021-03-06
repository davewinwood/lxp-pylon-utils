# LXP & PylonTech Monitoring in Ruby

The code in this repository is what I use to communicate with my LuxPower LXP 3600ACS inverter and a stack of PylonTech US2000 batteries (a cheaper version of a Powerwall).

## LXP 3600 ACS

The LXP protocol library itself has been extracted into a gem: https://rubygems.org/gems/lxp-packet.

The inverter by default sends information about itself to LuxPower every 2 minutes. It can optionally be configured with a second network endpoint; I set this to TCP Server with a port of 4346, which means you can connect to the inverter on that port and get the same information sent to you. You can also send it commands. So the "Network Setting" page of my inverter looks like this:

![LXP ACS Network Settings](https://i.imgur.com/teygH6h.png)

`config.ini` should look like this. Replace your real serials (they're placed into data packets) and the address/port are used to connect to.

```ini
[datalog]
serial = AB12345678

[inverter]
serial = 1234567890
address = 192.168.0.209
port = 4346

[influx]
database = foo
host = 192.168.5.100
```

`lxp_server.rb` opens a socket to the TCP server and writes some JSON containing the details I want into `/tmp/lxp_data.json`.

It runs a simple webapp that returns the contents of this JSON for any request, which can be graphed in Munin or whatever.

## PylonTech US2000 (maybe US3000 too)

The batteries can be communicated with over RS232 (console) or RS485. The Pylon class can be used with either with minimal modifications; I use RS485 as it can go a lot faster (115200bps vs the console's 1200bps by default).

Fortunately PylonTech appear a lot more hacker friendly than LuxPower, and I do have API documentation for the RS232/485 protocols for these. I'll add decoding of more packet types as I need them; for now analog and alarm data are done.

`pylon_server.rb` uses an USB-RS485 adaptor which is on `/dev/ttyUSB0`, and fetches new information regularly, storing it in `/tmp/pylon_data.json` (currently analog and alarm data are fetched). Again this is served over a HTTP webapp.

There's also a `monitor.rb` which watches for the pylon data JSON changing and renders it in a terminal. It looks a bit like this:

![monitor.rb screenshot](https://i.imgur.com/Fq0WrT0.png)

It's a bit knocked together, so a bit messy, and hardcoded for an 80x24 terminal, but it does the job for me. Because it watches the JSON file for changes it must run on the same machine as `pylon_server` but could be trivially modified to fetch over HTTP every minute instead.

On the left are individual cell voltages. These go yellow or red if the battery sends an alarm about them (voltage too low or too high). I added my own warnings to these too; if the difference between the lowest and highest cell is more than 10mV, the lowest will get a blue background and the highest will get a red background. Note this does not represent any action the BMS may be taking - it is entirely local to the Ruby app.

Below that are temperatures; the left-most is the BMS board, the next 4 are averages of various cells. Next to those is the lowest/highest cell voltage and the difference between them.

To the right is mostly abbreviations:

  * **CHG_CUR** is charge current; red if too high?
  * **DIS_CUR** is discharge current; red if too high?
  * **VOLTAGE** is entire pack voltage; red if too low
  * **UV** / **OV** are module undervoltage and overvoltage respectively
  * **C_OC** / **D_OC** are (dis)charge overcurrent
  * **C_OT** / **D_OT** are (dis)charge overtemperature
  * **C_FET** / **D_FET** are green when the (dis)charge MOSFETs are on. This seems to be electrical isolation for the batteries in some alarm situations
  * **D_EN** / **C_EN** are (dis)charge enable. These seem to signal an inverter to stop discharging or charging respectively
  * **CELL_UV** is cell undervoltage. Some of the voltage displays have probably gone red to indicate which one
  * **BUZ** means the alarm buzzer is sounding
  * **FULL** is green when the battery is full (100% SOC)
  * **ONBAT** is green when the module is being powered internally, from its own batteries
  * **EDC** / **ECC** are effective (dis)charge current. These seem to come on when the battery thinks it has enough current to be charging or discharging? Not sure
  * **CI1** / **CI2** are "charge immediately". 1 comes on when the SOC is 15%-19%, and 2 comes on at 9%-13%. Probably used by inverters to decide when to charge to stop the batteries going flat
  * **FCR** is full charge request. The Pylontech datasheet says this is to stop SOC calculations drifting too far from reality when the battery has not hada  full charge for 30 days. They suggest inverters might like to use grid charging when this comes on, to give the batteries a cycle
  * **S4** / **S5** are more cell status bits, if they're non-zero then I think a cell has been completely disconnected from the pack. May show this in the voltages in a later update (flashing voltage?)
