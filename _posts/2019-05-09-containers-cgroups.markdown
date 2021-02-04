---
layout: post
title: "WTF Containers: Part 2, Cgroups"
date: 2019-05-09 
categories: [containers]
---

Any time I read about a new topic I tend to turn to a few 'go to' blogs to see if they've written a primer. One of these is [Julia Evan's](https://jvns.ca/blog/2016/10/10/what-even-is-a-container/) and sure enough she has a post on containers. It's a pretty high level overview and I didn't really get the detail I wanted there but I recommend it as a primer on namespace/cgroup overview.

## Cgroups: Y tho

On a fundamental level cgroups are there to help you manage resource allocation. It is a 'control group' and it allows you to bunch processes together and define the resources they can access. Much like namespaces and pretty much everything in the linux kernel child processes inherit their parent processes memberships.

Why would you want to use them? If you're looking to isolate resource utilisation and visibility it stands to reason that you might just think about running virtual machines. Cgroups are providing some similar levels of resource limitation but they eliminate the overhead of running an entire VM (you have to perform the virtualisation overhead, have a host and guest kernel, pass through system calls etc. lots of extra work just to stop a rogue app from eating your RAM). So we gain some performance and flexibility since we can adjust cgroups in runtime, however we have to be conscious that we sacrifice some isolation. That's what we'll come back to later when we think about security of containers.

We can use cgroups to do a number of things that are neat and more flexible too, like resource limiting, but also priortization, accounting for each cgroups usage and control (freezing stopping and restarting)

## How do they work?
As usual the manpages provide us a great deal of [info](http://man7.org/linux/man-pages/man7/cgroups.7.html). Straight away I found something I didn't expect, and that was that the interface is provided via an pseudo filesystem, cgroupfs. News to me! We can view what the existing groups look like and amend them as well as create new ones by interacting with the cgroups filesystem.

As far as the actual implementation, I found there to be two logical separations, the implementation of the grouping, and then the tracking and limiting of resources based on that grouping. We refer to the grouping as cgroups and the implementation of resources are via subsystems or controllers.

Cgroups work entirely based on hierarchies, which are arranged as subdirectories. If you're using cgroups v1 (the default) then you're going to be able to have multiple different hierarchies with multiple top levels.

Each cgroup directory represents a cgroup. There's then files in there that control certain factors. These include:

- tasks (sorted by PID - add a pid to add it to the cgroup),
- cgroup.procs (list of thread group IDs),
- notify\_on\_release flag,
- release_agent (the path used for release notifications). You don't super need to go into these, unless you'll be hand creating them.

Notify on release is worth discussing simply to be aware of, it means, when the last thread completes run whatever is at the release_agent path in the hierarchy's root directory. You might use this to return a signal to an application and maybe docker utilises this for when a container stops(?), something to think about.

Interesting security aside is that you can set extended attributes for the cgroup filesystem, particularly around trusted and security types. Both require ```CAP_SYS_ADMIN``` to set, and these attributes are stored in kernel memory so we try to minimise them. SELinux sets these as does systemd for some metadata. There's a fair bit of discussion around ```CAP_SYS_ADMIN``` and container security so this was a relevant note to come back to.

## Practical Usage
Theory is nice but I want to break stuff. Neat thing, as always with Linux, is most of this is just echoing stuff to files, like this:
```
sudo mkdir /sys/fs/cgroup/memory/mycgroup
```


This will then create a directory for that new cgroup. Linux is smart here and creates all the constraint files available for you by standard. We can then go ahead and set a limit under it, so maybe we want that to be memory.

```
echo 200000 >  /sys/fs/cgroup/memory/mycgroup/memory.limit_in_bytes
```

No matter what you put in there it will pick to the nearest multiple of your page size.

We have a limit now, but we need to add a process into that cgroup, we can do this via amending cgroups.procs to contain our PID. Same thing here, just echo it into the file.
Now we can look at it via ps and ```ps -o cgroup $PID``` and find out what it lives in.

You can configure cgroups persistently in cgconfig.conf too. [Linuxjournal](https://www.linuxjournal.com/content/everything-you-need-know-about-linux-containers-part-i-linux-control-groups-and-process) has a handy guide on practical interaction with cgroups via CLI.

That is kind of the limit of 'need to know' in order to utilise cgroups, or at least to ground us for later thinking about containers and container security and performance.

Everything from this point on is just my own relentless desire to know more stuff about how the kernel works.

## What's up with the kernel?
So how does this work at kernel level? Obviously I want to know. As always I had to turn to [kernel.org](https://www.kernel.org/doc/Documentation/cgroup-v1/cgroups.txt) for that. The main thing that the kernel is doing here is creating the subsystems.

Every subsystem needs a cgroup_subsys object. This initialises it as a subsystem. All cgroup modifications are controled via a mutex lock, and subsystems will lock and unlock this depending on their tasks.

The subsystems themselves will add an entry into ```linux/cgroup_subsys.h``` so you can effectively see all the available subsystems by checking entries in that file.

Yes but how does it *work*. I wanted to know how these subsystems worked so I ended up browsing the linux kernel mailing list archive for cgroups [here](https://www.spinics.net/lists/cgroups/) and didn't this as usual lead me into a memory rabbit hole and made things more confusing. In the end I went and started with CPU restrictions first. I've found generally that memory subsystems are pretty complex, and not to say that CPU isn't complex but it's often an easier entry point into a new feature. I read through some more doco and particularly the[https://www.kernel.org/doc/Documentation/cgroup-v1/cpusets.txt](cpuset) documentation.

Thinking about this starting with CPU instead of memory was actually a better approach. That's because CPU resources already have a number of different ways to manipulate them that I'm already familiar with, primarily the scheduler and CPU node binding, Cgroups essentially just co-opt these pre-existing features.

In CPU cycles case this means they can be provided a minimum number of cycles, or if you would like an upper limit. CPU sets work similarly in that it takes the existing ```sched_setaffinity``` feature and ```set_mempolicy``` that are used for NUMA and CPU affinity and then extends them to be more precise. Particularly of note is that it will mask the requested CPUs in ```sched_setaffinity``` to just those allowed in the tasks subset and the same in ```set_mempolicy``` system calls. ```page_alloc.c``` and ```vmscan.c``` also get restricted to only allowed nodes. A neat thing to note here is you can actually see exactly what it's allowed to 'see' by looking at ```/proc/<pid>/status``` and you'll see something like:

```
  Cpus_allowed:   ffffffff,ffffffff,ffffffff,ffffffff
  Cpus_allowed_list:      0-127
  Mems_allowed:   ffffffff,ffffffff
  Mems_allowed_list:      0-63
```
The cpuset documentation has a load of really interesting information about spreading pagecache out and how to manage memory access as part of restricted sets.

The [memory](https://www.kernel.org/doc/Documentation/cgroup-v1/memory.txt) subsystem documentation is super interesting as well. Particularly around how you can set up OOM notifiers, and also monitor memory pressure within a cgroup. I think in an ideal world you would be able to then essentially create a cgroup where you have no OOM killer, and where when notified of memory pressure you create your own method for recovery.

This would allow you to keep OOM for the holistic top level system (where we do want the server to remain on at all times) but then within a specific grouping to say never let this OOM. Now the flip side is that if it doesn't OOM the application will just hang and you're likely to end up going in there and restarting the process anyway, so you might as well have let OOM kill it in the first place.
A possible good choice though if you have circumstances where an occasional bottleneck tends to invoke OOM but simply waiting a short time would have allowed your application to pick up operation again, perhaps some kind of saturation on pagecache write out etc or similar.



## NUMA NUMA NUMA
What about NUMA? NUMA is historically used for the same kind of affinity of scheduling and memory node availability. It sounded to me like you could potentially get into a strange situation with affinity stepping on its own toes. Well turns out cpuset builds in support or that essentially. The takeaway is that if you're using NUMA in conjunction with cgroups you should be aware of NUMA allocation, you'll actually end up allocating NUMA nodes to them as cpuset cpus.
[This](https://sthbrx.github.io/blog/2016/07/27/get-off-my-lawn-separating-docker-workloads-using-cgroups/) has some neat details, but essentially use ```lscpu``` to return your NUMA nodes grouping by CPU number, and then ensure that those CPUs are reflected in your cpusets for your cgrouped workload. As it notes, if you're using a container managment system like docker it's going to intelligently subdivide out those CPU sets to each container. Nice to be able to abstract that out.



## Cgroups V2

Cgroups v2 is basically a 'better' implementation, but it has some significant changes. I don't see too many people uptaking this yet by the sounds of things but I guess it's neat to know in general since it gives some idea as to the limitations of cgroups v1.

The key difference is that in cgroups v1 you had a bunch of different hierarchies, and in v2 you just have one main hierarchy. Frankly this makes more sense.

Each controller (per resources, so CPU, RAM etc) is mounted automatically when you initiate the cgroup. So they're mounted at /mnt/cgroup2 for example. Systemd will automatically mount the cgroup2 fs to ```/sys/fs/cgroup/unified``` during boot.

There's 6 main controllers in cgroups2: io, memory, pids, perf\_event, rdma, cpu

Within each of these live subtrees:

- cgroup.controllers: A read only file that exposes the available controlers in this cgroup. This matches the contents of cgroup.subtree_control in the parent cgroup.
- group.subtree_control: A list of active controllers in the cgroup. It will be a subset of the .controllers file. Active controllers are modified by writing strings to the file with + and - to enable and disable.

Sounds confusing, but actually it's alright. Basically what's in cgroup.controllers tells you what you *can* enable in cgroups further down the hierarchy. If you had multiple cgroups under each other you could then reduce the available controllers as you made your way down the hierarchy. This is there to kind of shoehorn v1s multiple hierarchies into a single hierarchy in an organised fashion that doesn't then get really confusing.

There's a few important implementation notes here, firstly is really around these hierarchies, I suspect it's easy to get twisted into a situation where you have a cgroup that's inherited a parent setting and you want a controller enabled but can't because of the hierarchy cascades. There's the ability to limit the number of cgroup descendents which is probably a neat thing to do near your top level cgroup to make sure you don't get into some kind of russian doll situation.

Secondly, the 'no internal processes' rule that's called out sounds pretty important. This says that apart form the root cgroup, processes may reside only in leaf nodes (cgroups that don't contain child groups). This helps you get out of that confusing 'why does this process get this amount but this other one doesn't. The main takeaway I got from this is keep your cgroups neat, and also if you ever see a directory that looks like ```/cgroupX/leaf``` it means 'these are some processes that I wanted to run in cgroupX but I also had child cgroups below that cgroupX so I've logically separated these into their own node.

I think this all sounds really complicated but actually isn't when you think of it as just cascading buckets of resources. I think that the no internal process rule goes for v1 as well, but I didn't find clarity on that. I think it mainly lives in v2 again as a side effect of trying to provide better clarity of what controllers are enabled for a given process when you're structuring everything in a single hierarchy.
