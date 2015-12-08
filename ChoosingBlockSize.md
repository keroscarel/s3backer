Here's a snippet from an email thread that may be helpful...

```
On Wed, Oct 15, 2008 at 9:22 PM, baboon <baboon...@gmail.com> wrote:
> > > How about increasing the block size? If it was 16k, the 75$ could
> > > already be reduced to 20$, which would more acceptable. However, I am
> > > running 32bit Linux and from what I have read I cannot increase the
> > > block size of my file system beyond the page size, which probably is
> > > 4k. I have not found a solution so far, could somebody else think of
> > > one?
> >
> > You could run s3backer with a 16k block size.  That would just mean that
> > you're packing four 4k file system blocks inside of an s3backer block.
> > This will incur a time penalty for contiguous block writes, but can be
> > tuned via the 'minWriteDelay' command line flag.  I've used 64k s3backer
> > blocks with  < 100 ms minWriteDelays before.  Another thought is that if
> > you're using RAID, you'll want to sync to your stripe size/stride to
> > keep your caches all in sync.
> 
> So what is a good s3 blocksize for arbitrary user files?

First let's review the rules :-)

$0.15 per GB-Month of storage used
$0.100 per GB – all data transfer in
$0.01 per 1,000 PUT, POST, or LIST requests
$0.01 per 10,000 GET and all other requests

If your goal is to reduce cost for uploads, consider:

    * Amazon charges for storage only by total size, not number of S3 objects.
      So the block size only really matters when actually transferring data.
    * Amazon charges for transfer by total bytes uploaded ($0.10 per GB) +
      $0.00001 per PUT

In general, since the total size and total bytes transferred are roughly the
same no matter what the s3backer block size is, to a first order approximation,
the larger the block size (fewer PUTs) the lower the cost.

Now if you have really large block sizes, e.g., 2GB, then even a tiny change
will cost you $0.20 + $0.00001 = $0.20 to upload and take a long time. Compare
that to a single 4k block upload which costs $00.0000003 + $0.00001 = $0.00 to
upload and will be very fast.

If you're going to do one initial backup then millions of tiny incremental rsync
updates, a smaller block size might make sense.

If you're going to do a smaller number and/or more substantial backups, a larger
block size makes sense.

So somewhere in there is probably a reasonable happy medium. With a 1MB block size,
your initial sync of 6GB, which will require 6144 PUTs, will cost only $0.60
+ $0.06144 = $0.66 or so. Each incremental update should cost much less than this.

Note that the current release of s3backer is very slow when dealing with large
block sizes (actually, it's the mismatch with the standard 4k kernel page size that
causes it). This has been fixed in SVN so you should use the latest version from
SVN [or version 1.2.1 or later] if you want to play with large block sizes.

Of course, with large block sizes, you must make very good use of the block cache
to avoid repeated writes of the same large block (which would be the worst of both
worlds).

Regarding filesystems, I like Reiser because it can pack multiple files' data into
a single block, which is good if you have a lot of small files, but I haven't really
compared the various filesystems much otherwise.

-Archie
```