+++
title = "Exploring Kernel Samepage Merging (KSM) with Modern Workloads: PyTorch, Redis, Memcached, and Stress-ng"
[[extra.authors]] 
name = "David Luo" 
+++


# Introduction
Kernel Samepage Merging (KSM) is a Linux feature that helps optimize memory usage by identifying and merging identical memory pages across processes. This can be really useful in environments and scenarios where memory efficiency is crucial. While KSM has been widely discussed in theory, its practical effectiveness often depends on workload characteristics and system configurations.

Testing KSM with more modern workloads such as PyTorch Redis, Memcached, and the stress-ng tool seemed straightforward enough, but various challenges got in the way of KSM detecting and merging the identical pages in each of the workloads. With each workload, KSM either failed to detect or successfully merge identical pages. This led to me going through the troubleshooting process as thorough as I could. Unfortunately, I wasn't able to successfully troubleshoot these workloads completely, but I may have figured out a few reasons as to why KSM wasn't merging any identical pages.


# Overview
KSM scans memory for identical pages, and if found, they're merged into a single read-only page. This heavily reduces memory consumption depending on the workload. Conceptually, it sounds pretty simple, but there is a lot more to making sure the workloads are taking advantage of KSM. The modern workloads used to test KSM were PyTorch, Redis, Memcached, and stress-ng. They were ran inside a VirtualBox Ubuntu VM that didn't support KVM. With each workload, the **madvise(MADV_MERGEABLE)** function  was used to designate what regions of memory were mergeable for KSM.


# PyTorch
The PyTorch workload would allocate large tensors, essentially an array of numerical values, and mark them as mergeable with the madvise function. This was done through modifying the source code to ensure that the tensors being allocated would be seen and scanned by KSM, but unfortunately, several issues appeared. At first, there were dependency issues from Python's externally managed environment restrictions that required setting up another virtual environment in the VM. Once getting past that obstacle, running the workload would fail, indicating that madvise calls were failing, returning -1. Getting past this obstacle wasn't too difficult either, as the workload ended up running. Checking the KSM statistics after the initial attempt showed that KSM was not in fact scanning any pages, let alone merging any pages.

Thus began the first troubleshooting session. It included checking each KSM statistic and configurations, ensuring that everything was set up correctly. Ranging from increasing **pages_to_scan**, to decreasing the **sleep_millisec**, to doublechecking the source code, making sure the madvise calls were being called correctly, none of these attempts made any dent in this process. 
 
 
# Redis
With Redis, it was a bit simpler as there weren't as many challenges. The original plan for this workload was to write a script that would apply the madvise function to the processes the benchmark was using while it was running. This ended up being ineffective because KSM was likely already looking at the memory used by the processes and ignored the madvise application after the benchmark began. The next step was to then modify the Redis benchmark to include the madvise function, directly applying it to its memory allocation functions, **zmalloc** and **zcalloc**. This was where the first complication happened. These functions use jemalloc, which may have affected KSM's ability to effectively merge pages. This interference shows how important it is to consider the interaction between memory allocators and KSM to ensure optimal performance.

The troubleshooting process for this started off by comparing the original source code and the modified code for the benchmark. After a few attempts at isolating the effects of jemalloc, there was no concrete evidence indicating that jemalloc was the sole reason for KSM's refusal to scan memory pages. Other potential factors were then explored, including the virtualization environment and  security mechanisms, but none of which were successfully identified as the cause.


# Stress-ng
The simplest approach in this experiment was to use stress-ng, a Linux tool designed to stress various system resources, including memory. When stressing memory specifically, the tool will continuously allocate memory and write to it. The plan for this workload was to do exactly that. The tool started 8 workers with 512 MB per worker. Letting it run should allow KSM to identify those pages, scan them, and merge any identical ones. Unfortunately, stress-ng doesn't seem to have its "workers" fill the pages with identical data.

**_full_scans: 3978_**
**_general_profit: 0_**
**_ksm_zero_pages: 0_**
**_pages_scanned: 12360116_**
**_pages_shared: 0_**
**_pages_sharing: 0_**
**_pages_skipped: 332609_**
**_pages_unshared: 0_**
**_pages_volatile: 0_**
**_stable_node_chains: 0_**
**_stable_node_dups: 0_**
**_Actual pages saved by KSM (pages_sharing + ksm_zero_pages): 0_**


As you can see in the KSM statistics above, roughly 12 million pages were scanned but none of them were merged. There were 300,000 pages skipped, which meant that they have previously not been merged. The missing 12 million pages are likely to be empty, which is why they were scanned, but nothing was done. They also weren't included in the **pages_skipped** statistic because they likely weren't considered as candidates for merging.

# Memcached
Memcached was a more controlled environment for testing KSM compared to the other workloads. The plan for this workload was to take a large amount of identical key-value store pairs and send them to a Memcached server after marking them as mergeable with the madvise function. This was automated using a script that would apply the madvise function to large amounts of identical key-value pairs, then sending them off to a Memcached server. Unfortunately, KSM didn't seem to like this that much either, not scanning or merging any of the pages, so I tried something else. I took YCSB (Yahoo! Cloud Serving Benchmark) and tried to modify it to include madvise. This also didn't work as there were various issues with the dependency modules. After some testing that included modifying the source code for the benchmark and exploring the built-in security mechanisms for Ubuntu, I realized one of the modules wasn't even included when I downloaded YCSB to the VM, and when I went to the Github to look for it, it wasn't there. That was when I realized that the code for YCSB and Memcached was likely to be outdated as the most recent commit for it was 6 years ago.

# Troubleshooting
The main problem for each workload, except for stress-ng was that KSM wasn't scanning or merging pages at all. The troubleshooting steps taken not specific to any workload was to check KSM's configuration. To ensure KSM was even running in the first place, the command _cat /sys/kernel/mm/ksm/run_ would be ran. If it was running, the command would return a 1 if it was running, and a 0 if it wasn't. KSM doesn't scan that many memory pages by default, about 100 pages, but you can increase the scan amount by increasing the number from 100, to however much you want it to scan.

There weren't many specific troubleshooting steps when it comes to each workload, as it was mainly check any modifications I made to the source code, and ensure I made the madvise function calls correctly. For Memcached, it seemed to just be outdated and I wasn't able to effectively update it to run. For stress-ng, there wasn't many issues because it just doesn't seem like it's a good workload for KSM. For Redis and PyTorch, it may have been a memory allocation conflict or an incorrect modification to include madvise.
 
To address these challenges, including workloads that have static memory pages, or pages that aren't constantly changing, would be a much better fit for KSM, though it may not be as applicable to real-world applications. Additionally, performing these experiments on either a host machine or a VM that includes KVM support would likely produce better results as well. 

# Conclusion
After an extensive testing and troubleshooting process, KSM did not effectively merge pages on any of the workloads tested. Something I noticed from this experiment is that KSM isn't something that can be applied to every workload, at least not easily. Workloads that frequently modify memory aren't necessarily the "right" workloads for KSM either as there will be little to no benefits because the merging process is much less effective and beneficial when there aren't many identical pages, especially if they're constantly changing. The type of memory allocation would also have to be taken into account as jemalloc would interfere with KSM's merging, limiting its potential. Using VirtualBox for virtualization may have also plated a role in reducing KSM's efficiency as without the KVM support, the performance may not have been its peak performance. 

Among the four workloads, it seemed like Memcached was the best candidate to test KSM. However, even in the Memcached workload, the expected page merging was not achieved. Even if Memcached is one of the more favorable workloads, it doesn't mean it's a great one as there are likely other factors that are preventing the merging process from being successful and efficient. 


# References
YCSB/Memcached Github: https://github.com/brianfrankcooper/YCSB/tree/master/memcached
KSM documentation: https://docs.kernel.org/admin-guide/mm/ksm.html
Pytorch: https://pytorch.org/
Redis: https://redis.io/
Memcached: https://memcached.org/
Stress-ng: https://github.com/ColinIanKing/stress-ng and https://wiki.ubuntu.com/Kernel/Reference/stress-ng
