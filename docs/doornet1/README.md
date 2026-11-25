# EmbedFire DoorNet1 on OpenWrt 25.12.6

This downstream port targets the RK3328 DoorNet1 booting from microSD. It
builds an ext4 disk image using the OpenWrt Rockchip image pipeline. The
25.12.6 upstream commit is `64900e756efa667774214f9c6b17c113222e01f7`.
The 2022 vendor image is a separate, hardware-tested recovery baseline.

## Build the base image

Start from the `doornet1-25.12.6` branch in a fresh checkout on Linux. Install
the [OpenWrt build prerequisites](https://openwrt.org/docs/guide-developer/toolchain/install-buildsystem)
first and run the build as an unprivileged user.

```sh
cp docs/doornet1/feeds.conf.v25.12.6 feeds.conf
./scripts/feeds update -a
./scripts/feeds install -a
cp docs/doornet1/base.config .config
make defconfig
make -j8 download
make -j8
```

Look in `bin/targets/rockchip/armv8/` for the DoorNet1
`*-ext4-sysupgrade.img.gz` image, package manifest, and checksums. For the
first install, write the image to a **spare** TF card and verify the write.
Do not upgrade the vendor install in place for the first board test.

This configuration contains LuCI and the RTL8153 USB Ethernet driver. It
does not include optional USB Wi-Fi. Board-specific U-Boot uses the
upstream RK3328 DDR4 TPL and reads the kernel from the SD card. Linux uses
the same board DTS without the vendor-only dynamic DDR frequency
settings. SD starts in high-speed mode without UHS voltage switching.
The WAN MAC is derived from the detected SD card CID when available, so
swapping cards may change it.

## Add USB cellular support

After verifying storage and both Ethernet ports, add the cellular fragment
to the same pinned build. Run the following after updating and installing
the pinned feeds as shown above.

```sh
cat docs/doornet1/base.config docs/doornet1/cellular.config > .config
make defconfig
make -j8 download
make -j8
```

For RNDIS, attach the detected USB Ethernet netdev to a DHCP interface
in the WAN firewall zone. For MBIM or QMI, choose ModemManager in LuCI
and set the device and APN for the actual modem. Select one manager for
a modem control port. These profiles include drivers, not a preset APN,
USB mode switch, or forced WAN failover policy.

## Hardware acceptance

Use a cold boot with serial at **1500000 baud, 8N1**. Confirm that U-Boot
loads the FIT, Linux enumerates the TF card, the root partition mounts,
and the console becomes available. Check that physical LAN is `eth1`,
leases an IPv4 address, and serves LuCI. Check WAN separately on `eth0`.
For the cellular build, confirm USB enumeration first and then test
RNDIS DHCP or MBIM/QMI dialing and recovery after unplugging the modem.

An image build and a checksum do not establish that these hardware
checks pass.
