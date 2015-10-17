---
layout: post
title:  "Tuning G1 GC for spark jobs"
date:   2015-10-14 12:48:50
categories: blog
tags: apache-spark G1GC
---
Using G1GC with spark jobs needs careful tuning to prevent the dreaded Full GC cycles. Recently while monitoring our spark jobs, we noticed that on loading the job with many queries (25 queries per second), frequent GCs were running on the spark driver.

Our initial driver config looked like this :

~~~
spark.driver.memory 10g
spark.driver.maxResultSize 2g
~~~

Following was used in `--driver-java-options`  for gc tuning:

`GC_OPTS=" -XX:+UseG1GC -XX:MetaspaceSize=$CLI_REPLACE_MAXPERMSIZE$ -Xloggc:$gcRelativePath"gc.log" -XX:+PrintGCDetails
-XX:+PrintGCDateStamps -XX:+UseGCLogFileRotation -XX:GCLogFileSize=500M -XX:NumberOfGCLogFiles=$NUM_GC_LOG_FILES "`

Simply put we were using G1GC with all its default configs. Since we are returning very less data per query, we can
ignore the affect of result size.

Observations
===

- Initial observation revealed that peak memory usage was anyways more than the 10g we had configured, so that had to be
  increased.
- We saw frequent Humoungous allocations, which soon led to a full gc, which was literally killing the subsecond
  queries:

~~~
2015-10-08T09:44:54.460+0000: 14820.612: [GC pause (G1 Humongous Allocation) (young) (to-space exhausted), 0.0083437 secs]
   [Parallel Time: 4.5 ms, GC Workers: 18]
      [GC Worker Start (ms): Min: 14820612.0, Avg: 14820612.1, Max: 14820612.1, Diff: 0.2]
      [Ext Root Scanning (ms): Min: 1.6, Avg: 1.8, Max: 4.2, Diff: 2.6, Sum: 33.1]
      [SATB Filtering (ms): Min: 0.0, Avg: 0.0, Max: 0.1, Diff: 0.1, Sum: 0.1]
      [Update RS (ms): Min: 0.0, Avg: 0.2, Max: 0.3, Diff: 0.3, Sum: 3.0]
         [Processed Buffers: Min: 0, Avg: 2.4, Max: 5, Diff: 5, Sum: 43]
      [Scan RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Code Root Scanning (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Object Copy (ms): Min: 0.1, Avg: 0.8, Max: 1.0, Diff: 0.9, Sum: 13.9]
      [Termination (ms): Min: 0.0, Avg: 1.4, Max: 1.8, Diff: 1.8, Sum: 25.9]
      [GC Worker Other (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.2]
      [GC Worker Total (ms): Min: 4.2, Avg: 4.2, Max: 4.3, Diff: 0.2, Sum: 76.2]
      [GC Worker End (ms): Min: 14820616.3, Avg: 14820616.3, Max: 14820616.3, Diff: 0.1]
   [Code Root Fixup: 0.4 ms]
   [Code Root Migration: 0.0 ms]
   [Code Root Purge: 0.0 ms]
   [Clear CT: 0.3 ms]
   [Other: 3.1 ms]
      [Evacuation Failure: 2.0 ms]
      [Choose CSet: 0.0 ms]
      [Ref Proc: 0.5 ms]
      [Ref Enq: 0.0 ms]
      [Redirty Cards: 0.0 ms]
      [Free CSet: 0.1 ms]
   [Eden: 8192.0K(1024.0M)->0.0B(1024.0M) Survivors: 0.0B->0.0B Heap: 15.3G(20.0G)->15.3G(20.0G)]
 [Times: user=0.06 sys=0.00, real=0.01 secs]

2015-10-08T09:44:54.470+0000: 14820.622: [Full GC (Allocation Failure)  15G->902M(20G), 6.2103923 secs]
   [Eden: 0.0B(1024.0M)->0.0B(12.0G) Survivors: 0.0B->0.0B Heap: 15.3G(20.0G)->902.6M(20.0G)], [Metaspace: 148800K->148033K(1179648K)]
 [Times: user=7.38 sys=0.12, real=6.21 secs]
~~~

- Freuqent full GCs led to numerous peaks in our query serve times, whenever a full gc got triggered.

Configuration Tuning
==

On exploring further about humongous allocations, I discovered that an allocation is considered humongous, if it
requests an allocation of size > 50% of heap region. G1GC divides the heap into a number of equal sized blocks called
regions. The region size by default is set as  nearest power of 2 of (TotalHeapSize/2048). Hence the region size can be
1,2,4,8,16,32 mb. In our case it was 8 mb, and one of the calls was requesting a 4.1 mb allocation, thereby leading to
humongous allocation. The problem with these allocations is that G1GC handles humongous objects differently. They are
allocated directly in the old gen (or the survivor space), and cleared only when a full GC runs (This has changed a bit
in newer versions of G1GC, but still the algorithm is not so good), thereby causing the problem.
Straightforward way is to increase the heap region size by this flag:
`-XX:G1HeapRegionSize=16M`
This changes our heap region size to 16mb, and hence all our allocations are no longer considered humongous allocations.

We also enabled a lot of gc logs so that it becomes easier to analze whats happening later :

~~~
-XX:+PrintFlagsFinal -XX:+PrintReferenceGC -XX:+PrintGCTimeStamps -XX:+PrintGCDateStamps -verbose:gc -XX:+PrintGCDetails
-XX:+PrintAdaptiveSizePolicy -XX:AdaptiveSizePolicyOutputInterval=1 -XX:+UnlockDiagnosticVMOptions
-XX:+G1SummarizeConcMark
~~~

We also set this `-XX:InitiatingHeapOccupancyPercent=45`, though this is the default value and no need to specify this
here, but it is a reminder that this one is a powerful parameter and can be tuned further, if our load changes later.
This flag sets the java heap occupancy thershold, which triggers the marking cycle. Lowering this value leads to
frequent minor GCs. We did this to delay a full GC as much as possible. This can be increased depending on how much
garbage is being generated, and how much you can allow delaying the marking cycle. We found that increasing this value
to 60 delivered almost the same results.

We also set `-XX:G1MixedGCLiveThresholdPercent=85`, which actually controls the occupancy threshold of an old region to
be included in a mixed garbage collection cycle. This helps in effective utilization of the old region, before it
contributes in a mixed gc cycle. This also helps in keeping the duration of mixed gc cycles small (will depend on what
rate your application really generates survivor objects) since our queries were mostly generating young gen objects.
Please note that this is an experimental VM option, so we need `-XX:+UnlockExperimentalVMOptions` as well.

We left out a few critical parameters that you must consider when tuning G1 GC, simply because the defaults worked well
for us. That might not be the case for everyone:

`-XX:MaxGCPauseMillis` is simply the most important parameter to tune in G1 GC. As you might be knowing, The G1 GC
has a pause time-target that it tries to meet (soft real time). This parameter helps in setting a target value for
desired maximum pause time. During young collections, the G1 GC adjusts its young generation (eden and survivor sizes) to 
meet the soft real-time target. During mixed collections, the G1 GC adjusts the number of old regions that are collected 
based on a target number of mixed garbage collections, the percentage of live objects in each region of the heap, and the 
overall acceptable heap waste percentage. Read [this][target-pause] post for a detailed discussion of how much G1 GC
adheres to this value, and what can be done to get optimum results. The default value of 200ms works good enough for us
though.

`-XX:ParallelGCThreads` is another useful parameter to tune, as it can help bring down mixed gc pauses, when carefully
tuned. It sets the number of threads used in parallel for processing during GC pauses. The default value is 5/6 of the number 
of logical cores. Again a bit of experimentation is needed to get the magic number that works best for you.

Our final configuration looked like this :

~~~
spark.driver.memory 20g
spark.driver.maxResultSize 2g
~~~

Following was used in `--driver-java-options` :

~~~
GC_OPTS=" -XX:+UseG1GC -XX:G1HeapRegionSize=16M -Xloggc:$gcRelativePath"gc.log" -XX:+PrintFlagsFinal 
-XX:+PrintReferenceGC -XX:+PrintGCTimeStamps -XX:+PrintGCDateStamps -verbose:gc -XX:+PrintGCDetails 
-XX:+PrintAdaptiveSizePolicy -XX:AdaptiveSizePolicyOutputInterval=1 -XX:+UnlockDiagnosticVMOptions 
-XX:+G1SummarizeConcMark -XX:+UseGCLogFileRotation -XX:GCLogFileSize=500M -XX:MaxDirectMemorySize=4096M 
-XX:-ResizePLAB -XX:+UseCompressedOops -XX:InitiatingHeapOccupancyPercent=45 -XX:+UnlockExperimentalVMOptions 
-XX:G1MixedGCLiveThresholdPercent=85 "
~~~

We were able to eliminate full GC cycles all-together, and no peaks were seen in the response time graph after the above
changes. GC tuning is a big factor in optimizing spark jobs (especially for long running ones), and one that needs
careful experimentation.

References
==

- Nice article from [Oracle][oracle] on G1GC tuning
- Humongous Allocations [explained][Humongous]


[oracle]:        http://www.oracle.com/technetwork/articles/java/g1gc-1984535.html
[Humongous]:     http://xmlandmore.blogspot.in/2014/11/g1-gc-humongous-objects-and-humongous.html
[target-pause]:  http://blog.mgm-tp.com/2014/04/controlling-gc-pauses-with-g1-collector/
