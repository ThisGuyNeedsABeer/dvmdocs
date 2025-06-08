
### Raspberry Pi Setup & Configuration
#### **Required Hardware**

* Raspberry Pi: [Raspberry Pi 3 Model B+ (1GB)](https://www.amazon.com/dp/B0BNJPL4MW) ($47.02)
* Power Supply: [GeeekPi 20W 5V 4A Power Supply for Raspberry Pi 4/Orange Pi 5/5B, UL Listed Type C Power Supply with ON/Off Switch](https://www.amazon.com/dp/B0BMGJNSVS) ($10.99)
* Micro SDHC Card: [SanDisk Ultra SDSQUNS-016G-GN3MN 16GB 80MB/s UHS-I Class 10 microSDHC Card](https://www.amazon.com/dp/B074B4P7KD) ($7.50)
* Raspberry Pi Case: [Duplex MMDVM & Raspberry Pi4 & 3B+ | Room for 0.96” Screen | Zebra DRPi-1S (Black Ice)](https://www.amazon.com/dp/B0D6J9TD4P) ($29.99)
* MMDVM Duplex Board: [NEW Soldered MMDVM DUPLEX hotspot Support P25 DMR YSF NXDN DMR SLOT 1+ SLOT 2 for Raspberry pi + OLED](https://www.aliexpress.us/item/3256806512480552.html) ($30.07)

**Total:** $125.57 (Plus Tax & Shipping)

#### **Software Requirements**

* Operating System: Raspbian 12 (x86/32-bit)
	* Installed via: [Raspberry Pi Imager](https://www.raspberrypi.com/software/)
* Internet connection
* SSH Enabled

#### Connect to & Configure the Raspberry Pi via SSH

```Shell
ssh user@raspberrypi
sudo systemctl disable bluetooth.service serial-getty@ttyAMA0.service
sudo systemctl mask serial-getty@ttyAMA0.service
grep '^dtoverlay=disable-bt' /boot/firmware/config.txt || echo 'dtoverlay=disable-bt' | sudo tee -a /boot/firmware/config.txt
sudo sed -i 's/^console=serial0,115200 *//' /boot/firmware/cmdline.txt
sudo reboot
```

**Note:** These steps were taken from the **dvmhost** GitHub: [https://github.com/DVMProject/dvmhost ](https://github.com/DVMProject/dvmhost )under the **Raspberry Pi Preparation Notes** section and updated to accommodate the current Raspbian 12 file structure. 

* `/boot/config.txt` -> `/boot/firmware/config.txt`
* `/boot/cmdline.txt` -> `/boot/firmware/cmdline.txt`

#### Prepare the Raspberry Pi for the MMDVM board.

```Shell
ssh user@raspberrypi
sudo shutdown now
```

1. Affix the **MMDVM_HS_HAT_DUPLEX** board to the **GPIO pins** on the **Raspberry Pi** so that the new board covers the **Raspberry Pi**. 
2. Screw on the provided antennas.
	1. **Note:** There is no designated antenna for transmit (TX) or receive (RX).

#### Power on the Raspberry Pi & Flash the DVM Firmware

1. Clone the `hsflash` GitHub repository

```Shell
git clone https://github.com/ThisGuyNeedsABeer/hsflash.git
```

2. Compile the source and make `hsflash` executable
```
cd hsflash/
gcc hsflash.c -o hsflash
chmod +x hsflash
```

3. Download the `dvm-firmware-hs-hat-dual.bin` firmware to the `hsflash` directory
```Shell
wget https://github.com/DVMProject/dvmfirmware-hs/releases/download/2025-03-21/dvm-firmware-hs-hat-dual.bin -O dvm-firmware-hs-hat-dual.bin
```

4. Flash the firmware, if `stm32flash` is missing, `hsflash` will attempt to install it.
```Shell
██╗  ██╗███████╗   ███████╗██╗      █████╗ ███████╗██╗  ██╗
██║  ██║██╔════╝   ██╔════╝██║     ██╔══██╗██╔════╝██║  ██║
███████║███████╗   █████╗  ██║     ███████║███████╗███████║
██╔══██║╚════██║   ██╔══╝  ██║     ██╔══██║╚════██║██╔══██║
██║  ██║███████║██╗██║     ███████╗██║  ██║███████║██║  ██║
╚═╝  ╚═╝╚══════╝╚═╝╚═╝     ╚══════╝╚═╝  ╚═╝╚══════╝╚═╝  ╚═╝
                  * HOTSPOT FLASH EZ *

[WARN] stm32flash is not installed.
Install stm32flash now? [Y/n]: y

... apt output omitted...

Enter BOOT0 GPIO pin [default 20]: 20
Enter NRST GPIO pin [default 21]: 21
Enter firmware filename (e.g. dvm-firmware-hs-hat-dual.bin): dvm-firmware-hs-hat-dual.bin
Enter serial port [default /dev/ttyAMA0]: /dev/ttyAMA0

===== Configuration Summary =====
BOOT0 GPIO      : 20
NRST GPIO       : 21
BIN_FILE        : dvm-firmware-hs-hat-dual.bin
SERIAL_PORT     : /dev/ttyAMA0
stm32flash      : found
=================================

Proceed with flashing? [Y/n]: y
[INFO] Proceeding with firmware flash setup...
[INFO] Setting BOOT0 high (GPIO20)...
[ raspi-gpio is deprecated - try `pinctrl` instead ]
[INFO] Asserting reset (GPIO21 low)...
[INFO] Releasing reset (GPIO21 high)...
[INFO] Flashing firmware...
stm32flash 0.7

http://stm32flash.sourceforge.net/

Using Parser : Raw BINARY
Size         : 57248
Interface serial_posix: 57600 8E1
Version      : 0x22
Option 1     : 0x00
Option 2     : 0x00
Device ID    : 0x0410 (STM32F10xxx Medium-density)
- RAM        : Up to 20KiB  (512b reserved by bootloader)
- Flash      : Up to 128KiB (size first sector: 4x1024)
- Option RAM : 16b
- System RAM : 2KiB
Write to memory
Erasing memory
Wrote and verified address 0x0800dfa0 (100.00%) Done.

Resetting device...
Reset done.

[INFO] Setting BOOT0 low (GPIO20)...
[INFO] Resetting MCU to boot from flash...
[SUCCESS] Flashing complete and STM32 restarted successfully.
```

5. Reboot to complete the process upon a successful flash.
```Shell
sudo reboot
```