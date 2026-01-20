# TP-Link Archer C6v2 EU flash capacity upgrade

This fork contains a branch called `16M_mod`, that adds support for TP-Link Archer C6v2 (EU) with a 16M flash chip upgrade.

Backstory: EU devices contain an absurdly small amount of flash (8MB) for OpenWrt 24.x, leaving only ~500kB for user packages. This problem can be solved rather easilly, if you have SMD soldering skills and some knowledge of electronics: just buy a new flash chip (I used W25Q128FVSIG) with a larger capacity and replace the factory one.

Obviously, you have to transfer the contents of the old chip to the new one first. My prefered way of duing this is TL866 parallel programmer + an SO-8 adapter, but you can get away with a CH340 programmer with a clamp. Solder the new chip into the router and it should run the old firmware just as it did with the old chip. This is where the "easy" part ended for me.

The last part is to somehow use the added flash space. Due to a lack of documentation and prior discussions of such cases, this part seemed tricky to me.

My first thought was to modify u-boot to move "tplink" and "art" partitions out of the way (binwalk + hex-editor, or recompile it from source from TP-Link's GPL code center: [see here](https://github.com/kutukvpavel/Archer_C6V2_EU_modernized)), and I even got pretty far on this journey: I learned to unpack and repack u-boot with binwalk and ancient LZMA v4.x from TP-Link's toolchain, as well as to recompile u-boot from GPL code on a Debian 8 VM. And yes, U-boot checks partition table integrity before launching Linux, so you won't be able to remove even the useless "tp-link" partition occupying 20KiB ("art" is atheros calibration data and is required for WiFi). U-boot on this device is compressed, so no environment sector is used, therefore you can't modify u-boot environment variables without recompiling it.

But then I found [this piece of documentation on extroot](https://openwrt.org/docs/guide-user/additional-software/extroot_configuration), and suddenly messing with u-boot started to look like an overly complicated solution, compared to creating a new UBIFS partition inside the freashly-added top 8 megabytes of flash. After all, recompiling modern OpenWrt from source is a much more pleasant task, than dealing with 20-year-old OEM GPL stuff. All that is required to add such partition is to add a new section into the device tree (`qca9563_tplink_archer-c6-v2.dts`):

```
partition@810000 {
	label = "extroot";
	reg = <0x810000 0x7e0000>;
};
```

I purposely left some padding around this area, just in case. Recompile, flash, reboot, just to find out 3 things:
 - OpenWrt build system does not use CCache by default,
 - OpenWrt build system does not play well with parallel compilation (sometimes),
 - OpenWrt built for a NOR-flash device lacks UBIFS tools, there is no separate package you can install to get UBI working (okay, there is `uvol` package and `ubiattach` and others can be enabled in BusyBox section of menuconfig, but my dumbass forgot to add all the package feeds after sysupgrade, so we will never know where this route leads).

You can't use ext4 or any other supported filesystem directly on a flash chip (mtd), since no other supported FS implements FTL (flash translation layer). You can't create a second JFFS2 partition, since again, there are no tools to do so from the device itself, and extroot does not seem to support it, too. Oh well.

The third idea came from extensive googling "mtd join/merge/concatenate", since all I in fact needed was to somehow stitch the two physical partitons into a single logical one, which led me to the "mtd-concat" feature of OpenWrt's kernel. This feature, AFAIK, is completely undocumented, save for a commit message, yet is used in contemporary OpenWrt codebase: `ar9344_huawei_ap6010dn.dts` device tree provides an excellent example of its usage. This was the perfect solution, that required only 2 more sections to be added to the device tree:

```
/ {

  <...>

	virtual_flash {
		compatible = "mtd-concat";
		devices = <&fwconcat0 &fwconcat1>;

		partitions {
			compatible = "fixed-partitions";
			#address-cells = <1>;
			#size-cells = <1>;

			partition@0 {
				label = "firmware";
				reg = <0x0 0xf80000>;
				compatible = "denx,uimage";
			};
		};
	};
};

<...>

&spi {
	
  <...>

	flash@0 {
		
    <...>

		partitions {
			
      <...>

			fwconcat0: partition@30000 {
				label = "fwconcat0";
				reg = <0x030000 0x7a0000>;
			};

			<...>

			fwconcat1: partition@810000 {
				label = "fwconcat1";
				reg = <0x810000 0x7e0000>;
			};
		};
	};
};
```

Just in case, I still made sure to compile an image of OpenWrt, that fits into the factory partition. AFAIK, sysupgrade is unable to resize partitons. Any required packages can be installed later.


# Original readme

![OpenWrt logo](include/logo.png)

OpenWrt Project is a Linux operating system targeting embedded devices. Instead
of trying to create a single, static firmware, OpenWrt provides a fully
writable filesystem with package management. This frees you from the
application selection and configuration provided by the vendor and allows you
to customize the device through the use of packages to suit any application.
For developers, OpenWrt is the framework to build an application without having
to build a complete firmware around it; for users this means the ability for
full customization, to use the device in ways never envisioned.

Sunshine!

## Download

Built firmware images are available for many architectures and come with a
package selection to be used as WiFi home router. To quickly find a factory
image usable to migrate from a vendor stock firmware to OpenWrt, try the
*Firmware Selector*.

* [OpenWrt Firmware Selector](https://firmware-selector.openwrt.org/)

If your device is supported, please follow the **Info** link to see install
instructions or consult the support resources listed below.

## 

An advanced user may require additional or specific package. (Toolchain, SDK, ...) For everything else than simple firmware download, try the wiki download page:

* [OpenWrt Wiki Download](https://openwrt.org/downloads)

## Development

To build your own firmware you need a GNU/Linux, BSD or macOS system (case
sensitive filesystem required). Cygwin is unsupported because of the lack of a
case sensitive file system.

### Requirements

You need the following tools to compile OpenWrt, the package names vary between
distributions. A complete list with distribution specific packages is found in
the [Build System Setup](https://openwrt.org/docs/guide-developer/build-system/install-buildsystem)
documentation.

```
binutils bzip2 diff find flex gawk gcc-6+ getopt grep install libc-dev libz-dev
make4.1+ perl python3.7+ rsync subversion unzip which
```

### Quickstart

1. Run `./scripts/feeds update -a` to obtain all the latest package definitions
   defined in feeds.conf / feeds.conf.default

2. Run `./scripts/feeds install -a` to install symlinks for all obtained
   packages into package/feeds/

3. Run `make menuconfig` to select your preferred configuration for the
   toolchain, target system & firmware packages.

4. Run `make` to build your firmware. This will download all sources, build the
   cross-compile toolchain and then cross-compile the GNU/Linux kernel & all chosen
   applications for your target system.

### Related Repositories

The main repository uses multiple sub-repositories to manage packages of
different categories. All packages are installed via the OpenWrt package
manager called `opkg`. If you're looking to develop the web interface or port
packages to OpenWrt, please find the fitting repository below.

* [LuCI Web Interface](https://github.com/openwrt/luci): Modern and modular
  interface to control the device via a web browser.

* [OpenWrt Packages](https://github.com/openwrt/packages): Community repository
  of ported packages.

* [OpenWrt Routing](https://github.com/openwrt/routing): Packages specifically
  focused on (mesh) routing.

* [OpenWrt Video](https://github.com/openwrt/video): Packages specifically
  focused on display servers and clients (Xorg and Wayland).

## Support Information

For a list of supported devices see the [OpenWrt Hardware Database](https://openwrt.org/supported_devices)

### Documentation

* [Quick Start Guide](https://openwrt.org/docs/guide-quick-start/start)
* [User Guide](https://openwrt.org/docs/guide-user/start)
* [Developer Documentation](https://openwrt.org/docs/guide-developer/start)
* [Technical Reference](https://openwrt.org/docs/techref/start)

### Support Community

* [Forum](https://forum.openwrt.org): For usage, projects, discussions and hardware advise.
* [Support Chat](https://webchat.oftc.net/#openwrt): Channel `#openwrt` on **oftc.net**.

### Developer Community

* [Bug Reports](https://bugs.openwrt.org): Report bugs in OpenWrt
* [Dev Mailing List](https://lists.openwrt.org/mailman/listinfo/openwrt-devel): Send patches
* [Dev Chat](https://webchat.oftc.net/#openwrt-devel): Channel `#openwrt-devel` on **oftc.net**.

## License

OpenWrt is licensed under GPL-2.0
