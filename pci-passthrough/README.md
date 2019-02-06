# On PCI-Passthrough
The need for performance dictates that the CEPH OSD, which handles raw storage, must have as direct access as possible to said hardware. 

While it is possible to give a virtualized chunk of hardware from xcp-ng (underlying LVM) to a CEPH OSD to manage, it would mean severely reduced performance since it would traverse a shared physical storage, a shared bus interface, and be virtualized through Xen.

PCI-passthrough gives us the ability to directly pass a PCI device to a running VM. It's a feature introduced in Xen (https://wiki.xenproject.org/wiki/Xen_PCI_Passthrough) that works for network interfaces, controllers etc. It is of special interest to us for its ability to pass an entire storage controller directly to a VM, presenting it to the VM as a native PCI device.

xenserver (which xcp-ng is derivated from) has more information on this here https://xenserver.org/blog/entry/pci-pass-through-on-xenserver-7-0.html

One of the premises of pci-passthrough (correct me if I'm wrong) is the requirement for the hardware to support Vt-d.
More on Vt-d at https://software.intel.com/en-us/blogs/2009/06/25/understanding-vt-d-intel-virtualization-technology-for-directed-io

Some more research into this topic led me to https://en.wikipedia.org/wiki/Input%E2%80%93output_memory_management_unit
In other words, we need an IOMMU (Vt-d for intel, IOV for AMD). This requirement is painfully absent from the official XenServer documentation and I discovered it only after setting up a bunch of spare (old) hardware that did not support it.

## Hardware used
My current test setup uses 3 Supermicro X9DRW with a PCIe LSI SAS2308 controller. A rewiring of some of the front disk bays is done to have the SAS controller manage these bays instead of the _onboard_ controller, making use of the SAS->SATA compatibility. 

## Checking for IOMMU 
The following shell command should return something.
```shell
# dmesg | grep IOMM
Using GPFN IOMMU mode, 1-to-1 offset is 0x3e00000000
XEN-PV-IOMMU: Using software bounce buffering for IO on 32bit DMA devices (SWIOTLB)
XEN-PV-IOMMU - completed setting up 1-1 mapping
```

## PCI Bus Id
We need to identify the correct ID in [BDF](https://wiki.xen.org/wiki/Bus:Device.Function_(BDF)_Notation) format and *make sure it is separate from the controller managing the base OS storage*.
We can have an identical Bus and Device as long as the _Function_ id is different. In my case, using an add-in cars solves my problem.

`lspci` will tell you what you need. In my case:
```shell
# lspci | grep -e SAS -e SATA
00:1f.2 SATA controller: Intel Corporation C600/X79 series chipset 6-Port SATA AHCI Controller (rev 06)
05:00.0 Serial Attached SCSI controller: LSI Logic / Symbios Logic SAS2308 PCI-Express Fusion-MPT SAS-2 (rev 05)
06:00.0 Serial Attached SCSI controller: Intel Corporation C602 chipset 4-Port SATA Storage Control Unit (rev 06)
```

In this case, *00:1f.2* is the onboard controller and *05:00.0* is my add-in card.
*06:00.0* is another onboard controller that would be perfect but I did not manage to make use of (requires a different mini-sas cable but even with that, drives were not seen by the base OS). Depending on your hardware, you may not need a separate add-in card at all.

## Telling XCP-ng to ignore a given PCI device
There are two ways to tell xen to ignore a given PCI device:
1. Modifying grub directly
2. Using a specific command line tool

In both cases a reboot is required

### Modifying grub directly
Simply edit `/etc/grub.cfg` and add `xen-pciback.hide=(05:00.0)` (replace with your own bus ID) to the command line

### Using a specific command line tool
Again, using your own device ID, use
```shell
/opt/xensource/libexec/xen-cmdline --set-dom0 "xen-pciback.hide=(05:00.0)"
```
You can then check `/etc/grub.cfg` for the change made.

## Passing the device to a given VM
Create a VM with any method you normally use. The VM must be created first, AFAIK there is no way to add the device at creation time. `xe` commands are used to glue the PCI device ID to the vm as follows (VM UUID is required):
```
xe vm-param-set other-config:pci=0/0000:05:00.0 uuid=<VM UUID>
```
fixme: VM reboot required?

If all goes well, the VM will show the device in `lspci`, quite possibly with _a different id_, in my case *00:06.0*
```
00:06.0 Serial Attached SCSI controller: LSI Logic / Symbios Logic SAS2308 PCI-Express Fusion-MPT SAS-2 (rev 05)
```
Disks associated with that controller should then be listed in `/dev/disk/by-path/` and any activity showing in `dmesg` on the *vm*, not on the *host*.

Here's the output of `dmesg` on the VM when I hotplug a disk on the controller with PCI ID 00:05.0 (on the host)
```
[82806.563511] scsi 2:0:0:0: Direct-Access     ATA      INTEL SSDSC2CW24 400i PQ: 0 ANSI: 6
[82806.563521] scsi 2:0:0:0: SATA: handle(0x0009), sas_addr(0x4433221104000000), phy(4), device_name(0x0000000000000000)
[82806.563524] scsi 2:0:0:0: enclosure logical id (0x500605b005cc6790), slot(7) 
[82806.563797] scsi 2:0:0:0: atapi(n), ncq(y), asyn_notify(n), smart(y), fua(y), sw_preserve(y)
[82806.563803] scsi 2:0:0:0: qdepth(32), tagged(1), simple(0), ordered(0), scsi_level(7), cmd_que(1)
[82806.598349] scsi 2:0:0:0: Attached scsi generic sg1 type 0
[82806.623408] sd 2:0:0:0: [sda] 468862128 512-byte logical blocks: (240 GB/223 GiB)
[82806.703570] sd 2:0:0:0: [sda] Write Protect is off
[82806.703575] sd 2:0:0:0: [sda] Mode Sense: 7f 00 10 08
[82806.723585] sd 2:0:0:0: [sda] Write cache: enabled, read cache: enabled, supports DPO and FUA
[82807.003575] sd 2:0:0:0: [sda] Attached SCSI disk
```

`/dev` shows the following now:
```
lrwxrwxrwx 1 root root 9 Feb  6 16:07 pci-0000:00:06.0-sas-0x4433221104000000-lun-0 -> ../../sda
lrwxrwxrwx 1 root root 9 Feb  6 16:07 pci-0000:00:06.0-sas-phy4-lun-0 -> ../../sda
```

SUCCESS! My physical disk is now handled directly by the VM, perfect for my CEPH OSD. Next step, CEPH.
