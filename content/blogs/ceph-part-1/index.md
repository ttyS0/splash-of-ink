---
title: "Ceph - Overview"
summary: "Part One in a series about setting up and using Ceph for my Home Lab environment."
showSummary: true
heroStyle: "basic"
date: "2020-04-03"
series: ["Ceph"]
series_order: 1
---

##  The Problem With Storage


One of the biggest Home Lab challenges is storage. The Home Lab is ideally a microcosm of a larger, organization financed, infrastructure. Many components nicely scale down to the home. Multi core Xeon servers with hundreds of Gigs of RAM can be represented by Intel NUCs and Raspberry Pis. Networking gear can be exactly the same in the case of managed switches, and pfSense can provide sophisticated firewall services while running on a much smaller box. When it comes to storage, however, things aren’t as cut and dry. Features such as LUNs, object stores, and replicated multi-write filesystems don’t really exist in the consumer market outside of some basic functionality baked into expensive NAS units. The options are narrow without also pushing the cost slider well beyond a reasonable Home Lab budget. The point of a Home Lab, after all, isn’t to actually build a data center in the basement. And while the NAS market is great for centralized file shares, it’s not so great with diverse or distributed storage.


I ran across this problem first when I had a vSphere cluster running on a collection of NUCs. In that case I was able to use VSAN, which is basically an object store. It has resiliency functionality that lines up with vSphere itself, and for the most part it worked well, albeit slowly, since the overhead of running a vSphere cluster with VSAN is significant. One other downside is it only exists as a component of a vSphere cluster, so any knowledge of VSAN is fundamentally limited to the realm of VMware. While I’m glad to have gained the experience, it all got left behind when I decided to replace the vSphere cluster with Kubernetes. It was a good lesson for when I started shopping around for a new storage solution.


When switching over to Kubernetes the `storage problem` showed up early. While it’s nice to think of container workloads as a stateless utopia, that notion, like all utopias, is only practical within narrow boundaries. State isn’t a bad thing, and almost all useful applications require some degree of it. Kubernetes storage is accessed via an abstraction layer called [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/). There are a number of built in Volume Plugins, with many of them being targeted at Cloud services. I wasn’t interested in dealing with NFS or iSCSI, so the storage field for my purposes rapidly constricted to either [GlusterFS](https://www.gluster.org/) or [Ceph](https://ceph.io/). ( If I was starting over today, I would also give [MooseFS](https://moosefs.com/) a hard look. ) Glancing over both of them it was clear that GlusterFS was the easiest to get going, so I started there. That lasted for about a day, as my first test of pulling a GlusterFS node out of the cluster lead to data corruption. Home Lab environments are relatively hostile to data and service integrity, and the notion of any system unexpectedly blinking out of existence is a matter of how often instead of when. To add to the chaos, any system needs to also be resilient to the various “what’s this do?” messes that I invariably cause. In the end, Ceph has been a solid solution for me, and survived so many self-inflicted mistakes that should have been catastrophic.

## How does it all work?


Ceph is a distributed storage system. That’s different from both a local and network filesystem. If you’re thinking that using Ceph is similar to using NFS or SMB shares, I want to stop you right there. While NAS style functionality is available, it’s a lot different. In short, if your level of interest with managing storage systems only rises to the configuration of an NFS export or a Synology configuration or using ZFS, then you might want to rethink entering the realm of distributed storage systems. Personally, I find storage solutions interesting in their own right, and correspondingly enjoy fiddling around with storage systems, so Ceph is a good fit for me. My point is running a Ceph cluster in a Home Lab environment is not a `set-it-and-forget-it` style undertaking. It requires care and feeding. The only reason I mention this stuff is I see so many posts extolling how “easy” some technology is based solely on the difficulty of the initial setup. I want to make it clear that while I find Ceph awesome, there’s not an `easy button`. Also, I won’t be covering all the aspects of Ceph and will be glossing over some of the complicated areas. Sometimes that will be because I don’t want to just rewrite [the documentation](https://docs.ceph.com/docs/master/), and other times because I don’t fully understand it myself. Okay, with all that preamble out of the way, let’s get started.

## Monitors, Managers & OSDs


There are three major components of a Ceph cluster. Monitors, Managers, and OSDs (Object Storage Daemon). There are other components, but if you have a serious problem with any of these you’re going to have a bad day. There should also be at least three of each, which is easy to achieve by having a minimum of three Ceph servers. For a Home Lab environment it’s fine to have the different components stacked on each system.


The Ceph Monitor runs as a daemon, and is responsible for maintaining the current state and status of the Ceph cluster. There should be a minimum of three instances in order to maintain quorum. If there are only two it becomes easy to fall into a split-brain situation that is difficult to recover from. If there is only one, then if that instance goes offline the entire Ceph cluster is also offline.


Next up is the Ceph Manager. This is a daemon that runs along side the Monitors, and is responsible for gathering much of the information that is subsequently used by the Monitors. Much of the information displayed by the `ceph status` comes from the Manager daemon. While Ceph Monitors use a quorum to maintain consistency, Ceph Managers use an active-standby mechanism. Only a single Manager daemon is active at any time. In the event that the active daemon goes offline one of the standby instances will immediately take over. Technically two Manager instances is enough, since only one is active at any moment, however it's still a good idea to run an instance on every system that is also running a Ceph Monitor.


Monitors and Managers are basically the accountants and schedulers of a Ceph cluster. Technically if you only have Monitor and Manager processes running you have an online Ceph cluster. The whole point, however, is to have storage available for use. That’s where the OSDs come in. Each OSD daemon is associated with a specific instance of block storage. So, if you have three Ceph servers, and each one has two hard drives attached for use with Ceph, then each Ceph server will have two OSD daemon processes, for a total of six OSD daemons cluster wide. A `replicas=3` setting (which is the default, and should be considered a minimum) means that every block will be live on a primary OSD, and be copied to two others. By default host boundaries are the highest priority, to the two copies will be stored on OSDs that are on different hosts. For example, block-1 will exist on host A, B, and C. The reason it’s so important to have at least three hosts is in order to determine data integrity Ceph needs to achieve a quorum for each block. If `block-1` on host A becomes corrupted, Ceph can detect and correct it because it’s able to say that since block-1 on hosts B & C match that host A is the aberrant copy. If there are only two hosts, then it becomes significantly more difficult, and sometimes impossible, to determine which block is the correct one in the event of a dispute. This also means that if you want 5TB of usable storage you will need 15TB of raw storage distributed as 5TB on each of the Ceph servers.


Under the covers, OSDs use a custom disk format called `Bluestore`, and the association of OSD ID to block device is achieved via LVM and Device Mapper. This means that if you end up getting drives shuffled up on a single system the OSD ID to block device mapping will be maintained. This is especially helpful when using drives attached via USB.


## Coming Up Next


In the next part, I’ll be covering how to stand up a Ceph cluster, including some recommended hardware. I’ll also get into Storage Pools, and two of the storage types that Ceph exposes, RBD and CephFS. I’ll make a mention of RGW, but won’t go into details as that’s a storage type that I don’t actually use.

