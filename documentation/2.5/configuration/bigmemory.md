---
---
# BigMemory

 

## Introduction
[BigMemory](http://www.terracotta.org/bigmemory?src=ehcache_off_heap_store) is a pure Java product from Terracotta that permits caches to use an additional
type of memory store outside the object heap.  It is packaged for use in Enterprise Ehcache as a snap-in job store called the "off-heap store." If Enterprise Ehcache is distributed in a Terracotta cluster, you can configure BigMemory in both Ehcache (the Terracotta client or L1) and in the [Terracotta Server Array (the L2)](http://terracotta.org/documentation/bigmemory/terracotta-server-array).

This off-heap store, which is not subject to Java GC, is 100 times faster than the DiskStore and allows
very large caches to be created (we have tested this with over 350GB).

Because off-heap data is stored in bytes, there are two implications:

* Only Serializable cache keys and values can be placed in the store, similar to DiskStore.
* Serialization and deserialization take place on putting and getting from the store. This means that the off-heap store is slower in an absolute sense (around 10 times slower than the MemoryStore), but this theoretical difference disappears due to two effects:
    * the MemoryStore holds the hottest subset of data from the off-heap store, already in deserialized form
    * when the GC involved with larger heaps is taken into account, the off-heap store is faster on average.

For a tutorial on Ehcache BigMemory, see [BigMemory for Enterprise Ehcache Tutorial](http://terracotta.org/documentation/bigmemory/get-started).


## Configuration


###  Configuring Caches to Overflow to Off-heap

Configuring a cache to use an off-heap store can be done either through XML or programmatically.

If using distributed cache with strong consistency, a large number of locks may need to be stored in client and server heaps. In this case, be sure to test the cluster with the expected data set to detect situations where OutOfMemory errors are likely to occur. In addition, the overhead from managing the locks is likely to reduce performance.

####  Declarative Configuration

The following XML configuration creates an off-heap cache with an in-heap store (maxEntriesLocalHeap)
of 10,000 elements which overflow to a 1-gigabyte off-heap area.

~~~
<ehcache updateCheck="false" monitoring="off" dynamicConfig="false">
    <defaultCache maxEntriesLocalHeap="10000"
                  eternal="true"
                  memoryStoreEvictionPolicy="LRU"
                  statistics="false" />

    <cache name="sample-offheap-cache"
           maxEntriesLocalHeap="10000"
           eternal="true"
           memoryStoreEvictionPolicy="LRU"
           overflowToOffHeap="true"
           maxBytesLocalOffHeap="1G"/>
</ehcache>
~~~


The configuration options are:


##### overflowToOffHeap

Values may be true or false. When set to true, enables the cache to utilize off-heap memory
storage to improve performance. Off-heap memory is not subject to Java
GC cycles and has a size limit set by the Java property MaxDirectMemorySize.
The default value is false.


##### maxBytesLocalOffHeap

Sets the amount of off-heap memory available to the cache and is in effect only if `overflowToOffHeap` is true. The minimum amount that can be allocated is 1MB. There is no maximum.

For more information about sizing caches, refer to [How To Size Caches](/documentation/configuration/cache-size).

<table markdown="1">
<caption>NOTE: Heap Configuration When Using an Off-heap Store</caption>
<tr><td>
You should set maxEntriesLocalHeap to at least 100 elements
when using an off-heap store to avoid performance degradation. Lower values for maxEntriesLocalHeap trigger a warning to be logged.
</td></tr>
</table>

#### Programmatic Configuration <a name="programmatic-configuration"/>

The equivalent cache can be created using the following programmatic configuration:

    public Cache createOffHeapCache()
    {
      CacheConfiguration config = new CacheConfiguration("sample-offheap-cache", 10000)
      .overflowToOffHeap(true).maxBytesLocalOffHeap("1G");
      Cache cache = new Cache(config);
      manager.addCache(cache);
      return cache;
    "/>


### Adding The License <a name="adding-the-license"/>

[Enterprise Ehcache trial download](http://terracotta.org/bigmemory) comes with a license key good for 30 days. Use this key to activate the off-heap store. It can be added to the classpath or via a system property.


#### Configuring the License in the Classpath <a name="configuring-the-license"/>

Add the `terracotta-license.key` to the root of your classpath, which is also where you add ehcache.xml. It will be
automatically found.


#### Configuring the License as a Java System Property <a name="configuring-license-as-java-system-property"/>

Add a `com.tc.productkey.path=/path/to/key` system property which points to the key location.

For example,

    java -Dcom.tc.productkey.path=/path/to/key


### Allocating Direct Memory in the JVM <a name="allocating-direct-memory"/>

In order to use these configurations, you must then use the ehcache-core-ee jar on your classpath and modify your JVM command-line
to increase the amount of direct memory allowed by the JVM. You must allocate at least 256MB more to direct memory than the total off-heap memory allocated to caches.

For example, to allocate 2GB of memory in the JVM:

    java -XX:MaxDirectMemorySize=2G ..."


<table markdown="1">
<caption>NOTE: Direct Memory and Off-heap Memory Allocations</caption>
<tr><td>
To accommodate server communications layer requirements, the value of maxDirectMemorySize must be greater than the value of maxBytesLocalOffHeap. The exact amount greater depends upon the size of maxBytesLocalOffHeap. The minimum is 256MB, but if you allocate 1GB more to the maxDirectMemorySize, it will certainly be sufficient. The server will only use what it needs and the rest will remain available.
</td></tr>
</table>

## Advanced Configuration Options <a name="advanced-configuration-options"/>

There are some additional configuration options which can be used for fine grained control.


### -XX:+UseLargePages <a name="xx-uselargepages"/>

This is a JVM flag which is meant to improve performance of memory-hungry applications.
In testing, this option gives a 5% speed improvement with a 1GB off-heap cache.

See <http://andrigoss.blogspot.com/2008/02/jvm-performance-tuning.html> for a discussion.


### Increasing the Maximum Serialized Size of an Element that can be Stored in the OffHeapStore <a name="increasing-serialized-size"/>

While the MemoryStore and the DiskStore do not have any limits, by default the OffHeapStore has a 4MB limit for classes with high quality hashcodes, and
256KB for those with pathologically bad hashcodes. The built-in classes such as the
`java.lang.Number` subclasses such as Long, Integer etc and and `String` have
high quality hashcodes.

You can increase the size by setting a system property `net.sf.ehcache.offheap.cache_name.config.idealMaxSegmentSize` to
the size you require.

For example, 

    net.sf.ehcache.offheap.com.company.domain.State.config.idealMaxSegmentSize=30M


### Avoiding OS Swapping <a name="avoiding-os-swapping"/>

Operating systems use swap partitions for virtual memory and are free to move less frequently used pages of memory to
the swap partition. This is generally not what you want when using the OffHeapStore, as the time it takes to swap a page
back in when demanded will add to cache latency.

It is recommended that you minimize swap use for maximum performance.

On Linux, you can set `/proc/sys/vm/swappiness` to reduce the risk of memory pages being swapped out. See <http://lwn.net/Articles/83588/>
for details of tuning this parameter. Note that there are bugs in this which were fixed in kernel 2.6.30 and higher.

Another option is to configure HugePages. See <http://unixfoo.blogspot.com/2007/10/hugepages.html>


Although there's a swappiness kernel parameter that can be set to zero in Linux, it is usually not enough to avoid swapping altogether. The only surefire way to avoid any kind of swapping is either (a) disabling the swap partition, with the undesirable consequences which that may bring, or (b) using HugePages, which are always mapped to physical memory and cannot be swapped out to disk.


### Compressed References <a name="xx-usecompressedoops"/>

The following setting applies to Java 6 or higher. Its use should be considered to make the most efficient use of memory in 64 bit mode. For the Oracle HotSpot, compressed references are enabled using the option `-XX:+UseCompressedOops`. For IBM JVMs, use `-Xcompressedrefs`. 
See <https://wikis.oracle.com/display/HotSpotInternals/CompressedOops> for details.


### Controlling Over-allocation of Memory to the OffHeapStore <a name="controlling-overallocation"/>

It is possible to over-allocate memory to the off-heap store and overrun the physical and even virtual memory available on the OS. See [Slow Off-Heap Allocation](#slow-off-heap-allocation) for how to handle this situation.


## Sample Application <a name="sample-application"/>

The easiest way to get started is to play with a simple, sample app. Download from  [here](http://d2zwv9pap9ylyd.cloudfront.net/ehcache-offheap-sample.zip) a simple Maven-based application that uses the ehcache off-heap functionality.

Note: You will need to get a [license key](http://www.terracotta.org/bigmemory?src=ehcache.org "BigMemory for Enterprise Ehcache  | Terracotta") and install it as
discussed above to run this.


## Performance Comparisons <a name="performance-comparisons"/>

Checkout <http://svn.terracotta.org/svn/forge/offHeap-test/> for a Maven-based performance comparison test between different store
configurations.

Note: To run the test, you will need to get a demo [license key](http://www.terracotta.org/bigmemory?src=ehcache.org "BigMemory for Enterprise Ehcache  | Terracotta") and install it as discussed
above.

Here are some charts from tests we have run on BigMemory.

The test machine was a Cisco UCS box running with Intel(R) Xeon(R) Processors. It had 6 2.93Ghz Xeon(R) CPUs for a total of 24 cores,
with 128GB of RAM, running RHEL5.1 with Sun JDK 1.6.0_21 in 64 bit mode.

We used 50 threads doing an even mix of reads and writes with 1KB elements. We used the default garbage collection settings.

The tests all go through a load/warmup phase and then start a performance run. You can use the tests in your own environments and extend
them to cover different read/write ratios, data sizes, -Xmx settings and hot sets. The full suite, which is done with
`run.sh` takes 4-5 hours to complete.

The following charts show the most common caching use case. The read/write ratio is 90% reads and 10% writes. The hot set is that 90%
of the time `cache.get()` will access 10% of the key set. This is representative of the the familiar Pareto distribution that is
very commonly observed.

There are of course many other caching use cases. Further performance results are covered on the [Further Performance Analysis](http://ehcache.org/documentation/configuration/bigmemory-further-performance-analysis) page.


### Largest Full GC <a name="largest-full-gc"/>

![Offheap Fullgc](/images/documentation/offheap_fullgc.png)

This chart shows the largest observed full GC duration which occurred during the performance run. Most non-batch applications have maximum response time
SLAs. As can be seen in the chart, as data sizes grow the full GC gets worse and worse for cache held on heap, whereas off-heap remains
a low constant.

The off-heap store will therefore enable applications with maximum response time SLAs to reliably meet those SLAs.


### Latency <a name="latency"/>

![Offheap Maxlatency](/images/documentation/offheap_maxlatency.png)

This chart shows the maximum observed latency while performing either a `cache.put()` or a `cache.get()`. It is very similar to the
Full GC chart because the full GCs are causing the on-heap latencies to rise dramatically, as all threads in the test app get frozen.

Once again the off-heap store can be observed to have a flat, low maximum latency, because any full GCs are tiny, and the cache has excellent
concurrency properties.

![Offheap Latency](/images/documentation/offheap_latency.png)

This chart shows the off-heap mean latency in microseconds. It can be observed to be flat from 2GB up to 40GB. Further in-house testing shows
that it continues to remain flat to the limits that we have tested.

Lower latencies are observed at smaller data set sizes because we use a `maxEntriesLocalHeap` setting which approximates to
200MB of on-heap store. On-heap, excluding GC effects, is faster than off-heap because there is no deserialization on gets.
At lower data sizes, there is a higher probability that the small on-heap store will be hit, which is reflected in the lower average
latencies.


### Throughput <a name="throughput"/>

![Offheap Tps](/images/documentation/offheap_tps.png)

This chart shows the cache operations per second achieved with off-heap. It is the inverse of average latency and shows much the same
thing. Once the effect of the on-heap store becomes marginal, throughput remains constant, regardless of cache size. 

Note that much larger throughputs than those shown in this chart are achievable. Throughput is affected by:

* the number of threads (more threads -> more throughput)
* the read/write ratio (reads are slightly faster)
* data payload per operation (more data implies a lower throughput in TPS but similar in bytes)
* CPU cores available and their speed (our testing shows that the CPU is always the limiting factor with enough threads. In other words, cache throughput can be increased by adding threads until all cores are utilised and then adding CPU cores - an ideal situation where the software can use as much hardware as you can throw at it.)


## Storage <a name="storage"/>


### Storage Hierarchy <a name="storage-hierarchy"/>

With the OffHeapStore, Enterprise Ehcache has three stores:

* MemoryStore - very fast storage of Objects on heap. Limited by the size of heap you can
comfortably garbage collect.
* OffHeapStore - fast (one order of magnitude slower than MemoryStore) storage of
Serialized objects off-heap. Limited only by the amount of RAM on your hardware and
address space. You need a 64 bit OS to address higher than 2-4GB.
* DiskStore - speedy storage on disk. It is two orders of magnitude slower than the
OffHeapStore but still much faster than a database or a distributed cache.

The relationship between speed and size for each store is illustrated below:

![Tiered Storage](/images/documentation/tiered_storage.png)


### Memory Use in Each Store <a name="memory-user-store"/>

As a performance optimization, and because storage gets much cheaper as you drop down through
the hierarchy, we write each put to as many stores as are configured. So, if all three are
configured, the Element may be present in MemoryStore, OffHeapStore and DiskStore.

The result is that each store consumes storage for itself and the other stores higher up the
hierarchy. So, if the MemoryStore has 1,000,000 Elements which consume 2GB, and the
OffHeapStore is configured for 8GB, then 2GB of that will be duplicate of what is in the
MemoryStore. And the 8GB will also be duplicated on the DiskStore plus the DiskStore will have
what cannot fit in any of the other stores.

This needs to be taken into account when configuring the OffHeap and Disk stores.

It has the great benefit, which pays for the duplication, of not requiring copy on eviction. On eviction from a
store, an Element can simply be removed. It is already in the next store down.

One further note: the MemoryStore is only populated on a read. Puts go to the OffHeapStore and then when read, are
held in the MemoryStore. The MemoryStore thus holds hot items of the OffHeapStore. This will result in a difference in what
can be expected to be in the MemoryStore between this implementation and the Open Source one. A "usage" for the purposes of the
eviction algorithms is either a put or a get. As only gets are counted in this implementation, some differences will be observed.


## Handling JVM Startup and Shutdown <a name="handling-jvm-lifecycle"/>

So you can have a huge in-process cache. But this is not a distributed cache, so when you shut down
you will lose what is in the cache. And when you start up, how long will it take to load the cache?

In caches up to a GB or two, these issues are not hugely problematic. You can often pre-load the cache
on start-up before you bring the application online. Provided this only takes a few minutes, there is
minimal operations impact.

But when we go to tens of GBs, these startup times are O(n), and what took 2 minutes now takes 20 minutes.

To solve this problem, we provide a new implementation of Ehcache's DiskStore, available in the Enterprise
version.

You simply mark the cache `diskPersistent=true` as you normally would for a disk persistent cache.

It works as follows:

* on startup, which is immediate, the cache will get elements from disk and gradually fill the MemoryStore and the OffHeapStore.
* when running elements are written to the OffHeapStore, they are already serialized. We write these to the DiskStore asynchronously in a write-behind pattern. Tests show they can be written at a rate of 20MB/s on server-class machines with fast disks. If writes get behind, they will back up and once they reach the `diskSpoolBufferSizeMB` cache puts will be slowed while the DiskStore writer catches up. By default this buffer is 30MB but can be increased through configuration.
* When the Cache is disposed, only a final sync is required to shut the DiskStore down.


## Using OffHeapStore with 32-bit JVMs

On a 32-bit operating system, Java will always start with a 32-bit data model. On 64-bit OSs, it will default to
64-bit, but can be forced into 32-bit mode with the Java command-line option `-d32`.
The problem is that this limits the size of the process to 4GB. Because garbage collection problems are generally
manageable up to this size, there is not much point in using the OffHeapStore, as it will simply be slower.

If you are suffering GC issues with a 32-bit JVM, then OffHeapStore can help. There are a few
points to keep in mind.

* Everything has to fit in 4GB of addressable space. If you allocate 2GB of heap (with `-Xmx2g`) then you have at most 2GB left for your off-heap caches.
* Don't expect to be able to use all of the 4GB of addressable space for yourself. The JVM process requires some of it for its code and shared libraries plus any extra Operating System overhead.
* If you allocate a 3GB heap with -Xmx as well as 2047MB of off-heap memory, the virtual machine certainly won't complain at startup, but when it's time to grow the heap you will get an OutOfMemoryError.
* If you use both -Xms3G and -Xmx3G with 2047MB of off-heap memory, the virtual machine will start but then complain as soon as the OffHeapStore tries to allocate the off-heap buffers.
* Some APIs, such as java.util.zip.ZipFile on Sun 1.5 JVMs, may &lt;mmap> files in memory. This will also use up process space and may trigger an OutOfMemoryError.

For these reasons we issue a warning to the log when OffHeapStore is used with 32-bit JVMs.

## Slow Off-Heap Allocation <a name="slow-off-heap-allocation"/>

Based on its configuration and memory requirements, each cache may attempt to allocate off-heap memory multiple times. If off-heap memory comes under pressure due to over-allocation, the system may begin paging to disk, thus slowing down allocation operations. As the situation worsens, an off-heap buffer too large to fit in memory can quickly deplete critical system resources such as RAM and swap space and crash the host operating system. 

To stop this situation from degrading, off-heap allocation time is measured to avoid allocating buffers too large to fit in memory. If it takes more than 1.5s to allocate a buffer, a warning is issued. If it takes more than 15s, then the JVM is halted with `System.exit()` (or a different method if the Security Manager prevents this). 

To prevent a JVM shutdown after a 15s delay has occurred, set the `net.sf.ehcache.offheap.DoNotHaltOnCriticalAllocationDelay` system property to true. In this case, an error is logged instead.


## Reducing Cache Misses <a name="reducing-cache-misses"/>

While the MemoryStore holds a hotset (a subset) of the entire data set, the off-heap store should be large enough to hold the entire data set. The frequency of cache misses begins to rise when the data is too large to fit into off-heap memory, forcing gets to fetch data from the DiskStore. More misses in turn raise latency and lower performance.

For example, tests with a 4GB data set and a 5GB off-heap store recorded no misses. With the off-heap store reduced to 4GB, 1.7 percent of cache operations resulted in misses. With the off-heap store at 3GB, misses reached 15 percent.


## FAQ <a name="faq"/>


### What Eviction Algorithms are supported?

The pluggable MemoryStore eviction algorithms work as normal. The OffHeapStore and DiskStore use a
Clock Cache, a standard paging algorithm which is an approximation of LRU.


### Why do I see performance slow down and speed up in a cyclical pattern when I am filling a cache?

This is due to repartitioning in the OffHeapStore which is normal. Once the cache is fully filled
the performance slow-downs cease.


### What is the maximum serialized size of an object when using OffHeapStore?

Refer to "Increasing the maximum serialized size of an Element that can be stored in the OffHeapStore" in the [Advanced Configuration Options](#advanced-configuration-options) section above.


### Why is my application startup slower?

On startup the CacheManager will calculate the amount of off-heap storage required for all caches
using off-heap stores. The memory will be allocated from the OS and zeroed out by Java. The time taken
will depend on the OS. A server-class machine running Linux will take approximately half a second per GB.

We print out log messages for each 10% allocated, and also report the total time taken.

This time is incurred only once at startup. The pre-allocation of memory from the OS is one of the reasons
that runtime performance is so fast.


### How can I do Maven testing with BigMemory?

Maven starts java for you. You cannot add the required `-XX` switch in as a `mvn` argument.

Maven provides you with a MAVEN_OPTS environment variable you can use for this on Unix systems.

For example, to specify 1GB of MaxDirectMemorySize and then to run jetty:

~~~
export MAVEN_OPTS=-XX:MaxDirectMemorySize=1G
mvn jetty:run-war
~~~