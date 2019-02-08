# To Pool or Not To Pool

## Why is it an issue
The use of resource pools with xenserver brings a number of somewhat desirable features but I am still not convinced that we entirely *should* or should *not* use them. 

It seems that, according from a number of issues reported to the various Xen related efforts, pools are almost a de-facto situation for people handling more than 1 server. The importance and omnipresence of pools is further demonstrated in Citrix's decision to keep pools (up to 3 nodes) in their highly [controversial decision](https://xenserver.org/blog/entry/xenserver-7-3-changes-to-the-free-edition.html) to takes features out of the "free" edition of xenserver in release 7.3.

For this current project, we need to carefuly weight the pros and cons of [resource pools](https://docs.citrix.com/en-us/xenserver/current-release/hosts-pools.html)(from hereon referred to as simply 'pools') since a pool must be configured early on in the deployment phase.

## Pros and Cons

Here is a list of the pros and cons I assembled from various sources. I am keen to hear more from other users, since we don't use pools in production.

### Pros
- High availability (restart VMs automatically in case of host failure)
- Automatic load balancing (migrates VM to use resources available and avoid cramming)
- Hosts in pools can access each other's libraries
- Provides heterogenous CPU set migration by automatically masking differences between hosts (specific vendor features required)
- Allows for fast vm migration using shared storage (xenmotion) without copying storage
- Allows for rolling updates
- Apparenty generally desirable
- Probably good to have for a "turn-key product"


### Cons
- Increases complexity
- Updates must apply to pool master first, always
- Cannot be created from hosts that have running VMs (must be shut down first)
- Cannot be created from hardware that it is too different
- Creates issues with LACP (especially for the management interface)
- Forces us to create the pool early on as VMs must otherwise be shut down
- A dedicated interface is recommended
- Shared storage should be available at pool creation (more on this below)
- We have lots of prod systems running standalone absolutely fine

### Also to consider
- xenmotion (live migration) also works outside a pool but storage must be copied over (takes time)
- adding/removing hosts from a pool has an unknown (for us) number of complications

## Pools in the HCI project

The following 3 requirements make it hard for us to use pools:
1. Pool can't be created with VMs running
1. Shared storage should be available before creating the pool
1. Shared storage is created from various VMs that need to run in the pool

If we forgo #2, then we can have a resrource pool, with OSD vms attached locally to their hosts (they should never migrate anyway since they use pci passthrough). In this case, we can use a pool by creating it *before* creating the OSD vms.

It would make sense to use pools in order to take advantage of all the features, especially the shared storage that CEPH will provide.

Decision was therefore made to initiate the project with a pool, and learn a few things along the way.
