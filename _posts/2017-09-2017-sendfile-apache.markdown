---
layout: post
title: "The Mysterious Case of Sendfile and Apache"
date: 2017-09-29
categories: [linux]
---
##sendfile() and why you shouldn't use it with apache 2.4

An odd question came up at work the other day, which was why we recommend against using sendfile with apache.
One of the guys was looking at our configuration guidelines and saw that we recommend not using it for either apache or nfs since it can cause 'performance problems' but our documentation didn't say what those problems were.

He started looking into it but couldn't seem to find a compelling answer. We tried old reliable [stackoverflow](https://stackoverflow.com/questions/46367130/why-does-apache-recommend-against-using-sendfile-with-nfs-on-linux) to no avail.

I totally forgot about it until today when I was looking into a frankly unrelated issue where someone crashed a server by running a bunch of git clones of respositories filled with single large 1GB files. A colleague idly mentioned something about checking if sendfile() was enabled which in the context of my issue didn't make sense but did remind me of our previously unanswered question. What better way to spend a Friday afternoon than looking into it.

I'll admit, I don't think I came up with a particularly compelling answer here. Or rather, I'm pretty sure it's the answer but i'm still not 100% as to what the underlying limitation is.

###### WTF is sendfile()

Sendfile is a system call that allows in kernel copying of data between two file descriptors. Usually used for copying between a file and a network socket, but since kernel 2.6.33 you can also copy between two files[1]. You can call this "zero-copy".

###### Right what's a zero-copy then?
Wrong question. Before we can answer this we need to know what actually happens normally when we read a file over a network.

###### Fine, What happens when we read a file over a network?

I found a *great* article about this on linuxjournal. I thoroughly recommend reading it in full, not just the weird summary i'm providing. You can find it at [Zero Copy I: User-Mode Perspective](http://www.linuxjournal.com/article/6345)

```
read(file, tmp_buf, len);
write(socket, tmp_buf, len);
```

What's happening behind the scenes though? A whole bunch, here's a really useful diagram in the article I read about this:
![normal-network-copy](http://www.linuxjournal.com/files/linuxjournal.com/linuxjournal/articles/063/6345/6345f1.jpg)

Esentially, we:

  - Are chilling (user)
  - Request a read (user)

  *switch to kernel*
  - syscall read() (kernel)
  - kernel DMA copies from the drive to its buffer

  *switch to user*
  - kernel CPU copies off into the user buffer (user)

  *switch to kernel*
  - syscall write(kernel)
  - CPU copies from user buffer into socket buffer
  - kernel DMA copies from socket to protocol engine

Okay that's a lot of stuff. What do we care about here? Technically all of it but for the purposes of answering today's question we really care about what we mean when we say 'kernel buffer'. Lot's of the documentation around sendfile likes to just say 'buffers' and run into the distance without specifying what buffer. Linux loves buffers and caches so there's always multiple buffers at play in most operations. In this case we're refering to a specific kernel address space buffer. We'll come back to this dude later.


######  What does this have to do with zero-copy?
Zero copy is trying to 'fix' the duplication of data in the above process. One way it does that is using nmap:

```
tmp_buf = mmap(file, len);
write(socket, tmp_buf, len);
```

Here's another picture:
![mmap](http://www.linuxjournal.com/files/linuxjournal.com/linuxjournal/articles/063/6345/6345f2.jpg)

TL;DR here is that mmap copies the file straight into a kernel buffer from the DMA engine and then *shares* that with the user buffer, and the CPU will copy that across from one kernel buffer to another kernel buffer associated with the destination socket. Cool we haven't duplicated data.

This is where we see the underlying problemo with this solution though. This happens, as explained by Dragan Stancevic,

> when you memory map a file and then call write while another process truncates the same file. Your write system call will be interrupted by the bus error signal SIGBUS, because you performed a bad memory access.

I'm not super interested in how to fix this because it's a digression from our main question here which is what is sendfile actually up to, and to be honest it's also a little deeper than my understanding at this point.

###### Back to sendfile()
So now we have sendfile trying to solve the same problem in a similar way.

```
sendfile(socket, file, len);
```

It's picture time again!
![sendfile_copy](http://www.linuxjournal.com/files/linuxjournal.com/linuxjournal/articles/063/6345/6345f3.jpg)

So Stancevic states here again that there's issues around processes truncating files that we're sending. Apparently in kernel 2.4 there's a change made to socket buffer descriptors that basically means it will support gather operations. So the data being transmitted doesn't need to be in consecutive memory but can be located all around the place.

This lets us do this:
When copying from the kernel buffer to the socket buffer we don't actually move any data. We just add descriptors within information about where the data is located to the socket buffer. So the DMA then passes directly from itself to the protocol engine, no CPU copy involved at all.

Here's the kicker though. It does still pass into the kernel buffer. It's still copied from the disk to the memory and from the memory to the wire. We aren't actually avoiding kernel buffers, we're just avoiding *two* kernel buffers. We've saved performance by reducing context switches, reducing CPU data cache pollution and no CPU checksum calculations.

######  I asked about apache though?
I know, long diversion.
![84_years_gif](https://m.popkey.co/53257f/0DmrW.gif)

We're back to it now though. So we've learned that sendfile() is still using buffers, but it's using a whole lot less, just one in fact! Also, if we look at the man page for sendfile we can find out some other stuff that might y'know be important like this little nugget:

> sendfile() will transfer at most 0x7ffff000 (2,147,479,552) bytes,
       returning the number of bytes actually transferred.  (This is true on
       both 32-bit and 64-bit systems.)

There's a 2GB limitation. Now here's the assumption, the apache documentation says:

> With a network-mounted DocumentRoot (e.g., NFS, SMB, CIFS, FUSE), the kernel may be unable to serve the network file through its own cache[2]

So when it says 'the kernel may be unable to server the file' I think we might be referring here to the inherent limit on file size that sendfile has.

As for apache not default enabling sendfile support, the reason is because the normal package isn't compiled to enable the server to switch between using sendfile() for files <2GB and using regular read() write() for files >2GB. It either uses one or the other, so you would only turn on sendfile functionality if you knew you weren't serving files larger than the limitation.

There's a few posts around the same topic on other webservers mailing lists thttpd mentions it [here](https://marc.info/?l=thttpd&m=103799658531220&w=2). Some reading around this implies that you could custom compile a webserver that would intelligently use sendfile() for files below the 2G threshold and regular read + write for larger files.

## Y tho
![y-tho-meme](https://i.imgur.com/dY16kHi.png)

Is it not enough that I gave a reasonable explanation with little to no evidence?! I guess the short answer here is, "I'm not sure". I looked around to see if I could find the buffer that it's copying too, as I assumed that buffer must have a maximum limit that aligns with that 2GB value, but I couldn't actually find the piece of code that specifies it.
So I haven't answered my question of why that 2GB limit exists.

This is at the point that I get a bit confused, I'm defeated here by the limits of my understanding of C, so what follows is really just some random bits I grabbed from different sources that I'm mentioning here so that I can come back at a later date and perhaps pick this back up again.

 I read [this](http://blog.superpat.com/2010/06/01/zero-copy-in-linux-with-sendfile-and-splice/) that mentions that the underlying operation of sendfile is based on splice(). That article links out to some crazy in depth explanations, as if it wasn't already a super in depth discussion. Fun, but again, pushing up against the limits of my understanding at this stage. What I gathered here was really that sendfile is maybe pulling some of splice's functionality or ideas and making it a bit more robust.

Here's the description of that syscall from the man pages

> splice() moves data between two file descriptors without copying
 between kernel address space and user address space. It transfers up to len bytes of data from the file descriptor fd_in to the file descriptor fd_out, where one of the file descriptors must refer to a pipe.

So, looking into splice() a bit more I found some discussion on the kernel mailing lists between Linus and Ingo Molnar which is way above my pay grade, but essentially does clarify something about sendfile, although this seems to be implying that sendfile is kind of using the same idea as splice but a different way of implementing it. Linus says:

>    By using the page cache directly, sendfile() doesn't need any extra
   buffering, but that's also why sendfile() fundamentally _cannot_ work
   with anything else. You cannot do "sendfile" between two sockets to
   forward data from one place to another, for example. You cannot do
   sendfile from a streaming device.

He then says this about splice:

>    The pipe is just the standard in-kernel buffer between two arbitrary
   points. Think of it as a scatter-gather list with a wait-queue. That's
   what a pipe _is_. Trying to get rid of the pipe totally misses the
   whole point of splice().

So I guess what we've learned here is that sendfile isn't that based on splice? In fact it's using the page cache directly? I still wasn't sure where that 2GB limit was coming from because the page cache size can be modified in Linux, so I went and found where sendfile is implemented in linux/fs/read_write.cache
See a copy [here](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/plain/fs/read_write.c?id=HEAD)

In that we see a declaration there for loff_t maxsize which is maybe what's restricting it? Perhaps it's not the overall size of the page cache but rather how much of My ability to read C completely flunks out at this point so I guess that level of investigation is going to have to wait for another day. Maybe i'll come back to this in a year or two and be able to dig a bit deeper.

#####Conclusion
For now the takeaway here seems to be, keep sendfile turned off in apache 2.4 unless you can guarantee you'll never serve files larger than 2GB since this is the limit of the file size that can be copied using sendfile().


[1]http://www.linuxjournal.com/article/6345
[2]http://blog.superpat.com/2010/06/01/zero-copy-in-linux-with-sendfile-and-splice/
