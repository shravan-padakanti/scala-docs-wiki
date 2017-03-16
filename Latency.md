

In the Parallel Programming course we learned about:

* Data Parallelism in the single machine, multi-core, multiprocessor world.
* _Parallel Collections_ as an implementation of this paradigm.

Here we will learn:

* Data Parallelism in a distributed (multi node) setting
* _Distributed collections_ abstraction from Apache Spark as an implementation of this paradigm.

Because of the **distribution**, we have 2 _new_ issues:

1. Partial Failure: crash failures on a subset of machines in the cluster.
2. Latency: network communication causes higher latency in some operations - **cannot be masked and always present; impacts programming model as well as code directly as we try to reduce network communication**. 

**Apache Spark** stands out in the way it handles these issues.

### Latency Numbers Every Programmer Should Know

```
Latency Comparison Numbers
--------------------------
L1 cache reference                           0.5 ns
Branch mispredict                            5   ns
L2 cache reference                           7   ns                      14x L1 cache
Mutex lock/unlock                           25   ns
Main memory reference                      100   ns                      20x L2 cache, 200x L1 cache
Compress 1K bytes with Zippy             3,000   ns        3 us
Send 1K bytes over 1 Gbps network       10,000   ns       10 us
Read 4K randomly from SSD*             150,000   ns      150 us          ~1GB/sec SSD
Read 1 MB sequentially from memory     250,000   ns      250 us
Round trip within same datacenter      500,000   ns      500 us
Read 1 MB sequentially from SSD*     1,000,000   ns    1,000 us    1 ms  ~1GB/sec SSD, 4X memory
Disk seek                           10,000,000   ns   10,000 us   10 ms  20x datacenter roundtrip
Read 1 MB sequentially from disk    20,000,000   ns   20,000 us   20 ms  80x memory, 20X SSD
Send packet US -> Europe -> US     150,000,000   ns  150,000 us  150 ms
```

* See that reading 1MB sequentially from disk is 100x more expensive than reading 1MB sequentially from memory.
* Also, sending packet over the network from US -> Europe -> US is a million times expensive than a main memory reference.
* The general trend is: 
    * memory operations  = fastest
    * disk operations    = slow
    * network operations = slowest

### Latency numbers visual
[latency_numbers]

### Humanized numbers

Lets multiply all these durations by a billion:

```
# Minute:
L1 cache reference                  0.5 s         One heart beat (0.5 s)
Branch mispredict                   5 s           Yawn
L2 cache reference                  7 s           Long yawn
Mutex lock/unlock                   25 s          Making a coffee

# Hour:
Main memory reference               100 s         Brushing your teeth
Compress 1K bytes with Zippy        50 min        One episode of a TV show (including ad breaks)

# Day
Send 2K bytes over 1 Gbps network   5.5 hr        From lunch to end of work day

# Week
SSD random read                     1.7 days      A normal weekend
Read 1 MB sequentially from memory  2.9 days      A long weekend
Round trip within same datacenter   5.8 days      A medium vacation
Read 1 MB sequentially from SSD    11.6 days      Waiting for almost 2 weeks for a delivery

# Year
Disk seek                           16.5 weeks    A semester in university
Read 1 MB sequentially from disk    7.8 months    Almost producing a new human being
The above 2 together                1 year

# Decade
Send packet CA->Netherlands->CA     4.8 years     Average time it takes to complete a bachelor's degree
```

