*_This project is currently in the early stages and consists mostly of documentation and notes_*

# xcpng-ceph-hci
An attempt at bringing hyperconverged [hci](https://en.wikipedia.org/wiki/Hyper-converged_infrastructure) xen virtualization and ceph without [openstack](https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/12/html-single/hyper-converged_infrastructure_guide/index).

## Key concepts
The idea is to provide a 3-hardware node cluster running each [XCP-ng](https://xcp-ng.org/) with [PCI passthrough](https://xenserver.org/blog/entry/pci-pass-through-on-xenserver-7-0.html) enabling direct access to the storage disks for a CEPH OSD. This will in turn propulate a [CEPH type SR](https://github.com/xcp-ng/xcp/wiki/Ceph-on-XCP-ng-7.5-or-later) that can be used by any VM in or outside the cluster. 

The concept is similar to [XOSAN](https://xen-orchestra.com/docs/xosan.html) which uses Gluster, but another difference is PCI passthrough to avoid speed bottlenecks (XOSAN uses virtualised storage layers). The availability of CEPH-SR makes it possible to have a solution that doesn't use openstack.

## Key requirements
* xcp-ng (current target: 7.5, due to some 7.6 upstream issues)
* ceph (current LTS: Luminous)
* 3 identical hardware nodes (identical in order CPU differences that can prevent VM migrations)
  * [Vt-d](https://software.intel.com/en-us/blogs/2009/06/25/understanding-vt-d-intel-virtualization-technology-for-directed-io) enabled (not just VT, this is required for PCI passthrouugh) or equivalent [IOMMU](https://en.wikipedia.org/wiki/Input%E2%80%93output_memory_management_unit)
  * At least 2x GB interfaces (10GB interfaces preferred), 1 dedicated to internode replication
  * Enough memory to accomodate XCP-NG, base CEPH images (OSD, Monitor, Manager) and other guest VMs
  * PCI dedicated controller for passthrough, separate from the one used for the XCP-NG base SR
  
## Other considerations
### Redundancy
XCP-NG v7.5.0-2 and above allows for RAID1 installs for the base system, providing survivability for a single disk failure.
Power supply redundancy and network interface redundancy can be factored in with the type of hardware but this is not beign explored at the moment in this project.

Ceph allows for a certain number of configurations but the one we are using for now is a single OSD running is a single VM tied to a given node. It is possible to run mutliple OSDs, one for each disk, in that same VM (due to PCI passthrough, only that VM can access the PCI bus ID hosting the controller for the drives). The risk being, if that single VM (or the host itself) becomes unavailable, multiple OSD will fail at the same time. For now, we are using a 3 node, single disk OSD setup for experimentation. That setup should still survive with a single node total failure.

### For more performance
The 3 node setup can be expanded with more nodes, giving more OSDs and more capacity and performance (read and write). 

## Current status
* Initial setup and experimentation
  * Hardware setup
  * Base OS setup
  * PCI passthrough configuration and testing
  * CEPH cluster VMs setup and configuration
  * CEPH-SR configuration
* Testing, a lot

## Other things
* clean OSD maintenance state on host shutdown
* pool settings for proper shared storage

## Next phase
* Ansible automatic deployments
  * CEPH has support for ansible already
  * xen is getting support in ansible in the next (2.8) release [it seems](https://xcp-ng.org/forum/topic/159/deploy-vms-using-ansible/10)
 
A turnkey solution would be nice, putting it all together. Ideally it would perform the following tasks;
* Supplying 3 ip addresses and the root password for the XCP-ng nodes, provision a config-manager VM
* config-manager then 
  * sets up an ansible repository
  * sets up all 3 hardware nodes under ansible control
  * sets up pci-passthrough
  * sets up a ceph cluster with necessary VMs
  * sets up a shared SR with CEPH as backend
  * sets up a monitoring and control VM
  * applies monitoring changes to all the systems
  
Further improvements and possibilities are endless.
