# Flatpack2 CAN Protocol

## Conventions in This Document
MSG is a message from the power supply to the controller.  
CMD is a message from the transceiver to the power supply.  
XX is ID of PSU (Set during login)

##  Hardware
The Flatpack2's CAN bus runs at 125kbit/s, using an extended ID field, referenced to the PSU's negative rail.  
It is necessary to connect the GND contact of your CAN bus to the CAN bus GND contact of your power supply.  

## Notes
Everything written in this document was tested exclusively on the Flatpack2 HE 48V/2000W. I use the MSP2551 transceiver.

## Operating Principle
When turned on, the PSU sends a Hello packet approximately every 2 seconds (containing the power supply serial number).
After you sending the Login command, the PSU sends a Status packet approximately every 0.2 seconds (containing the current voltages and temperatures).
After 64 Status packets, a logout occurs. After that, a Login request packet arrives approximately every 10 seconds. After sending the Login command, you will receive the Status packet again.
Without the Login command, the PSU sets the default voltage and has no current limit. After the Login command, you set the voltage and current limit. After the Logout, it reverts to the default voltage and no current limit. To ensure your parameters remain consistent, send the Login command before receiving the 64 Packet Status.
In Login mode, the PSU responds to your requests; in Logout mode, the PSU ignores everything except Login.


## MSG Hello packet, 0x0500XXXX (0x1b,serial[1-6],0x00)
PSU sends a Hello packet approximately every 2 seconds (containing the power supply serial number) if not logged in; XXXX = last digits of the serial number

> :memo: **Example:** 
MSG 05005566, 1b 11 22 33 44 55 66 00 - Serial is 112233445566

## CMD Log in command, 0x050048XX (serial[1-6],0x00,0x00)	
Log in to a power supply and assign it a CAN ID. Serial number payload chooses which supply to target.
PSU ID is assigned by setting XX to (ID * 4), e.g. sending message ID 0x05004804 assigns PSU ID of 0x01 for future commands.
Allowable ID range is 0x01 to 0x3F , or an XX of 0x04 through 0xFC , and the power supply will log out if no login packet is received for (64 * 0.2) seconds.

> :memo: **Example:** 
CMD 05004804, 11 22 33 44 55 66 00 00 - Login, serial is 112233445566, ID is 01

## MSG Status packet, 0x05XX40YY (intemp, Iout[2], Vout[2], Vin[2], outtemp)
After you sending the Login command, the PSU sends a Status packet approximately every 0.2 seconds (containing the current voltages and temperatures). After 64 Status packets, a logout occurs. XX is PSU ID. 
Current, output voltage and input voltage are stored in little endian (LSB first). All temperatures are in degrees Celsius. Current is in deciamps (i.e. 21.2A is 212). Output voltage is in centivolts (i.e. 48.52V is 4852). Input voltage is in volts.

#### YY is Status flags:
:------:|:--------------------------
0x04 | normal (constant voltage)
0x08 | warning (constant current)
0x0c | alarm
0x10 | walk in

> :memo: **Example:** 
MSG 05014004, 23 64 00 68 15 e6 00 37 - intemp 35(0x23) °C, Iout - 10A(100=0x64), Vout - 54.8V(5480=0x1568), Vin - 230V(0xe6), outtemp - 55(0x37) °C

## MSG Log in request packet, 0x05XX4400 (serial[1-6],0x00,0x00) xx=ID каждые 10 сек
After 64 Status packets, a logout occurs. After that, a Login request packet arrives approximately every 10 seconds. Similar to the CAN hello packet, but uses supply's pre-set ID. After sending the Login command, you will receive the Status packet again.
XX is PSU ID.

> :memo: **Example:** 
MSG 05014400, 11 22 33 44 55 66 00 00 - serial is 112233445566, ID is 01

## CMD Alert request command, 0x05XXbffc (0x08, 0x04 warn/0x08 alarm, 0x00)
This is sent to the PSU to request information about current warnings and alarms. The command should be sent after receiving message 0x05XX40YY, where YY = 08 or YY = 0C (i.e., when a warning or alarm is present). The second byte determines which information is returned: warning or alarm data.

> :memo: **Example:** 
CMD 0501bffc, 08 04 00 - 0x04 - warnings request 

## MSG Alert information packet, 0x05XXbffc (0x0e, 0x04/0x08, 0x00, Alert flag1, Alert flag2, 0x00, 0x00)
Sent by the PSU in response to receiving message 0x05XXBFFC. It contains information about any active alarms or warnings, depending on the content of the request message. 2nd byte indicates whether flag bits are warning or critical, and is equal to 2nd byte of CMD packet.
XX is PSU ID. Bit 0 is the LSB.

#### Warnings/Alarms
Bit | Warning/alarm flag1 | Warning/alarm flag2
:---:|:--------------------|:--------------------
0 | OVS Lock Out | Internal Voltage
1 | Mod Fail Primary | Module Fail
2 | Mod Fail Secondary | Mod Fail Secondary
3 | High Mains | Fan 1 Speed Low
4 | Low Mains | Fan 2 Speed Low
5 | High Temp | Sub Mod1 Fail
6 | Low Temp | Fan 3 Speed Low
7 | Current Limit | Inner Volt

> :memo: **Example:** 
MSG 0501bffc, 0e 04 00 80 00 00 00 - Warning, Current Limit

## CMD Set voltage and current limits command, 0x05XX4004 (max_cur[2], meas_vol[2], des_vol[2], OVP_vol[2])
Sent to the PSU to immediately set output voltage and current limits.
If the supply logs out, these settings will be lost - default voltage will apply, and current limit will be set to factory maximum. Max current (max_cur[2]) is the point at which the supply will switch from CV to CC modes. 
Desired voltage is the output voltage setpoint.
Measured voltage is for calibration/feedback; if you do not have a feedback voltage source, it should be equal to desired voltage.
OVP voltage is the voltage at which over-voltage protection will enable & cause the supply to shut down. Set this to your supply's maximum rated output voltage.
XX is PSU ID. Set XX to ff to send all PSU

> :memo: **Example:** 
CMD 05ff4004, 64 00 68 15 68 15 e0 15 - Max 10A(100=0x64), meas_vol[2]=des_vol[2], MaxV - 54.8V(5480=0x1568), OVP - 56V(5600=0x15e0)

## CMD Set default voltage command, 0x05XX9c00 (0x29, 0x15, 0x00, new_volt[2])
Sent to the power supply to set its default voltage. Does not take effect until the supply is logged out. If the supply is logged in when the command is sent, the voltage is set when the log in times out. If it is not logged in, the voltage will be set when the supply logs in then times out. The voltage is stored in little-endian and is in centivolts (i.e. 48.52V is 4852).
XX is PSU ID.

> :memo: **Example:** 
CMD 05019c00, 29 15 00 68 15 - DefV - 54.8V(5480=0x1568)

## CMD Information request command, 05XXbc00 (0x50, ZZ, 0x00)
Sent to the PSU information request. 
ZZ is 0x00 - Model, 0x04 - Part No, 0x08 - Serial No, 0x0c - Revision
XX is PSU ID. 

> :memo: **Example:** 
CMD 0501bc00, 50 00 00 - Model request command
CMD 0501bc00, 50 04 00 - Part No request command
CMD 0501bc00, 50 08 00 - Serial No request command
CMD 0501bc00, 50 0c 00 - Revision request command

## MSG Information response packet, 05XXbc00
The response arrives in several packets. Below are examples of responses. XX is PSU ID. 

> :memo: **Example:** Model response - FLATPACK2 48/2000 HE
'''
MSG 0501bc00, 53 00 86 46 4C 41 54 50 - FLATP
MSG 0501bc00, 53 00 05 41 43 4B 32 20 - ACK2 
MSG 0501bc00, 53 00 04 34 38 2F 32 30 - 48/20
MSG 0501bc00, 53 00 03 30 30 20 48 45 - 00 HE
MSG 0501bc00, 53 00 02 00 00 00 00 00 - 
MSG 0501bc00, 53 00 01 00 00 90 FB 3F - \x90\xfb?
'''

> :memo: **Example:** Part No response - 241115.105
MSG 0501bc00, 53 04 83 32 34 31 31 31 - 24111
MSG 0501bc00, 53 04 02 35 2E 31 30 35 - 5.105
MSG 0501bc00, 53 04 01 00 00 90 FB 3F - \x90\xfb?

> :memo: **Example:** Serial No response - 112233445566
MSG 0501bc00, 53 08 82 11 22 33 44 55 - 
MSG 0501bc00, 53 08 01 66 C4 90 FB 3F - \x84Đ\xfb?

> :memo: **Example:** Revision response - 3.2
MSG 0501bc00, 53 0C 82 33 2E 32 00 00 - 3.2
MSG 0501bc00, 53 0C 01 00 C4 90 FB 3F - Đ\xfb?
