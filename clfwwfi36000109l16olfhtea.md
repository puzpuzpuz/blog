---
title: "The Secret Life of fsync"
seoDescription: "What kind of durability guarantees fsync system call provides in Linux? Let's find out."
datePublished: Fri Mar 31 2023 18:49:29 GMT+0000 (Coordinated Universal Time)
cuid: clfwwfi36000109l16olfhtea
slug: the-secret-life-of-fsync
tags: linux, databases

---

Several times I've heard opinions that many mass-market SSDs and HDDs don't provide sufficient durability guarantees and Linux can do nothing with that. Namely, after an [`fsync`](https://man7.org/linux/man-pages/man2/fsync.2.html)`()` call recently modified data can still sit in the drive's volatile write cache and, thus, it may be lost in case of a power failure. If you want any meaningful durability, you should go for enterprise-grade drives that have a battery/capacitor so that they can flush the data to persistent storage on power loss. Is it really so? Let's find out.

First, let's check what POSIX.1-2017 [specification](https://pubs.opengroup.org/onlinepubs/9699919799/) says about `fsync`:

> The *fsync*() function shall request that all data for the open file descriptor named by *fildes* is to be transferred to the storage device associated with the file described by *fildes*. The nature of the transfer is implementation-defined.

The above description is rather vague. If the OS issues operations to write the data to the disk's volatile cache, that's a "transfer", so formally such OS would be POSIX-compliant. The informative section of the spec sheds more light on what a proper `fsync` implementation should do:

> The *fsync*() function is intended to force a physical write of data from the buffer cache, and to assure that after a system crash or other failure that all data up to the time of the *fsync*() call is recorded on the disk. Since the concepts of "buffer cache", "system crash", "physical write", and "non-volatile storage" are not defined here, the wording has to be more abstract.

OK, that's much more specific. Once an `fsync` call is made, the data should become durable in the face of a system crash, e.g. due to a power loss. But is it really something Linux does on an `fsync`?

As with many other FS-related system calls, most (if not all) file systems have their own implementation of `fsync`. To keep things simpler, we're going to check the ext4 implementation in the recent 6.x kernel code base. We should be looking at the [\`ext4\_sync\_file()\`](https://github.com/torvalds/linux/blob/62bad54b26db8bc98e28749cd76b2d890edb4258/fs/ext4/fsync.c#L129-L187) function which is invoked on an `fsync()` call. It involves the following steps:

1. First, it writes all dirty pages belonging to the file that corresponds to the input file descriptor to the disk. That's done by the \`file\_write\_and\_wait\_range()\` function. As a result, the data may be sitting in the disk volatile cache, so that's not what we're looking for.
    
2. Next, it writes the inode's metadata to the disk. Depending on whether journaling is enabled on the FS or not, it's done via a specific function, e.g. \`ext4\_fsync\_journal()\`. Again, not something we're in search of.
    
3. Finally, if the `needs_barrier` variable is true, it calls the `blkdev_issue_flush()` function. That's probably what we need, isn't it?
    

Let's leave the `needs_barrier` variable out of the equation for now and check what `blkdev_issue_flush()` does. This [function](https://github.com/torvalds/linux/blob/62bad54b26db8bc98e28749cd76b2d890edb4258/block/blk-flush.c#L462-L468) queues a flush operation to the block device and waits until it's finished. The operation has the `REQ_PREFLUSH` bit set among the flags. If we open kernel docs, we'll find [some information](https://docs.kernel.org/block/writeback_cache_control.html#explicit-cache-flushes) on this flag (and not only):

> In addition the REQ\_PREFLUSH flag can be set on an otherwise empty bio structure, which causes only an explicit cache flush without any dependent I/O. It is recommend to use the blkdev\_issue\_flush() helper for a pure cache flush.

As we expected, the `REQ_PREFLUSH` flag (as well as the `REQ_FUA` flag) tells the block device that it should flush its volatile cache to the persistent storage. Drivers for any well-behaved disk with a volatile write cache [should handle](https://docs.kernel.org/block/writeback_cache_control.html#implementation-details-for-request-fn-based-block-drivers) this flag properly. Obviously, disks without such cache don't need to bother with these operations and flags.

Now, what's the buzz with `needs_barrier`? In both [journaled](https://github.com/torvalds/linux/blob/62bad54b26db8bc98e28749cd76b2d890edb4258/fs/jbd2/journal.c#L638-L670) and [non-journaled](https://github.com/torvalds/linux/blob/62bad54b26db8bc98e28749cd76b2d890edb4258/fs/ext4/fsync.c#L98-L99) ext4 code paths, it appears to be nothing more, but an optimization to avoid sending flush requests to the disk multiple times. Note that you can also configure ext4 not to issue flush operations. For example, in non-journaled mode it's done via EXT4\_DEFM\_NOBARRIER mount option.

Other file systems have their own specifics, but the overall logic should be close enough to the ext4's one. So unlike [macOS](https://news.ycombinator.com/item?id=30370551), Linux does its best to transfer the data to persistent storage on `fsync`. Of course, this doesn't protect you from a flawed driver implementation written for a cheap no-name drive, but it also means that if you have a decent SSD from a well-known brand, you may be fine without an enterprise-grade disk.