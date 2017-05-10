
## 1. SUMMARY
> We have implemented a Cache Simulator for analyzing how different Snooping-Based Cache Coherence Protocols perform
under various workloads. Given any program, we can use our simulator to compare the performance of various protocols,
based on number of Bus Transactions, Memory Requests, Memory Write-Backs and Cache-to-Cache Transfers.

## 2. BACKGROUND
> We have studied about different snooping based Cache Coherence Protocols in class. Whenever a processor wants to read
or write something, it tries to use its own cache to avoid having to go to the memory each time (as it’s very slow).
But, when we have multiple processors, we need to synchronize the caches, so that all processors have a coherent view
of memory. For this, one approach is to use a snooping cache, where each cache monitors the memory reads and writes done
by other caches and takes some action based on those requests. MSI is one simple choice but there are other protocols
too which offer different kinds of benefits under specific workloads - MESI, MOSI, MOESI, Dragonfly, etc.
> With this project we basically aim to study and demonstrate the advantages of each protocol over the others based on
some real-world, and some synthetic memory traces. We have also implemented a split-bus memory access, so that we can
mimic memory transactions similar to how they happen in real machines, instead of using an atomic bus. This gives us a
clear understanding of the performance of each snooping based protocol.
> The inputs for analysis have been derived from programs that are concurrent in nature. We created two kinds of
programs - one based on locks, so that the read/writes are in a FIFO manner; and other based on random (aka wild)
access to the memory, which neither maintain consistency of data nor the order of transactions. We also designed some
artificial test inputs to be able to clearly differentiate the performance of each snooping based protocol.

## 3. APPROACH

> The protocols that we have implemented are: <br>
> Write-Invalidate Protocols <br>
> 1\. MSI <br>
> ![alt text][MSI.png] <br>
> 2\. MESI <br>
> ![alt text][MESI.png] <br>
> 3\. MOSI <br>
> MOSI - Processor Transactions: <br>
> ![alt text][MOSI-processor.png] <br>
> MOSI - Snooped Bus Transactions: <br>
> ![alt text][MOSI-snooping.png] <br>
> 4\. MOESI <br>
> ![alt text][MOESI.png] <br>
> Write-Update Protocols <br>
> 1\. Dragonfly (write-back update) <br>
> ![alt text][Dragon.png] <br>
> All of these protocols use write-allocate write-back caching. <br>
### 3.1. Working:
> We use Intel’s pintool to instrument programs and generate memory traces for them. For this, we wrote our own pintool,
which for each instruction, checks whether it has a memory access or not. If it has a memory access, then at runtime we
record the following information: <br>
> 1\. Whether the access is a read or a write <br>
> 2\. The memory address for the load/store <br>
> 3\. The thread-ID for the thread which performed the load/store <br> <br>
> This is saved in a file in the following format : <br>
> ```javascript
> [Thread-ID] [R/W] [Memory_Address]
> ```
> Example: <br>
> ```javascript
> 0 R 0x1024
> 0 W 0x1024
> 1 R 0x1088
> 1 W 0x1088
> 2 R 0x1152
> ```
> For improved analysis of a program, we have written another pintool which allows demarcating the code which we want to
 analyze using the simulator. This is to help in the scenario, where a program is very large and there are only specific
  parts of the program that we want to analyze. We do this by calling two dummy functions: dummy_instr_start() and
   dummy_instr_end(). The pintool recognizes calls to these functions, so when it sees a dummy_instr_start(), it
   starts recording memory accesses and on dummy_instr_end(), it stops recording them. <br>
> Once we have the memory trace generated by one of the pintools, we pass it to our Cache Simulator. <br>
### 3.2. Cache Simulator Design:
> Our Cache Simulator allows varying the following parameters: <br>
> -c Number of Processor Cores/Caches <br>
> -s Cache Size (in MB) <br>
> -a Set Associativity <br>
> -p Cache Coherence Protocol to use (MSI/MESI/MOSI/MOESI/Dragon) <br>
> -t The trace file to use <br>
 <br> <br>
> At initialization, we create caches for each processor, and we create two pThreads for each cache. One thread is
responsible for handling requests from the processor and initiating bus transactions if necessary (processor
transactions), while the other thread is responsible for responding to bus transactions (snooped bus transactions). For
accesses to memory, we have simulated a split-transaction bus. We create a pThread to act as the memory controller. We
have also added a small delay(of the order of couple ms), so that each memory request has some considerable latency
associated with it. This also means that while a memory request is pending, another cache might request the same
cacheline. This causes some interesting correctness issues that we will discuss later. <br>
> The high level workflow of our simulator is as follows: <br>
> The main thread is responsible for reading the memory trace and instructing the appropriate caches to perform the
required memory read/write. This is implemented using a signalling mechanism (mutex and condition variables). <br>
> When a cache worker receives the signal for the memory access, it checks the state of the cacheline which has the
address and executes the protocol transitions if any. If the transition requires a snooping bus transaction, it
contends for the bus and broadcasts the action it wants to perform. The snooping threads for the other caches read
this action and depending on the protocol, they might change the status of the cache line, invalidate it, flush it to
memory, etc. If the cache worker who is performing the action does not have the cacheline in a valid state, one of the
other caches can place the data on the bus for this cache to read. Once the snooping is done, the cache worker might be
done with the handling of the memory access, or it might need to go to memory to fetch the cacheline (if no other cache
had the cacheline). In this case, we use the split transaction bus to request the cacheline from memory. <br>
> This continues till all caches are done with the memory accesses they were required to perform. <br>
> Based on the actions that a cache performed, we update the number of snooping bus transactions, memory requests,
memory writebacks and cache to cache transfers. <br>
### 3.3. Metrics Used for Comparison
> 1\. Number of Snooping Bus Transactions - The number of bus transactions that the protocol implementation requires
for the program. <br>
> 2\. Number of Memory requests - The number of cacheline loads that have to be served by main memory (and not another
cache).  <br>
> 3\. Number of Memory writebacks - The number of times any cacheline has to be written back to main memory. <br>
> 4\. Number of Cache to Cache Transfers - The number of cacheline loads that are served by another cache. <br>
 <br> <br>
> Ideally, we want as few bus transactions, memory requests, memory write-backs or cache to cache transfers as possible
because all of them will have some amount of latency associated with them. If it is a choice between a memory request
and a cache to cache transfer, we always prefer the latter, because a memory request will have much higher latency
compared to getting data from another cache. <br>
### 3.4. Interesting Synchronization and Implementation Issues:
> As described earlier, we create separate threads to simulate various parts of the cache. To maintain and ensure
correctness, there are certain issues we need to take care of:
> #### 3.4.1. Split Transaction Bus
> Having a split transaction bus means that when there is a request from one cache pending, another cache can request
load of the same cacheline. In this case, the memory does not need to service the same request twice. It just needs to
provide the data for the cacheline to both the caches, or in other words, both the caches need to be listening for the
memory’s servicing of the same request. For implementing this, we create a separate condition variable for each
request. When a cache needs something serviced from memory, it first checks whether the access already exists in the
request table. If it already exists, instead of creating another memory request in the table, it simply waits on the
condition variable of the existing request. When the memory worker is done servicing the request, it performs a
broadcast on the condition variable associated with the request, so that all caches that were waiting for that request
can continue with their work.
> #### 3.4.2. Cache Worker and Snooping Worker
> We have two workers (pThreads) for each cache. One for servicing load/store requests from the processor and one for
servicing for snooping bus transactions. Both these threads require concurrent access to the same cache, and hence the
accessed need to be synchronized. Our implementation uses a single copy of the cache with pthread mutexes for
synchronization.
> #### 3.4.3. Snooping Bus
> The current implementation of the snooping bus uses pthread mutices and condition variables. The order in which caches
gain access to the bus is completely dependent on the implementation of pthread mutex and condition variables. Based
on our implementation, we can easily enforce a specific ordering on the arbitration, but we decided to use the simple
default implementation. To explain in brief, we have a mutex for getting access to the snooping bus for broadcasting
actions. Another mutex is required for other snooping caches to respond to this action. This also involves the
snooping caches informing the cache worker whether the cacheline would be in the exclusive or shared state, or if the
cache worker does not have the cacheline, whether the snooping cache can provide the data for the cacheline. A snooping
caches can also NACK the transaction, if its own cache has a conflicting pending transaction that is waiting to be
serviced by the split transaction bus. <br>
### 3.5. Protocols Low Level Design Choices:
> We have designed all the aforementioned protocols so that a cache miss will try to be serviced from another cache
(if a cache already has the data). This reduces the number of memory requests required. <br>
> In MOSI and MOESI implementations, memory writebacks only happen when a cacheline in Modified or Owner state is
evicted. If a cacheline is invalidated due to the protocol, the cache transfers the cacheline to the new owner of the
cacheline. (We deem the cache which has a line in the Modified or Owner state to be the owner.) This enhances the
effect of the additional Owner state. <br>
> In our Dragon implementation, when a line is in the SharedClean or SharedModified state and tries to perform a write,
we first check whether it is the only cache holding the line or not. If it is the only one holding the cacheline, then
instead of having to perform a BusUpdate and moving to the SharedModified state, it can simply move to the Modified
state. This helps in the case where other caches have evicted that line causing only one cache to have the line. This
saves on unnecessary bus transactions.

## 4. THE CHALLENGE
> It would be interesting to implement various snooping cache based protocols and measure their performance
under different kinds of workloads. We wish to be able provide memory traces to our analyzer and by simulating
the various protocols, we will be able to determine the behavior of each protocol under varying circumstances.
This will help us gain a better understanding of how each protocol might have been implemented, and also what
the strong and weak points of each are.

## 5. RESOURCES
> We will be writing the code from scratch. And since this is an analysis project, we don’t need any additional
hardware. We will be able to run it our own laptops.
We still need to figure out how to obtain/generate the memory traces. Currently, we are thinking that we might
generate our own memory traces.

## 6. GOALS AND DELIVERABLES
> ### 6.1 Plan to achieve
> Cache Coherence Protocols <br>
> We plan to do the analysis based on comparisons between the following 4 protocols: <br>
> 1\. MSI <br>
> 2\. MESI <br>
> 3\. MOSI <br>
> 4\. MOESI <br>
>  <br> If time permits we will try to add a few more protocols to our analysis results: <br>
> 1\. Dragon <br>
> 2\. MESIF <br>
> 3\. Firefly <br>
>  <br> We will try to analyse how these protocols respond to various work loads. We will first begin with a fixed
system configuration (fixed cache size, set associativity, LRU replacement policy, number of processors) to
study the effects of different protocols. And further expand it to make the system configurable for better
analysis. <br>
>### 6.2 Memory Traces
> As mentioned earlier, we are still unsure of the kind of memory traces we will be using to compare the
various protocols. Generating manual traces seems favorable because we will be able to control the workload
properties. For example, are the accesses primarily reads, or is it write oriented, is just one of the cores
modifying or are all of them trying to modify. Generating workloads that mimic such different properties will
lead to more interesting comparisons between the protocols  <br>
>### 6.3 Metrics
> To compare the performance of the various protocols, we can think of two metrics.
Number of memory accesses - This includes all memory writebacks, flushes and reads. Memory accesses are slow,
hence a system that requires lower number of memory requests as compared to cache should give better performance.
This of course assumes that the interconnect used for cache coherence has much lower latency as compared to
memory. <br>
> Number of bus transactions - The total number of bus transactions that are needed for executing the memory
trace. The bus messages used in snooping cache coherence protocols have to be broadcasted so that every cache
controller can see them. This generates a lot of traffic on the interconnect. Depending on the the size of the
system and the available bandwidth, there is some latency associated with each bus transaction. We would prefer
a system that minimizes the number of bus transactions. <br>

## 7. PLATFORM CHOICE
> We will be using C++ as the programming language, as this project mainly involves being able to read the
memory traces from input files and dumping them into an output file. This can be easily done in C++.

## 8. SCHEDULE
> Apr 11 - Apr 17 - Implement a simple LRU cache for a single processor <br>
> Apr 18 - Apr 24 - Add support for cache coherence protocols - MSI, MESI <br>
> Apr 25 - May 01 - Add additional protocols - MOSI, MOESI, Dragon, Firefly, etc. <br>
> May 02 - May 07 - Generate different types of memory workloads and Perform Analysis <br>
> May 09 - May 11 - Project Presentation Preparation <br>

[//]: # (Links to images below this line)

[MSI.png]: http://15418.courses.cs.cmu.edu/spring2017content/lectures/10_cachecoherence1/images/slide_028.jpg "MSI Protocol"
[MESI.png]: http://15418.courses.cs.cmu.edu/spring2017content/lectures/10_cachecoherence1/images/slide_033.jpg "MESI Protocol"
[MOSI-processor.png]: https://upload.wikimedia.org/wikipedia/commons/4/4d/MOSI_Processor_Transactions.png "MOSI-1 Protocol"
[MOSI-snooping.png]: https://upload.wikimedia.org/wikipedia/commons/5/50/MOSI_Bus_Transactions_Updated.png "MOSI-2 Protocol"
[MOESI.png]: http://wiki.expertiza.ncsu.edu/images/thumb/4/4e/MOESIfig.jpg/450px-MOESIfig.jpg "MOESI Protocol"
[Dragon.png]: http://15418.courses.cs.cmu.edu/spring2017content/lectures/10_cachecoherence1/images/slide_038.jpg "Dragon Protocol"
[basic_stats]: https://kshitizdange.github.io/418CacheSim/images/Basic_stats.png "Basic Stats Graph"


