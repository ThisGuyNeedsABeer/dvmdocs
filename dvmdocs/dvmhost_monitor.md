### Hotspot Display Setup & Configuration
#### What is the role of the dvmhost_monitor?

* Display pertinent information on the I2C display attached to the MMDVM hat. Such information can include, but is not limited to:
	* Connect status or network addresses of adapters such as eth0, wlan0, tun0
	* FNE network addresses
	* System Identity Name
	* Site name
	* Last message received details
		* Radio ID
		* Talkgroup ID

#### Prerequisites

1. An Internet connect Raspberry Pi running Raspbian 12 (Bookworm) with a MMDVM hat flashed with the latest firmware from the DVMProject. 
	1. This guide will assume it is a Raspberry Pi Model 3B+
	2. This guide will assume that there are no `iptable` rules or other firewalls currently installed within Raspbian 12.
2. The latest `dvmhost_monitor` source from GitHub.
	1. [https://github.com/DVMProject/dvmhost_monitor](https://github.com/DVMProject/dvmhost_monitor)
3. A configured `dvmhost` with a valid `config.yml` that associates with an **FNE**.

#### Connect to the Raspberry Pi & Download The dvmhost_monitor Release

```bash
cd /opt/dvm/
git clone https://github.com/DVMProject/dvmhost_monitor.git
```

#### Install Node.js & NPM

```Shell
sudo apt update
sudo apt install nodejs npm -y
```

##### Verify the installation(s)

```Shell
node -v
npm -v
```

#### Install I2C Bus Node.js Module

```Bash
cd /opt/dvm/dvmhost_monitor/display/
sudo npm install i2c-bus
```

#### Configure Raspberry Pi I2C Interface

```Bash
sudo raspi-config
```

##### Enable I2C

Within `raspi-config`, navigate using the arrow keys and enter key: 

**`Interface Options` → `I2C` → `Yes (To Enable)`**

This step directly **enables the I2C hardware interface** on your Raspberry Pi, making it available for use by connected devices and software.
##### Verify I2C Device

```Bash
ls /dev/i2c-1
```

#### Update configuration in run.js


```Shell
cd /opt/dvm/dvmhost_monitor/display/
nano run.js
```

The `run.js` file contains the main script for the **dvmhost_monitor** and requires specific configuration updates for your environment. You'll need to edit this file to provide the correct paths and display names for your system.

Within **`run.js`**, locate and update the following variables:

```JavaScript
const wlaninterface = 'wlan0';
```

This line specifies the name of your **wireless local area network (WLAN) interface**. Common examples include **wlan0**. The monitor uses this to display WLAN-related information.

```JavaScript
const ethinterface = 'eth0';
```

This line specifies the name of your **Ethernet interface**. Common examples include **eth0**. The monitor uses this to display Ethernet-related information.

```JavaScript
const vpninterface = 'tun0';
```

This line specifies the name of your **Virtual Private Network (VPN) interface**. Common examples include **tun0** or **ppp0**. The monitor uses this to display VPN connection status if applicable.

```JavaScript
const sysName = '     NODE-NAME     ';
```

This line sets the **display name** for your system, which will be shown on the monitor's screen. Choose a descriptive name to easily identify your system.

#### Example `run.js`

```JavaScript
/**
 * Digital Voice Modem - Host Monitor
 * GPLv2 Open Source. Use is subject to license terms.
 * DO NOT ALTER OR REMOVE COPYRIGHT NOTICES OR THIS FILE HEADER.
 *
 * @package DVM / Host Monitor
 *
 */
/*
*   Copyright (C) 2022 Steven Jennison KD8RHO
*
*   This program is free software; you can redistribute it and/or modify
*   it under the terms of the GNU General Public License as published by
*   the Free Software Foundation; version 2 of the License.
*
*   This program is distributed in the hope that it will be useful,
*   but WITHOUT ANY WARRANTY; without even the implied warranty of
*   MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
*   GNU General Public License for more details.
*/
// CONFIGURATION

// Location of dvmhost YML file
const ymlLocation = "/opt/dvm/config.yml";

// WLAN interface name
const wlaninterface = "wlan0";

// Ethernet interface name
const ethinterface = "eth0";

// VPN Interface name
const vpninterface = null;

// System name
// const sysName = centerLine("MY HOST");

/**
 * DO NOT MODIFY BELOW THIS POINT
 */

 const i2c = require('i2c-bus'),
    i2cBus = i2c.openSync(1),
    font = require('oled-font-5x7'),
    yaml = require('js-yaml'),
    fs   = require('fs'),
    readline = require('readline'),
    rl = readline.createInterface({
        input: process.stdin,
        output: process.stdout,
        terminal: false
    });

var oled = oled = require('oled-i2c-bus');

class BigDecimal {
    // Configuration: constants
    static DECIMALS = 18; // number of decimals on all instances
    static ROUNDED = true; // numbers are truncated (false) or rounded (true)
    static SHIFT = BigInt("1" + "0".repeat(BigDecimal.DECIMALS)); // derived constant
    constructor(value) {
        if (value instanceof BigDecimal) return value;
        let [ints, decis] = String(value).split(".").concat("");
        this._n = BigInt(ints + decis.padEnd(BigDecimal.DECIMALS, "0")
                .slice(0, BigDecimal.DECIMALS))
            + BigInt(BigDecimal.ROUNDED && decis[BigDecimal.DECIMALS] >= "5");
    }
    static fromBigInt(bigint) {
        return Object.assign(Object.create(BigDecimal.prototype), { _n: bigint });
    }
    add(num) {
        return BigDecimal.fromBigInt(this._n + new BigDecimal(num)._n);
    }
    subtract(num) {
        return BigDecimal.fromBigInt(this._n - new BigDecimal(num)._n);
    }
    static _divRound(dividend, divisor) {
        return BigDecimal.fromBigInt(dividend / divisor
            + (BigDecimal.ROUNDED ? dividend  * 2n / divisor % 2n : 0n));
    }
    multiply(num) {
        return BigDecimal._divRound(this._n * new BigDecimal(num)._n, BigDecimal.SHIFT);
    }
    divide(num) {
        return BigDecimal._divRound(this._n * BigDecimal.SHIFT, new BigDecimal(num)._n);
    }
    toString() {
        const s = this._n.toString().padStart(BigDecimal.DECIMALS+1, "0");
        return s.slice(0, -BigDecimal.DECIMALS) + "." + s.slice(-BigDecimal.DECIMALS)
            .replace(/\.?0+$/, "");
    }
}

rl.on('line', function(line){
    console.log("DVMHOST:" + line);
    // [I: 2022-02-27 23:01:17.598    RX Frequency: 900974976Hz]
    if(line.includes("    RX Frequency: "))
    {
        // [I: 2022-02-27 23:01:17.598    RX Frequency: 900974976Hz]
        let frq = line.replace(/.*RX Frequency: /g,"");
        // [900974976]
        frq = frq.replace("Hz","");
        // in Hz
        // Round to clean up dvmhost interpretation
        let tempNum = Math.round(frq/1000);
        // Make bigdec
        let frequency = new BigDecimal(tempNum);
        // Divide by 1000 to get mhz
        frequency = frequency.divide(1000);
        // Ensure frequency is a string for manipulation
        let freqString = frequency.toString();

        // Check if there's a decimal point
        if (freqString.indexOf('.') === -1) {
            // No decimal point, so add .000
            freqString += '.000';
        } else {
            // Decimal point exists, check how many digits are after it
            const parts = freqString.split('.');
            const decimalPart = parts[1] || ""; // Get the part after the decimal, or empty string if none

            if (decimalPart.length === 0) {
                freqString += '000'; // e.g., "123." becomes "123.000"
            } else if (decimalPart.length === 1) {
                freqString += '00';  // e.g., "123.4" becomes "123.400"
            } else if (decimalPart.length === 2) {
                freqString += '0';   // e.g., "123.45" becomes "123.450"
            }
            // If decimalPart.length is 3 or more, it already has enough decimal places.
        }

        //rxFrq = centerLine("R:"+frequency.toString()+" MHz");
          rxFrq = centerLine("R: "+freqString.toString()+" MHz");
    }
    if(line.includes("    TX Frequency: "))
    {
        // [900974976Hz]
        let frq = line.replace(/.*TX Frequency: /g,"");
        // [900974976]
        frq = frq.replace("Hz","");
        // in Hz
        // Round to clean up dvmhost interpretation
        let tempNum = Math.round(frq/1000);
        // Make bigdec
        let frequency = new BigDecimal(tempNum);
        // Divide by 1000 to get mhz
        frequency = frequency.divide(1000);
        // Ensure frequency is a string for manipulation
        let freqString = frequency.toString();

        // Check if there's a decimal point
        if (freqString.indexOf('.') === -1) {
            // No decimal point, so add .000
            freqString += '.000';
        } else {
            // Decimal point exists, check how many digits are after it
            const parts = freqString.split('.');
            const decimalPart = parts[1] || ""; // Get the part after the decimal, or empty string if none

            if (decimalPart.length === 0) {
                freqString += '000'; // e.g., "123." becomes "123.000"
            } else if (decimalPart.length === 1) {
                freqString += '00';  // e.g., "123.4" becomes "123.400"
            } else if (decimalPart.length === 2) {
                freqString += '0';   // e.g., "123.45" becomes "123.450"
            }
            // If decimalPart.length is 3 or more, it already has enough decimal places.
        }

        //txFrq = centerLine("T:"+frequency.toString()+" MHz");
        txFrq = centerLine("T: "+freqString.toString()+" MHz");
    }
    if(line.includes("(HOST) Host is up and running"))
    {
        controlStatus = true;
        rptStatus = true;
    }
    if(line.includes("(NET) Connection to the master has timed out, retrying connection"))
    {
        LNK = false;
    }
    if(line.includes("(NET) Logged into the master successfully"))
    {
        LNK = true;
    }
    if(line.includes("P25 RF unit registration request from"))
    {
        var reg = line.replace(/.*from /g,"");
        alertLn = centerLine(`R:${reg}`)
    }
    if(line.includes("P25 RF group affiliation request from "))
    {
        // 121501 to TG  50101
        let data = line.replace(/.*from /g,"").split("to");
        let unit = data[0].trim();
        let tg = "TG"+data[1].replace("TG ","").trim();
        alertLn = centerLine(`A:${unit}>${tg}`);
    }
    if(line.includes("P25 RF group grant request from "))
    {
        // 121501 to TG  50101
        let data = line.replace(/.*from /g,"").split("to");
        let unit = data[0].trim();
        let tg = "TG"+data[1].replace("TG ","").trim();
        alertLn = centerLine(`C:${unit}>${tg}`);
    }
    if(line.includes("P25 Net network voice transmission from "))
    {
        // 121501 to TG  50101
        let data = line.replace(/.*from /g,"").split("to");
        let unit = data[0].trim();
        let tg = "TG"+data[1].replace("TG ","").trim();
        alertLn = centerLine(`ID:${unit}>${tg}`);
    }
})

var opts = {
    width: 128,
    height: 64,
    address: 0x3C
};

oled = new oled(i2cBus, opts);
var statusLn1 = "NET:OK LNK:!! CC:!!"
var statusLn2 =[
    "     OFFLINE       ", // [   CC lcn 00-0000  ]
    "     OFFLINE       ", // [   MODE: P25T-CC   ]
    "     OFFLINE       ", // [   T:123.456mhz    ]
    "     OFFLINE       ", // [     R:123.456     ]
    "WLAN OFFLINE       ", // [WL:111.111.111.111 ]
    "ETH OFFLINE        ", // [ET:111.111.111.111 ]
    "VPN OFFLINE        "  // [VP:111.111.111.111 ]
]
var alertLn =   "                   ";

var siteName =  ""
var rxFrq = "";
var txFrq = "";
var NET = false;
var LNK = false;
var control = false;
var controlStatus = false;
var rptStatus = false;
var FNE = "";

function getConfigData()
{
    try {
        const doc = yaml.load(fs.readFileSync(ymlLocation, 'utf8'));
        return doc;
    } catch (e) {
        console.log(e);
    }
}

var statusLineCurrent = 0
// Update Status Line
setInterval(()=>{
    var newLine = "";
    newLine+="NET:";
    if(NET)
    {
        newLine+="OK";
    }
    else
    {
        newLine+="!!";
    }
    newLine+=" LNK:";
    if(LNK)
    {
        newLine+="OK";
    }
    else
    {
        newLine+="!!";
    }
    if(control)
    {
        newLine+=" CC:";
        if(controlStatus)
        {
            newLine+="OK";
        }
        else
        {
            newLine+="!!";
        }
    }
    else
    {
        newLine+=" RPT:";
        if(rptStatus)
        {
            newLine+="OK";
        }
        else
        {
            newLine+="!!";
        }
    }
    statusLn1 = newLine;
},1000) // Dispay scroll speed in MS

function centerLine(name)
{
    while(name.length<19)
    {
        name = " "+name;
        if(name.length<19)
        {
            name=name+" ";
        }
    }
    return name;
}

// Update LCN/Mode
setInterval(()=>{
    let config = getConfigData();
    let mode = "";
    let status = "";
    if(config.protocols.p25.enable && config.protocols.p25.control.enable && config.protocols.p25.control.dedicated)
    {
        mode = "   MODE: P25T-CC   ";
        status = `   CC LCN ${config.system.config.channelId}-${config.system.config.channelNo}  `;
        control = true;
    }
    else if(config.protocols.dmr.enable && config.protocols.dmr.control.enable && config.protocols.dmr.control.dedicated)
    {
        mode = "   MODE: DMRT-CC   ";
        status = `   CC LCN ${config.system.config.channelId}-${config.system.config.channelNo}  `;
        control = true;
    }
    else if(config.protocols.p25.enable && config.protocols.dmr.enable)
    {
        mode = " MODE: DMR/P25 VC  ";
        status = `   VC LCN ${config.system.config.channelId}-${config.system.config.channelNo}  `;
    }
    else if(config.protocols.p25.enable)
    {
        mode = centerLine("MODE: P25");
        status = centerLine(`CH ID: ${config.system.config.channelId} CH No: ${config.system.config.channelNo}`);
    }
    else if(config.protocols.dmr.enable)
    {
        mode = "     MODE: DMR VC    ";
        status = `   VC LCN ${config.system.config.channelId}-${config.system.config.channelNo}  `;
    }
    statusLn2[0] = status;
    statusLn2[1] = mode;
    statusLn2[2] = txFrq;
    statusLn2[3] = rxFrq;
    siteName = centerLine(`${config.system.config.rfssId}-${config.system.config.siteId} ${config.system.identity}`);
    FNE = ` ${config.network.address}`
},5000)

// Update IP info
setInterval(()=>{
    const { networkInterfaces } = require('os');

    const nets = networkInterfaces();
    const results = Object.create(null); // Or just '{}', an empty object

    for (const name of Object.keys(nets)) {
        for (const net of nets[name]) {
            // Skip over non-IPv4 and internal (i.e. 127.0.0.1) addresses
            if (net.family === 'IPv4' && !net.internal) {
                if (!results[name]) {
                    results[name] = [];
                }
                results[name].push(net.address);
            }
        }
    }
    if(wlaninterface==null)
    {
        statusLn2[4] = centerLine(`${wlaninterface}: DISABLED`);
    }
    else
    {
        statusLn2[4] = centerLine(`${wlaninterface}: OFFLINE`);
        try{
            if(results[wlaninterface]!==undefined && results[wlaninterface].length>0) {
                statusLn2[4] = centerLine(`${wlaninterface}: ${results[wlaninterface][0]}`);
            }
        }
        catch {
            //
        }

    }
    if(ethinterface==null)
    {
        statusLn2[5] = centerLine(`${ethinterface}: DISABLED`);
    }
    else
    {
        statusLn2[5] = centerLine(`${ethinterface}: OFFLINE`);
        try{
            if(results[ethinterface]!==undefined && results[ethinterface].length>0) {
                statusLn2[5] = centerLine(`${ethinterface}: ${results[ethinterface][0]}`);
            }
        }
        catch {
            //
        }
    }
    if(vpninterface==null)
    {
        statusLn2[6] = centerLine(`VPN: DISABLED`);
    }
    else
    {
        statusLn2[6] = centerLine(`${vpninterface}: OFFLINE`);
        try{
            if(results[vpninterface]!==undefined && results[vpninterface].length>0) {
                statusLn2[6] = centerLine(`${vpninterface}: ${results[vpninterface][0]}`);
            }
        }
        catch {
            //
        }
    }
    require('dns').resolve('www.google.com', function(err) {
        if (err) {
            NET=false;
        } else {
            NET=true;
        }
    });
})

let inversion = 0;
let inverted = false;

setInterval(()=>{
    inversion++;
    if(inversion === 4)
    {
        inversion = 0;
        inverted = !inverted;
        oled.invertDisplay(inverted);
    }
    var status = "";
    if(statusLineCurrent>statusLn2.length-1)
    {
        statusLineCurrent = 0;
    }
    if(alertLn.length>0)
    {
        status = statusLn2[statusLineCurrent];
    }
    statusLineCurrent++;
    oled.clearDisplay();
    oled.setCursor(1, 1);
    oled.writeString(font, 1, `${statusLn1}`,1,false);
    oled.drawLine(1, 8, 128, 8, 1);
    oled.setCursor(1, 10);
    oled.writeString(font, 1, `${status}`, 1, false);
    //oled.setCursor(1, 20);
    //oled.writeString(font, 1, ``, 1, false);
    oled.setCursor(1, 30);
    oled.writeString(font, 1, `${alertLn}`, 1, false);
    //oled.setCursor(1, 30);
    //oled.writeString(font, 1, `${siteName}`, 1, false);
    oled.setCursor(1, 40);
    oled.writeString(font, 1, `${siteName}`, 1, false);
    oled.setCursor(1, 50);
    oled.writeString(font, 1, `FNE:${FNE}`, 1, false);
},1000)
```

My modified configuration makes some UI changes and introduces some additional code to make output look neater and more uniform. Notably, if you using frequency pairs that end in **000** such as **464.000** the script would originally display this as **RX: 464. mhz.** The code I added will display this as **RX: 464.000 MHz**. My configuration also adds an `FNE` variable and displays that at the very bottom of the screen. I made use of the `centerLine()` function as well to appease my brain so that when the details would scroll on the screen everything lined up nice and neat.

#### Verify configuration and launch dvmhost_monitor

```Shell
sudo node run.js
```

#### Configure the dvmhost_monitor to start at boot.

##### Open the System Cron Manager
```Shell
sudo crontab -e
```

##### Add the @reboot entry.

1. At the very end of the file append the following line. Ensure the paths referenced below match the paths on your Pi.

```Shell
@reboot /usr/bin/journalctl -f -u dvmhost.service | /usr/bin/grep --line-buffered dvmhost | /usr/bin/node /opt/dvm/dvmhost_monitor/display/run.js >> /var/log/dvmhost_display_cron.log 2>&1
```