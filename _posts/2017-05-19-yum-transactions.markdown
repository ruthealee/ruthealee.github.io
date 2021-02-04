---
layout: post
title: "Yum Transaction Ordering"
date: 2017-05-19 
categories: [linux]
---
Today I got an interesting question from a team member about yum's behaviour around the yuminstall_only= field in yum.conf

We had a server with a small /boot partition that was mainly full. It had the standard 5 retained kernel versions kept there, and we had reduced this to 2 by changing this in yum.conf. My colleagues' idea was that we could run the yum update command and not worry about the low space on /boot since he assumed that yum would drop the 3 extra old kernels before starting to unpack the new one.

That didn't make much sense to me logically and I wanted to give him a clear answer for why it wouldn't do that which wasn't just "because I think doing it that way would be dumb". And thus, I spent some of Friday morning investigating it with one of the guys.

The ordering of yum transactions is interesting. Technically, after some digging into the source code, this is done by rpm being called out to by yum, so that's what's taking care of the order in which packages are manipulated. To be honest making sense of that would have been a lot of work so we can actually work out yum's behaviour more simply.

**Step 1**: What do we *think* is going to happen?

From a purely logical perspective, if I'm writing the code for yum, i'm going to want to remove the old packages after i've installed the new ones. That's a pretty standard safety check, don't scrap your rollback plan before you have confirmed you've successfully completed the new install. If we removed these before then, in a circumstance where our transaction failed to complete for some reason we could end up with less than the number of retained kernels specified in that config file which seems counter intuitive.

**Step 2**: Prove what I theoretically think happens does actually happen (because Lord knows Linux doesn't always do things logically!). Couple of ways to do this, so lets look at all of them.

**Method A**: Look at the dependencies declared during an install (semi-convincing proof)
One of the major things we know yum and rpm are doing is managing dependencies. Dependencies can be specified for two purposes mainly: (a) declared by a package as a dependency, for example mod_ssl requires httpd. (b) used by yum to deal with ordering of the installs of groups of packages, yum will do some cursory ordering of groups of packages to be installed in an intelligent manner, and it does this by createing a tree of dependencies and using them to ensure packages aren't modified in the wrong order. Knowing that, let's look at a transaction log from a transaction I quit out of where I try to install another kernel: 2.6.32-696

```
[root@athena ~]# yum install kernel-2.6.32-696.el6.x86_64 --disableexcludes=all --showduplicates
Loaded plugins: fastestmirror
Setting up Install Process
Loading mirror speeds from cached hostfile
Resolving Dependencies
--> Running transaction check
---> Package kernel.x86_64 0:2.6.32-696.el6 will be installed
--> Finished Dependency Resolution
--> Running transaction check
---> Package kernel.x86_64 0:2.6.32-642.15.1.el6 will be erased
---> Package kernel.x86_64 0:2.6.32-696.1.1.el6 will be erased
--> Finished Dependency Resolution

Dependencies Resolved

======================================================================================================================
 Package                 Arch                    Version                              Repository                 Size
======================================================================================================================
Installing:
 kernel                  x86_64                  2.6.32-696.el6                       base                       32 M
Removing:
 kernel                  x86_64                  2.6.32-642.15.1.el6                  @updates                  131 M
 kernel                  x86_64                  2.6.32-696.1.1.el6                   @updates                  131 M

Transaction Summary
======================================================================================================================
Install       1 Package(s)
Remove        2 Package(s)

Total download size: 32 M
Is this ok [y/N]: ^CExiting on user Command
Your transaction was saved, rerun it with:
 yum load-transaction /tmp/yum_save_tx-2017-05-19-10-42kpqPll.yumtx
[root@athena ~]# cat /tmp/yum_save_tx-2017-05-19-10-42kpqPll.yumtx
321:948eb17674fd22d90b6de338e86184c6cdb98499
0
5
base:6706:1490724196
epel:12304:1495064580
extras:64:1489012224
pgdg96:307:1494498242
updates:282:1494864180
3
mbr: kernel,x86_64,0,2.6.32,696.1.1.el6 20
  repo: installed
  ts_state: e
  output_state: 40
  isDep: False
  reason: user
  reinstall: False
  depends_on: kernel,x86_64,0,2.6.32,696.el6@a
mbr: kernel,x86_64,0,2.6.32,696.el6 70
  repo: base
  ts_state: i
  output_state: 30
  isDep: False
  reason: user
  reinstall: False
mbr: kernel,x86_64,0,2.6.32,642.15.1.el6 20
  repo: installed
  ts_state: e
  output_state: 40
  isDep: False
  reason: user
  reinstall: False
  depends_on: kernel,x86_64,0,2.6.32,696.el6@a
```

So, from the above we can basically see the following. We're installing "2.6.32-696.el6", but we can see that the old kernels that will be removed (due to the retention limit set in yum conf) have the kernel that we are installing listed as a dependency. This isn't a hard dependency as declared by the package, but rather a dependency that's added by yum for this transaction (i.e. look don't touch the older kernels unless the kernel we're installing here is present)

Now that doesn't 100% answer our question because it's not clear whether we mean 'is present in the transaction thus allowing successful dependency resolution' or whether we mean 'is literally installed at this point in time'. So, question now becomes, what order do transactions take?

**Method B**: Use debug mode and trust the yum output (More convinced but still not iron clad)
We can look to the output of a yum transaction with debug enabled to see some more information around this. We know that yum provides a log of its actions, because if we are doing an update and we ctrl c out of it while it's installing a specific package we can look to that output to see what was successfully installed and what wasn't, as well as what we quit out of. We can surmise then that it's a chronologically accurate log of the actions the program is taking.

Switching debug mode on we can see this:
```
Running Transaction
  Installing : kernel-2.6.32-696.el6.x86_64                                                                       1/3
  Cleanup    : kernel.x86_64                                                                                      2/3
warning:    erase unlink of /lib/modules/2.6.32-642.15.1.el6.x86_64/modules.order failed: No such file or directory
warning:    erase unlink of /lib/modules/2.6.32-642.15.1.el6.x86_64/modules.networking failed: No such file or directory
warning:    erase unlink of /lib/modules/2.6.32-642.15.1.el6.x86_64/modules.modesetting failed: No such file or directory
warning:    erase unlink of /lib/modules/2.6.32-642.15.1.el6.x86_64/modules.drm failed: No such file or directory
warning:    erase unlink of /lib/modules/2.6.32-642.15.1.el6.x86_64/modules.block failed: No such file or directory
  Cleanup    : kernel.x86_64                                                                                      3/3
warning:    erase unlink of /lib/modules/2.6.32-696.1.1.el6.x86_64/modules.order failed: No such file or directory
warning:    erase unlink of /lib/modules/2.6.32-696.1.1.el6.x86_64/modules.networking failed: No such file or directory
warning:    erase unlink of /lib/modules/2.6.32-696.1.1.el6.x86_64/modules.modesetting failed: No such file or directory
warning:    erase unlink of /lib/modules/2.6.32-696.1.1.el6.x86_64/modules.drm failed: No such file or directory
warning:    erase unlink of /lib/modules/2.6.32-696.1.1.el6.x86_64/modules.block failed: No such file or directory
/var/cache/yum/x86_64/6/base/packages/kernel-2.6.32-696.el6.x86_64.rpm removed
  Verifying  : kernel-2.6.32-696.el6.x86_64                                                                       1/3
  Verifying  : kernel-2.6.32-696.1.1.el6.x86_64                                                                   2/3
  Verifying  : kernel-2.6.32-642.15.1.el6.x86_64                                                                  3/3
VerifyTransaction time: 2.277
Transaction time: 67.796

Removed:
  kernel.x86_64 0:2.6.32-642.15.1.el6                        kernel.x86_64 0:2.6.32-696.1.1.el6

Installed:
  kernel.x86_64 0:2.6.32-696.el6
```

Looking at this it you can see then basically that it's first installing, and then cleaning up (removing the older kernels to bring us in line with the limit set in yum.conf)


**Method C**: Test it! (Can't get more definitive than this!)

We could also prove this by testing it on a box, first let's arbitrarily reduce the space on the /boot partition. I did this with fallocate, fallocate -l xxxM yum-test-file where the value was something that would bring us to only having around 20MB of space free on the fs. I then set the number of kernels to be retained down to 2, so theoretically, if it erased prior to the kernel installation process there should have been space free to unpack the kernel. As we can see this fails out:

```
[root@localhost boot]# yum install kernel-3.10.0-514.16.1.el7 --disableexcludes=all
Loaded plugins: fastestmirror
base                                                                                           | 3.6 kB  00:00:00
epel/x86_64/metalink                                                                           |  22 kB  00:00:00
epel                                                                                           | 4.3 kB  00:00:00
extras                                                                                         | 3.4 kB  00:00:00
updates                                                                                        | 3.4 kB  00:00:00
Loading mirror speeds from cached hostfile
 * base: centos.serverspace.co.uk
 * epel: www.mirrorservice.org
 * extras: centos.serverspace.co.uk
 * updates: mirror.sov.uk.goscomb.net
Resolving Dependencies
--> Running transaction check
---> Package kernel.x86_64 0:3.10.0-514.16.1.el7 will be installed
--> Finished Dependency Resolution
--> Running transaction check
---> Package kernel.x86_64 0:3.10.0-327.el7 will be erased
---> Package kernel.x86_64 0:3.10.0-327.36.3.el7 will be erased
--> Finished Dependency Resolution

Dependencies Resolved

======================================================================================================================
 Package                Arch                   Version                                Repository                 Size
======================================================================================================================
Installing:
 kernel                 x86_64                 3.10.0-514.16.1.el7                    updates                    37 M
Removing:
 kernel                 x86_64                 3.10.0-327.el7                         @anaconda                 136 M
 kernel                 x86_64                 3.10.0-327.36.3.el7                    @updates                  136 M

Transaction Summary
======================================================================================================================
Install  1 Package
Remove   2 Packages

Total size: 37 M
Is this ok [y/d/N]: y
Downloading packages:
Running transaction check
Running transaction test


Transaction check error:
  installing package kernel-3.10.0-514.16.1.el7.x86_64 needs 15MB on the /boot filesystem

Error Summary
-------------
Disk Requirements:
  At least 15MB more space needed on the /boot filesystem.
```

So, there's our answer. Basically the documentation for this sucks but we can pretty much trust best practice most of the time, and there's a couple of different ways we can double check our hunch is right.

To be honest, I was hoping to hero level this and read the python code for yum and be like 'well clearly in line blah blah blah we can see it does this' but it turns out yum is complicated and my python is poor at best! Luckily I think doing this just by observing yum's behaviour and running a few tests still gives us a definitive answer. Now, as for whether it behaves in the same way for regular non-kernel packages, that's another question. (To which I suspected the answer is yes, but processes transactions in little chunks since the source code indicates its using something called transaction groups, so that it's only ever doubling up the necessary disk space for a subset of packages in the total transaction).

In the end after forcing my much better at python friend to read through a lot of the yum and rpm source he found this in the rpm source code:

```
/** \ingroup rpmts
* Determine package order in a transaction set according to dependencies.
*
* Order packages, returning error if circular dependencies cannot be
* eliminated by removing Requires's from the loop(s). Only dependencies from
* added or removed packages are used to determine ordering using a
* topological sort (Knuth vol. 1, p. 262). Use rpmtsCheck() to verify
* that all dependencies can be resolved.
*
* The final order ends up as installed packages followed by removed packages,
* with packages removed for upgrades immediately following the new package
* to be installed.
*
* @param ts            transaction set
* @return              no. of (added) packages that could not be ordered
*/
int rpmtsOrder(rpmts ts);

```

So I think that's case shut on this one. We install and then we remove, and the grouping of packages provided by yum's sorting of dependencies provides us the ability to do this without doubling the disk space we're consuming during the time it takes for the various stages of the transaction to complete.

Not too bad for a Friday.

 

