### Hotspot Setup & Configuration
#### What is the role of the dvmhost?

* Host software that connects to the DVM modems (both air interface for repeater and hotspot or P25 DFSI for commerical P25 hardware) and is the primary data processing application for digital modes. [See configuration](https://github.com/DVMProject/dvmhost#dvmhost-configuration) to configure and calibrate.
	* Source: [https://github.com/DVMProject/dvmhost](https://github.com/DVMProject/dvmhost)

#### Prerequisites

1. An Internet connect Raspberry Pi running Raspbian 12 (Bookworm) with a MMDVM hat flashed with the latest firmware from the DVMProject. 
	1. This guide will assume it is a Raspberry Pi Model 3B+
	2. This guide will assume that there are no `iptable` rules or other firewalls currently installed within Raspbian 12.
2. The `dvmhost` binary from the latest `dvmhost` release for your CPU architecture.
	2. At the time of writing, **R04H31 2025-05-25** was the latest release.
		1. [dvmhost-2025-05-25-amd64.tar.gz](https://github.com/DVMProject/dvmhost/releases/download/2025-05-25/dvmhost-2025-05-25-amd64.tar.gz)

#### Connect to the Raspberry Pi & Download The dvmhost Release For Your CPU Architecture

1. Download the archive from the [DVMPRoject GitHub]([https://github.com/DVMProject/dvmhost](https://github.com/DVMProject/dvmhost))
```Shell
ssh user@raspberrypi
cd /opt/
sudo wget https://github.com/DVMProject/dvmhost/releases/download/2025-05-25/dvmhost-2025-05-25-amd64.tar.gz
```

2. Extract & remove the archive
```Shell
sudo tar zxvf dvmhost-2025-05-25-amd64.tar.gz

# Optional step
sudo rm /opt/dvmhost-2025-05-25-arm.tar.gz

```

#### Organize the working directory

```Shell
cd /opt/dvm/
sudo mkdir /opt/dvm/examples/
sudo mv /opt/dvm/*.example.* /opt/dvm/examples/
```
#### Copy the following example configurations as live versions.

```Shell
sudo mkdir /opt/dvm/rules/
sudo cp /opt/dvm/examples/rid_acl.example.dat /opt/dvm/rules/rid_acl.dat
sudo cp /opt/dvm/examples/talkgroup_rules.example.yml /opt/dvm/rules/talkgroup_rules.yml
sudo cp /opt/dvm/examples/peer_list.example.dat /opt/dvm/rules/peer_list.dat
```
#### Review & Populate the iden_table.dat

The example `iden_table.dat` includes the **UHF R2** range, if you are using frequencies that fall into this range, there is no additional calculation needed and you will use `2` as the `channelId` later in `config.yml`.

If you need to calculate a frequency pair, perform the following steps:

1. Navigate to: [https://dvmproject.io/iden-calc-web/](https://dvmproject.io/iden-calc-web/)
2. Enter the appropriate values into each of the following fields:
	1. **Downlink Frequency (MHz)**
		1. This field represents the frequency on which the hotspot will  receive transmissions. 
	2.  **Base Frequency (MHz)**
		2. This field defines a starting or reference frequency for a specific block or range of channels within the P25 system's frequency plan. It is used in a formula, along with the **Channel ID** and **Spacing**, to calculate the specific frequency for a given channel.
	3. **Spacing (kHz)**
		1. This field indicates the separation between the center frequencies of adjacent channels, often called the channel step or channel raster. It is typically measured in kilohertz (kHz) (e.g., 12.5 kHz, 6.25 kHz for P25). This value is crucial for calculating the exact frequency of a channel based on its ID and the base frequency.
	4. **Offset (MHz)**
		1. This field most commonly refers to the standard frequency difference between the **uplink** and **downlink frequencies** for a repeater channel pair in a specific band (e.g., +5 MHz or -5 MHz). It is measured in megahertz (MHz) and ensures that radios transmit on one frequency and receive on another, allowing for simultaneous transmission and reception through a repeater. 
	5. **Channel ID (dec)**
		1. This field is a numerical identifier for a specific radio channel within the P25 system. This decimal value is typically used in a formula with the **Base Frequency** and **Spacing** to determine the precise operational frequency (often the **downlink**) for that channel.
	6. **Uplink Frequency (MHz)**
		1. Calculated automatically.

#### Copy the base configuration file to the  working directory.

```Shell
sudo cp /opt/dvm/examples/config.example.yml /opt/dvm/config.yml
```

#### Define the settings for P25 communication in config.yml

```Shell
sudo nano /opt/dvm/config.yml
```

These dotted paths below represent nodes within the YAML syntax of `fne-config.yml`

1. **network.address:** [ADDRESS-OF-FNE]
2. **network.id:** [6-DIGIT-ID-OF-HOTSPOT]
3. **network.password:** [PASSWORD-FROM-FNE-CONFIG.YML-ON-FNE]
4. **system.identity:** MYHOTSPOT
5. **protocols.dmr.enable:** false
6. **protocols.nxdn.enable:** false
7. **system.modem.protocol.type:** uart
8. **system.modem.protocol.mode:** air
9. **system.modem.protocol.uart.port:** /dev/ttyAMA0
10. **system.modem.protocol.uart.speed:** 115200
11. **system.config.channelId:** 2 
	1. If you defined your own `iden_table.dat` use your associated `channelId`
12. **system.config.channelNo:** [VALUE-CALCULATED-FROM-IDEN-CALC-WEB]
13. **system.config.voiceChNo.channelId:** 2
	1. If you defined your own `iden_table.dat` use your associated `channelId`
14. **system.config.voiceChNo.channelNo:** [VALUE-CALCULATED-FROM-IDEN-CALC-WEB]
15. **systtem.config.iden_table.file:** /opt/dvm/rules/iden_table.dat
16. **system.config.radio_acl.file:** /opt/dvm/rules/radio_acl.dat
17. **system.config.talkgroup_id.file:** /opt/dvm/rules/talkgroup_rules.yml

#### Optional: Enable ACLs based on radio ID or talkgroup ID

```Shell
sudo nano /opt/dvm/config.yml
```

1. Radio ID
	1. system.config.radio_id.acl: true
2. Talkgroup ID
	1. system.config.talkgroup_id.acl: true

#### Verify settings in the TUI

**Note:** You can use your mouse in the TUI to click through the menu's and visually confirm the settings set in `config.yml`

```Shell
sudo /opt/dvm/bin/dvmhost -c /opt/dvm/config.yml --setup
```

##### Connect to the modem

1. Press F8
	1. The modem should initialize and connect successfully.
	2. This is where alignment would take place if necessary. Alignment is generally needed if the Hotspot does not activate when receiving a transmission from a subscriber radio. At this time this guide will not cover offsets or alignments on the hotspot.
2. Press F2
	1. This will save your current settings and "flatten" your `config.yml` file removing any comments.
3. Press F3
	1. This will take you back to the console.

#### Start the dvmhost process as a daemon.

```Shell
/opt/dvm/start-dvm.sh /opt/dvm/config.yml
```

If the start is successful, you should see an entry as denoted in **Log Entry Examples** subsection, **Peer Connecting.**

#### Configure dvmhost to start as a system service.

```Shell
sudo nano /etc/systemd/system/dvmhost.service
```

##### dvmhost.service Example

```Shell
[Unit]
Description=dvmhost
After=network.target

[Service]
ExecStart=/opt/dvm/bin/dvmhost -c /opt/dvm/config.yml -f
User=root
Type=forking
Restart=on-abnormal
TimeoutSec=infinity

[Install]
WantedBy=multi-user.target
```

##### Enable and start the dvmhost service
```Shell
sudo systemctl daemon-reload
sudo systemctl enable dvmhost.service
sudo systemctl start dvmhost.service
```