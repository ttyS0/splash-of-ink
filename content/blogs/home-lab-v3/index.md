---
title: "Home Lab v3"
summary: "The Home Lab gets a full renovation."
showSummary: true
heroStyle: "basic"
date: "2022-01-30"
series: ["Home Lab"]
series_order: 3
---

It's been quite a while since I reworked my Home Lab environment. 

{{< article link="/2020/03/home-lab-v2/" >}}

While minor changes happened along the way, like adding more storage and changing enclosures for the RaspberryPis, it hadn't been completely torn down and rebuilt for a couple of years. While I'm not a devout advocate of the "everything needs to be rebuilt from scratch every `$N` `$time_period`" philosophy, at times it can be a valuable exercise. 

In my case, we were planning a massive house remodel that would have necessitated us moving out. To prepare for that, I migrated smaller workloads to a [K3s](https://k3s.io) instance running on cloud VMs, and collapsed the large workloads into a single local NUC, also running K3s. This meant my [Ceph](https://ceph.io) cluster needed to be taken down, so all the data was either migrated to the cloud instances or to a ZFS array attached to the local NUC. After all this was done, the plans for the remodel basically fell apart. That meant there was no longer a need to move out, and I could pull everything back to a local cluster.

While I considered standing up an environment just like the previous one, instead I took some time to think about the parts that had worked well, along with the parts that hadn't. 

Briefly I considered sticking with a K3s cluster, but that didn't last long. One of the reasons I like to run Kubernetes is to gain expertise that can be used elsewhere. While K3s is nice, it is also different enough that I would worry about missing elements that only exist in an actual Kubernetes cluster. 

With the previous cluster, I ran with an HA control-plane. I'm glad I learned how that configuration works, but for my Home Lab environment it is overkill. Pods don't cycle that much, and if something is unable to relaunch immediately because the control-plane is offline for a while that's not a serious issue. My Home Lab does _not_ have five nines. 😎 So for the new cluster I decided to go with a single control-plane node.

The next major consideration was storage. Previously I ran an external Ceph cluster on four dedicated NUCs, each with its own storage enclosure and internal SSD. The internal SSDs were used to form a "fast" storage pool, while the enclosures with spinning disks formed a "bulk" storage pool. Depending on the workload's needs, an RBD was created from one of the pools, or a CephFS mount was used in the case of large bulk storage. [Ceph-CSI](https://github.com/ceph/ceph-csi) was used for dynamic provisioning, and on the whole things worked well. It did, however, mean that not only did I need to maintain a Kubernetes cluster, but I also needed to maintain a full blown Ceph storage cluster. 

I've always liked storage technology in general, so I enjoyed learning the ins and outs of Ceph. However, when reconsidering storage for a new Kubernetes cluster, I wasn't so sure it was worth it. After investigating the dynamic storage provisioners that would make sense in my Home Lab environment, it was pretty clear that for provisioning block storage, Ceph really is the best option. 

In my case, however, the vast majority of my storage lives in the "bulk" storage realm. That is a scenario where the dynamism of storage provisioning is less important. I started to look into NFS, but not from a provisioner point of view, but rather a static PV/PVC configuration for the few mounts I would need. After some tire kicking, I decided that NFS would be fine for bulk storage. For that, I decided to take a NUC, put FreeBSD on it, and attach three of the enclosures. This would allow for a ZFS  `RAIDZ2` striped array, where each `RAIDZ2` vdev was an enclosure. Initially I considered using striped mirrors, but the improved performance wasn't worth the higher risk of data loss. With striped mirrors, if I lost the _wrong_ two drives, the whole storage pool would be gone. In the case of a striped `RAIDZ2` configuration, it becomes the wrong _three_ drives. It may sound like a minor distinction, but it is a trade off that I was willing to make. Effectively, the system becomes a DIY NAS with shares provided by NFS, Samba, or both. 

Now that the bulk storage was squared away I had a new Ceph dilemma. For SSD backed RBD storage a separate Ceph cluster really didn't make any sense. After much deliberation, I decided to revisit [Rook](https://rook.io). I had tried it before, and abandoned it. Part of the reason was my general lack of understanding of both Kubernetes and Ceph at the time. There are still aspects of Rook that I'm not a fan of, but since all I now required was basic RDB storage backed by an equally basic `replicas=3` storage pool, I was willing to give it another go.

With Kubernetes and storage design decisions out of the way, the next thing to decide was whether I would again run a mixed architecture cluster. When I was running a separate Ceph cluster, that consumed four of the NUCs, just for storage. To provide some extra worker nodes in the Kubernetes cluster, I used RaspberryPis. This worked out well, and helped me gain some expertise around dealing with a mixed architecture cluster. Since Rook is a Ceph instance that runs inside the Kubernetes cluster, and since I'm only using SSDs for Ceph, and since I'm only using a single control-plane node, the need for the RaspberryPis as worker nodes effectively vanishes. 

The hardware inventory, at the end, looked like this:

|System Type|OS|RAM|Purpose|
|-|-|-|-|
|NUC 7|FreeBSD 13|64GB|ZFS based NAS|
|RPi4|FreeBSD 13|8GB|ZFS based backup NAS|
|NUC 7|Ubuntu 20.04 LTS|32GB|K8s control-plane & Ceph node|
|NUC 7|Ubuntu 20.04 LTS|32GB|K8s worker & Ceph node|
|NUC 7|Ubuntu 20.04 LTS|32GB|K8s worker & Ceph node|
|NUC 8|Ubuntu 20.04 LTS|64GB|K8s worker|
|NUC 8|Ubuntu 20.04 LTS|64GB|K8s worker|
|NUC 8|Ubuntu 20.04 LTS|64GB|K8s worker|

I removed the `control-plane` taint, so effectively there are six nodes that can have work scheduled. Node labels are used to make sure Rook (Ceph) is only deployed to the `NUC 7` systems, since they are the ones with SSDs that are used by the Ceph OSDs. 

I'm going to write other posts detailing how I did the configurations, including the FreeBSD systems, and how I incorporated [Kustomize](https://kustomize.io) into the reworked manifests.

