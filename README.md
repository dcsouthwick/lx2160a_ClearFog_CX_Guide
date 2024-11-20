
# ClearFog CX LX2160A

<img src="images/casefit_small.jpg"  align="right" width="400"/>

- [ClearFog CX LX2160A](#clearfog-cx-lx2160a)
  - [Resources](#resources)
    - [Quickstarting](#quickstarting)
    - [Booting](#booting)
    - [Finish install](#finish-install)
    - [make dpmac auto-configure on boot](#make-dpmac-auto-configure-on-boot)
  - [Gotchas, Fixes, headaches](#gotchas-fixes-headaches)
    - [Bug: Rebooting with the serial cable attached causes u-boot to hang, fans running on max](#bug-rebooting-with-the-serial-cable-attached-causes-u-boot-to-hang-fans-running-on-max)
    - [Bug: `fdtfile` is not set correctly during u-boot, Retimers not set for 25/50/100G, general QSFP errors](#bug-fdtfile-is-not-set-correctly-during-u-boot-retimers-not-set-for-2550100g-general-qsfp-errors)
    - [Bug: CRC boot error](#bug-crc-boot-error)
    - [Bug: Docker/make/X doesn't work](#bug-dockermakex-doesnt-work)
    - [u-boot shows wrong serdes lane configs](#u-boot-shows-wrong-serdes-lane-configs)
    - [Get IP address in u-boot](#get-ip-address-in-u-boot)
    - [notes on dtb/dts conversion \& debugging with QEMU](#notes-on-dtbdts-conversion--debugging-with-qemu)
  - [Override default SERDES configs](#override-default-serdes-configs)
    - [Not working yet](#not-working-yet)
  - [mITX cases](#mitx-cases)
    - [Other potentially passive ITX cases worth considering](#other-potentially-passive-itx-cases-worth-considering)
  - [Thermal Calculations](#thermal-calculations)

<br clear="right">

## About

This is a collection of helpful information for setting up and using the ClearFog CX / Honeycomb LX2 from
[Solidrun](https://www.solid-run.com/embedded-networking/nxp-lx2160a-family/cex7-lx2160/#carrier-boards).
This mini-ITX board is marketed as a desktop for ARM developers working in graphical environments,
however this guide is focused on leveraging the *considerable* networking capabilities of the CPU. Finally,
the only difference between the ClearFox CX and the Honeycomb LX2 is the presence of a QSFP connector.

<img src="images/newboard_small.jpg"  width="400"/>  <img src="images/specs_comparison.png"  width="413"/>

## Web Resources

Below are related guides and information for the Honeycomb/Clearfog boards:

- [Solid-Run ClearFog CX LX2160A Quick Start Guide](https://solidrun.atlassian.net/wiki/spaces/developer/pages/197494288/HoneyComb+LX2+ClearFog+CX+LX2+Quick+Start+Guide) (sometimes outdated)
- [SolidRun lx2160a u-boot repo](https://github.com/SolidRun/lx2160a_build)
- [Wooty-B's collection of fixes/info from discord](https://github.com/Wooty-B/LX2K_Guide)
- [Solid-Run Discord](https://t.co/MaJevmjlC5)

### Quickstarting

Start with [provided ubuntu images](https://images.solid-run.com/LX2k/lx2160a_build/). By default they are SERDES
(serial-deserializer) `18_5_2`, where SD1=18, SD2=5, SD3=2.
This enables QSFP28 at 10Gx4* (with octopus) and the 4xSFP+ ports at 10Gx4.

*In 2024, the defualt build target switched from `8_5_2` (also 8x10G), thanks to upstream patches that finally enabled runtime
switching of port speeds. There are some caveats however - more on that later.*

SERDES lanes map H-E (lane0-3) to QSFP, and the rest to the 4x cage. See page 1912 of the lx2160a Reference Manual.

> Images are raw. use `dd if=` or [Balena Etcher](https://www.balena.io/etcher/) to write

### Booting

Boot pre-built images from NVME:

```sh
# load ubuntu-core.img to address a4000000 in memory (RAM) from mmc card at 0:1 (uSD-card)
# load [interface] [dev:part] [addr] [filename] 
=> load mmc 0:1 0xa4000000 ubuntu-core.img

# load image from memory (address 0xa4000000) into NVME drive
=> nvme scan
=> nvme write 0xa4000000 0 0xd2000

=> boot
```

alternatively write to eMMC

```sh
# show mmc devices
=> mmc list
# load image from mmc0 (uSD)
=> load mmc 0:1 0xa4000000 ubuntu-core.img
# switch to device 1
=> mmc dev 1
# mmc write [start mem-addr] [start blk offset] [blk count]
# writes the above image we wrote to RAM to eMMC blk0
=> mmc write 0xa4000000 0 0xd2000

=> boot
```

### Finish install

##### Delete and install to disk (NVME)

```sh
fdisk /dev/nvme0n1
# list with p
d # delete partition1
n # new 
p # primary
1 # default
First Sector: 131072
Last Sector: # default whole drive if you like, or smaller
Remove Signature? Yes
w # write

# use new partition as disk
resize2fs /dev/nvme0n1p1
```

##### Delete and install to disk (eMMC)

```sh
fdisk /dev/mmcblk1
# list with p
d # delete partition1
n # new 
p # primary
1 # default
First Sector: 131072
Last Sector: # default whole drive if you like, or smaller
Remove Signature? No
w # write

# use new partition as disk
resize2fs /dev/mmcblk1p1
```

Connect onboard nic to DHCP network:

```sh
dhclient # (eth0/dpmac.17 should show active IP)
# dhclient -r to renew
```

Update system time & write to hw clock (here using swiss ntp, ymmv):

```sh
ntpdate -buv ch.pool.ntp.org # flags -bu for time more than 1000s out of spec, unprivileged ports
hwclock -w
```

Finally, don't forget to unpack:

```sh
unminimize
```

### make dpmac auto-configure on boot

For using any SFP ports, you can either add them to .dsti configs, or use restool wrappers to configure
them once linux is running. Here is an example one-shot service for brining up 25G QSFP on Lane0/H SerDes 17
(see above guide from Wooty):

`/usr/lib/systemd/system/dpmac.service`

```desktop
[Unit]
Description=SFP+ Network Interfaces
Before=basic.target
After=local-fs.target sysinit.target
DefaultDependencies=no

[Service]
Type=oneshot
RemainAfterExit=true
ExecStart=/bin/sh -c '/usr/bin/ls-addni --label=QSFP_25G dpmac.3; sleep 0.2'

#ExecStart=/usr/local/bin/ls-listni

ExecStop=/bin/bash -c '/usr/bin/ls-delete $(ls-listmac |grep dpmac.3 | sed "s/.*\\(dpni\\.[0-9]\\).*/\\1/g"); sleep 0.2'

ExecStop=/usr/local/bin/ls-listni

[Install]
WantedBy=basic.target
```

```sh
systemctl daemon-reload
systemctl enable --now dpmac.service
```

## Gotchas, Fixes, Headaches

Solid-Run provides a [u-boot build repo](https://github.com/SolidRun/lx2160a_build),
which is used for producing the prebuilt images for getting started. No testing of other configs
seems to be done as of 2024. You might be able to get support from #solidrun [on discord](https://t.co/MaJevmjlC5),
but most devs are using the [uefi bootloader](https://images.solid-run.com/LX2k/lx2160a_uefi), and not u-boot.

### Bug: Rebooting with the serial cable attached causes u-boot to hang, fans running on max

This is caused by the serial buffer not being completely empty on reboot. A quite annoying bug, especially if you use this device as a router or embedded device. This can be fixed in several ways:

- cold cycle the device with the cable and power disconnected, allowing the buffer to drain/erase.
- Change the autoboot interrupt behavior in u-boot to be a specific key(s)

  ```sh
  #define CONFIG_AUTOBOOT_KEYED
  #define CONFIG_AUTOBOOT_PROMPT "autoboot in %d seconds\n", bootdelay
  #define CONFIG_AUTOBOOT_STOP_STR "f"
  ```

### Bug: `fdtfile` is not set correctly during u-boot, Retimers not set for 25/50/100G, general QSFP errors

If during testing u-boot doesn't set `fdtfile` correctly, ie you see `fls-layerscape-lx2160a` during boot,
or the retimers for the QSFP serdes are not being setup, the problem is in u-boot late-init
(`u-boot/board/solidrun/lx2160a/eth_lx2160acex7.c`), which is responsible for setting the env `fdtfile`,
among other things:

```C
int fsl_board_late_init(void) {
  ...
  printf ("fsl_board_late_init\n");
  if (env_get("fdtfile"))
    return 0;

  if (!protocol_supported(1, srds_s1) || !protocol_supported(2, srds_s2))
    return 0;
  ...
  /* Setup 25gb retimer on lanes e,f,g,h */
  setup_retimer_25g(4);
}
```

You can see a few things above:

1. If `fdtfile` is set before `fsl_board_late_init()`, the retimers will not be enabled.
2. If your serdes protocol is not supported (eg not in the struct below), retimers will not be set
3. *There is no notification of these failure cases*

Problem 2 & 3 causes u-boot to leave `fdtfile` unset, which it later attempts to load some default based on
`${SOC}-${BOARD}.dtb`, where `SOC=fsl-layerscape`, `BOARD=lx2160a`. Fix this by adding your missing serdes:

```diff
+++ b/board/solidrun/lx2160a/eth_lx2160acex7.c
@@ -217,6 +217,7 @@ static struct serdes_configuration {
        {1, 19, true},
        {1, 20, true},
+       {1, 21, true},

        /* Serdes block #2 */
        {2, 0, false},
```

Here I've added serdes 21 to the valid serdes list (which needs retimers)

### Bug: CRC boot error

[u-boot reference](https://u-boot.readthedocs.io/en/stable)

If you see lines like:

```log
Loading Environment from MMC... *** Warning - bad CRC, using default environment
```

Set serdes env correctly to fix CRC boot errors (using your SERDES, here 17_5_2 as an example):

```sh
=> env set serdes0 17
=> env set serdes1 5
=> env set serdes2 2
=> env save
```

### Bug: Docker/make/X doesn't work

Many default kernel configs needed for the lx2160a were not included and until 6.x, so we need to build with
custom includes. For example Docker:

```sh
# from a running lx2160a node check what you are missing:
wget --no-check-certificate https://github.com/moby/moby/raw/master/contrib/check-config.sh
chmod +x check-config.sh
$ ./check-config.sh
info: reading kernel config from /proc/config.gz ...

Generally Necessary:
- cgroup hierarchy: properly mounted [/sys/fs/cgroup]
- CONFIG_NAMESPACES: missing
- CONFIG_NET_NS: missing
- CONFIG_PID_NS: missing
- CONFIG_IPC_NS: missing
- CONFIG_UTS_NS: missing
- CONFIG_CGROUPS: enabled
- CONFIG_CGROUP_CPUACCT: enabled
- CONFIG_CGROUP_DEVICE: enabled
- CONFIG_CGROUP_FREEZER: enabled
- CONFIG_CGROUP_SCHED: enabled
- CONFIG_CPUSETS: enabled
- CONFIG_MEMCG: enabled
- CONFIG_KEYS: enabled
- CONFIG_VETH: enabled (as module)
- CONFIG_BRIDGE: enabled
- CONFIG_BRIDGE_NETFILTER: enabled
- CONFIG_IP_NF_FILTER: enabled
- CONFIG_IP_NF_TARGET_MASQUERADE: enabled
- CONFIG_NETFILTER_XT_MATCH_ADDRTYPE: enabled (as module)
- CONFIG_NETFILTER_XT_MATCH_CONNTRACK: enabled (as module)
- CONFIG_NETFILTER_XT_MATCH_IPVS: missing
- CONFIG_NETFILTER_XT_MARK: missing
- CONFIG_IP_NF_NAT: enabled
- CONFIG_NF_NAT: enabled
- CONFIG_POSIX_MQUEUE: enabled
- CONFIG_CGROUP_BPF: missing
```

You can enable the above missing configs for your software (docker in this case) by adding them to
`/configs/linux/ls2k_addition.config` & rebuilding

verify what configs are enabled:

```sh
zcat /proc/config.gz
```

### u-boot shows wrong serdes lane configs

> Potentially fixed in `develop-ls-5.15.71-2.2.0` branch

If you are using anything other than the pre-provided images from solid-run, you might notice in u-boot incorrect SERDES lane information:

```
DPMAC3@xgmii, DPMAC4@xgmii, DPMAC5@xgmii, DPMAC6@xgmii, DPMAC7@xgmii, DPMAC8@xgmii, DPMAC9@xgmii, DPMAC10@xgmii, DPMAC17@rgmii-id [PRIME], DPMAC18@rgmii-id
```

This is due to u-boot reading a device tree list, and picking up the wrong one: `u-boot/arch/arm/dts/fsl-lx2160a-cex7-8-x-x.dts`

For now, I've edited this file to report the correct SERDES config. I'll try and fix the dynamic loading (as worked in LSDK20.12) later.

### Change u-boot default boot order

> The default boot order is `boot_targets=usb0 mmc0 mmc1 scsi0 scsi1 scsi2 scsi3 pxe dhcp`

This can be changed in two ways:

1. Edit `BOOT_TARGET_DEVICES` variable in `u-boot/include/configs/lx2160acex7.h`
2. During boot time, edit `boot_targets` variable to make persistent:

### Get IP address in u-boot

Get IP address in u-boot:

```sh
=> setenv autoload no; dhcp;
```

### notes on dtb/dts conversion & debugging with QEMU

Step 1:
Dump the default dtb file

`qemu-system-aarch64 -machine virt dumpdtb=virt_default.dtb <followed by other options>`

Step 2:
Decompile dtb file to dts file

`dtc -I dtb -O dts virt_default.dtb > virt.dts`

Step 3:
Add "next-level-cache" to each cpu node. For how to add that, just type `grep -r "next-level-cache" ./` in your Kernel source code. You should get a plenty of samples.

Step 4:
Use dtc to compile the modified dts file to dtb file.

Step 5:
Load this new dtb file in Qemu.
`qemu-system-aarch64 -dtb virt_new.dtb <followed by other options>`

## Override default SERDES configs

**WARNING WARNING WARNING** : Expermimental land! Support ends here!

<https://www.nxp.com/docs/en/application-note/AN13022.pdf>

The above application note documents the steps to override a default SERDES config. I've chosen to overide SERDES17 to the following:

| SRDS_PRTCL_S1 |   H/0  | G/1 | F/2 | E/3 |    D/4    |   C/5   |   B/6  |   A/7   |    PLL   |
|:-------------:|:------:|:---:|:---:|:---:|:---------:|:-------:|:------:|:-------:|:--------:|
| 17 (starting) | 25GE.3 | "   | "   | "   | PCIe.2 x4 | "       | " | " | FFFFSSSS |
| 31 (target)   | 25GE.3 | "   | "   | "   | USXGMII.7   | USSGMII.8 | USXGMII.9 | USXGMII.10 | FFFFFFFF |

Supposedly, the override must be done at the register level, so .pbi hex segments in RCW:

> SerDes1 base address: 0x01EA_0000

> Commands must be 32bit

**This serdes config has been validated by NXP** and is safe to use!

1. Disable all PCIe
    > PCC0 (offset 0x1080)
    - `write 0x01EA1080, 0x00000000`
2. Enable USXGMII XFI mode for lanes A-D
    > PCC9 (offset 0x10B0)
    - `write 0x01EA10B0, 0x99990000`
3. Assume 161.1328125 MHz reference clock for PLLS for 10GE operation
    >PLLSCR0 (offset 0x0504)
    - `write 0x1EA0504, 0x00040000`
4. Configure 10.3125G clock net frequency for PLLS
    >PLLSCR1 (offset 0x0508)
    - `write 0x1EA0508, 0x86100008`
5. Set PLLS clock for XFI
    > PLLSCR3 (offset 0x0510)
    - SSC_SEL=00 non pci
    - SSC_SLP_OFF=0 non pci
    - Bit13=1
    - Bit12=1
    - `write 0x1EA0510, 0x00003000`
6. Set PLLS clock for XFP (pt 2)
    > PLLSCR4(offset 0x0514)
    - `write 0x1EA0514, 0x00001000`
7. change protocol for lane A-D from PCIe to XFI
    - `write 0x1EA0800, 0x00000052`
    - `write 0x1EA0900, 0x00000052`
    - `write 0x1EA0A00, 0x00000052`
    - `write 0x1EA0B00, 0x00000052`
8. Set PLL assingment for XFI on the transmitter for lanes A-D
    - `write 0x01EA0824,0x10000000`
    - `write 0x01EA0924,0x10000000`
    - `write 0x01EA0A24,0x10000000`
    - `write 0x01EA0B24,0x10000000`
9. Configure the transmit equalization for lanes A-D for XFI
    - `write 0x01EA0830,0x1080830`
    - `write 0x01EA0930,0x1080830`
    - `write 0x01EA0A30,0x1080830`
    - `write 0x01EA0B30,0x1080830`
10. set the PLL assignment for XFI on the receiver for lanes A-D
    - `write 0x01EA0844,0x10000000`
    - `write 0x01EA0944,0x10000000`
    - `write 0x01EA0A44,0x10000000`
    - `write 0x01EA0B44,0x10000000`
11. Set the recommended XFI settings for lanes A-D
    - `write 0x01EA0848,0x10000000`
    - `write 0x01EA0948,0x10000000`
    - `write 0x01EA0A48,0x10000000`
    - `write 0x01EA0B48,0x10000000`
12. Disable the receive equalization gain overrides for lanes A-D
    - `write 0x01EA0850,0x00000000`
    - `write 0x01EA0950,0x00000000`
    - `write 0x01EA0A50,0x00000000`
    - `write 0x01EA0B50,0x00000000`
13. Set the recommended receive equalization gains for XFI lanes A-D
    - `write 0x01EA0858,0x81000020`
    - `write 0x01EA0958,0x81000020`
    - `write 0x01EA0A58,0x81000020`
    - `write 0x01EA0B58,0x81000020`

All summed up, paste this into `rcw/lx2160acex7/configs/lx2160a_SD1_17.rcwi, after the includes:

(notice: added REF_CLK flags as the solid-run implementation has swapped the PLLF & PLLS)

```c
SRDS_PRTCL_S1=17
#include <configs/lx2160a_cex7_hc_sd1_lanes_e_f.rcwi>
#include <configs/lx2160a_cex7_hc_sd1_lanes_g_h.rcwi>

SRDS_INTRA_REF_CLK_S1=1 /* PLLF used for PLLS */
SRDS_PLL_REF_CLK_SEL_S1=2

.pbi
write 0x01EA1080,0x00000000
write 0x01EA10B0,0x99990000

write 0x01EA0504,0x00040000
write 0x01EA0508,0x86100008
write 0x01EA0510,0x00003000
write 0x01EA0514,0x00001000

write 0x01EA0800,0x00000052
write 0x01EA0900,0x00000052
write 0x01EA0A00,0x00000052
write 0x01EA0B00,0x00000052
write 0x01EA0824,0x10000000
write 0x01EA0924,0x10000000
write 0x01EA0A24,0x10000000
write 0x01EA0B24,0x10000000
write 0x01EA0830,0x10808307
write 0x01EA0930,0x10808307
write 0x01EA0A30,0x10808307
write 0x01EA0B30,0x10808307
write 0x01EA0844,0x10000000
write 0x01EA0944,0x10000000
write 0x01EA0A44,0x10000000
write 0x01EA0B44,0x10000000
write 0x01EA0848,0x10000000
write 0x01EA0948,0x10000000
write 0x01EA0A48,0x10000000
write 0x01EA0B48,0x10000000
write 0x01EA0850,0x00000000
write 0x01EA0950,0x00000000
write 0x01EA0A50,0x00000000
write 0x01EA0B50,0x00000000
write 0x01EA0858,0x81000020
write 0x01EA0958,0x81000020
write 0x01EA0A58,0x81000020
write 0x01EA0B58,0x81000020
.end 
```

**BUT WAIT!** *There's more!*

You also need to modify `u-boot/arch/arm/cpu/armv8/fsl-layerscape/lx2160a_serdes.c` to create a new SERDES `31` (`0x1f`) with the values:

```c
{0x15, {_25GE10, _25GE9, PCIE2, PCIE2, _25GE6, _25GE5, _25GE4,
_25GE3 } },
{0x16, {XFI10, XFI9, PCIE2, PCIE2, XFI6, XFI5, XFI4, XFI3 } },
{0x1f, {XFI10, XFI9, XFI8, XFI7, _25GE6, _25GE5, _25GE4, _25GE3 } },
```

then modify the SERDES value in RCWSR29 to reflect this, via *magic* and **undocumented** override register: `0x7_00100170`!

(in `eth_lx2160acex7.c`, after retimers are enabled in the board-specific folder of u-boot)

```c
u32 srds_raw;
srds_raw = in_le32(0x1E00170); // should read 0x08b10000, or SD1=17, SD2=5, SD3=2
printf ("rcw_status = 0x%x, srds_s1 = 0x%x (%i), address_srds: 0x%x, RAW=0x%x\n",  rcw_status, srds_s1, srds_s1, &gur->rcwsr[28], srds_raw);
// write new SD1=31
out_le32(0x700100170, 0x08BF0000); // write '31' to SERDES1 magic register
srds_raw = in_le32(0x1E00170);
printf ("srds_raw (after) = 0x%x\n", srds_raw);
```

### Not working yet

Still waiting on response from NXP on how to set lane speeds for the new serdes via restool- to be updated. Currently, the override correctly sets the new lane behavior, but we are unable to configure the DPAA2 objects without help from restool :-/

## mITX cases

> mITX is 170x170 outer , mounting holes 157.48 x 154.95

I purchased this one:
<https://www.fischerelektronik.de/web_fischer/en_GB/cases/M1.08/Heat%20dissipating%20case/$catalogue/fischerData/PG/EMB175/search.xhtml>

![EMB175](images/emb175.jpg)

The carrier board connects via M3x40mm screws, and the chip die is bonded to the top of the case via 15mm copper block where the stock (*loud*) heatsink used to be. With a bit of thermal grease its totally passive, and pretty neat*.

### Other potentially passive ITX cases worth considering

<http://www.isg.com.tw/product_speclist.php?typ=IPC%20Chassis&ser=Mini-ITX%20Enclosure>
1 mini, others hot-swap, tower

<https://www.fischerelektronik.de/web_fischer/en_GB/cases/M1/Cases/index.xhtml>
Industrial supplier, BTO - user stockist?

- this manufacturer seems to supply cases similar to many OEMs (SuperMico, etc)
- supplies many OEM heatsinks of various dimmensions

<https://www.takachi-enclosure.com/cat/heatsink_enclosures>
save as above  (RS)

Takachi EXH • EXHF19-6-18 (43mm internal height)
Takachi EXH • EXHF19-8-18 (63mm internal height)

## Thermal Calculations

[LX2160A datasheet](https://www.mouser.cn/datasheet/2/302/LX2160A-1919583.pdf) Section 5, Thermal

Package 40x40mm (30x30 effective)

Substrate #3-2-3 stackup, 600um core

Case outline number: 98ASA01023

Junction to case thermal resistance $R_{\Theta JC} = 0.15$ &deg;C/Watt

Heatsink force max = 47f-lb / 209N
