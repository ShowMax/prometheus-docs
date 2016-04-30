 ---
 title: When (not) to use varbit chunks
 created_at: 2016-04-30
 kind: article
 author_name: Björn “Beorn” Rabenstein
 ---

 The embedded time-series database (TSDB) of the Prometheus server organizes
 the raw sample data of each time series in chunks of constant 1024 bytes
 size. In addition to the raw sample data, a chunk contains some meta-data,
 which allows the selection of a different encoding for each chunk. The most
 fundamental distictinction is the encoding version. You select the version for
 newly created chunks via the command line flag
 `-storage.local.chunk-encoding-version`. So far, there were only two supported
 versions: 0 for the original delta encoding, and 1 for the improved
 double-delta encoding. With release
 [0.18.0](https://github.com/prometheus/prometheus/releases/tag/0.18.0), we
 added version 2, which is another variety of double-delta encoding. We call it
 _varbit encoding_ because it involves a variable bit-width per sample within
 the chunk. While version 1 is superior to version 0 in almost every aspect,
 there is a real trade-off between version 1 and 2. This blog post will help
 you to make that decision. Version 1 remains the default encoding, so if you
 want to try out version 2 after reading this article, you have to select it
 explicitly via the command line flag. There is no harm in switching back and
 forth, but note that chunks will not change their encoding version once they
 have been created.

## What is varbit encoding?

From the beginning, we designed the chunked sample storage for easy addition of
new encodings. When Facebook published a
[paper on their in-memory TSDB Gorilla](http://www.vldb.org/pvldb/vol8/p1816-teller.pdf),
we were intrigued by a number of similarities between the independently
developed approaches of Gorilla and Prometheus. However, there were also many
fundamental differences, which we studied in detail, wondering if we could get
some inspiration from Gorilla to improve Prometheus.

At the rare occurrence of a free weekend ahead of me, I decided to give it a
try. In a coding spree, I implemented what would later (after a considerable
amount of testing and debugging) become the varbit encoding.

In a future blog post, I will describe the technical details of the
encoding. For now, you only need to know a few characteristics for your
decision between the new varbit encoding and the traditional double-delta
encoding. (I will call the latter just “double-delta encoding” from now on but
note that the varbit encoding also uses double-deltas, just in a different
way.)

## What are the advantages of varbit encoding?

In short: It offers a way better compression ratio. While the double-delta
encoding needs about 3.3 bytes per sample for real-life data sets, the varbit
encoding went as far down as 1.28 bytes per sample on a typical large
production server at SoundCloud. That's almost three times more space efficient
(and even slightly better than the 1.37 bytes per sample reported for Gorilla –
but take that with a grain of salt as the typical data set at SoundCloud might
look different from the typical data set at Facebook).

Now think of the implications: Three times more samples in RAM, three times
more samples on disk, only a third of disk ops, and since disk ops are
currently the bottleneck for ingestion speed, it will also allow three times
faster ingestion. Indeed, the recently reported new ingestion record of 800,000
samples per second was only possible with varbit chunks – and with an SSD,
obviously. With spinning disks, the bottleneck is reached far earlier, and thus
the 3x gain matters even more.

All of this sounds too good to be true…

## So where is the catch?

For one, the varbit encoding is more complex. The computational cost to encode
and decode values is therefore somewhat increased, which fundamentally effects
everything that writes or reads sample data. Luckily, it is only a proportional
increase of something that usually contributes only a small part to the total
cost of an operation.

Another property of the varbit encoding is potentially way more problematic:
samples in varbit chunks can only be accessed sequentially, while samples in
double-delta encoded chunks are randomly accessible by index. This obviously
only affects reading of sample data, and the impact depends heavily on the
nature of the originating PromQL query.

A pretty harmless case is the retrieval of all samples within a time
interval. This happens when evaluating a range selector or rendering a
dashboard with a resolution similar to or higher than the scrape frequency. The
Prometheus storage engine needs to find the starting point of the
interval. With double-delta chunks, it can perform a binary search, while it
has to scan sequestially through a varbit chunk. However, once the starting
point is found, all remaining samples in the interval need to be decoded
sequentially anyway, which is only slightly more expensive with the varbit
encoding.

The tradeoff is different if only one sample is retrieved from a chunk, or a
few chunks that are not near each other within the chunk. 
