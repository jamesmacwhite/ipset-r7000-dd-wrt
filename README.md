# ipset support for the Netgear R7000 running DD-WRT

Additional packages and kernel modules for ipset support.

## Introduction

`ipset` support is a somewhat challenging on DD-WRT because several key components for it are not currently built into the firmware. While DD-WRT can be extended by compiling additional userland tools and rolling your own firmware builds with the DD-WRT toolchain, this is a rather complex and overkill approach for adding additional features like `ipset`. This repository is a collection of additional ipk packages and kernel modules for ipset support on DD-WRT all in one place.

**Note:** These packages and modules are provided as is with no warranty/support. They have been tested on a R7000 (used in my own network) running DD-WRT firmware, other router models may vary. 

## Project directory structure

This explains what each directory is for and its purpose is related to ipset support.

### ipk directory

The additional packages are mostly taken from the [Entware-ng](https://github.com/Entware-ng/Entware-ng) project compiled with a compatible ARM toolchain that works on DD-WRT as well. You will need several additional packages for `ipset`. They are:

* ipset (version 6) - Compiled with the matching DD-WRT kernel source tree
* dnsmasq-full - To replace the version of dnsmasq built into DD-WRT
* iptables - A newer version of iptables to use commands like `--match-set`

The ipk of the dnsmasq provides the following compile time options (version may vary):

```
root@NETGEAR-R7000:~# /opt/sbin/dnsmasq -v
Dnsmasq version 2.77rc5  Copyright (c) 2000-2016 Simon Kelley
Compile time options: IPv6 GNU-getopt no-RTC no-DBus no-i18n no-IDN DHCP DHCPv6 no-Lua TFTP conntrack ipset auth DNSSEC no-ID loop-detect inotify
```

While DD-WRT comes with both `dnsmasq` and `iptables` already, the `dnsmasq` version is compiled without `ipset` support, in addition the `iptables` version is a custom 1.3.7 build which is too old for some `ipset` based firewall rules.

`ipset` itself is compiled using the build system in Entware-ng (which uses the OpenWRT buildroot) but with DD-WRT kernel sources to be compatible where needed.

These packages can be installed with the `opkg` command.

Example:

```
opkg install /path/to/package.ipk
```

In some cases you may also need to use the `--force-checksum` flag.

### xt_set.ko kernel module

In order for ipset and iptables to work together the `xt_set.ko` kernel module is needed. This will not be present in any DD-WRT build currently. This is compiled using the DD-WRT kernel sources and matches the latest firmware kernel branch of the R7000 (currently `linux-4.4`).

Getting the right kernel source and toolchain is important when building modules, otherwise when attempting to load them you may kernel panic and crash your router. Likewise, you cannot simply use a module compiled on the 3.10 kernel compared to the 4.4.x kernel and vice versa, you'll also likely crash your router upon attempting to load the module. This project includes the required kernel for both `linux-3.10` and `linux-4.4` builds.

### opt directory

If you don't have `opkg` installed, you can alternatively copy the entire contents of the `/opt` directory to your routers JFFS partition or a seperate mounted USB device. You will need to make sure you place the contents that matches the DD-WRT $PATH variable. Typically, this is what the DD-WRT path variable looks like:

```
/bin:/usr/bin:/sbin:/usr/sbin:/jffs/sbin:/jffs/bin:/jffs/usr/sbin:/jffs/usr/bin:/mmc/sbin:/mmc/bin:/mmc/usr/sbin:/mmc/usr/bin:/opt/sbin:/opt/bin:/opt/usr/sbin:/opt/usr/bin
```

This however is not recommended unless you know what you are doing, you'll also need to make sure you copy over the /opt folder preserving symlinks. 99.9% of users should install via the .ipk packages provided, its a lot easier!

## Adding ipset support to DD-WRT

For `dnsmasq` and `iptables` you can either "overwrite" the built in versions using mount:

```
mount -o bind /opt/usr/sbin/dnsmasq /usr/sbin/dnsmasq
mount -o bind /opt/usr/sbin/iptables /usr/sbin/iptables
mount -o bind /opt/usr/sbin/ip6tables /usr/sbin/ip6tables
```

This can however cause problems.

Alternatively, if you are using Entware-ng already, you can actually take advantage of running `dnsmasq` via the init.d script included: at `/opt/etc/init.d/S56dnsmasq`

Change `ARGS` to match the arguments used by DD-WRT:

```
-u root -g root --conf-file=/tmp/dnsmasq.conf --cache-size=1500
```

This allows you to essentially use the `dnsmasq` binary with `ipset` support while using the DD-WRT firmware `dnsmasq.conf` file allowing you to still make changes to `dnsmasq` normally. Finally, you'll need to stop the DD-WRT `dnsmasq` server and start the Entware-ng version. You'll want to put this in your `.rc_startup`.

```
stopservice dnsmasq
/opt/etc/init.d/S56dnsmasq start
```

For `iptables` you can place additional rules within the usual `.rc_firewall` but ensure you reference to the full path to the newer `iptables` binary e.g. `/opt/sbin/iptables`

The kernel module for `xt_set.ko` can be placed within /jffs/usr/lib/modules and be loaded at boot time, using `insmod`:

```
insmod /jffs/usr/lib/modules/4.4/xt_set.ko
```

If everything was successful, you will get no prompt returned, running the same command again should yield something similar to:

```
insmod: cannot insert '/jffs/usr/lib/modules/4.4/xt_set.ko': File exists
```

This means the module is now loaded. You can also confirm this by running `lsmod | grep "xt_set"`.

Ensure to change the kernel version in the `insmod` command to whatever kernel source it was built from as stated in the repo. The kernel module is versioned by kernel source because its important to keep track of what Linux kernel base source its come from. You can mostly get away with using a kernel module that is from a slightly different sublevel, but in most cases not an entirely different kernel version altogether.

JFFS storage is better for kernel modules as its a storage partition available early in the boot process. Alternatively you can also use /opt but you may have to delay executing code related to this module till a bit later on in the boot process, to ensure the USB device holding the /opt mountpoint is available.

Sometimes its also good to insmod custom kernel modules manually first, rather than adding `insmod` commands straight into to your `.rc_startup`. The reason being is if the module doesn't load properly and causes a kernel panic, you will essentially throw your router into a boot loop as it will keep trying to load the module causing the problem on startup.

### Example tutorials using ipset

Once you've got `ipset` support setup on DD-WRT, you'll now want to start using it. Here's a few tutorials on what you can do. Common examples are country blocklists and selective VPN routing using domain based rules.

* [Basic ipset tutorial](https://www.dd-wrt.com/phpBB2/viewtopic.php?t=279586)

