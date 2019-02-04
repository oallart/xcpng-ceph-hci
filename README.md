# xcpng-ceph-hci
An attempt at bringing hyperconverged xen virtualization and ceph without openstack

## Key concepts
The idea is to provide a 3-hardware node cluster running each xcp-ng with PCI passthrough enabling direct access to the storage disks for a CEPH OSD. This will in turn propulate a CEPH type SR that can be used by any VM in or outside the cluster. The concept is similar to XOSAN which uses Gluster, but another difference is PCI passthrough to avoid speed bottlenecks (XOSAN uses virtualised storage layers). The availability of CEPH-SR makes it possible to have a solution that doesn't use openshift.

## Key requirements
* xcp-ng (current target: 7.5, due to some 7.6 upstream issues)
* ceph (current LTS: Luminous)
* 3 identical hardware nodes (identical in order CPU differences that can prevent VM migrations)
  * Vt-d enabled (not just VT, this is required for PCI passthrouugh)
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
Experimentation and initial setup
TEsting, a lot

## Next phase
Ansible automatic deployments

