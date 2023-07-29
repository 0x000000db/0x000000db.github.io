---
title: "Why does shared_ptr sometimes lock?"
date: 2023-07-28T16:08:17+01:00
draft: true
slug: why-does-shared-ptr-sometimes-lock
---
# Introduction
In this post I want to explore why std::shared_ptr sometimes uses a mutex instead of using atomic instructions. I discovered this a while back when implementing a lazy resource cache for a game, which essentially stored a table of weak_ptr references to already loaded data from disk, encouraging sharing of memory and not an expensive load from disk again. However when it came to profiling I noticed that `weak_ptr::lock` had climbed to the top with a mutex lock, why would this be? It's worth noting I was running on an intel based Mac at the time, also that this was some time ago, and is likely not relevant on modern CPU systems. 

# The details
A shared_ptr is typically made from 2 important components, the resource tracking part (how many refefrences there are aka `ptr.use_count()`) and the data it points to. The first part is what we're interested in here, it stores two important components: a strong reference count and a weak reference count. The rules are if the strong reference count reaches 0 then the data pointed to component is destructed (memory _may_ retain), and if both strong and weak counts reach 0, then everything can be deallocated. This is important because a weak count can remain whilst strong reaches zero, meaning a `weak_ptr::lock` will fail. 

# Implementation
A reference tracking block could look like the following:
```
struct ref_tracking { 
    size_t strong_count;
    size_t weak_count;
};
```
For a `shared_ptr` copy/assignment we can reason that `strong_count` is non-zero (otherwise it would be invalid) so simply an atomic increment will do. The same logic applies to `weak_ptr` and `weak_count`. But what now if we want to create a `strong_ptr` from a `weak_ptr`? The difference here is that a `strong_count` may be zero, an atomic increment will not do. What about compare and swap? This seems to be possible since we only need to operate on `strong_count` with two conditions: If 0, the pointer is no longer alive return nullptr, otherwise increment and return a strong reference. The important distinction is that if an atomic add from 0 to 1 happened then it will corrupt use on other threads if 1 was read but the value had dropped to 0. 

It becomes a little more complicated for decrement, we must only destroy the data once. Which means that we need to ensure if we are the last decrement then it is safe to do so, decrementing either `strong_count` or `weak_count` can result in this but again would require a compare and swap to track if we were the reason both counts were equal to zero. For this reason, we would require a double-wide atomic which we can see a hint of [here](https://github.com/gcc-mirror/gcc/blob/5ffa9d0a5e22f6f763b7f04877a940689e7abcba/libstdc%2B%2B-v3/include/bits/shared_ptr_base.h#L317). Are double-wide atomics a widely supported feature? The instruction for x86 is `cmpxchg16b` which requires data to be 16-byte aligned also (spec [here](https://www.felixcloutier.com/x86/cmpxchg8b:cmpxchg16b.html)). 

# Conclusion
The main details for whether shared_ptr is lock-free appear to be around support for atomic instructions, I'm sure most modern systems are capable but it was an interesting find. My macbook pro at the time most likely did not have the requirements, which means shared_ptr is not a lock-free implementation! 