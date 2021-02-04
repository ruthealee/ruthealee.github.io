---
layout: post
title: "WTF Containers: Part 1, Namespaces"
date: 2019-04-24 
categories: [containers]
---
Having avoided them for the most part for the previous few years I am now beginning to need to know about containers, and additionally Kubernetes. This series will be a wandering (probable misunderstanding) through current documentation and other smarter peoples' blogs that hopefully ends up with me understanding some key concepts better. We'll see.


One of the traditional 'descriptions' of containers is as these standalone boxes that your application runs in. That's a nice way to envisage them, but it's fundamentally pretty inaccurate. The first half of Alice Goldfuss' Lead Dev talk, [The Container Operator's Manual](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=1&cad=rja&uact=8&ved=2ahUKEwiKiNbKrdnhAhWQAHwKHb5tBSwQwqsBMAB6BAgIEAQ&url=https%3A%2F%2Fwww.youtube.com%2Fwatch%3Fv%3DsJx_emIiABk&usg=AOvVaw17sJwMobpq6MqRg6Y2LgEy) does a great job of explaining this. After watching this I read https://blog.jessfraz.com/post/containers-zones-jails-vms/ which went on and describes essentially, different approaches to achieve isolation.

I then went off and tried to read another post about hard multitenancy which was really interesting but I didn't understand at all. So my starting point here is:

- Containers are basically ways to isolate processes on a shared host
- Isolation can be achieved by combining namespaces and cgroups
- Namespaces control what a process can see on the OS
- Cgroups control what resources a process can utilise on the OS
- Linux Security Modules or LSM deal with some security aspects (AppArmor, SELinux etc.)

That was a distinction I hadn't quite picked up in the chatter around containers and it's a pretty important one, the next step then seemed to be better understanding how namespaces and cgroups work. I'm approaching this whole thing from an angle of being interested in what kind of security and scaleability concerns we should have when implementing containers. The aim is that over the next few months I'll get a better grip on container architecture and theory, and that will lead to security and best practice and out the end with Kubernetes hopefully. None of this is really going to be about implementation so much as it is architecture, no doubt I'll get some things wrong and come back and revisit this with corrections later.


### Let's start with Namespaces

Linux has a number of namespaces that are built into it, you can combine these to create process isolation. The available namespaces are:

      Namespace   Constant          Isolates
       Cgroup      CLONE_NEWCGROUP   Cgroup root directory
       IPC         CLONE_NEWIPC      System V IPC, POSIX message queues
       Network     CLONE_NEWNET      Network devices, stacks, ports, etc.
       Mount       CLONE_NEWNS       Mount points
       PID         CLONE_NEWPID      Process IDs
       User        CLONE_NEWUSER     User and group IDs
       UTS         CLONE_NEWUTS      Hostname and NIS domain name

Here's one confusing thing, there's a cgroups namespace, which controls what cgroups your process can see (This is I think why I didn't realise these were separate things)

Each of these functions in a slightly different way to achieve its goals, but we can look at one example here, which is the process namespace. This is probably the most interesting to us because it's a pretty fundamental way to ensure that processes can't see one another (important for containers!)
Linux boots, and the root process starts up PID 1. All the PIDs of any given task are in stored in ``` struct pid ```


Now, with namespaces, we actually assign two pids, one is the PID, and the other is a upid. As usual LWN comes through and [describes](https://lwn.net/Articles/259217/) how this works. Here's the code:

```
    struct upid {
	int nr;					/* moved from struct pid */
	struct pid_namespace *ns;		/* the namespace this value
						 * is visible in
						 */
	struct hlist_node pid_chain;		/* moved from struct pid */
    };

    struct pid {
	atomic_t count;
	struct hlist_head tasks[PIDTYPE_MAX];
	struct rcu_head rcu;
	int level;				/* the number of upids */
	struct upid numbers[0];
    };
```

Now, each PID may have several values, with each one being valid in one namespace. That is, a task may have PID of 1024 in one namespace, and 256 in another.

Another good example I looked at is the mount namespace. This is another common one that we are likely to want to use. I'm familiar with chroot from working with SFTP servers previously, although I'll admit it always seemed slightly dark arts to me. For me what's interesting here with mount namespaces is that the views in ```/proc/[pid]/mounts``` and other associated /proc files correspond to the mount namespace in which the process with the PID resides.

So you can see what your process sees by looking at those. These function similarly to forks when a new mount namespace is created since it just clones its own namespace and then subsequent changes are reflected in the new namespace. There's also a way to have shared subtrees, which means you can create a peer group and then if in one namespace you mount a new location, that will propagate to the other peers. You could use this as a way to share a location across different namespaces.

 Members are added to a peer group when a mount point is marked as
       shared and either:

       *  the mount point is replicated during the creation of a new mount
          namespace; or

       *  a new bind mount is created from the mount point.
       In both of these cases, the new mount point joins the peer group of
       which the existing mount point is a member.



The thing I took away from this essentially that you can cascade down mounts into a newly spawned namespace, but equally if you then make changes in one of the namespaces that reflects across all of them. I feel like that's something that would potentially bite me at some point.

Now we know a little about how these namespaces work, we may want to know how we utilise them. To do this we use the namespaces API that's provided.

```
clone - New process
setns - Join an existing namespace
unshare - Move to a new namespace
ioctl - discover info about namespaces
```

A neat thing to know is that when the last process allocated to a specific namespace terminates or leaves the namespace then the namespace is removed. [The namespaces man page](http://man7.org/linux/man-pages/man7/namespaces.7.html)


I found another good post on the actual manipulation of these at [Prefetch.net](https://prefetch.net/blog/2018/02/22/making-sense-of-linux-namespaces/), and this details that we can actually see a list of namespace types in the kernel by searching for ```CLONE_NEW``` in ```/usr/include/linux/sched.h``` and that this will actually provide us with the bit pattern called for an NSTYPE. This I think is probably something that would be useful in debugging, since as that post mentions you might find ```sys_setns``` calls that are assigning processes new namespaces, but the details of that just state ```NSTYPE: 0x4000000``` which on it's own is meaningless but when correlated with sched.h tells us it's ```UTSNAME```.

That article also had some useful information about how kubernetes operates on top of these, mainly discussing pod and pause containers. I'm going to circle back to that blog again once I've got a firmer grip on cgroups and docker containers as well as kubernetes basics.

So that's namespaces, next up...cgroups
