# Chapter 3: Memory Management

**Parkinson's Law**: "Programs expand to fill the memory available to hold them"

**memory hierachy**: cache > main memory > storage

**memory manager**: track, allocate, deallocate memory to processes

## 3.1: No Memory Abstraction

- one process at a time
- variations:
	1. user program over OS in RAM
	2. user program below OS in ROM
	3. drivers on top in ROM, user program in middle, OS in RAM
- threads can be a solution to parallelism (limited however)
-  no abstraction? just save contents of memory to disk, or just introduce special hardware
-  static relocation
## 3.2: Memory Abstraction: Address Spaces
- exposing physical memory to processes is dangerous
### 3.2.1 The Notion of an Address Space
- just as the process concept creates an abstract CPU to run programs, the address space creates a kind of abstract memory for programs to live in
- **address space**: a set of addresses a process can use to address memory
- **dynamic relocation**: map each process' address space onto a different part of physical memory in a simple way 
- **base/limit** registers: CPU registers that store process addresses for dynamic relocation (base addr added to instruction and compared; limit addr to prevent writing outside of memory range)
### 3.2.2 Swapping
- **swapping**: bringing in each process in its entirety, running it for a while, and putting it back on the disk (deals with memory overload )
	+ **memory compaction**: moving all allocated memory downwards to fill holes (takes alot of time)
	+ a problem occurs when a process needs to grow in memory, can allocate extra memory when a process swapped in or moved (with stack and heap segments)
- **virtual memory**: allows programs to run even when they are only partially in main memory
### 3.2.3 Managing Free Memory
- **bitmaps**: memory is divided into allocation units, each allocated unit is mapped to a bit
	+ the smaller the allocation unit, the larger the bitmap
	+ must search bitmap for a run of given length k (slow)
- **free lists**: linked list of allocated/free memory segments, where a segment either contains a process (P) or is a hole (H) (start addr, length, next ptr) (sorted by address)
	+ algos: first fit, next fit, best fit, worst fit, quick fit
	+ can separate process and hole lists to speed up allocation, but additional complexity and slowdown when deallocating (with sort by size on holes)
- 
## 3.3 Virtual Memory
- overlays: splitting programs into little pieces and managed by an overlay manager (the only thing loaded into memory)
- virtual memory: each program has its own address space, which is broken up into chunks called pages, each page being a contiguous range of addresses which are then mapped onto physical memory at separate times
### 3.3.1 Paging
- **memory management unit**: maps virtual addresses onto physical memory addresses
- **virtual addresses**: address generation via the program's indexing, base registers, segment registers, other ways (forms the **virtual address space**)
	+ virtual address space consists of fixed-size units called **pages**
		* corresponding units in physical memory are called **page frames**
- **page fault**: CPU trap to the OS when referencing an unmapped page, OS picks a little-used page frame and writes content to disk, then fetches the referenced page into the freed page frame and changes the map and restarts the trap
### 3.3.2 Page Tables
- page table: table that maps the page to the corresponding page frame number or present/absent bit
- example 32 bit page table entry: [caching][referenced][modified][protection][present][frame #]
	+ modified also known as **dirty**,, must be written back to disk 
### 3.3.3 Speeding Up Paging
1. mapping must be fast
2. virtual address space is large, page table is large

- translation lookaside buffers (associative memory): small hardware device for mapping virtual addresses to physical addresses without going through the page table (usually inside the MMU)
	+ can also be software, TLB miss generates a TLB fault and goes to OS
- tlb misses: soft miss (page ref not in tlb, but in memory), hard miss (page itself not in memory, need disk access)
- page table walk: looking up mapping in page table hierarchy
	+ page faults: minor fault (remap), major fault (disk access), segmentation fault (invalid address)

### 3.3.4 Page Tables for Large Memories
- multilevel page table: avoids keeping all page tables in memory all the time
- page directory pointer table: 64 bits, 4 entries, 512 in each directory, 512 each table (4gb max)
- page map level 4: 512 entries in all tables
- inverted page tables: one entry per page frame rather than one entry per page of virtual address space (saves space, but virtual-to-physical translation is much harder, must serach whole table on every memory reference)
	+ use TLB for translation, but misses use a hash table search
	
### 3.4 Page Replacement Algorithms
1. optimal: page w/ highest label should be removed 
2. not recently used: removes a page at random from the lowest-numbered non empty class (R and M bits)
3. fifo: self-explanatory (oldest page may be useful still is the problem)
4. second-chance: fixes fifo by checking R bit and then moving to end of list if 1 then clears
5. clock: hand tracks oldest page, then does second-chance but moves hand instead of page after clearing
6. least recently used: close to optimal, uses linked list to track usage, must be updated on every memory reference, can use special hardware to implement (counter attached to page entries)
7. not frequently used: LRU but uses a software counter
8. working set: find a page not in the working set and evict it
	- demand paging
	- locality of reference
	- working set
	- thrashing
	- prepaging
9. wsclock: 
## 3.5 Design Issues for Paging Systems
### 3.5.1 Local versus Global Allocation Policies
- local: pages for certain process
	+ can cause thrashing if used and working set grows
- global: all pages in memory
- page fault frequency: manages allocation dynamically 
### 3.5.2 Load Control
- swapping out processes to disk and freeing the pages they hold to fix thrashing
### 3.5.3 Page Size
- large page sizes cause internal fragmentation, which is a wastage of a page when segments dont fill a page completely
- small pages needs that a program needs many pages and thus a large page table (and uses TLB space)
### 3.5.4 Separate Instruction and Data Spaces
- I-space (instructions)
- D-space (data)
- both address spaces pageable and are independent
### 3.5.5 Shared Pages
- two processes share an i-page, if one process terminates the i-page will be evicted and thus a page fault will occur and the entire i-page must be reloaded in
- unix fork is used to share both instructions and data, usually gives each process own page table and have them point to the same set (READ ONLY)
	+ if written to a trap occurs and a copy is made with r/w protection (copy on write)
### 3.5.6 Shared Libraries
- no static linking; linker uses a routine to bind called function at runtime
- shared libs can be either loaded when program is loaded or when functions are called for the first time
	+ paged in page by page
- combine with position-independent code to prevent relocation
### 3.5.7 Mapped Files
- memory-mapped files: process can issue a system call to map a file onto a portion of its virtual address space
### 3.5.8 Cleaning Policy
- paging daemon: sleeping process awakened periodically to inspect state of memory for eviction
### 3.5.9 Virtual Memory Interface
- allowing programmers to control memory map
- can be used to allow processes to share memory
## 3.6 Implementation Issues
### 3.6.1 OS Involvement with Paging
- four events w/ os intervention: process creation time, process execution time, page fault time, and process termination time
### 3.6.2 Page Fault Handling
1. hardware traps to kernel, program counter on stack
2. routine to save registers and other volatile information
3. os discovers page fault, attempts to discover what page is needed
4. check virtual address that called fault (valid and protection), looks for free page frame if successful or looks to replace
5. if page frame is dirty, write page to disk and context switches
6. page frame is cleaned, os looks up disk address where needed page is, schedule disk op to bring in
7. disk interrupt, page tables updated, frame marked normal
8. faulting instruction gets backed up, pc is reset
9. faulting process is scheduled, os returns to calling routine
10. routine reloads registers and other state information and continues execution
### 3.6.3 Instruction Backup
- attempting to figure out which instruction called the page fault and thus which one to return to
### 3.6.4 Locking Pages in Memory
- locking/pinning pages engaged in I/O in memory so they dont get removed
### 3.6.5 Backing Store
- how pages get put on disk?
- swap partition with block numbers instead of a normal file system (usually on unix)
- space allocated with process image (either swapped in or out)
- windows uses large, preallocated files in the normal file system to deal with the lack of a swap partition
### 3.6.6 Separation of Policy and Mechanism
- splitting policy from mechanism
1. low level mmu handler
2. page fault handler part of kernel
3. external pager running in user space
## 3.7 Segmentation
- providing the machine with many completely independent address spaces (segments) that have changing lengths during execution
- simplifies handling of data structures that are growing or shrinking alongside other advantages
### 3.7.1 Implementation of Pure Segmentation
- segments are capable of fragmentation (checkboarding/external fragmentation)
### 3.7.2 Segmentation with Paging: MULTICS
- paging segments for memory efficiency
- allows for uniform page size, not having to keep whole segment in memory, ease of programming, modularity, protection, and sharing
