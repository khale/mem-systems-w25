+++
title = "SimplePIM: A Software Framework for Productive and Efficient Processing-in-Memory"
[extra]
bio = """ """
[[extra.authors]]
name = "Nanda Velugoti (leader, blogger)"
[[extra.authors]]
name = "Benjamin Knutson (scribe)"
[[extra.authors]] 
name = "David Luo" 
[[extra.authors]] 
name = "Rabecka Moffit"
[[extra.authors]] 
name = "Eugene Cohen"
+++

# Introduction

PIM stands for Processing-In-Memory and as the name suggests the idea here is that, instead of doing compute in CPU, it can be done in the DRAM by embedding a processing unit within it, this is usually referred to as DPU (DRAM Processing Unit) [1]. The obvious benefit of having such a mechanism is that, memory bound workloads avoid the DRAM-to-CPU (and vice versa) communication overhead. This is shown in many common workloads where offloading such memory bound compute to DPUs results in better throughput and overall performance [2].

# Design and Implementation

SimplePIM is a software framework that provides a convenient userspace API that allows programmers to easily implement PIM features into their applications. SimplePIM does this by providing a set of PIM programming primitives for managing, communicating and processing data structures between host (CPU) and DPU.

### SimplePIM API

- Management Interface: `simple_pim_array_lookup()`, `simple_pim_array_register()`, `simple_pim_array_free()`
- Communication Interface: `simple_pim_array_broadcast()`, `simple_pim_array_scatter()`, `simple_pim_array_gather()`, `simple_pim_array_allreduce()`, `simple_pim_array_allgather()`
- Processing Interface: `simple_pim_create_handle()`, `simple_pim_array_map()`, `simple_pim_array_red()`, `simple_pim_array_zip()`

### Optimizations
- Strength reduction
- Loop unrolling
- Avoiding boundary checks
- Function inlining
- Adjustment of data transfer sizes


# Results

SimplePIM results in, up to 83% reduction in lines of code and improves productivity up to 5.93x. Along with these improvements, SimplePIM speeds up workloads by 1.10x - 1.43x when compared to their hand-optimized implementations.

# Discussion

- **Why is PIM being explored more now?** Things like graph analytics and computations that do not have a lot of cache access can benefit from this type of hardware
- **Do you know of any other companies that are making PIM other than UPMEM?** Sk hinex and samsung, samsung is not generally purpose however.
- **Have you taken any functional programming courses?** They might help with this paper due to the idea of arrays and maps
- **Do we know what a histogram calculation is?** Counting the number of occurrences in a data set, if there are repeating entries it will tell you how many times those entries have repeated
- Strong scaling vs weak scaling: when you increase the number of cores or nodes that is strong scaling the program stays fixed, you end up dividing the problem over a number of ever increasing cores. weak scaling is meant to change the program, the nodes always have a fixed amount of work to do, the problem is growing as you scale up since each node always has the same amount of work to do.
- SimplePIM, is it compatible with the current versions of the Samsung PIM, not currently you would probably have to work to implement them
- Compare the performance to hand optimized code seems like a bit of a weakness but I’m not sure what else they would do. They point out it’s from a library, from nanda’s experience Yeah, they have to compare it to that code since they have nothing else to compare it to. **What if they rigged the hand optimized code?** hand optimized is quite a subjective term and is rather arbitrary.
- Lots of compiler papers from the 80’s was like this hand optimized code section since compilers were a lot worse back then
- **You are compensating for the fact that you have a strong processor(code optimizations slide), did they put the correct type of processor in the memory? Was this the correct compromise? Did they have enough space on the memory physically?** We could implement caches into these to get rid of some of the issues. Not sure if they picked the best CPU
- **Why is this on the DRAM bus, would PCI/mem better for this type of work?** That is a fair point, we can consider expanding the hardware to have interconnection and native floating point.
- If you talk to a computer architect, they would say this is programming near memory not in
- X86 architecture, DPU RISC-V so there is a 5-10% overhead just to ship the data between the two
- **Are they adjusting the runtime manually?** We think they adjust them via the compiler to use large DMA transfer sizes when all the accessed data is going to be used. Otherwise manually
- **Do they talk about the hardware in this paper?** There might be UPMEM in the paper,
- **What are the internals of the UPMEM engine?** They don't publish it
- I think there might be a point where GPU and PIM might merge, dont you eventually just approximate PIM? GPU and memory get so close to where we just use one to do the other. Would we ever want to do this or is there some law of physics where it makes it impossible?
- The new tesla supercomputer (dojo) might be interesting to look at

# References
1. [A Modern Primer on Processing-In-Memory](https://arxiv.org/pdf/2012.03112)
2. [Benchmarking a New Paradigm: An Experimental Analysis of a Real Processing-in-Memory Architecture](https://arxiv.org/abs/2105.03814)
3. Paper: [https://people.inf.ethz.ch/omutlu/pub/SimplePIM_pact23.pdf](https://people.inf.ethz.ch/omutlu/pub/SimplePIM_pact23.pdf)
4. Slides: [https://events.safari.ethz.ch/micro-pim-tutorial/lib/exe/fetch.php?media=realpimtutorial-micro23-simplepim-juan-slides.pdf](https://events.safari.ethz.ch/micro-pim-tutorial/lib/exe/fetch.php?media=realpimtutorial-micro23-simplepim-juan-slides.pdf)
