---
title: "Atomic ring buffer experiment"
date: 2023-08-3T16:08:17+01:00
draft: false
slug: atomic-ring-buffer-experiment
--- 
# Introduction
Inspired by the [LMAX Disruptor](https://lmax-exchange.github.io/disruptor/disruptor.html), I decided to have a go at writing a fast ring buffer. For the purposes of this post I only used 1 producer and 1 consumer but it would work with multiple threads (with some caveats, discussed later). The interface to this queue is a blocking `push` and `pop` operation, internally there is a cell for each pre-allocated ring buffer element and a head and tail pointer. 

# A queue using only atomics 
Here we use only atomic operations to transmit state about the queue, to allocate a cell we perform an atomic increment operation. However this is not sufficient for readers to know when the value has successfully been written, we need to also track a commit state. The commit state is handled by a bitmask here, because it allows for multiple writers to write/commit at the same time (there is a caveat here going beyond 1 producer/consumer discussed below). 

# Code
To implement, simply using an atomic number for the read and write heads to "allocate" a spot to write or read from is sufficient. However this is not enough to notify others that you have successfully written to the queue (as it is allocate then write), however the alternative is compare-and-swap which comes with a different cost profile. Let's look at the code
```
class ring_buffer { 
    std::atomic_int32_t write_head; // Increment to allocate write slot
    std::atomic_int32_t read_head; // Increment to allocate read slot
    
    std::vector<int> data; // Buffer storage
    std::unique_ptr<std::atomic_uint64_t[]> write_mask; // A mask indicating which cells are written
public:
    ring_buffer(size_t size /* multiple of 64 */)
        : write_head{0}
        , read_head{0} { 
        data.resize(size);
        write_mask.reset(new std::atomic_uint64_t[size/64]);
        for (int i = 0; i < size/64; i++) { 
            write_mask[i].store(0);
        }
    }
    void push_blocking(int entry) { 
        int32_t slot = write_head++; // Allocate
        int32_t index = slot % data.size();
        int32_t mask_index = index / 64;
        uint64_t bit_mask = static_cast<uint64_t>(1) << (index-mask_index*64);
        while ((write_mask[mask_index].load() & bit_mask) == bit_mask);
        data[index] = entry;
        // Publish
        write_mask[mask_index] += bit_mask;
    }

    void pop_blocking(int& entry) { 
        int32_t slot = read_head++; // Allocate read slot
        int32_t index = slot % data.size();
        int32_t mask_index = index / 64;
        uint64_t bit_mask = static_cast<uint64_t>(1u) << (index-mask_index*64);
        while ((write_mask[mask_index].load() & bit_mask) == 0);
        entry = data[index];
        // Publish
        write_mask[mask_index] -= bit_mask;
    }
};
```

One of the tricks here is that the publish step can be written using an atomic increment/decrement function rather than needing to stall the queue until some conditions are met. The reason this may be more useful is that with some careful planning the queue may allow some partial unsorting of the insert/pop to allow faster throughput, if allowed by the data/algorithm running. For instance, if a cell is allocated and an interrupt happens before the publish, the queue is blocked until this thread is scheduled again. Which may be a problem when using blocking waits (like we do above), however it will allow faster throughput when the queue is active (also dependent on the data/processing). For instance, if a queue element takes a second to process, the type of queue is largely insignificant as long as it's sufficiently fast (compared to 1 second). 

# Performance
A very quick and unrealitic environment benchmark on my Macbook Pro (2018 Intel) shows the following performance results:
```
512 queue length
mean:   197360 ns
stddev: 625515 ns
p99:    2736800 ns
p90:    429700 ns
p50     58400 ns 
```

I measured using `clock_gettime(CLOCK_REALTIME, ...)` which may have some inaccuracy, perhaps using hardware counters would provide a more accurate insight. Although the p99 latency may be indicative of the problem described above where a busy waiter is stuck busy waiting for a writer, this can lead the system to get into a suboptimal rhythm where the queue size dictaces when these threads are doing work. The high p99 latency is observed at multiple sizes, however I have not profiled other queue characteristics for comparison, perhaps for a future post. 

# Improvements
- Using acquire/release semantics
- Whether the algorithm can be tweaked to allow out-of-order read/writes and whether this is a nessesary improvement
- Profiling other queue types

# More than 1 producer & 1 consumer 
To go to more threads producing and consuming there needs to be some considerations made, the algorithm which allocates and publishes writes to a cell could hold up the queue if an interrupt happens in between allocation and comitting the write (bitmask write). I don't know how much of a problem this would be but it seems it would become increasingly likely as use increases, although there is not much that can be done here about that. Perhaps an algorithm change could overcome this - such as reading from the first written bit in a range. 

On the read side, the bit mask check would need to ensure that all previous readers were done - meaning that a maximum number of consumers were allowed (and known ahead of time). It shouldn't matter too much if the number of threads were lower than the max but exceeding it would cause problems. 