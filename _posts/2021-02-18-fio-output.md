---
layout: post
title:  "fio默认输出结果中的难点解析"
date:   2020-02-18 16:00:00
categories: fio
---

fio的默认输出结果中有很多参数，有一些不太好理解，本文主要记录不好理解的部分。

## 数理统计

     slat (usec): min=2, max=260, avg= 4.26, stdev= 2.91
     clat (usec): min=10, max=4954, avg=72.49, stdev=183.47
      lat (usec): min=22, max=4957, avg=76.84, stdev=183.51
    bw (  KiB/s): min=169355, max=220256, per=97.20%, avg=201025.67, stdev=27636.92, samples=3
    iops        : min=42338, max=55064, avg=50256.00, stdev=6909.55, samples=3

fio 使用了min, max, avg, stdev等参数来描述读写过程中各项指标的整体状态。min, max, avg比较好理解，分别表示最小值，最大值，平均值。stdev是标准差，用来衡量数据相对于平均值的波动。它的值越小波动就越小。`bw`还有一个`per`参数，它表示采样平均值与聚合平均值的比率。平均值是在采样的过程中动态计算的，聚合平均值是根据IO总大小和时间计算的。

## clat p-th percentile

    clat percentiles (usec):
    |  1.00th=[   31],  5.00th=[   34], 10.00th=[   36], 20.00th=[   37],
    | 30.00th=[   38], 40.00th=[   39], 50.00th=[   40], 60.00th=[   43],
    | 70.00th=[   46], 80.00th=[   50], 90.00th=[   89], 95.00th=[  143],
    | 99.00th=[  857], 99.50th=[ 1352], 99.90th=[ 2671], 99.95th=[ 3195],
    | 99.99th=[ 4293] 

p-th percentiles 分布图，表示占比为p的IO的延迟不超过它对应的值。p的值用户可以指定最多20个。下面是英文解释[p-th percentile](https://www.itl.nist.gov/div898/software/dataplot/refman2/auxillar/percenti.htm)

    The p-th percentile of a data set is defined as that value where p percent of the data is below that value and (1-p) percent of the data is above that value. For example, the 50th percentile is the median. 

### 计算方法

fio设计了一个哈希表来存储某一段时间范围的延迟的IO数量。每一个区间的时间差是不相同的，前面的精度高于后面的。每个区间都包含64个buckets。 在采样时，将时间转化成索引找到对应的bucket，累加计数。最后统计时扫描表将索引转化成它代表的时间的平均值。

stat.h有一段注释Aggregate latency samples for reporting percentile(s) 详细描述了设计原理。

    Group   MSB     #discarded      range of                #buckets
                    error_bits      value
    ----------------------------------------------------------------
    0*      0~5     0               [0,63]                  64
    1*      6       0               [64,127]                64
    2       7       1               [128,255]               64
    3       8       2               [256,511]               64
    4       9       3               [512,1023]              64
    ...     ...     ...             [...,...]               ...
    28      33      27              [8589934592,+inf]**     64
 
 
## 延迟分布图

    lat (usec)   : 20=0.02%, 50=79.99%, 100=12.18%, 250=4.23%, 500=1.85%
    lat (usec)   : 750=0.60%, 1000=0.33%
    lat (msec)   : 2=0.61%, 4=0.18%, 10=0.02%

延迟分布图显示延迟区间对应的IO样本百分比。`=`左边的值对应下一区间的开始不包含在当前区间内。 usec和nsec表示的lat各有10个区间表示对应时间单位范围1－1000的延迟，msec表示的则有12个，多出[1000,2000)和>=2000的两个区间。

| Index | lat       | Range            | Unit    |
|-------|-----------|------------------|---------|
| 1     | 2         | [0, 2)           | n/u/m   |
| 2     | 4         | [2, 4)           | n/u/m   |
| 3     | 10        | [4, 10)          | n/u/m   |
| 4     | 20        | [10, 20)         | n/u/m   |
| 5     | 50        | [20, 50)         | n/u/m   |
| 6     | 100       | [50, 100)        | n/u/m   |
| 7     | 500       | [100, 500)       | n/u/m   |
| 8     | 750       | [500, 750)       | n/u/m   |
| 9     | 1000      | [750, 1000)      | n/u/m   |
| 10    | 2000      | [1000, 2000)     | m       |
| 11    | >=2000    | [2000, )         | m       |

## IO队列分布

    IO depths    : 1=0.1%, 2=0.1%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
       submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
       complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%

IO depths统计io的发起，submit和complete根据函数调用返回的结果进行统计。

`man fio` 中的解释

    IO depths
      The distribution of I/O depths over the job lifetime. The numbers are divided into powers of 2 and each entry
      covers depths from that value up to those that are lower than the next entry -- e.g., 16= covers depths  from
      16  to 31. Note that the range covered by a depth distribution entry can be different to the range covered by
      the equivalent submit/complete distribution entry.
    
    IO submit
      How many pieces of I/O were submitting in a single submit call. Each entry denotes that amount and below, un‐
      til  the  previous  entry  --  e.g., 16=100% means that we submitted anywhere between 9 to 16 I/Os per submit
      call. Note that the range covered by a submit distribution entry can be different to the range covered by the
      equivalent depth distribution entry.
    
    IO complete
      Like the above submit number, but for completions instead.
