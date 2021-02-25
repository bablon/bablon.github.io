---
layout: post
title:  "FIO CPU时钟源"
date:   2020-02-25 23:07:00
categories: fio
---

常用的获取时间的函数是gettimeofday和clock_gettime，这两个函数是系统调用。这里没有列出time系统调用，因为它返回的时间单位是秒, 不适用精度要求较高的场合。FIO获取时间的函数是fio_gettime, 它根据fio_clock_source的值来从不同的源来获取时间。FIO启动时会选取最优源。为了减少系统调用提升性能，FIO对于CPU有高性能高精度时钟支持的平台，会在启动时较准CPU时钟，如果稳定性足够，就会选择CPU作为时钟源。

get_cpu_clock获取CPU时间ticks，x86平台上是利用rdtsc汇编指读取，然后将当前数值与起始基准时间的差值转化成单调纳秒时间.

    t = get_cpu_clock();
    t -= cycles_start;

    multiples = t >> max_cycles_shift;
    nsecs = multiples * nsecs_for_max_cycles;
    nsecs += ((t & max_cycles_mask) * clock_mult) >> clock_shift;

有五个参数是在校准CPU时钟的时候得到的, 这样的方法避免了使用浮点数带来的性能损耗。

* max_cycles_shift和max_cycles_mask是不超过一小时的ticks的最高偏移和掩码。
* clock_mult和clock_shift用于将ticks转化成纳秒。
* nsecs_for_max_cycles由max_cycles_shift, clock_mult, clock_shift计算生成。

## clock_mult和clock_shift

    max_ticks = MAX_CLOCK_SEC * cycles_per_msec * 1000ULL;
    max_mult = ULLONG_MAX / max_ticks;

MAX_CLOCK_SEC定义为1小时的秒数，max_ticks是对应的CPU时钟ticks。 mux_mult是不溢出时max_ticks的最高倍数，可以理解为1个tick所花的纳秒时间最高可以放大多少倍。
 
    tmp = max_mult * cycles_per_msec / 1000000;
    while (tmp > 1) {
          tmp >>= 1;
          sft++;
    }

cycles_per_msec / 1000000表示1纳秒有几个tick, tmp表示1纳秒的放大。

    clock_shift = sft;
    clock_mult = (1ULL << sft) * 1000000 / cycles_per_msec;

clock_shift是不超过tmp的最高偏移。(1 << shift)可理解为调整后的1纳秒放大, clock_mult则是调整后的1个tick所用纳秒放大。其值不超过max_mult。
 
## max_cycles_shift和nsecs_for_max_cycles

    max_cycles_shift = max_cycles_mask = 0;
    tmp = MAX_CLOCK_SEC * 1000ULL * cycles_per_msec;
    while (tmp > 1) {
          tmp >>= 1;
          max_cycles_shift++;
    }
 
    nsecs_for_max_cycles = ((1ULL << max_cycles_shift) * clock_mult) >> clock_shift;