# OSEnergy Programming Guide

##
This document will assist in developing devices to participate in an OSEnergy compliant system.  Both Battery Masters and Charging Devices are covered:
&nbsp;&nbsp;&nbsp;- Battery Master  (BatMan):  Coding needed to enable a node to function as the system wide conductor/coordinator
&nbsp;&nbsp;&nbsp;- Charging Device (ChgDev):  Coding needed to allow a node to participate in a cooperative way for charging an associated battery.

Please refer to the `OSEnergy Design Guide` for additional architectural details on how each type of device is utilized in a complete system.  Do note that a single device may provide for more then one OSEnergy role, example, an alternator regulator is a Charging Device, but may (if capable) also serve as a potential Battery Manager.


The following sections outline the steps each type device type needs to take to support the OSEnergy specification.  Do note that it is possible for one physical device to potential support multiple functions.


<br><br>
## Common Requirements
All OSEnergy CAN connected devices will need a Control Area Network controller able to support the CAN specification 2.0B data link layer as well as a transceiver able to support ISO-11898-2 electrical specifications.  A  paring of the MCP2515 & MCP2561 is a common way to meet these hardware.

In operation, the CAN configured for:
- 250Kbps
- 29 bit (extended) addressing

Refer to the `OSEnergy Design Guide` for additional hardware details, including cabling and termination recommendations.


All common CAN conventions (including the use of all 1's to indicate a non-existing value) are used.
  
 <br><br>
### J1939 and RV-C foundation
 The OSEnergy protocol is built upon the J1939 standard, specifically on the variant of J1939 as implemented in the RV-C standard  ([http://www.rv-c.com/?q=node/75](http://www.rv-c.com/?q=node/75).   (Relevant copy of this specification are located in the OSEnergy github repository)  It is recommended to review this document carefully.  At a minimum, OSEnergy devices must support the following DGNs/PGNs:
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1.  0xEA00 - Address Request
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2.  0xEE00 - Address Clamed
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3.  0xEA00 - DGN/PGN Request 
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;4.  0xECFF - Multi-packet initiator
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;5.  0xEBFF - Multi-packet subsequent packet
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;6.  0xFEEB - Product identification message
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;7.  0x1FECA - Operational Status and Diagnostics Message
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;8.  0x1FED6 - Manufacturer-Specific ADDRESS_CLAIM Request
 
Other DGNs / PGNs which are recommended to support include:
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;9.  0xE800 - Acknowledgment xx
&nbsp;&nbsp;&nbsp;&nbsp;10.  0x17E00 - Terminal (ASCII) messages


<br><br>
####DERIVATIONS FROM RV-C SPECIFICATION
 OSEnergy as currently implemented utilizes the open CAN standard RV-C for communication between devices.  DGN/PGN references are more completely described in the RV-C specification referenced above.

There are two notable deviations to the RV-C specification:
&nbsp;&nbsp;- Addition of `Charger Status 2`
&nbsp;&nbsp;- Definition of `Charger Type` 

The current release of RVCMessages contains one proposed addition to the RV-C specification, *Charger Status 2.*  
It is enabled by reusing the retired PGN @ 1FF9Dh.   

```  
// Charger Status2 - 1FF9Dh  (PROPOSED, USING OLD BRIDGE_DGN_LIST DGN #)  
// Input:  
//  - Instance              Instance of charger  
//  - DC Source Instance    DC Instance (bus) ID associated with  
//  - Device Priority       Relative ranking of DC charging Source  
//  - DC Voltage            0..3212.5v, in 50mV steps  
//  - DC Current            -1600..1612.5a, in 50mA steps (0x7D00 = 0A)  
//  - Temperature           -40..210 in deg-C, in 1C steps
```


In addition, a ```Charger Type``` designation has been added to all the 'Charger' DGNs via the upper nibble of the existing Instance byte. (6.21.x in December 2015 release of RV-C spec).  This allows for a common approach to all charging sources, while being able to segregate the different type of chargers:

```
 enum tRVCChrgType  {                                   /*  EXTENSION -  INSTANCE TYPE CLASSIFICATION OF CHARGER DGNs */
                            RVCDCct_Default=0,          /* Default value, for backwards compatibility */
                            RVCDCct_ACSourced=0,
                            RVCDCct_Solar=1,
                            RVCDCct_Wind=2,
                            RVCDCct_Engine=3,           /* Engine alternator, DC Generator, etc */
                            RVCDCct_FuelCell=4,
                            RVCDCct_Water=5,
                            RVCDCct_VD01=13,            /* Vender Defined types */
                            RVCDCct_VD02=14,
                            RVCDCct_Unknown=0x0F        /* 4-bit field - all 1's indicated undefined value */
                          };
```
                          
 
 
<br><br>
#### Resolving Device Priorities
A key concept of OSEnergy is Device Priority.  When resolving relevant priorities, any nodes with conflicting / equivalent priority will be decided bases on the assigned source ID to resolve the conflict. The node with the higher assigned ID will be recognized as the higher priority.






<br><br>
#### J1939 convention for undefined values
It is convention to utilize all 1's for a value to indicate it is not available / not supported.   A 16 bit value containing  0xFFFF would indicate there is no data present.   RV-C and OSEnergy makes use of this convention to allow partial transmission of information, example:  When a Battery Master is sending battery voltage and current, but does not have a temperature sensor installed the temperature field will contain all 1's
















<br><br>
## Battery Master (BatMan)
The Battery Master is responsible for determining the batteries needs and communicating those needs (as well as status) via the CAN for subsequent use by charging sources associated with the same battery ID.

In addition to the common PGNs above, all BatMan devices must transmit the following DGNs/PGNs:
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1. 0x1FEC9 - DC-Status4  (Requested battery needs)

DC-Status4 is used to resolve which device is to be recognized as the Battery Master to be followed.  All devices should monitor for the presence of DC Status4 matching its battery ID.  The highest priority device transmitting DC Status4 for a given battery ID should be recognized as the Battery Master for that battery.  DC Status4 messages must be transmitted at least every 5000mS per the specification.  If a given node fails to transmit subsequent DC Status4 messages for two or more time slots (10,000mS) devices should look to resolve to a new Battery Master.

Once a device is acting as Battery Master and is transmitting DC Status4 messages, it should begin to also transmit these other battery status messages:
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2. 0x1FFFD - DC-Status 1 (Battery Current)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3. 0x1FEC9 - DC-Status 2 (Battery Temperature)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;4. 0x1FEC8 - DC-Status 5 (Battery Voltage - higher precision then DC-Status 1)

Note that this is not an restrictive list of messages which may originate from a Battery Master.   Depending on the capability of the Battery Master, other DC status messages may be sent (ala, SOC, SOH, etc from a proper BMS device)

<br><br>

At startup a Battery Master will perform needed J1939 address claim procedures and start monitoring the CAN.   If another Battery Master is detected for the same Battery ID and which has higher priority the device will place its self into a dormant role and not transmit any battery DC status CAN messages.  One exception to this is if a PGN Request is received, then even a dormant Battery Master will reply with the requested messages.

#### Power Reduction Options
DC Status4 is the principal message needed to establish and retain a Battery Master role for a given battery ID.  In order to conserve power, a Battery Master may optionally monitor for the presence of a charging device before sending out the other DC Status messages.  The presence of Charger Status - 1FFC7h messages associated with the same battery ID and a charger state other then 1 (Do not charge) may be used to trigger the resumption of all required DC Status messages.  Battery Masters should also respond to individual requests for messages.

Charging Devices will utilize DC-Status4 to resolve which Battery Master to follow, but will also require the presence of the other required DC-Status messages before validating a given Battery Master.



<br><br><br><br>
## Charging device (ChrDev)
Charging devices supply energy to the battery, as well as support any concurrent house loads.  Charging Devices must look for a valid Battery Master to provide direction on the battery needs, only upon failing to recognize a valid Battery Master - Charging Devices may fall back to local decisions as best as their capability allows.  In addition, a Charging Device must communicate its status and utilization as well as monitor for other charging devices associated with the same battery.  Charging Devices must support the following:
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1. 0x1FED0 - DC Disconnect Status  (Monitor for battery disconnect)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2. 0x1FECF - DC Disconnect Command
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3. 0x1FFC7 - Charger Status
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;4. 0x1FF9D - Charger Status 2


It is critical that all Charging Sources monitor for the DC disconnect messages and immediately stop charging in the event of a battery disconnect.

Other PGNs as supported by Charging Devices may be supported, specifically those involved in the configuration of charging source.  Note that J1939 requires a device to reply NAK in the event it is sent a PGN which it does not support.

<br><br>
#### Power Reduction Options
When a Charging Device is not active it may reduce its periodic messages to only the  Charger Status (0x1FFC7) message with the charger state set = 1 (Do not charge).  Or it may completely stop any periodic CAN message transmissions.  A Charging Device should however respond to any message requests, and resume all status messages upon entering an active charging state.








<br><br>
<br><br>
<br><br>
# Critical timing values
The following are suggested critical timing values and timeouts, in mS.  These are used when establishing Battery Masters as well as recognizing higher and lower priority charging sources.

```
#define REMOTE_CAN_MASTER_STABILITY 10000UL                             // We need to see 10 seconds of messaged from any potential 'remote' device wishing us to lock onto it.
#define REMOTE_CAN_MASTER_TIMEOUT     750UL                             // And if we do not hear SOMETHING from it for over 750mS second, we figure it is dead and need to start over.
                                                                        // (Should be broadcasting every 500mS max per spec)

#define REMOTE_CAN_LPCS_TIMEOUT       7500UL                             // If we do not hear from any Lower Priority Charging Sources . . figure WE are it.
#define REMOTE_CAN_HPUUCS_TIMEOUT    7500UL                             // Charger Status 1 (which has utilization %) comes every 5000mS..

#define RBM_REMASTER_IDLE_PERIOD    2 * REMOTE_CAN_MASTER_STABILITY     // When looking to establish a new RMB, wait at least 2x the stability timeout period.  To give some time for network errors to clear
#define RBM_REMASTER_IDLE_MULTI     50UL                                // Also, hold off a little longer based on your 'priority' level.  Let 'smarter' devices try 1st for king...

```



<br><br>
##Other CAN protocols (Raw J1939,  NMEA-2000^TM^, etc)
OSEnery is based on the J1939 protocol.  J1939 is very common and widely adopted in the transportation sector.  Many diesel engines provide for J1939 defined messages to communicate their operations status.  Devices may optional be designed to receive these messages, an example an Alternator Regulator may utilize J1939 - PGN 61444  to pick up the engine RPMs.  NMEA2000^TM^ is another common CAN protocol built upon J1939, and OSEnergy devices may find value in providing CAN messages which support status outputs in a compatible form.  However one needs to take caution that there is no conflicts created by the different standards, and that the presence of more than one protocol on the same physical CAN bus does not cause unintended errors.  In an extreme case, a CAN bridge or router may be needed.

