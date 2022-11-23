---
title: "A Tale of Two Home Labs"
summary: "A transition form vSphere to Kubernetes"
showSummary: true
heroStyle: "basic"
date: "2018-05-31"
series: ["Home Lab"]
series_order: 1
---

### [And then the dinosaurs came…](https://www.youtube.com/watch?v=MabLtuTpDw8)

In one of my previous jobs I worked for a small tech company where approximately half of the employees worked remotely. I was one of those remotes, and perhaps one day I’ll write up my thoughts on being a remote employee. From having worked in larger companies, I was used to having a playground area that I could use as a learning tool. The small tech company didn’t have such an environment, which made sense in their case. Since I was at home all the time, I decided to build up a home lab.

It’s a bit of a balancing act when attempting to stand up environments that roughly approximate something that might be found in a production environment. The overhead and maintenance costs just don’t make any sense. I had been down that road before. In the early 2000s I had a T1 line run to my house, and setup a small room in the garage as a mini data center. It was a great learning experience from both a technological and a “yeah, this really isn’t such a great idea” point of view. Thankfully, the technology has progressed immensely since the early 2000s, and now it’s possible to get power compute in small quiet and relatively power efficient components.

I started with the virtualization platform, which at the time was vSphere, as it was looking like the company was going to be using VMware to migrate off of a legacy Xen infrastructure. To that end, I signed up for the [VMUG Advantage](https://www.vmug.com/VMUG-Join/VMUG-Advantage) program, which is an amazing deal. It provides for personal use almost the entire suite of VMware products. I was most interested in a vSphere cluster backed by VSAN. From previous jobs I had extensive experience with vSphere, but I was years and major versions behind.

### [Virt! Virt! Virt!](https://youtu.be/RB6MvSEaMKI?t=6m6s)

Having decided on vSphere and VSAN, the next step was finding some hardware that could support it. While cost wasn’t a critical issue, I also wasn’t interested in spending a ton of money. It’s very easy for me to slip right into the “oh hey, I’m  building a mini data center” mode. In the end, I settled on Intel NUCs. The SkyLake models had just been released, so I gathered up three of them with M2 and SATA SSD storage to accommodate the capacity and cache tiering features of VSAN. As a side note, I also decided to put in a real firewall, router, and switch. The firewall and router functions are handled by a small NUC-like system running pfSense, and I picked up a [Cisco SG300](https://amzn.to/2LaLXnd) for the switching.

I got everything setup, and was quite pleased with the performance of the vSphere cluster with VSAN. Each NUC has 32GB of RAM, so the overhead of vSphere pieces (vCenter, NSX, Harbor, et al) created a bit of a constraint, especially when I started entertaining the notion of standing up an ELK cluster on the cluster. Storage really wasn’t an issue, especially with VSAN compression and deduplication. CPU wasn’t an issue either, since it was a home lab environment and it rarely got hit with anything hard. I also had a pile of Raspberry Pis that I couldn’t directly integrate, but was able to use as an extension of some of the services I was running, like [Consul](https://consul.io).

### [A New Career In The Same Town](https://www.youtube.com/watch?v=kZssy0IiyMA)

After a year or so, I left the small tech company. It boiled down to mutual bad timing. Where I wanted to be in my professional development just didn’t line up with their vision of their infrastructure. I ended up going back to a large environment where a handful of people that I had previously worked with had landed. Sometimes it’s devil you know. In any event, I kept the vSphere cluster running even though it didn’t have any direct application to anything I was working on at the new job, where my primary focus was on cloud services.

I had already been playing around with Docker, but was wanting to branch out into either Swarm, Kubernetes, or some other orchestration. To that end, I finally settled on Kubernetes, and started looking at getting it setup on a collection of VMs running on my vSphere cluster. As is usual with my learning process there were a *lot* of starts, stalls, stops, scorched earth, restarts, reevaluation, reconsidering, and so on. During one of those cycles I realized that one of the things working against me was the vSphere cluster itself and the whole management thinking that went along with that. I was managing images and operating systems via configuration management that was all still rooted into decades old thinking. I’m sure that having to adjust my thinking about infrastructure because of *cloud* played a big part.

There’s a constant struggle in tech between the [“The Appeal to Tradition”](https://en.wikipedia.org/wiki/Appeal_to_tradition) and [“The Appeal to Novelty”](https://en.wikipedia.org/wiki/Appeal_to_novelty). Typically, I tend to tilt towards the former, having learned early in my career that the hot new tech also comes with its own collection of hot messes. That tends to foster an internal resistance to radical changes in thinking about solving problems. Add some sprinkling of rationalization, and *boom*.

> “I’m not stuck in my ways, just look at these new vSphere features I’m using! I just need more hardware for what I want to do. It’s not a design problem.”

### [And now for something completely different.](https://www.youtube.com/watch?v=FGK8IC-bGnU)

During my struggle to figure out how to best put a virtualized application layer on top of a virtualized hardware layer, I ultimately decided to just get rid of the latter piece. This is a technique that I usually come around to after reaching some indeterminate threshold of struggling with trying to make all the dice and marbles stack up nicely.

> “Wait, what if I just got rid of the marbles?

Part of my struggle with Kubernetes involved the way I tend to learn things. The installation documentation is clearer when one understands the components, but I have a difficult time understanding things until I’ve had a chance to directly interact with them. I learn best when I can smash something to bits and then poke around at the pieces. If there is a prerequisite for putting something together before I’m able to smash it up, then there’s a bit of a chicken and egg problem. While I started down the path of trying to translate [kubernetes the hard way](https://github.com/kelseyhightower/kubernetes-the-hard-way) to a bare metal installation, I ultimately settled on using [kubeadm](https://github.com/kubernetes/kubeadm). While easier, it was far from a seamless process, especially since I had three NUCs to work with and absolutely wanted a multi master configuration.

My first stab was a single master with `etcd` running in pods, and only a single Kubernetes master. A number of rebuilds later, I finally got to an multi master and multi architecture cluster. Currently the three NUCs are Kubernetes masters, and my collection of RaspberryPis are member nodes. I still have `etcd` running as an external service co-resident on the three NUCs. Once `etcd` was setup and functioning properly, I’ve taken as close to a hands-off approach as possible with it. It’s the one piece that still seems out of place. Oh, and for persistent storage, I setup a [Ceph](https://ceph.com) cluster, also co-resident on the NUCs. In one of the early iterations, I used GlusterFS, and while it was easier to get started, it was problematic and difficult to repair. I’ve inadvertently smashed up Ceph a few times, and was able to recover every time with minimal fuss. I’ve gone through a few upgrade cycles of all the components, and am reasonably confident in the resiliency of the new Home Lab.

The only major thing I need to do with the Home Lab now is get it moved into the basement, but that’s dependent on a number of house jobs to be completed. In the meantime, the whole setup is resting comfortably (mostly) on a shelf of an IDEA table.

### Now What?

So I have a Kubernetes cluster running, and I’ve put a handful of services on it. Now What? I really don’t know. Becoming proficient in one or more programming languages is still something that I feel I need to do. Somehow I managed to get this far with hacking up scripts in various languages that just sat around and did what they needed to do. The notion of making something that I would consider an actual application seems other worldly and scary. That’s the strongest sign that it’s the direction I should be pointing.

As I get into the habit of posting things to this space, I’ll consider putting together some HOWTOs for the technical stuff. In the meantime, I’ve been putting a decent amount of things on [GitHub](https://github.com/ttyS0).

