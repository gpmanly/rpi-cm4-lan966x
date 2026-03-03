# Integrating LAN966x to Raspberry Pi OS
This document describes the process of integrating the **LAN966x PCIe network switch driver** into the standard Raspberry Pi OS, running on a Raspberry Pi Compute Module 4 (CM4). The official RPi OS doesn't natively support LAN966x PCIe, so this build bridges that gap by compiling Microchip's Linux kernel fork with the necessary drivers and configurations, allowing users to run a full-featured Raspberry Pi OS alongside the LAN966x switch hardware.

While Microchip provides a basic Board Support Package (BSP), it lacks the full richness of the standard Raspberry Pi OS ecosystem, things like package management, and familiar tooling.

This project solves that by using the standard 64-bit Raspberry Pi OS as the base, then builds and installs a custom kernel derived from Microchip's Linux fork (v6.12).

> Disclaimer Note: This integration is intended as a functional proof-of-concept and has not been performance-optimized. In its current form, throughput is limited to approximately ~619 Mbps using `iperf`.
## General Idea

The core idea is to **replace the stock Raspberry Pi kernel with a custom-built one that includes LAN966x PCIe support**, while keeping everything else about the Raspberry Pi OS intact. The process flows like this:

The build starts by cloning Microchip's Linux kernel repository and checking out the v6.12 branch. Rather than building from scratch, it borrows the `bcm2711_defconfig` (the standard RPi 4 kernel config) from the official Raspberry Pi kernel fork as a baseline. On top of that, a set of additional kernel config options are enabled, primarily the LAN966x switch driver, DSA (Distributed Switch Architecture) framework, SerDes PHY, and various I2C and pinctrl components that the switch depends on.

Device tree sources and overlays are similarly pulled from the Raspberry Pi fork to ensure proper hardware description at boot, and a custom device tree (`bcm2711-rpi-cm4-lan966x.dtb`) is used to describe the combined CM4 + LAN966x hardware. A minor patch is also applied to fix the `ranges` property of the pcie node.

Once built, the kernel image, modules, and device tree blobs are installed onto the RPi boot media, and `config.txt` is updated to point to the new kernel and device tree. After booting, the LAN966x switch enumerates over PCIe and its ports appear as standard Linux network interfaces (`eth1`–`eth4`), fully manageable with familiar tools like `ip` and `ethtool`.

---
## Hardware:
* EVB-LAN9662-NIC
* Raspberry Pi Compute Module 4 (CM4)
* Raspberry Pi IO Board

| ![](raspi-cm4-1.png) | ![](lan9662-nic.png) |
| -------------------- | -------------------- |

Overall set up this:

![](lan9662-rpicm4.png)

## Build Environment:
* Ubuntu Linux 24.04 LTS
* **Target Kernel Version v6.12.48-v8+**

## Install Raspberry Pi OS to the Boot Media (SD Card):
Install the Raspberry Pi OS 64-bit to the Boot Media (SD Card) by following [Install using Imager](https://www.raspberrypi.com/documentation/computers/getting-started.html#raspberry-pi-imager). Use the following options

* _Raspberry Pi Device:_ **Raspberry Pi 4**
* _Operating System:_ **Raspberry Pi OS (64-bit)**
* [`2025-12-04-raspios-trixie-arm64.img`](https://downloads.raspberrypi.com/raspios_arm64/images/raspios_arm64-2025-12-04/2025-12-04-raspios-trixie-arm64.img.xz)


**Boot-up the Raspberry Pi OS as per usual**: 
* Set up credentials
* Set up configurations
* Set up time and date
* Update using `sudo apt update && sudo apt upgrade -y`

Once done, remove the Boot Media.

---
## **Build the Microchip Linux Kernel:**
We will need a Linux cross-compilation host. We will use **Ubuntu Linux 24.04 LTS** with the following packages installed.
```shell
sudo apt install bc bison flex libssl-dev make libc6-dev libncurses5-dev crossbuild-essential-arm64
```

The Linux Kernel we will use is from the Microchip's Fork. The kernel version we'll be using is `v6.12`--the latest version as of this writing. We will also use the `bcm2711_defconfig` from Raspberry Pi's Fork as the base config.
### 1. Prepare the Linux-UNG Repository
```shell
git clone https://github.com/microchip-ung/linux linux-ung
```
Checkout the Kernel Version 6.12
```shell
cd linux-ung
git checkout bsp-6.12-2025.12
```
### 2. Prepare the `defconfig` from the Raspberry Pi Repository
Add the Raspberry Pi remote and fetch the `rpi-6.12.y`
Overwrite `bcm2711_defconfig` file from Raspberry Pi Repository to `arch/arm64/configs/`
```shell
# Add the raspberrypi remote
git remote add rpi https://github.com/raspberrypi/linux.git

# Fetch only the specific branch (no need to fetch everything)
git fetch rpi rpi-6.12.y

# Copy/overwrite the configs from that branch into the working tree
git checkout rpi/rpi-6.12.y -- arch/arm64/configs/bcm2711_defconfig
```
> **Note:** The version of the defconfig should match with our target version.
### 3. Build Configuration
```shell
# for 64-bit
KERNEL=kernel8
make -j12 ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- bcm2711_defconfig
```
> **Note:** The `bcm2711_defconfig` loads the default Linux configuration for the Raspberry Pi  

```shell
make -j12 ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- menuconfig
```

Add the LAN966x drivers and related components in the Linux config:
> Tip: Inside menuconfig, press **`'/'`** key. Type the config, i.e., **`LAN966X_SWITCH`**. It'll display all the config option having this string. Press the number against the desired option. It'll take to that specific hardware configuration. Change configuration with a **`'space'`**, **`'Y'`**, or **`'N'`** key.
```c
# Tick [Y] to Add
<*> The IPv6 protocol                         (IPV6 [=y])
[*] Distributed Switch Architecture           (NET_DSA [=y])
<*> 802.1d Ethernet Bridging                  (BRIDGE [=y])
[*] Data Center Bridging support              (DCB [=y])
<*> Lan966x switch driver                     (LAN966X_SWITCH [=y])
<*> Microchip LAN966X Support                 (MFD_LAN966X_PCI [=y])
<*> SerDes PHY driver for Microchip LAN966X   (PHY_LAN966X_SERDES [=y])
<*> SFP cage support                          (SFP [=y])
[*] Microchip Sparx5 reset driver             (RESET_MCHP_SPARX5 [=y])
<*> Microsemi MIIM interface support          (MDIO_MSCC_MIIM [=y])
{*} I2C bus multiplexing support              (I2C_MUX [=y])
<*> pinctrl-based I2C multiplexer             (I2C_MUX_PINCTRL [=y])
<*> pinctrl-based I2C demultiplexer           (I2C_DEMUX_PINCTRL [=y])
<*> GPIO-based bitbanging I2C                 (I2C_GPIO [=y])
<*> GPIO-based I2C multiplexer                (I2C_MUX_GPIO [=y])
<*> I2C device interface                      (I2C_CHARDEV [=y])
<*> Atmel AT91 I2C Two-Wire interface         (I2C_AT91 [=y])
<*> Microchip AT91 I2C experimental slave mode (I2C_AT91_SLAVE_EXPERIMENTAL [=y])
<*> Broadcom BCM2835 I2C controller           (I2C_BCM2835 [=y])
<*> BRCM Settop/DSL I2C controller            (I2C_BRCMSTB [=y])
[*] Atmel Flexcom                             (MFD_ATMEL_FLEXCOM [=y])
<*> Pinctrl driver for Microchip Serial GPIO  (PINCTRL_MICROCHIP_SGPIO [=y])
<*> Pinctrl driver for the Ocelot and Jaguar2 SoCs (PINCTRL_OCELOT)
<*> High-availability Seamless Redundancy     (HSR [=y])
[*] IP-VLAN support                           (IPVLAN [=y])
```
Remove the following component in the Linux config:
```c
#Tick [N] to Remove
[ ] Memory-mapped io interface driver for DW SPI core  (SPI_DW_MMIO [=n])
```

Save and Exit
```
<Save>
<Exit>
```
### 4. Prepare the Device-Tree
Copy the device tree sources `dts` and overlays `dtso` from Raspberry Pi Repository:
```shell
# Copy the arm64 dts from Raspberry Pi Repository into the working tree
git checkout rpi/rpi-6.12.y -- arch/arm64/boot/dts/broadcom

# Copy the arm dts from Raspberry Pi Repository into the working tree
git checkout rpi/rpi-6.12.y -- arch/arm/boot/dts/broadcom

# Copy the dtso from Raspberry Pi Repository into the working tree
git checkout rpi/rpi-6.12.y -- arch/arm64/boot/dts/overlays
git checkout rpi/rpi-6.12.y -- arch/arm/boot/dts/overlays

# Copy the Makefile from Raspberry Pi Repository into the working tree
git checkout rpi/rpi-6.12.y -- arch/arm64/boot/dts/Makefile

git checkout rpi/rpi-6.12.y -- scripts/Makefile.lib
git checkout rpi/rpi-6.12.y -- scripts/Makefile.build

```
Copy some dependencies from the `include` to properly build the `dts` and `dtso`
```shell
# Copy the dependencies from Raspberry Pi Repository into the working tree
git checkout rpi/rpi-6.12.y -- include/dt-bindings/clock/rp1.h
git checkout rpi/rpi-6.12.y -- include/dt-bindings/gpio/gpio-fsm.h
git checkout rpi/rpi-6.12.y -- include/dt-bindings/mfd/rp1.h
```

We need to correct the `ranges` property in the PCIe node of the device tree `bcm2711.dtsi`. 
```shell
nano arch/arm/boot/dts/broadcom/bcm2711.dtsi
```
The relevant DT node (/scb/pcie@7d500000) should have its inbound mapping changed from:
```
ranges = <0x02000000 0x0 0xf8000000 0x6 0x00000000
				  0x0 0x04000000>;
```
To:
```
/* Correct - identity map for LAN966x FDMA */
ranges = <0x02000000 0x0 0xc0000000 0x6 0x00000000
				  0x0 0x40000000>;
```

The device tree to use is `bcm2711-rpi-cm4-lan966x`. So edit the `Makefile` to include `bcm2711-rpi-cm4-lan966x` during build:
```shell
nano arch/arm64/boot/dts/broadcom/Makefile
```
Insert `bcm2711-rpi-cm4-lan966x.dtb` on this location like so:
```c
dtb-$(CONFIG_ARCH_BCM2835) += bcm2711-rpi-cm4-lan966x.dtb
```

### 5. Build
Run the following command to build a 64-bit kernel, modules and the device-tree blob:
```shell
make -j12 ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- Image modules dtbs
```
> The build may take a while so grab a coffee ☕

---

## **Install the Microchip Linux Kernel**
Having built the kernel, copy it onto the Raspberry Pi OS 64-bit Image and install the modules.
### 1. Mount Boot Media
First, run `lsblk`. Then, connect the boot media. Run `lsblk` again; the new device represents the boot media. See output similar to the following:
```shell
$ lsblk
sdc
   sdc1
   sdc2
```

Mount these partitions as `mnt/boot` and `mnt/root`, adjusting the partition letter to match the location of the boot media:
```shell
mkdir mnt
mkdir mnt/boot
mkdir mnt/root
sudo mount /dev/sdc1 mnt/boot
sudo mount /dev/sdc2 mnt/root
```
### 2. Next, install the kernel modules onto the boot media:
```shell
sudo env PATH=$PATH make -j12 ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- INSTALL_MOD_PATH=mnt/root modules_install
```
### 3. Install Kernel onto the boot media:
Run the following commands to create a backup image of the current kernel, install the fresh kernel image, overlays.
```shell
sudo cp mnt/boot/kernel8.img mnt/boot/kernel8-backup.img
sudo cp arch/arm64/boot/Image mnt/boot/kernel8.img
sudo cp -r arch/arm64/boot/dts/broadcom/*.dtb* mnt/boot/
sudo cp -r arch/arm64/boot/dts/overlays/*.dtb* mnt/boot/overlays/
```
Modify `config.txt` to define the new kernel and device-tree at boot
```shell
sudo nano mnt/boot/config.txt
```
Add the following texts at the bottom of `config.txt`, save then exit.
```shell
[all]
#use 64-bit
arm_64bit=1

#boots the newly built kernel
kernel=kernel8.img

#this overlay prevents firmware from extending the PCIe inbound window beyond 32-bit
dtoverlay=pcie-32bit-dma

#target DTB to be loaded
device_tree=bcm2711-rpi-cm4-lan966x.dtb

#Disable OTG mode of the USB
otg_mode=0
```
Save and Exit
```
<Save>
<Exit>
```
Unmount the partitions:
```shell
sudo umount mnt/boot
sudo umount mnt/root
```

Finally, connect the boot media to the Raspberry Pi and connect it to power to run the freshly-compiled kernel.

---

## **Verification**
Boot the Raspberry Pi with the newly modified Boot Media
### Show Linux Version
```shell
uname –a

#It should return the following
Linux raspberrypi 6.12.48-v8+ #36 SMP PREEMPT Mon Mar  2 15:18:32 PST 2026 aarch64 GNU/Linux
```
### Show that the LAN966x driver is loaded and enumerated.
```shell
$ lspci -vv
```
It should return **`Kernel driver in use: microchip_lan966x_pci`**:
```shell
lspci -vv
00:00.0 PCI bridge: Broadcom Inc. and subsidiaries BCM2711 PCIe Bridge (rev 20) (prog-if 00 [Normal decode])
        Device tree node: /sys/firmware/devicetree/base/scb/pcie@7d500000/pci@0,0
        Control: I/O- Mem+ BusMaster+ SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx-
        Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
        Latency: 0
        Interrupt: pin A routed to IRQ 19
        Bus: primary=00, secondary=01, subordinate=01, sec-latency=0
        Memory behind bridge: 00000000-043fffff [size=68M] [32-bit]
        Prefetchable memory behind bridge: [disabled] [64-bit]
        Secondary status: 66MHz- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- <SERR- <PERR-
        BridgeCtl: Parity- SERR- NoISA- VGA- VGA16- MAbort- >Reset- FastB2B- 
                PriDiscTmr- SecDiscTmr- DiscTmrStat- DiscTmrSERREn-
        Capabilities: <access denied>
        Kernel driver in use: pcieport

01:00.0 Ethernet controller: Microchip Technology / SMSC Device 9660
        Subsystem: Microchip Technology / SMSC Device 9660
        Device tree node: /sys/firmware/devicetree/base/scb/pcie@7d500000/pci@0,0/switch@0,0
        Control: I/O- Mem+ BusMaster+ SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx-
        Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
        Latency: 0
        Interrupt: pin A routed to IRQ 19
        Region 0: Memory at 600000000 (32-bit, non-prefetchable) [size=32M]  
        Region 1: Memory at 602000000 (32-bit, non-prefetchable) [size=16M]
        Region 2: Memory at 603000000 (32-bit, non-prefetchable) [size=8M]   
        Region 3: Memory at 603800000 (32-bit, non-prefetchable) [size=8M]   
        Region 4: Memory at 604000000 (32-bit, non-prefetchable) [size=128K] 
        Region 5: Memory at 604020000 (32-bit, non-prefetchable) [size=8K]   
        Capabilities: <access denied>
        Kernel driver in use: microchip_lan966x_pci
```
IP Command
```shell
$ ip a
```
Returns:
```shell
$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether d8:3a:dd:db:e2:99 brd ff:ff:ff:ff:ff:ff
    inet 10.161.140.95/24 brd 10.161.140.255 scope global dynamic noprefixroute eth0
       valid_lft 6768sec preferred_lft 6768sec
    inet6 fe80::902:abcd:669d:8f2c/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether f6:8c:d7:1d:08:d1 brd ff:ff:ff:ff:ff:ff
    inet 10.0.1.2/32 scope global eth1
       valid_lft forever preferred_lft forever
4: eth2: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc mq state DOWN group default qlen 1000
    link/ether f6:8c:d7:1d:08:d2 brd ff:ff:ff:ff:ff:ff
5: eth3: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc mq state DOWN group default qlen 1000
    link/ether f6:8c:d7:1d:08:d3 brd ff:ff:ff:ff:ff:ff
6: eth4: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc mq state DOWN group default qlen 1000
    link/ether f6:8c:d7:1d:08:d4 brd ff:ff:ff:ff:ff:ff
7: wlan0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc fq_codel state DOWN group default qlen 1000
    link/ether d8:3a:dd:db:e2:9a brd ff:ff:ff:ff:ff:ff
```
Ethtool Command
```shell
$ ethtool -i eth1
```
Returns:
```shell
driver: lan966x-switch
version: 6.12.48-v8+
firmware-version:
expansion-rom-version:
bus-info: 602000000.switch
supports-statistics: yes
supports-test: no
supports-eeprom-access: no
supports-register-dump: no
supports-priv-flags: no
```
### 4. Test the Ethernet interface.
Connect the Ethernet to a link partner and then ping:
```shell
# Assign address to eth1
$ sudo ip addr add 10.0.1.2/24 dev eth1

# Ping the link partner
$ ping 10.0.1.3 -I eth1
PING 10.0.1.3 (10.0.1.3) from 10.0.1.2 eth1: 56(84) bytes of data.
64 bytes from 10.0.1.3: icmp_seq=1 ttl=128 time=1.73 ms
64 bytes from 10.0.1.3: icmp_seq=2 ttl=128 time=1.82 ms
64 bytes from 10.0.1.3: icmp_seq=3 ttl=128 time=1.81 ms
64 bytes from 10.0.1.3: icmp_seq=4 ttl=128 time=1.59 ms
64 bytes from 10.0.1.3: icmp_seq=5 ttl=128 time=1.65 ms
^C
--- 10.0.1.3 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4006ms
rtt min/avg/max/mdev = 1.591/1.718/1.819/0.088 ms
```

```shell
# IPERF TEST
$ iperf -c 10.0.1.3
------------------------------------------------------------
Client connecting to 10.0.1.3, TCP port 5001
TCP window size: 16.0 KByte (default)
------------------------------------------------------------
[  1] local 10.0.1.2 port 41082 connected with 10.0.1.3 port 5001
[ ID] Interval       Transfer     Bandwidth
[  1] 0.0000-10.0168 sec   740 MBytes   619 Mbits/sec
```

## Screenshot of the Actual Verification
![Test](results.png)
