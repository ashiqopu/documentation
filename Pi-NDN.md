## Introduction
The NFD and it's related tools and software packages can be installed on platforms mentioned in the official **[NFD](http://named-data.net/doc/NFD/current/)** documentation. However, a successful installation on constrained hardware like the **[Raspberry Pi](https://www.raspberrypi.org/)** with the official [Raspbian OS](https://www.raspberrypi.org/downloads/raspbian/) is tricky and sometimes suggested to perform a cross-compilation and install the generated binaries. Although it's possible to install [Ubuntu MATE](https://ubuntu-mate.org/download/) and install the relevant pre-compiled binaries directly, one might need to go through the painstaking process of the MATE environment setup, and not be able to modify code to test/experiment self-written features.

In this documentation, I will talk about how to perform a step-by-step (as noob as possible) installation of NFD and related software from source on the Raspberry Pi running official Raspbian OS. The trick is two-folds-

* Install **[Raspbian Lite](https://www.raspberrypi.org/downloads/raspbian/)** instead of full Raspbian Stretch with Pixel desktop to save a lot of memory.
* Increase the Swap partition to avoid virtual memory overflow issue.

The process discussed here has been successfully tested with both Raspberry Pi model 2 and 3b with wireless interface networking.

## Prepare the Pi
* Follow official [Raspbian installation](https://www.raspberrypi.org/documentation/installation/installing-images/README.md) guide to install Raspbian Lite on the SD card.
* Connect Raspberry Pi with HDMI to an external display, attach power cord and a keyboard.
* Boot into your Pi device.
* Login:
	* default user : **pi**
	* default password : **raspberry**
* **Prerequisites for Model 2 only**
	* Plug in USB Wifi dongle.
	* Follow this tutorial on [How to setup usb wifi on Raspberry Pi 2](https://www.electronicshub.org/setup-wifi-raspberry-pi-2-using-usb-dongle/).
	* Reboot Pi using 
```bash
sudo reboot
```
* Connect to wifi by using (do a little google for details or follow [this](https://www.raspberrypi.org/documentation/configuration/wireless/wireless-cli.md)) 
```bash
sudo raspi-config
```

## Increase swap to avoid virtual memory overflow
*This is the most important step to avoid virtual memory overflow during compilation.*
* Modify the file **`/etc/dphys-swapfile`** as follows:
```bash
sudo nano /etc/dphys-swapfile
```
Increase the **`CONF_SWAPSIZE`** from **`200`** megabytes to **`1024`** megabytes. After modification, the file should look like this:
```bash
# /etc/dphys-swapfile - user settings for dphys-swapfile package
# author Neil Franklin, last modification 2010.05.05
# copyright ETH Zuerich Physics Departement
#   use under either modified/non-advertising BSD or GPL license

# this file is sourced with . so full normal sh syntax applies

# the default settings are added as commented out CONF_*=* lines


# where we want the swapfile to be, this is the default
#CONF_SWAPFILE=/var/swap

# set size to absolute value, leaving empty (default) then uses computed value
#   you most likely don't want this, unless you have an special disk situation
CONF_SWAPSIZE=1024

# set size to computed value, this times RAM size, dynamically adapts,
#   guarantees that there is enough swap without wasting disk space on excess
#CONF_SWAPFACTOR=2

# restrict size (computed and absolute!) to maximally this limit
#   can be set to empty for no limit, but beware of filled partitions!
#   this is/was a (outdated?) 32bit kernel limit (in MBytes), do not overrun it
#   but is also sensible on 64bit to prevent filling /var or even / partition
#CONF_MAXSWAP=2048
```
Save and reboot your Pi. This should be enough for building **`ndn-cxx`**, **`NFD`** and **`ndn-tools`** but higher values might be required for building other packages. However, do not go over **`CONF_MAXSWAP=2048`**. Once the builds are complete, you can reduce the value or leave as it is. Using a large **`swap`** can definitely decrease the SD card lifetime but it's going to be used for experimental purposes anyway.

## Installing NFD and related NDN software

```bash
sudo apt-get update
sudo apt-get upgrade
```

#### Prerequisites
Note: minimum boost library requirement is **1.58**.
```bash
sudo apt-get install git build-essential pkg-config libboost1.58-all-dev \
                     libsqlite3-dev libssl-dev libpcap-dev
```

#### Manpage and API docs
Note: I skipped them. Details on how to install are available on [NFD](http://named-data.net/doc/NFD/current/INSTALL.html) installation guide.
```bash
sudo apt-get install doxygen graphviz python-sphinx
```

## Download and install ndn-cxx
```bash
git clone https://github.com/named-data/ndn-cxx
cd ndn-cxx
./waf configure
./waf
sudo ./waf install
sudo ldconfig
```

## Download and install NFD
```bash
git clone --recursive https://github.com/named-data/NFD
cd NFD
./waf configure
./waf
sudo ./waf install
```
After installing NFD from source, you need to create a proper config file. If default location for ./waf configure was used, this can be accomplished by simply copying the sample configuration file:
```bash
sudo cp /usr/local/etc/ndn/nfd.conf.sample /usr/local/etc/ndn/nfd.conf
```
To make NFD properly work, you also need to either disable IPv6 on NFD config file or enable IPv6 probing on Raspberry Pi. I disabled IPv6 in NFD config for now by setting **`enable_v6 no`** to all relevant entries.

## Install ndn-tools (optional)
```bash
git clone https://github.com/named-data/ndn-tools
cd ndn-tools
./waf configure
./waf
sudo ./waf install
```
### Full set of documentation (tutorials + API) in build/docs
`./waf docs`

### Only tutorials in `build/docs`
`./waf sphinx`

### Only API docs in `build/docs/doxygen`
`./waf doxgyen`

## General
All other documentations can be found at the [official NFD install page](http://named-data.net/doc/NFD/current/).
