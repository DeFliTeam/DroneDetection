# DroneDetection 

**A Project Contained Witin the DEFLI NETWORKS decentralized infrastructure to utilize low cost SDR's and bespoke processing technologies to identify drones using RF monitoring** 

**Hardware** 
Base Boards: Raspberry Pi4 or Jetson Nano. Alternatively most mini-PC's can be utilized. 
Wifi Receiver: One or both of Intel 8265 Card or Panda Wireless Wifi Card (USB) 

**Radio Receiver Architecture** 
Wi-Fi signals can be strongly attenuated with distance and the presence of obstacles. To ensure drone detection from significant distances, the detection system should detect weak Wi-Fi signals. An RF amplification chain is then required. 

If using the Panda WiFi Card the architecture is: 

Panda > 2437MHZ Filter > LNA > Rx 

If using the Intel WiFi Card the architecture is: 

Intel Connection 1 > 2400 - 2484MHZ Filter > LNA > Rx 
Intel Connection 2 > 5725 - 5875MHZ Filer > LNA > Rx 

If using both boards to monitor all WiFi Frequencies the architecture is: 

Panda > 2437MHZ Filter > Two Way RF Spliiter (1) > LNA (1) > Rx 
Intel Connection 1 > 2400 - 2484MHZ Filter > Two Way RF Splitter (1) > LNA (1) > Rx 
Intel Connection 2 > 5725 - 5875MHZ Filer > LNA > Rx 

**WiFi Packet Capture Tools** 

Aircrack-ng (https://www.aircrack-ng.org/) 
Wireshark (https://www.wireshark.org/) 
Kismet (https://www.kismetwireless.net/)  

**Identifying WiFi Drone Nodes** 

Another utility of the “airckrack-ng” tool is the use of the “airodump-ng” program to capture raw IEEE Wi-Fi 802.11 frames

**Methodology**  

To begin with, we put the Panda and the Intel wireless cards into monitor mode. This is achieved using the “aircrack-ng” tool with the following Linux command: “sudo airmon-ng start $<name_wireless_card>”. Once in monitor mode, the cards capture all traffic. To focus on analyzing specific packets, a series of filters is applied. 

*Decoding Wi-Fi RDID Packet* 

The RDID is broadcast unencrypted via the IEEE 802.11 wireless management flag as a beacon frame with an OUI of “6a:5c:35”. To isolate Wi-Fi beacon frames from the captured traffic, the command “wlan.fc.type_subtype == 0x08” is used. In addition, filtering is achieved with the BSSID command “wlan.bssid[0:3] == 60:60:1F”. To confirm the presence of the RDID, we check whether the OUI tag “6a:5c:35” is present in the decoded packet 

The image below displays the Mavic 2 Pro RDID packet capture (PCAP) file after applying these Wireshark commands. 

<a href="https://imgbb.com/"><img src="https://i.ibb.co/crSt8YN/Screenshot-from-2023-10-09-14-27-50.png" alt="Screenshot-from-2023-10-09-14-27-50" border="0"></a>

The PCAP contains information about the drone’s location, altitude, speed, heading, and other related parameters. The packet has been decoded and contains various tags that describe the different parameters. The Radiotap header provides information about the captured packet, such as the interface, ID, and length. The IEEE 802.11 beacon frame indicates that this packet contains wireless management information. The fixed parameters include the timestamp, beacon interval, and capabilities information. The tagged parameters provide additional details about the drone, such as the service set identifier (SSID)(DJI-1581###############), latitude, longitude, altitude above mean sea level (AMSL), altitude above ground level (AGL), latitude takeoff, longitude takeoff, horizontal speed, and heading. Furthermore, there are vendor-specific tags that provide information about the drone manufacturer and serial number.

*Decoding Enhanced Wi-Fi DJI Drone ID Packet* 

DJI Wi-Fi drones include a standard packet addition known as a DJI Drone ID, which is broadcast through an OUI tag of “26:37:12” in the IEEE 802.11 beacon. Two packet types are used; packets with a subcommand of 0 × 10 include flight telemetry and location, while packets with a subcommand of 0 × 11 include user-entered information about the drone and the flight. 
These packets are alternately sent down the Wi-Fi link every 200 ms. Thanks to the OUI tag and the knowledge of the subcommands, Kismet can detect a DJI Drone ID packet data and classify it as “uav.device”. Kismet allows tracking using a list of drone MAC addresses by retrieving a UAV’s telemetry history. 

Knowing that a DJI enhanced Wi-Fi Drone ID occupies a 5 MHz bandwidth, Wi-Fi channels can be scanned with Kismet from 1 to 177 in the 2.4 GHz and 5.8 GHz frequency bands using the following bash command 

kismet -c  wlan0mon:name=channel0, ht_channels=false, vht_channels=false 
channels=\"1W5, 2W5, 3W5, 4W5, 5W5, 6W5, 7W5, 8W5, 9W5, 10W5, 11W5, 12W5, 13W5, 14W5, 140W5, 149W5, 153W5, 157W5, 161W5, 165W5, 169W5, 173W5, 177W5\" 

By using the Kismet_Rest application programming interface (API) on Python, it is possible to retrieve decoded DJI Drone ID packet data from the Kismet server in the form of a JavaScript object notation (JSON) file as shown in the image below 

<a href="https://imgbb.com/"><img src="https://i.ibb.co/ft0b7nm/Screenshot-from-2023-10-09-14-43-01.png" alt="Screenshot-from-2023-10-09-14-43-01" border="0"></a>

This packet contains information about a DJI drone, including its serial number, model, and frequency histogram usage. Additionally, telemetry data are included such as the drone’s pitch, yaw, and roll, as well as its speed and altitude. It also includes a timestamp indicating when the telemetry data were recorded. This information can be used to track and analyze drone movements and behavior.
The Python code connects to a Kismet server using the kismet_rest API to retrieve information about a wireless detected device with a specific MAC address. As shown in the flowchart below, the code performs several steps. First, the code imports the required kismet_rest module. Then, it creates a KismetConnector instance and establishes a connection to the Kismet server. Next, the code defines a list of MAC address masks, stored in the mac_list variable, for the devices that need to be detected. If a device is detected, the code retrieves information such as the device name, full MAC address, and manufacturer using the kismet_rest.Devices.by_mac function and providing the MAC list. Additionally, the code employs the kismet.device.base.seenby function to access multiple dictionary attributes. 

<a href="https://imgbb.com/"><img src="https://i.ibb.co/17WjkW9/Screenshot-from-2023-10-09-14-44-33.png" alt="Screenshot-from-2023-10-09-14-44-33" border="0"></a> 

