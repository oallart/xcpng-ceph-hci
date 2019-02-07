# To Pool or Not To Pool

## Why is it an issue
The use of resource pools with xenserver brings a number of somewhat desirable features but I am still not convinced that we entirely *should* or should *not* use them. 

It seems that, according from a number of issues reported to the various Xen related efforts, pools are almost a de-facto situation for people handling more than 1 server. The importance and omnipresence of pools is further demonstrated in Citrix's decision to keep pools (up to 3 nodes) in their highly [controversial decision](https://xenserver.org/blog/entry/xenserver-7-3-changes-to-the-free-edition.html) to takes features out of the "free" edition of xenserver in release 7.3.

For this current project, we need to carefuly weight the pros and cons of [resource pools](https://docs.citrix.com/en-us/xenserver/current-release/hosts-pools.html)(from hereon referred to as simply 'pools') since a pool must be configured early on in the deployment phase.

## Pros and Cons

Here is a list of the pros and cons I assembled from various sources. I am keen to hear more from other users, since we don't use pools in production.
