### FNE Setup & Configuration
#### What is the role of the FNE?

- A network "core", this provides a central server for `dvmhost` instances to connect to and be networked with, allowing relay of traffic and other data between `dvmhost` instances and other `dvmfne` instances. [See configuration](https://github.com/DVMProject/dvmhost#dvmfne-configuration) to configure.
	- Source: [https://github.com/DVMProject/dvmhost](https://github.com/DVMProject/dvmhost)

#### Prerequisites

1. A workstation, server, or virtual machine running Linux to act as the FNE.
	1. This guide will showcase a remote VPS running Debian 12 (Bookworm).
2. An open port for the hotspot devices to connect to
	1. This guide is using the default port: 62031
	2. A site-to-site VPN could also be established from the hotspot to the VPS negating the need for an exposed port; this guide will not be covering that topology at this time.
3. The `dvmfne` binary from the latest `dvmhost` release for your CPU architecture.
	1. At the time of writing, **R04H31 2025-05-25** was the latest release.
		1. [dvmhost-2025-05-25-amd64.tar.gz](https://github.com/DVMProject/dvmhost/releases/download/2025-05-25/dvmhost-2025-05-25-amd64.tar.gz)

#### Connect to the VPS & Download The `dvmhost` Release For Your CPU Architecture

1. Download the archive from the [DVMProject GitHub]([https://github.com/DVMProject/dvmhost](https://github.com/DVMProject/dvmhost))
```Shell
ssh user@remotevps
cd /opt/
sudo wget https://github.com/DVMProject/dvmhost/releases/download/2025-05-25/dvmhost-2025-05-25-amd64.tar.gz
```

2. Extract & remove the archive
```Shell
sudo tar zxvf dvmhost-2025-05-25-amd64.tar.gz

# Optional step
sudo rm /opt/dvmhost-2025-05-25-amd64.tar.gz

```

#### Organize the `/opt/dvm` working directory

```Shell
cd /opt/dvm/
sudo mkdir /opt/dvm/examples/
sudo mv /opt/dvm/*.example.* /opt/dvm/examples/
```

#### Copy the base configuration file to the `/opt/dvm` working directory.

```Shell
sudo cp /opt/dvm/examples/fne-config.example.yml /opt/dvm/fne-config.yml
```

#### Define the settings for P25 communication in `fne-config.yml`

```Shell
sudo nano /opt/dvm/fne-config.yml
```

These dotted paths below represent nodes within the YAML syntax of `fne-config.yml`

1. **log.filePath:** /opt/dvm/log
2. **log.activityFilePath:** /opt/dvm/log
3. **master.peerId:** 1234567
4. **master.password:** 00AABBCCDDEEFF112233445566778899
5. **master.allowDMRTraffic:** false
6. **master.allowP25Traffic:** true
7. **master.allowNXDNTraffic:** false
8. **master.parrotGrantDemand:** true
9. **parrotOnlyToOrginiatingPeer:** true _This typo is present in the live config_
10. **master.talkgroup_rules.file:** /opt/dvm/rules/talkgroup_rules.yml
11. **system.radio_id.file:** /opt/dvm/rules/rid_acl.dat

#### Copy the following example configurations as live versions.

```Shell
sudo mkdir /opt/dvm/rules/
sudo cp /opt/dvm/examples/rid_acl.example.dat /opt/dvm/rules/rid_acl.dat
sudo cp /opt/dvm/examples/talkgroup_rules.example.yml /opt/dvm/rules/talkgroup_rules.yml
sudo cp /opt/dvm/examples/peer_list.example.dat /opt/dvm/rules/peer_list.dat
```

#### Populate radio_acl.dat

```Shell
sudo nano /opt/dvm/rules/radio_acl.dat
```

```Shell
#
# This file sets the valid Radio IDs allowed on a repeater.
#
# Entry Format: "RID,Enabled (1 = Enabled / 0 = Disabled),Optional Alias,Optional IP Address,<newline>"
#
#1234,1,RID Alias,IP Address,
5432,1,Name-Of-User-1,
9999,1,Name-Of-User-2,
4321,1,Name-Of-User-3,

[ensure-to-leave-a-new-line-at-the-end]
```

#### Populate peer_list.dat

```Shell
sudo nano /opt/dvm/rules/peer_list.dat
```

```Shell
#
# This file sets the valid peer IDs allowed on a FNE.
#
# Entry Format: "Peer ID,Peer Password,Peer Link (1 = Enabled / 0 = Disabled),Peer Alias (optional),Can Request Keys (1 = Enabled / 0 = Disabled),<newline>"
1234567,99887766554433221100FFEEDDCCBBAA,1,NAME-1,1,
7654321,99887766554433221100FFEEDDCCBBAA,1,NAME-2,1,

[ensure-to-leave-a-new-line-at-the-end]
#1234,,0,,1,
#5678,MYSECUREPASSWORD,0,,0,
#9876,MYSECUREPASSWORD,1,,0,
#5432,MYSECUREPASSWORD,,Peer Alias 1,0,
#1012,MYSECUREPASSWORD,1,Peer Alias 2,1,
```

#### Populate talkgroup_rules.yml

```Shell
sudo nano /opt/dvm/rules/talkgroup_rules.yml
```

These dotted paths below represent nodes within the YAML syntax of `talkgroup_rules.yml`

1. Primary Channel 
	1. **groupVoice.name:** MYTGNAME
	2. **groupVoice.alias:** MYTGNAME
	3. **groupVoice.config.active:** true
	4. **groupVoice.source.tgid:** 12345
2. Parrot Channel
	1. **groupVoice.name:** MYPARROTTG
	2. **groupVoice.alias:** MYPARROTTG
	3. **groupVoice.config.active:** true
	4. **groupVoice.config.parrot:** true
	5. **groupVoice.source.tgid:** 54321

#### Start the FNE daemon

```Shell
sudo /opt/dvm/start-dvm-fne.sh fne-config.yml
```

#### Log Entry Examples

##### Peer Connecting
```Shell
I: (NET) PEER 1234567 started login from, 192.168.100.100:32960
I: (NET) PEER 1234567 RPTL ACK, challenge response sent for login
I: (NET) PEER 1234567 RPTK ACK, completed the login exchange
I: (NET) PEER 1234567 RPTC ACK, completed the configuration exchange
I: (NET) PEER 1234567 reports identity [IDENTITY]
I: (NET) PEER 1234567 reports software DVM_R04H31
I: (NET) PEER 1234567 (IDENTITY) sending ACL list updates
U: 001234567 ( IDENTITY) M: 2025-06-06 01:16:05.775 (NET) PEER 1234567 RPTC ACK, master commanded alternate port for diagnostics and activity logging, remotePeerId = 9039148
U: 001234567 ( IDENTITY) I: 2025-06-06 01:16:10.717 (HOST) Host is running on a hotspot modem! Fixed mode is forced.
U: 001234567 ( IDENTITY) M: 2025-06-06 01:16:10.717 (HOST) [ OK ] host:rpc
U: 001234567 ( IDENTITY) M: 2025-06-06 01:16:10.718 (HOST) [ OK ] host:watchdog
U: 001234567 ( IDENTITY) M: 2025-06-06 01:16:10.718 (HOST) [ OK ] host:modem
U: 001234567 ( IDENTITY) M: 2025-06-06 01:16:10.719 (HOST) [ OK ] p25d:frame-r
U: 001234567 ( IDENTITY) M: 2025-06-06 01:16:10.719 (HOST) [ OK ] p25d:frame-w
U: 001234567 ( IDENTITY) I: 2025-06-06 01:16:10.720 (HOST) [ OK ] Host is up and running on Linux 6.12.25+rpt-rpi-v7 armv7l
U: 001234567 ( IDENTITY) M: 2025-06-06 01:16:10.720 (HOST) [ OK ] host:site-data
I: (NET) PEER 1234567 (IDENTITY) ping, pingsReceived = 1, lastPing = 1135288524, now = 1135298546
U: 001234567 ( IDENTITY) W: 2025-06-06 01:16:15.799 (NET) PEER 1234567 pong, time delay greater than 360ms, now = 1749186975799, server = 1749186988018, dt = 12219
```

##### Peer Disconnecting
```Shell
I: (NET) PEER 1234567 (IDENTITY) disconnected
```

##### Parrot Communication
```Shell
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:26:39.149 (RF) P25, HDU (Header Data Unit), dstId = 38002, algo = $80, kid = $0000
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:26:39.150 (RF) P25, LDU1 (Logical Link Data Unit 1), audio, mfId = $00 srcId = 3820, dstId = 38002, group = 1, emerg = 0, encrypt = 0, prio = 4, errs = 2/1233 (0.2%)
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:26:39.329 (RF) P25, LDU2 (Logical Link Data Unit 2), audio, algo = $80, kid = $0000, errs = 1/1233 (0.1%)
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:26:39.508 (RF) P25, LDU1 (Logical Link Data Unit 1), audio, mfId = $00 srcId = 3820, dstId = 38002, group = 1, emerg = 0, encrypt = 0, prio = 4, errs = 2/1233 (0.2%)
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:26:39.687 (RF) P25, LDU2 (Logical Link Data Unit 2), audio, algo = $80, kid = $0000, errs = 1/1233 (0.1%)
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:26:39.872 (RF) P25, LDU1 (Logical Link Data Unit 1), audio, mfId = $00 srcId = 3820, dstId = 38002, group = 1, emerg = 0, encrypt = 0, prio = 4, errs = 6/1233 (0.5%)
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:26:40.051 (RF) P25, LDU2 (Logical Link Data Unit 2), audio, algo = $80, kid = $0000, errs = 7/1233 (0.6%)
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:26:40.230 (RF) P25, LDU1 (Logical Link Data Unit 1), audio, mfId = $00 srcId = 3820, dstId = 38002, group = 1, emerg = 0, encrypt = 0, prio = 4, errs = 0/1233 (0.0%)
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:26:40.409 (RF) P25, LDU2 (Logical Link Data Unit 2), audio, algo = $80, kid = $0000, errs = 0/1233 (0.0%)
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:26:40.588 (RF) P25, LDU1 (Logical Link Data Unit 1), audio, mfId = $00 srcId = 3820, dstId = 38002, group = 1, emerg = 0, encrypt = 0, prio = 4, errs = 2/1233 (0.2%)
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:26:40.772 (RF) P25, LDU2 (Logical Link Data Unit 2), audio, algo = $80, kid = $0000, errs = 1/1233 (0.1%)
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:26:40.951 (RF) P25, LDU1 (Logical Link Data Unit 1), audio, mfId = $00 srcId = 3820, dstId = 38002, group = 1, emerg = 0, encrypt = 0, prio = 4, errs = 1/1233 (0.1%)
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:26:41.130 (RF) P25, LDU2 (Logical Link Data Unit 2), audio, algo = $80, kid = $0000, errs = 2/1233 (0.2%)
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:26:41.309 (RF) P25, LDU1 (Logical Link Data Unit 1), audio, mfId = $00 srcId = 3820, dstId = 38002, group = 1, emerg = 0, encrypt = 0, prio = 4, errs = 0/1233 (0.0%)
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:26:41.488 (RF) P25, LDU2 (Logical Link Data Unit 2), audio, algo = $80, kid = $0000, errs = 5/1233 (0.4%)
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:26:41.668 (RF) P25, LDU1 (Logical Link Data Unit 1), audio, mfId = $00 srcId = 3820, dstId = 38002, group = 1, emerg = 0, encrypt = 0, prio = 4, errs = 1/1233 (0.1%)
M: (NET) P25, Parrot Playback will Start, peer = 9039148, srcId = 3820
M: (NET) P25, Call End, peer = 9039148, srcId = 3820, dstId = 38002, duration = 2, streamId = 3063698845, external = 0
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:26:41.673 (RF) P25, TDU (Simple Terminator Data Unit), total frames: 15, bits: 18496, undecodable LC: 0, errors: 31, BER: 0.1676%
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:26:41.707 (RF) P25, TDULC (Terminator Data Unit with Link Control), CALL_TERM (Call Termination), srcId = 3820, dstId = 38002
I: (NET) PEER 9039148 (IDENTITY) ping, pingsReceived = 2860, lastPing = 1200712208, now = 1200713943
M: (NET) P25, Parrot Grant Demand, peer = 9039148, srcId = 3820, dstId = 38002
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:26:43.883 (NET) P25, HDU (Header Data Unit), dstId = 38002, algo = $80, kid = $0000
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:26:43.884 (NET) P25, LDU1 (Logical Link Data Unit 1) audio, mfId = $00, srcId = 3820, dstId = 38002, group = 1, emerg = 0, encrypt = 0, prio = 4, sysId = $001, netId = $BB800
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:26:43.884 (NET) P25, LDU2 (Logical Link Data Unit 2) audio, algo = $80, kid = $0000
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:26:44.058 (NET) P25, LDU1 (Logical Link Data Unit 1) audio, mfId = $00, srcId = 3820, dstId = 38002, group = 1, emerg = 0, encrypt = 0, prio = 4, sysId = $001, netId = $BB800
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:26:44.242 (NET) P25, LDU2 (Logical Link Data Unit 2) audio, algo = $80, kid = $0000
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:26:44.467 (NET) P25, LDU1 (Logical Link Data Unit 1) audio, mfId = $00, srcId = 3820, dstId = 38002, group = 1, emerg = 0, encrypt = 0, prio = 4, sysId = $001, netId = $BB800
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:26:44.595 (NET) P25, LDU2 (Logical Link Data Unit 2) audio, algo = $80, kid = $0000
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:26:44.774 (NET) P25, LDU1 (Logical Link Data Unit 1) audio, mfId = $00, srcId = 3820, dstId = 38002, group = 1, emerg = 0, encrypt = 0, prio = 4, sysId = $001, netId = $BB800
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:26:44.959 (NET) P25, LDU2 (Logical Link Data Unit 2) audio, algo = $80, kid = $0000
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:26:45.143 (NET) P25, LDU1 (Logical Link Data Unit 1) audio, mfId = $00, srcId = 3820, dstId = 38002, group = 1, emerg = 0, encrypt = 0, prio = 4, sysId = $001, netId = $BB800
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:26:45.317 (NET) P25, LDU2 (Logical Link Data Unit 2) audio, algo = $80, kid = $0000
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:26:45.501 (NET) P25, LDU1 (Logical Link Data Unit 1) audio, mfId = $00, srcId = 3820, dstId = 38002, group = 1, emerg = 0, encrypt = 0, prio = 4, sysId = $001, netId = $BB800
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:26:45.680 (NET) P25, LDU2 (Logical Link Data Unit 2) audio, algo = $80, kid = $0000
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:26:45.879 (NET) P25, LDU1 (Logical Link Data Unit 1) audio, mfId = $00, srcId = 3820, dstId = 38002, group = 1, emerg = 0, encrypt = 0, prio = 4, sysId = $001, netId = $BB800
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:26:46.037 (NET) P25, LDU2 (Logical Link Data Unit 2) audio, algo = $80, kid = $0000
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:26:46.409 (NET) P25, LDU1 (Logical Link Data Unit 1) audio, mfId = $00, srcId = 3820, dstId = 38002, group = 1, emerg = 0, encrypt = 0, prio = 4, sysId = $001, netId = $BB800
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:26:46.415 (NET) P25, TDU (Simple Terminator Data Unit), srcId = 3820
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:26:46.547 (RF) P25, TDULC (Terminator Data Unit with Link Control), CALL_TERM (Call Termination), srcId = 3820, dstId = 38002
E: (NET) PEER 9000990, retrying master login, remotePeerId = 0
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:26:48.406 (NET) talkgroup hang has expired, lastDstId = 38002
```

##### Talkgroup Communication
```Shell
[COMING SOON]
```

##### Talkgroup Communication (Encrypted)
```Shell
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:28:13.439 (RF) P25, HDU (Header Data Unit), dstId = 38001, algo = $84, kid = $0003
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:28:13.440 (RF) P25, LDU1 (Logical Link Data Unit 1), audio, mfId = $00 srcId = 3820, dstId = 38001, group = 1, emerg = 0, encrypt = 1, prio = 4, errs = 0/1233 (0.0%)
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:28:13.618 (RF) P25, LDU2 (Logical Link Data Unit 2), audio, algo = $84, kid = $0003, errs = 1/1233 (0.1%)
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:28:13.619 (RF) P25, LDU2 (Logical Link Data Unit 2), Enc Sync, MI = C1 7C 1E 45 F0 F6 94 44 00
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:28:13.798 (RF) P25, LDU1 (Logical Link Data Unit 1), audio, mfId = $00 srcId = 3820, dstId = 38001, group = 1, emerg = 0, encrypt = 1, prio = 4, errs = 0/1233 (0.0%)
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:28:13.977 (RF) P25, LDU2 (Logical Link Data Unit 2), audio, algo = $84, kid = $0003, errs = 3/1233 (0.2%)
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:28:13.978 (RF) P25, LDU2 (Logical Link Data Unit 2), Enc Sync, MI = 9C 03 CE 4D 6C AD AA 96 00
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:28:14.157 (RF) P25, LDU1 (Logical Link Data Unit 1), audio, mfId = $00 srcId = 3820, dstId = 38001, group = 1, emerg = 0, encrypt = 1, prio = 4, errs = 2/1233 (0.2%)
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:28:14.341 (RF) P25, LDU2 (Logical Link Data Unit 2), Enc Sync, MI = 20 12 E1 85 ED 94 20 96 00
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:28:14.341 (RF) P25, LDU2 (Logical Link Data Unit 2), audio, algo = $84, kid = $0003, errs = 8/1233 (0.6%)
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:28:14.520 (RF) P25, LDU1 (Logical Link Data Unit 1), audio, mfId = $00 srcId = 3820, dstId = 38001, group = 1, emerg = 0, encrypt = 1, prio = 4, errs = 2/1233 (0.2%)
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:28:14.699 (RF) P25, LDU2 (Logical Link Data Unit 2), audio, algo = $84, kid = $0003, errs = 3/1233 (0.2%)
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:28:14.700 (RF) P25, LDU2 (Logical Link Data Unit 2), Enc Sync, MI = C2 51 17 3A E6 C1 C5 0A 00
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:28:14.878 (RF) P25, LDU1 (Logical Link Data Unit 1), audio, mfId = $00 srcId = 3820, dstId = 38001, group = 1, emerg = 0, encrypt = 1, prio = 4, errs = 4/1233 (0.3%)
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:28:15.057 (RF) P25, LDU2 (Logical Link Data Unit 2), audio, algo = $84, kid = $0003, errs = 2/1233 (0.2%)
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:28:15.058 (RF) P25, LDU2 (Logical Link Data Unit 2), Enc Sync, MI = 2E 49 2A 15 DC 59 B6 A6 00
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:28:15.236 (RF) P25, LDU1 (Logical Link Data Unit 1), audio, mfId = $00 srcId = 3820, dstId = 38001, group = 1, emerg = 0, encrypt = 1, prio = 4, errs = 3/1233 (0.2%)
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:28:15.420 (RF) P25, LDU2 (Logical Link Data Unit 2), audio, algo = $84, kid = $0003, errs = 1/1233 (0.1%)
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:28:15.421 (RF) P25, LDU2 (Logical Link Data Unit 2), Enc Sync, MI = 8E 30 5D 4F 91 09 1E 68 00
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:28:15.600 (RF) P25, LDU1 (Logical Link Data Unit 1), audio, mfId = $00 srcId = 3820, dstId = 38001, group = 1, emerg = 0, encrypt = 1, prio = 4, errs = 3/1233 (0.2%)
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:28:15.779 (RF) P25, LDU2 (Logical Link Data Unit 2), audio, algo = $84, kid = $0003, errs = 4/1233 (0.3%)
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:28:15.779 (RF) P25, LDU2 (Logical Link Data Unit 2), Enc Sync, MI = E0 79 45 C2 8B A2 AC 2D 00
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:28:15.958 (RF) P25, LDU1 (Logical Link Data Unit 1), audio, mfId = $00 srcId = 3820, dstId = 38001, group = 1, emerg = 0, encrypt = 1, prio = 4, errs = 5/1233 (0.4%)
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:28:16.137 (RF) P25, LDU2 (Logical Link Data Unit 2), audio, algo = $84, kid = $0003, errs = 8/1233 (0.6%)
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:28:16.138 (RF) P25, LDU2 (Logical Link Data Unit 2), Enc Sync, MI = 50 B7 D2 37 12 4B E3 7A 00
M: (NET) P25, Call End, peer = 9039148, srcId = 3820, dstId = 38001, duration = 2, streamId = 746414736, external = 0
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:28:16.143 (RF) P25, TDU (Simple Terminator Data Unit), total frames: 16, bits: 19729, undecodable LC: 0, errors: 49, BER: 0.2484%
U: PEER-ID ( IDENTITY) M: 2025-06-06 19:28:16.151 (RF) P25, TDULC (Terminator Data Unit with Link Control), CALL_TERM (Call Termination), srcId = 3820, dstId = 38001
```