---
layout: post
title: "That Time I Was Wrong About Sendfile & Apache"
date: 2018-12-17
categories: [linux]
---
A while back I looked into why Apache recommends turning sendfile() off when serving over NFS.

I hypothesised it was due to an inherent limit in the size of the buffer that sendfile copies to and that Apache didn't have a way to distinguish between the size of the files it was serving and then choose to utilise or not utilise sendfile() based on that data.

In the time since I wrote that and now, someone came up with a much more compelling answer on the original stack overflow [post](https://stackoverflow.com/questions/46367130/why-does-apache-recommend-against-using-sendfile-with-nfs-on-linux).

TL;DR it looks like this does and doesn't relate to the page cache in the way I thought it did. The page cache *is* involved however the issue here is due to the way this passes through the page cache you can end up serving stale data. This is really the case where you have a filesystem that isn't configured to support sendfile callbacks.

In a circumstance where this isn't enabled it means that we can have cached data that's stale in the page cache and end up serving that even when on disk data has been edited. We need the send file call back option enabled so that when the on disk data is edited it knows to dirty it in the page cache. This is interesting, since it then means your data may be stale or fresh based on whether you're bypassing the cache when reading from it.

In conclusion, this was cool because I was totally wrong in my guess. I had said I was going to come back to this and it's gratifying to get a real explanation for this. Looking back my assumption didn't make sense because if what I'd guessed was right this would have affected all cases of sendfile, not just those on NFS or remote filestorage systems.


