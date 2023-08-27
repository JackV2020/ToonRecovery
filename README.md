# ToonRecovery

I followed https://github.com/IgorYbema/ToonRecovery and used a Raspberry Pi 4 with bullseye and things were a little bit different.

This is why I forked and created "Pi 4 bullseye recovery Toon 1.txt" which describes the whole process which starts with downloading and installing bullseye on my Raspberry Pi 4.

## What it does
This application boots a recovery environment for your Toon 1 from a Raspberry Pi or another device with a JTAG debugger attached.

At the moment, the following is implemented:
 - Detection of the U-Boot version
 - Loading of modified U-Boot version using JTAG
 - Logging in to supported versions of U-Boot
 - Setting up the U-Boot environment so the Toon boots into the recovery console
 - Set your own boot server IP address and/or the gateway IP address

## Where is the recovery environment loaded from?

The recovery will be loaded over NFS from the server which you provide (using option --serverip). You must download the NFS server image and unpack the tar.gz file into /srv/nfs. The image file is stored in github using LFS. However LFS (in a free plan) has limited bandwith available and blocks if too many bandwith is already used this month. So, you can also download it from the qutility.nl webserver. Just run this command to download and unpack the recovery image to your NFS directory:

```
mkdir -p /srv/nfs
cd /srv/nfs
curl -Nks "http://qutility.nl/toon-recovery-nfs-server-image.tar.gz" | tar zxvf -
```

Then enable NFS on your server and enable NFSv2. This is needed for the Toon uboot which only supports NFSv2. Check how to enable NFS v2 in the manuals of your linux distro. Check if NFSv2 is enabled with
```
cat /proc/fs/nfsd/versions
```

Then enable the NFS export of the NFS directory with this in the /etc/exports file. Don't forget that you need to reload the exports if you change it.
```
/srv/nfs *(rw,no_subtree_check,async,no_root_squash)
```

## How to use it?

Just like the Toon Rooter it at least requires a serial connection to the Toon. If your Toon 1 is from a newer generation with U-Boot after version 2010-R8, it also requires a JTAG connection. Serial and JTAG connection can be used from the GPIO of a Raspberry PI.

Make sure there's no power going in to either of the devices, and double check the connections
before powering up again.
Connect your Toon's debugging header to a Raspberry Pi according to the following pin assignments:

| Toon | Signal | Pi   |
|:----:|:------:|:----:|
|  1   |  RTCK  |      |
|  2   |  TRST  |  24  |
|  3   |  GND   |  25  |
|  4   |  TCK   |  23  |
|  5   |  GND   |  20  |
|  6   |  TMS   |  22  |
|  7   |  SRST  |  18  |
|  8   |  TDI   |  19  |
|  9   |  Vt    |      |
|  10  |  TDO   |  21  |
|  11  |  RxD   |  8   |
|  12  |        |      |
|  13  |  TxD   |  10  |
|  14  |  GND   |  9   |


Then make sure the serial port on the Pi is enabled and the serial console is disabled
using `raspi_config` and reboot if necessary. Install the dependencies mentioned in the
[Dependencies](#dependencies)-section.

Then get and run this application:
```bash
sudo apt install python-serial
git clone https://github.com/ToonSoftwareCollective/ToonRecovery.git
cd ToonRecovery
sudo python . --jtag-available
```

Als you need to connect the Toon to your network using the builting ethernet port. It is not possible to boot from wifi.

Then reset your Toon and let the magic happen :) After it is finished you must connect to your toon over serial and you will see that your Toon is booted into the recovery console.

## Recovery
When the Toon is booted into the recovery environment start your favourite serial terminal emulator. You will presented a menu like this one.

```
Welcome to the Toon recovery environment.
--> Your Toon has hostname: eneco-001-xxxxxx
--> We have a VPN config file in the backup location
=========================================
1) Backup Toon VPN config
2) Format Toon filesystem
3) Recover Toon filesystem
4) Restore Toon hostname and VPN config from backup
9) Reboot Toon
0) Drop to shell
=========================================
Choose an option:
```
The first option will mount your Toon filesystem (if possible) and make a backup of your VPN config file. This is the only thing which makes your Toon unique and necessary to have your Toon connected to the Eneco server later on (for example, for updates).

The second option will format your Toon filesystem. Be sure you have a backup of the VPN config and agree with formatting the filesystem. You will loose every history of your Toon!

The third option will allow you to recover your Toon from a few supplied firmware versions.

The fourth option will recover your Toon VPN config and hostname after the recovery.


## But I don't have a Pi

You should definitely get a Pi.

However, if you're adamant that you want to recover your Toon from another device and
you have a JTAG debugger lying around that works with OpenOCD, you should be able to
use this script without issue. Just put the configuration file for your debugger in the
`assets/adapters` directory (make sure it has a `.cfg` extension) and pass the name
of the file (without extension) to the script using the `--jtag-hardware` argument.
I'm pretty sure Windows is not going to work though, so you should use a Linux
(virtual) machine.

## Command line arguments

```
usage: sudo python . [-h] [--serial-port PATH] [--gatewayip IP] [--serverip IP]
                  [--output-level INFO|DEBUG] [--jtag-available] [--jtag-hardware TYPE]
                  [--dont-check-uboot] [--boot-only]

Recover your Toon.

optional arguments:
  -h, --help            show this help message and exit
  --serial-port PATH    The path of the serial port to use. Per default it will use /dev/serial0 
  --output-level INFO|DEBUG
                        The level of output to print to the console
  --gatewayip IP        Set a gateway IP address if DHCP is not providing the gateway IP to your toon.
  --serverip IP         Set the NFS server IP address where the recovery image is located. Defaults to
                        where this script is running on.
  --jtag-available      Indicates you have a JTAG debugger connected to your
                        Toon's JTAG headers
  --jtag-hardware TYPE  The JTAG debugger type that we're working with. The
                        default is to autodetect the JTAG debugger (which
                        currently only works on Raspberry Pi). Supported
                        values are: auto, rpi1, rpi2, rpi3
  --dont-check-uboot    Don't check whether we can access the installer
                        version of U-Boot before using JTAG to start up the
                        custom one.
  --boot-only           Don't auto-recover, just boot into the serial
                        console for manual recovery
```

## Dependencies

- Python 2.7

- OpenOCD from git (for newer Toons) (see [instructions](#install-openocd))

## Install OpenOCD
If your Toon has a newer U-Boot version than 2010-R8, a JTAG interface is required to
upload a bootloader that we have access to through the serial console. To do this,
you need to build a version of OpenOCD (at the time of writing the version in apt
doesn't support using the Pi's headers as JTAG debugger).

```bash
git clone --recursive git://git.code.sf.net/p/openocd/code openocd
cd openocd
sudo apt install make libtool libtool-bin pkg-config autoconf automake texinfo libusb-1.0 libusb-dev
{
./bootstrap &&\
./configure --enable-sysfsgpio\
     --enable-bcm2835gpio \
     --prefix=/usr\
&&\
make -j4
} 2>&1 | tee openocd_build.log
sudo make install
```
> these instructions were based on the instructions posted [here](https://www.domoticaforum.eu/viewtopic.php?f=87&t=11230&start=210#p83745) by rboers

## Thanks
This application is based on ToonRooter from Marten Jacobs https://github.com/martenjacobs/ToonRooter and information provided by
- TheHogNL
- MarcelR
