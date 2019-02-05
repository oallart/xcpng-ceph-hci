# On PCI-Passthrough
The need for performance dictates that the CEPH OSD, which handles raw storage, must have as direct access as possible to said hardware. 

While it is possible to give a virtualized chunk of hardware from xcp-ng (underlying LVM) to a CEPH OSD to manage, it would mean severely reduced performance since it would traverse a shared physical storage, a shared bus interface, and be virtualized through Xen.

PCI-passtrhough gives us the ability to directly pass a PCI device to a running VM. It's a feature introduced in Xen (https://wiki.xenproject.org/wiki/Xen_PCI_Passthrough) that works for network interfaces, controllers etc. It is of special interest to us for its ability to pass an entire storage controller directly to a VM, presenting it to the VM as a native PCI device.

xenserver (which xcp-ng is derivated from) has more information on this here https://xenserver.org/blog/entry/pci-pass-through-on-xenserver-7-0.html

One of the premises of pci-passthrough (correct me if I'm wrong) is the requirement for the hardware to support Vt-d.
More on Vt-d at https://software.intel.com/en-us/blogs/2009/06/25/understanding-vt-d-intel-virtualization-technology-for-directed-io

Some more research into this topic led me to https://en.wikipedia.org/wiki/Input%E2%80%93output_memory_management_unit
In other words, we need an IOMMU (Vt-d for intel, IOV for AMD). This requirement is painfully absent from the official XenServer documentation and I discovered it only after painfully setting up a bunch of spare (old) hardware that did not support it.

## Checking for IOMMU 
```shell
# dmesg | grep IOMM
Using GPFN IOMMU mode, 1-to-1 offset is 0x3e00000000
XEN-PV-IOMMU: Using software bounce buffering for IO on 32bit DMA devices (SWIOTLB)
XEN-PV-IOMMU - completed setting up 1-1 mapping
```
