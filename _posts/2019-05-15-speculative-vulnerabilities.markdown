---
layout: post
title: "The Ghost of Vulnerabilities Past: Spectre/Meltdown Revisit"
date: 2019-05-16
categories: [security]
---

# Transient Execution Attacks

A recent glut of speculative attacks on CPU chips prompted a re-examination of the category. I've been spending a fair bit of time looking at memory in the Linux kernel, particularly around how pagecache operates, but a side affect of this was reviewing over the more general background of virtual memory and page mapping. 

When we saw meltdown/spectre I was able to get a rough understanding of the issue so this was a good opportunity to return to it with a better grasp of the underlying memory access mechanisms and see if I can get a better read on it. 

Turns out it is plenty complex still, we're assuming a lot of background knowledge here. I've added a reading list at the end of the post with each topic and what I think are the top resources for a thorough understanding of each of these next to them. Later in the post there's a link to [this](https://medium.com/@mattklein123/meltdown-spectre-explained-6bc8634cc0c2) medium article that also does a brief refresher. Other resources are called out inline. 

I also found the 2.1 section from the [Zombieload](https://zombieloadattack.com/zombieload.pdf) paper was a nice summary of these. 






## Transient Execution
First port of call for these new vulnerabilities is to go back and understand how initial transient execution attack vectors operated and the remediation of these. 

These attacks rely on two fundamental things. CPU microarchitecture, and cache attacks. Both of them heavily rely on exploiting caches, in this case at CPU level. The microarchitecture it takes advantage of is around out of order and speculative execution of instructions. 

CPUs do not always execute instructions in order. There's two performance increasing features that they utilise. One is out of order execution and the other is speculative execution. 

Out of order execution is when a CPU breaks down an instruction stream into smaller micro-operations. These are executed out of order, with results stored in a dedicated buffer. Once they all complete they are re-assembled and returned from the CPU in a way that gives the impression of in order execution. 

Similarly speculative execution is allows for the CPU to optimize operation pre-executing code that it predicts will be requested. If the results turn out not to be needed it just discards them. 

Both of these are theoretically super nice, the risk comes with how we store those speculative results, and also the possibilty of tricking the CPU into executing a branch that reveals information we want to read. When a CPU executes instructions two important things happen, firstly the location of those instructions ends up being loaded in from memory, that will change entries in the CPU local caches (L1, L2 etc) as well as potentially the contents of the TLB. There's a neat diagram and simple explanation at [usenix](https://www.usenix.org/legacy/publications/library/proceedings/usenix03/tech/fraser/fraser_html/node3.html) but essentially the takeaway is that the contents of this speculation or out of order action will end up populating buffers. 

A further issue is that this microarchitectural level has not historically segregated between applications or privilege levels. That logical separation tends to happen at an architectural or OS level, and it's this combination of out of order and speculative executions along with the temporary (transient) storing of this information in locations that don't distinguish between privilege levels or applications that allows for cache side channel attacks. 

A nice top level summary of the two exploits is essentially

- Meltdown: Relies on exploiting transient execution after a CPU exception. 
- Spectre: Relies on exploiting misprediction in speculative execution. 

You could pretty much stop reading here and have enough information. 



### High level
In the [MDS whitepaper](https://mdsattacks.com/files/ridl.pdf) there's also a good top level review, that points out that the initial vulnerability was limited within address boundaries. 

We need to know about page mapping in the Linux kernel here. When a process is active in Linux it will load in its own page table mapping, traditionaly the entirety of the kernel page maps are included within each process to allow them to access entry points to kernel functions (aka systemcalls). That reveals the first attack vector, which is that if we have those pages mapped in our userprocess we suddenly are relying on a security bit being set corectly to prevent reading those memory maps for the kernel pages. Spectre and Meltdown rely on the fact that side channel attacks allowed for the leaking of this memory. Refresher on side channel attacks: This relies on working out when things change to be able to tell us something about what's being accessed. In our case, we monitor the caches and TLB to work out certain parameters that allow us to attack cryptographic implementations. 

### Detail
Meltdown is actually the 'simpler' of these attacks and so it's the main one to understand before heading into the latest vulns. Since the TLB and various CPU caches are shared across a system, meltdown advantage of the fact that if you can train a CPU to pre-execute certain code, then it will load the result into the cache. We can then use access times to work out where that result lives in kernel memory. Knowing this, and the fact that all kernel address space is available within processes you could read the entire physical memory in this manner. 
The access to kernel memory via this method was a specific bug in intel processers, particularly around the TLB. Thre's a really nice breakdown by Matt Klein on [medium](https://medium.com/@mattklein123/meltdown-spectre-explained-6bc8634cc0c2) of this. This covers both meltdown and spectre. Of course there's way more information in the orginal [Spectre whitepaper]https://spectreattack.com/spectre.pdf) too. 

It took me a while to work this through in my head, that medium post was really helpful, but a nice way to think of it is this. Again this is highly cribbed from that Matt Klein article so I highly recommend reading that as well. I'm just writing it out in my own words to ensure I have a copy. 

- You can easily predict the location of some random byte of kernel region memory. Just guess an arbitrary value that sets you well above the predicted range of the offset. Most security features are around not being able to locate specific kernel addresses, but the general range is not hard to guess inside of. 
- Then we do something like the below:

```
init an array with defined number of positions of which x is one.
smoking_gun = ()
  foo = (x)(n) <-- this will page fault since we aren't allowed access that memory
    bar = foo * 4096 <-- speculated
    smoking_gun += ('bar') <-- speculated   
```
The idea is simple, we initiate an empty array with 256 (for arguments sake) possible values. We then pick an arbitrary int within that range, and we try and trick the kernel into pre-executing some maths on that and caching the result. If we were returned the result that would be ideal, but obviously the kernel won't give us that since it's protected. 

Instead we use that array to perform a side channel attack. We know the kernel cached it, so now we just go ahead and iterate over the array. One of those values is going to return much faster since it's already cached. As soon as that happens we know what the value of x was, and so then we can know the value of n. 

That same approach can be used to read the entirety of physical memory. 

Spectre is similar in that it uses both pre-execution and cache timing attacks. It's more difficult because you have to actually trick the CPU into pre-executing your code. That requires training the CPU via code execution. It poisons the pre-execution engine essentially. This is pretty damning because it's not just a CPU bug, it's a holistic attack on how CPU and OS operate. 


### Mitigation
Linux tried to offset this risk by using something called Kernel Page Table Isolation or KPTI after metldown. Instead of our previous mapping of kernel space within a process, with access secured via security bits, we stop keeping the kernel memory maps in memory. KTPI stops us keeping both kernel-space and user-space addresses in memory when running in user mode. Instead in userspace we keep a very minimal set of kernel space mappings that provide only the access needed (entry points for system calls, interrupts etc). The rest of the kernel operation is then only mapped when running in kernel space. The upside here is it's more secure and probably a better way to do it in general. The downside is that it means system calls become more expensive since switching into kernel space means a total flush of the TLB and reload. It's actually the KPTI implentation that slows systems, not as many people and myself presumed, that we disabled speculative execution. That is a key distinction for understanding how the new vulnerabilities are then possible. 

The mitigation for spectre is where we actually get very interesting. Here we have to start preventing CPU caches from revealing or leaking information. 

Some of the mitigation looks at how we actually write to and from the stack. Google published their implementation of this in a [whitepaper](https://support.google.com/faqs/answer/7625886). My assembly is nonexistent so this is pretty dark magic to me, but essentially it means that sepculative execution doesn't end up actually writing to the stack (I think?). Think of it as a kind of quarantine for particularly sensitive binaries, usually OS or hyp processes that should never really be exposed to pre-execution related risk. This is patched therefore really at a hypervisor level for those of us running on third party cloud providers. So AWS or GCP level. 

CPU microcode updates have been another significant mitigation strategy for this and it's primarily around retooling how the branch prediction code operates. Previously it seems that branch prediction was shared across all threads, and this meant that user mode code could affect kernel code since they're all sharing the same prediction cache. The ultimate 'fix' for this would be to have per thread branch prediction. Right now most of the mitigation just flushes the CPU caches when switching between threads of different privileges. Of course there's performance tradeoffs here, and that's where we see the big hit. Readers of Drepper's paper will remember the graphing around the impact of caches on CPU performance and know how large that impact can be. 
o
### Reading List
General Background Refresh

- CPU caches and on chip memory. Read Ulrich Drepper [What Every Programmer Should Know About Memory](https://people.freebsd.org/~lstewart/articles/cpumemory.pdf). The whole thing. But if not, then at least sections, 2 and 3-3.2 on CPU Caches and 4-4.3.1 on TLBs
- Linux and OS virtual memory management (particularly around privilege and page mapping). Read [this](https://www.tldp.org/LDP/tlk/mm/memory.html) fully detailed rundown or [watch](https://www.youtube.com/watch?v=EWwfMM2AW9g) this talk.
- Program memory, specifically stack access. Read this [post](https://manybutfinite.com/post/anatomy-of-a-program-in-memory) on the Anatomy of a Program in Memory and [review](https://en.wikibooks.org/wiki/X86_Disassembly/The_Stack) how we manipulate a stack.
- How speculative execution works. Read [this](https://www.usenix.org/legacy/publications/library/proceedings/usenix03/tech/fraser/fraser_html/node3.html) nice quick summary.

