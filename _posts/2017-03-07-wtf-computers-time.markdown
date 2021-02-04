---
layout: post 
title: "Time, Leap Seconds and NTP" 
date: 2017-03-07
categories: time
---
There are plenty of times I sit at my desk and approach what I think will be something simple that immediately rabbit holes into hours of reading culminating in a single moment at which I exclaim "What the fuck?" and am answered by a colleague "computers" accompanied by a shrug. 99% of the time it turns out the simple thing was actually super interesting and I learned something cool. This is the first in what I'm sure will be an endless string of posts chronicling the latest thing that made me swear at my computer.

######Time is terrible

With the next leap second insertion swiftly approaching on the 31st of December we were starting to think about possible effects on systems. Previously (2012) there were a few issues that cropped up with the leap second, one of which was high CPU usage. This should have been fixed in later kernel releases from RHEL 6.4 and later. If experience has taught me anything it's not to trust when someone says they've fixed something. So, off I went to understand why the original issue happened and decide if it was actually fixed or not. That problem turned out to be pretty interesting and a good excuse to think about how Linux and other systems handle the problem of time. 

If you'd like to really unbalance your perception of the world and potentially have an existential crisis I highly recommend reading about time. For the purposes of what we're talking about here we'll assume that atomic clocks provide us with an accurate measure of time, and further we'll assume that leap seconds are a reasonable way to handle the problem of keeping these in line with the Earth's axial rotation. [1] As far as systems go there's two aspects of time we care about: What time is it relative to other systems, and what time is it relative to system boot. The former is used to co-ordinate communication and exchange between systems and the latter to co-ordinate system tasks and processes. 

The problem we saw with leap seconds in 2015 was based on the interaction between these two 'types' of time and how they manage themselves in relation to one another.

**System Time**

System time is going to be measured in a few ways, but commonly you'll have this synchronised by a service like ntpd. ntpd on your server uses a reference clock to provide the time. It does some neat little tricks considering things like offset and jitter in order to preserve as close to accurate a time as possible. We'll assume that it is correctly doing so and staying synchronised with the reference clock. When it notices offsets or drift, it will correct this. With ntp it's generally safe to assume that your clock is accurate-ish. 


**Timers and Timer subsystems**

When you run multiple processes on a CPU you need a way to ensure each process gets time on the CPU. There's a couple of different schedulers that you'll commonly see like CFS but one thing they all need is the ability to measure time. In this case we don't care about our time relative to other systems, merely time relative to our own system. To measure this the kernel uses something called 'jiffies'. Jiffies are the number of 'ticks' since system boot. Tick rates are based on the HZ defined by different architectures, you can find this in &lt;asm/param.h&gt; but you can assume it to be 1000 on x86-64 systems. The standard kernel timer API utilises jiffies as its units. 

The smaller the unit of time, the more accurately and efficiently you can schedule tasks. Ideally you would use the smallest unit that it's is possible to measure on the realtime clock of the processor. This may or may not match the unit being used for the jiffie, usually it will be more accurate (smaller) than what the jiffie can provide. &nbsp;On systems where you need a higher resolution and more accuracy you have two options, increase the HZ value appropriately thus reducing the length in nanoseconds of each jiffie or utilise one of the high resolution timers. While it might seem obvious that you should just set a higher value for HZ this can introduce additional overhead across the whole system. High resolution timers allow you to be constrained not by jiffies but by the maximum accuracy the hardware provides. hrtimer() is a common implementation, have a look at it here.[2] On an high level it does this by matching offsets between system clocks and the realtime clock of the processor.[3]

**So what does this have to do with leap seconds?**

Leap seconds affect system time, we'll assume here that we're using ntpd to synchronise this. There's a few ways that leap seconds are applied, you can either 'step', which is essentially either stepping back a second, repeating your last timestamp, or you can insert a duplicate timestamp, or an impossible timestamp. Alternatively you might choose to smear or slew the time. With a smear the ntp server smudges the extra second across a period of time, distributing the additional second in millisecond increments throughout that period, a slew is the same thing but done locally on the server. Smears and slews kind of solve some of the issues you see with stepping time but they do mean that your time will remain 'incorrect' for a longer period which may not be ideal. In this case it's the step that caused the problem. With a step your system time will increment similarly to below, either repeating a second, or adding an additional second, e.g.:
```
  23:59:59 
  23:59:59 <-- leap second 
  00:00:00

  23:59:59 
  23:59:60  <-- leap second 
  00:00:00
````

That's fine for the kernel and ntp, they'll both adjust and go 'sweet, inserted a leap second' before trundling along with their regular life. Where we get a problem is the way in which system time interacts with the timer subsystems we looked at earlier. If we had been using the kernel's timer API we would be measuring in jiffies from boot and not care what the system clock was doing because we already have a built in consistently incrementing value independent of system time. &nbsp;

This only becomes a problem when we are using high resolution timers. This is because hrtimer and other high resolution timers are essentially bridging system time with processors realtime clocks. The issue seen in 2015 was with hrtimer and how it implements this integration with the system clock. An overly simple explanation here is that via the use of a number of internal time values that it offsets from the system clock hrtimer takes system time and translates it into the appropriate value for processors realtime clocks, basically syncing up the smaller units of the processors realtime clock (milliseconds) with the larger units of the system clock (seconds). It's not using jiffies, so it's anchoring it's internal time bases off that system clock as opposed to an independent counter. 

That works just grand until you change the system time, because it's going to need to adjust itself to take that into account. Since there's things like drift with ntpd there's already a way to do that. The kernel will change the time and call clock\_was_set() which will notify hrtimer to go check and adjust itself. 

The problem in 2015 was that earlier kernels failed to make this call, and hrtimer was never notified that the system time changed. This meant that the subsystem was immediately a second in the future, causing all the timers to go off a second sooner than they should. Not necessarily a huge problem for a single application, if however you have a bunch of timers set for &lt;1 second in the future (which you probably do if you cared enough to utilise a high resolution timer in the first place) those are all going to wake up at exactly the same time because they'll immediately expire. They'll wake up and request CPU time. Most of them will reset immediately, for another &lt;1 second value, which of course will immediately expire again since the subsystem is still ahead of the system clock. Repeate ad infinitum.&nbsp;

The immediate 'fix' that was used and publicised for this was to restart ntpd, which is kind of misleading because it implies the issue was with ntpd. The reason this worked is because a restart of ntpd caused the kernel to send the clock\_was_set() syscall as part of ntpd's initialisation, thus hrtimer worked out that it's offset was wrong and corrected itself. If we didn’t know any better you’d think it was something to do with the nptd, but actually it turned out to be a much more interesting case that provides a nice excuse for digging through how different elements of the system require different ways of tracking time and how those interact.&nbsp;

 [1] Turns out this is an enormous assumption and actually is a huge simplification of how time and space operates and basically everything is inaccurate, time is meaningless and nothing matters. 

 [2][ hrtimer source](http://lxr.free-electrons.com/source/include/linux/hrtimer.h)

 [3] It's actually more complicated than that and can be set to be bound to either a monotonic clock (similar to jiffies in that it's an incrementing counter) or to CLOCK_REALTIME which is self explanatory. We're presuming you're binding to CLOCK_REALTIME since that's where this bug crops up. More details [here](https://lwn.net/Articles/167897/).

_Other reading:_
[Original bug report](https://bugzilla.redhat.com/show_bug.cgi?id=840950)

[Interesting breakdown](http://marc.info/?l=linux-kernel&amp;m=134113577921904&amp;w=2)

[General leap second info from RH](https://access.redhat.com/articles/15145)&nbsp;

[Google's Policy on NTP smears](https://developers.google.com/time/smear)
