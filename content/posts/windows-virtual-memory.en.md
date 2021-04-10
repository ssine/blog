---
title: 'Windows Virtual Memory Intro'
date: '2018-06-13'
dateformat: 'Y-m-d H:i'
tags:
    - windows
    - 课程
categories:
    - 课程
---

__Note: This passage is mainly translated by Google translator, please turn to the Chinese version for more accurate expression if possible.__

Virtual memory provides a logical view of memory, it does not correspond to the layout of physical memory. During operation, the memory manager maps virtual addresses to physical addresses with the help of hardware support.

<!-- more -->

Each process has its own virtual address space, and it will feel that it monopolizes this large address space. In a 32-bit x86 system, the total virtual address space is 4GB. Because the 32-bit pointer can represent the value between 0X00000000 and 0XFFFFFFFF. But by default, Windows will give 2GB of address space to the process as its private address space, called user mode space; while the other half (the higher half of the address space, from 0X80000000 to 0XFFFFFFFF) is used as its own The protected kernel usage is called kernel mode space.

Virtual memory has three benefits:

* Programs can use a series of adjacent virtual addresses to access non-adjacent memory blocks in physical memory.
* The program can access memory buffers that are larger than the available physical memory. When the supply of physical memory becomes small, the memory manager saves physical memory pages (usually 4 KB in size) to disk files. Pages are swapped in and out between physical memory and disk as needed.
* The virtual addresses used by different processes are isolated from each other. Code in one process cannot access the physical memory being used by another process.

There are three ways for applications to use memory:

1. The virtual memory allocation method based on pages is suitable for large objects or structure arrays;
2. Memory mapping file method, suitable for large data stream files and data sharing between multiple processes;
3. The memory heap method is suitable for a large number of small memory applications.

There are a series of Win32 APIs that can be used for memory-related allocation, use, and query operations. For details, you can query Microsoft's documentation. The following mainly introduces the first type of method.

Pages in the virtual address space of a Windows process have three states: free, reserved, and committed. The process can also lock and unlock a page, and the locked memory will have a high probability of being retained in the physical memory (not guaranteed). The relationship between the states is as follows:

<!-- <center> -->

![Win32 memory](https://sine-img-bed.oss-cn-beijing.aliyuncs.com/autoup/OS-virtual-memory-states.svg)

<!-- </center> -->

Windows can set the type of protection for memory. Several examples used in the experiment are listed below:

|Macro Definition|Type|
|---|---|
|PAGE_READONLY|read only|
|PAGE_READWRITE|Read and Write|
|PAGE_EXECUTE|Execute|
|PAGE_EXECUTE_READ|write+execute|
|PAGE_EXECUTE_READWRITE|Read and Write+Execute|

Performing operations outside the authority on a page will cause an error.
