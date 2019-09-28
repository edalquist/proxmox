# GPU Passthrough working on ProxMox 6 & Windows 10

## Hardware
 * Ryzen 9 3900X
 * ASUS Pro WS-X570 ACE
 * XFX Radeon RX 580 GTS XXX Edition (passthrough)
 * VisionTek Radeon 5450 (host)

## BIOS/Firmware
 * Motherboard: Version 1001 - 2019/09/25

## Setup Steps

### Add iommu flags to EFI config

Edit `/etc/kernel/cmdline` and add `iommu=pt amd_iommu=on video=efifb:off` mine looks like:
```
root=ZFS=rpool/ROOT/pve-1 boot=zfs iommu=pt amd_iommu=on video=efifb:off rootdelay=10
```

Update the EFI config
```
$ pve-efiboot-tool refresh
```

**Reboot**

### Find the Card

Run `lspci` and look through the results for your card, there are two entries,
one for the GPU and one for the HDMI audio device.

```
$ lspci -v
...
0a:00.0 VGA compatible controller: Advanced Micro Devices, Inc. [AMD/ATI] Ellesmere [Radeon RX 470/480] (rev e7) (prog-if 00 [VGA controller])
	Subsystem: XFX Pine Group Inc. Ellesmere [Radeon RX 470/480/570/570X/580/580X/590]
	Flags: bus master, fast devsel, latency 0, IRQ 160
	Memory at c0000000 (64-bit, prefetchable) [size=256M]
	Memory at d0000000 (64-bit, prefetchable) [size=2M]
	I/O ports at f000 [size=256]
	Memory at fce00000 (32-bit, non-prefetchable) [size=256K]
	Expansion ROM at fce40000 [disabled] [size=128K]
	Capabilities: [48] Vendor Specific Information: Len=08 <?>
	Capabilities: [50] Power Management version 3
	Capabilities: [58] Express Legacy Endpoint, MSI 00
	Capabilities: [a0] MSI: Enable+ Count=1/1 Maskable- 64bit+
	Capabilities: [100] Vendor Specific Information: ID=0001 Rev=1 Len=010 <?>
	Capabilities: [150] Advanced Error Reporting
	Capabilities: [200] #15
	Capabilities: [270] #19
	Capabilities: [2b0] Address Translation Service (ATS)
	Capabilities: [2c0] Page Request Interface (PRI)
	Capabilities: [2d0] Process Address Space ID (PASID)
	Capabilities: [320] Latency Tolerance Reporting
	Capabilities: [328] Alternative Routing-ID Interpretation (ARI)
	Capabilities: [370] L1 PM Substates
	Kernel driver in use: vfio-pci
	Kernel modules: amdgpu

0a:00.1 Audio device: Advanced Micro Devices, Inc. [AMD/ATI] Ellesmere HDMI Audio [Radeon RX 470/480 / 570/580/590]
	Subsystem: XFX Pine Group Inc. Ellesmere HDMI Audio [Radeon RX 470/480 / 570/580/590]
	Flags: bus master, fast devsel, latency 0, IRQ 159
	Memory at fce60000 (64-bit, non-prefetchable) [size=16K]
	Capabilities: [48] Vendor Specific Information: Len=08 <?>
	Capabilities: [50] Power Management version 3
	Capabilities: [58] Express Legacy Endpoint, MSI 00
	Capabilities: [a0] MSI: Enable- Count=1/1 Maskable- 64bit+
	Capabilities: [100] Vendor Specific Information: ID=0001 Rev=1 Len=010 <?>
	Capabilities: [150] Advanced Error Reporting
	Capabilities: [328] Alternative Routing-ID Interpretation (ARI)
	Kernel driver in use: vfio-pci
	Kernel modules: snd_hda_intel
...
```

Note the device address, `0a:00` in this case and then get the device ids

```
# lspci -n -s 0a:00
0a:00.0 0300: 1002:67df (rev e7)
0a:00.1 0403: 1002:aaf0
```

The IDs in this case are `1002:67df` and `1002:aaf0`.


### Verify IOMMU configuration

Execute `dmesg` and grep the results, it should look something like this:
```
$ dmesg | grep -e DMAR -e IOMMU
[    1.248478] AMD-Vi: IOMMU performance counters supported
[    1.252322] AMD-Vi: Found IOMMU at 0000:00:00.2 cap 0x40
[    1.253794] perf/amd_iommu: Detected AMD IOMMU #0 (2 banks, 4 counters/bank).
[   15.970395] AMD-Vi: AMD IOMMUv2 driver by Joerg Roedel <jroedel@suse.de>
```

See if the card is in a group by itself:

```
$ for a in /sys/kernel/iommu_groups/*; do find $a -type l; done | sort
...
/sys/kernel/iommu_groups/23/devices/0000:06:00.7
/sys/kernel/iommu_groups/24/devices/0000:0a:00.0
/sys/kernel/iommu_groups/24/devices/0000:0a:00.1
/sys/kernel/iommu_groups/25/devices/0000:0b:00.0
...
```

In this case the card is in group 24 by itself.


### Configure vfio drivers to be loaded

```
$ echo vfio >> /etc/modules \
    echo vfio_iommu_type1 >> /etc/modules \
    echo vfio_pci >> /etc/modules \
    echo vfio_virqfd >> /etc/modules
```

### Configure vfio Driver

This sets up `vfio.conf` to make the vfio driver load BEFORE the amd drivers so they capture the card we want to pass
through.

```
$ vim /etc/modprobe.d/vfio.conf
```

```
# Make vfio-pci a pre-dependency of the usual video modules
softdep amdgpu pre: vfio-pci
softdep radeon pre: vfio-pci

# Have vfio-pci grab the XFX 580 device IDs on boot
options vfio-pci ids=1002:67df,1002:aaf0 disable_vga=1
```


Update Kernel config:
```
$ update-initramfs -u
```

**Reboot**

### Create the Windows VM

Create a new virtual machine inside of Proxmox. During the wizard make sure to select these things:

* Create the VM using “SCSI” as the Hard Disk controller
* Under CPU select type “Host”
* Under Netwerk select Model “VirtIO”

After the wizard completes, we need to change a few things:

* Next to linking the default DVD-ROM drive to a Windows 10 ISO (if you are passing through to windows), create a
  second DVD-ROM drive and link the VFIO driver ISO , you will need it while installing windows. Make sure the
  second DVD-ROM drive is assigned IDE 0
* Under Options, change the BIOS to “OVMF (UEFI)”
* Then under Hardware click Add and select “EFI Disk”
* The next step I only know how to do using CLI, for this open an SSH session to the Proxmox host and run the
  following:

```
$ cd /etc/pve/qemu-server
$ ls
$ vim 100.conf
```

Change the `machine` line to `machine: pc-q35-3.1`. Ideally this is just `q35` but there is a bug that prevents
the audio device from passing through. See: https://forum.proxmox.com/threads/gpu-passthrough-hdmi-audio.55740/

**Boot the VM**

Install windows and all of the [VirtIO drivers](https://www.linux-kvm.org/page/WindowsGuestDrivers/Download_Drivers)

Make sure you can remote control. I use Chrome Remote Desktop for this, RDP/VNC/etc will work. You **WILL NOT** have
control via the Proxmox Console.

**Shutdown the VM**

### Configure the PCI Passthrough

Change the device ID to whatever you found in the earlier in this process.

```
$ echo hostpci0: 0a:00,pcie=1,x-vga=1 >> /etc/pve/qemu-server/100.conf
```

Try starting again!


### Debugging :(

Whenever I tried booting the VM I'd see the following in the `dmesg` output:

```
[  177.198320] vfio-pci 0000:0a:00.0: enabling device (0002 -> 0003)
[  177.198649] vfio_ecap_init: 0000:0a:00.0 hiding ecap 0x19@0x270
[  177.198658] vfio_ecap_init: 0000:0a:00.0 hiding ecap 0x1b@0x2d0
[  177.198664] vfio_ecap_init: 0000:0a:00.0 hiding ecap 0x1e@0x370
[  177.220653] vfio-pci 0000:0a:00.1: enabling device (0000 -> 0002)
[  178.309130] dpc 0000:00:03.2:pcie008: DPC containment event, status:0x1f01 source:0x0000
[  178.309136] dpc 0000:00:03.2:pcie008: DPC unmasked uncorrectable error detected
[  178.309147] pcieport 0000:00:03.2: PCIe Bus Error: severity=Uncorrected (Non-Fatal), type=Transaction Layer, (Requester ID)
[  178.309149] pcieport 0000:00:03.2:   device [1022:1483] error status/mask=00100000/04400000
[  178.309152] pcieport 0000:00:03.2:    [20] UnsupReq               (First)
[  178.309154] pcieport 0000:00:03.2:   TLP Header: 34000000 0a000010 00000000 80008000
[  178.444946] vfio_bar_restore: 0000:0a:00.1 reset recovery - restoring bars
[  178.460801] vfio_bar_restore: 0000:0a:00.0 reset recovery - restoring bars
[  178.564702] pcieport 0000:00:03.2: AER: Device recovery successful
```

I was able to pass through the *Radeon 5450* but that is not a useful card to pass through. After lots of searching I
found [this post](https://www.reddit.com/r/Amd/comments/ckr5f4/amd_ryzen_3000_series_linux_support_and/ezvahy7/) that
suggested suspending the host (hibernate) and then resuming it after booting. That sounds insane right? Well, it works.

```
$ systemctl suspend
```

Then hit a key on the keyboard.

Boot the Windows VM ... ***IT WORKS***




## Guides I followed to figure this out

* https://www.reddit.com/r/VFIO/comments/cnm7pe/amd_gpu_passtrough_code_43/
* https://blog.quindorian.org/2018/03/building-a-2u-amd-ryzen-server-proxmox-gpu-passthrough.html/
* https://www.reddit.com/r/Amd/comments/ckr5f4/amd_ryzen_3000_series_linux_support_and/ezvahy7/
* https://www.reddit.com/r/VFIO/comments/73og2w/cant_get_amdgpu_driver_to_stop_grabbing_2nd_gpu/
