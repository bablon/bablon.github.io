---
layout: post
title:  "Linux Sparse内存区管理"
date:   2021-03-02 23:50:00
categories: linux
---

本文的Sparse以内核三个配置`CONFIG_SPARSEMEM`, `CONFIG_SPARSEMEM_VMEMMAP`和`CONFIG_SPARSEMEM_EXTREME`为基础。

Sparse将物理内存分区管理，用结构体struct mem_section表示。每个区有一个固定的大小，比如在x86_64平台上，这个大小是128M。全局变量mem_section用于管理所有的区，它是页的数组，每个页存储固定数目的内存区结构体。
CONFIG_SPARSEMEM_EXTREME配置开启时，它的成员是动态分配的，一次分配一个页面。

    include/linux/mmzone.h
    
    struct mem_section {
            unsigned long section_mem_map;
    
            struct mem_section_usage *usage;
    };

    struct mem_section_usage {
    #ifdef CONFIG_SPARSEMEM_VMEMMAP
            DECLARE_BITMAP(subsection_map, SUBSECTIONS_PER_SECTION);
    #endif
            /* See declaration of similar field in struct zone */
            unsigned long pageblock_flags[0];
    };

mem_section结构体里的section_mem_map表示对应的struct page结构体数组的虚拟地址的基地址。 CONFIG_SPARSEMEM_VMEMMAP开启时，它始终等于vmemmap宏。此外这个成员的低几位用于额外存储一些标志信息。usage表示区内内存的使用情况。一个区分成多个大小相同的子区，这个大小是2M。 subsection_map用来标记子区的使用情况，pageblock_flags用来存储子区的迁移标志, 每个区占NR_PAGEBLOCK_BITS。

    include/linux/pageblock-flags.h

    #define PB_migratetype_bits 3
    /* Bit indices that affect a whole block of pages */
    enum pageblock_bits {
            PB_migrate,
            PB_migrate_end = PB_migrate + PB_migratetype_bits - 1,
            PB_migrate_skip,
            NR_PAGEBLOCK_BITS
    };

Sparse初始化过程函数调用图

    setup_arch
     x86_init.paging.pagetable_init
      sparse_init
       memblocks_present
       sparse_init_nid
        __populate_section_memmap /* 为struct page数组虚拟地址建立页表分配物理内存 */
        sparse_init_one_section   /* 初始化section_mem_map */
      zone_sizes_init
       free_area_init
        free_area_init_node
         free_area_init_core
          memmap_init_zone
           memmap_init
            memmap_init_zone
             set_pageblock_migratetype  /* 为子区设置迁移类型 */
    mm_init
     mem_init
       memblock_free_all
        free_low_memory_core_early
         __free_memory_core
          __free_pages_memory
           memblock_free_pages
            __free_pages_core
             __free_pages_ok
              free_one_page
               __free_one_page