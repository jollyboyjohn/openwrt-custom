### INTRODUCTION

OpenWRT 21.02 on a TD-W8980 v1 is on the limit of the minimum specs:
* Storage: 8MB
* Memory: 64MB
More detail here: https://openwrt.org/toh/hwdata/tp-link/tp-link_td-w8980

As a result, there's little space to upload additional DSL firmware for vectoring, which isn't supplied due to distribution issues.

To make space, the following _could_ be removed if the W8980 is only used as a modem:
* __IPv6__ -- additional, unused
* __DNS__ -- not required for a passive device
* __Firewall__ -- DSL is passthru over Cat5e
* __DSL firmware__ -- doesn't have vectoring
* __Wireless__ -- unused, as the W8980 doesn't do 802.11AC
* __PPP__ -- DSL is passthru over Cat5e

Suggested modem port bridge configuration:
```
            + DSL port
br-modem:   |
            + LAN/WAN port

br-lan:     + LAN1/2/3 ports (Management on 192.168.1.2)
```

### REQUIREMENTS

Get a Debian 10 virtual machine.

### OPTIONAL: DSL CUSTOM FIRMWARE

Sources of vectoring-specific firmware can be found here:
https://xdarklight.github.io/lantiq-xdsl-firmware-info/

The vectoring software of the NetGear DM200 (v1.0.0.34 / 7.6.A+7.1.C) seems to work well and is easy to obtain.

For other firmware sources, tools like p7zip-full and Freetz need to be installed.

To compile Freetz if required:
```
$ sudo apt-get install p7zip-full autoconf libtool-bin pkg-config bison bc flex zlib1g-dev libreadline-dev libcap-dev libacl1-dev gcc-multilib
$ cd ~/
$ git clone https://github.com/Freetz/freetz
$ cd freetz
$ make tools
```

### OPENWRT BASE BUILD SYSTEM

Download the dependencies:
```
# apt-get install build-essential git libncurses5-dev gawk rsync unzip
```

Download the source and set it up:
```
$ git clone https://git.openwrt.org/openwrt/openwrt.git
$ cd openwrt
$ ./scripts/feeds update -a
$ ./scripts/feeds install -a
```

### CONFIGURATION

Configuration is done via the ncurses configuration:
```
$ make menuconfig
```

Set the profile of the platform to be built:

- [x] Target System (**Lantiq**)
- [x] Subtarget (**XRX200**)
- [x] Target Profile (**TP-Link TD-W8980 v1**)

Kernel options to enable/disable for a modem:

**Global Build Settings**
- [ ] Enable IPv6 support in packages

**Base system**
- [x] bridge
- [ ] dnsmasq
- [ ] firewall

**Firmware**
- [ ] dsl-vrx200-firmware-xdsl-a
- [ ] dsl-vrx200-firmware-xdsl-b-patch

**Kernel modules -> Wireless drivers**
- [ ] kmod-owl-loader
- [ ] kmod-ath9k

**Network -> Firewall**
- [ ] ip6tables
- [ ] iptables

**Network -> Wireless APD**
- [ ] wpad-basic-openssl

**Network**
- [ ] ppp

**Utilities**
- [ ] iwinfo

**Kernel modules -> Netfilter Extensions**
- [ ] kmod-ipt-conntrack
- [ ] kmod-ipt-nat
- [ ] kmod-ipt-offload
- [ ] kmod-nf-conntrack6
- [ ] kmod-nf-flow
- [ ] kmod-nf-nat
- [ ] kmod-nf-conntrack
- [ ] kmod-ipt-core
- [ ] kmod-nf-ipt
- [ ] kmod-nf-reject

### TOOLCHAIN + IMAGE BUILD

To build in additional files (configs, firmware), move them to openwrt/files/.

For example:
```
$ mkdir -p files/lib/firmware/
$ cp vr9-B-dsl.bin files/lib/firmware/
```

To compile:
```
$ make
```

This builds the full build toolchain and the image.

Do *not* perform a "make clean", otherwise the entire toolchain will be blown away, which takes a substantial amount of time.

If the build fails, try with a single thread:
```
$ make -j1
```

Once complete, the image is written into the bin/ directory.
```
$ cd bin/targets/lantiq/xrx200/
$ scp openwrt-lantiq-xrx200-tplink_tdw8980-squashfs-sysupgrade.bin username@hostname:
```

### UPGRADING

Copy the firmware to /tmp on the device:
```
user@buildhost:~$ scp openwrt-*-squashfs-sysupgrade.bin root@openwrt:/tmp
```
Commence the upgrade:
```
root@openwrt:~# sysupgrade -v /tmp/openwrt-lantiq-xrx200-tplink_tdw8980-
squashfs-sysupgrade.bin
```
Testing to see if the management port comes back:
```
$ ping 192.168.1.2
```

Once up, test the modem, looking the firmware first:
```
root@openwrt-dsl:~# /etc/init.d/dsl_control dslstat
firmware_version": "5.7.6.10.0.7",
```

Note: It can take some time for the DSL firmware to be active.

After a few mins, using the same command, the power_state should go from L3 to L0:
```
root@openwrt-dsl:~# /etc/init.d/dsl_control dslstat
"power_state": "L3 - No power"

root@openwrt-dsl:~# /etc/init.d/dsl_control dslstat
"power_state": "L0 - Synchronized",
```

Use the bridge utility to check that the DSL and LAN/WAN ports are bridged:
```
root@openwrt-dsl:~# brctl show
bridge name     bridge id               STP enabled     interfaces
br-lan          7fff.f4f26d897b3d       no              eth0.1
br-modem        7fff.f4f26d897b3d       no              dsl0.101
                                                        eth0.2
```
