---
title: "NUMA balancing"
date: 2020-04-24T19:04:30-04:00
author: Peter
categories:
  - CFS (Completely Fair Scheduler)
tags:
  - Performance
  - Benchmarks
---

NUMA balancing impact on common benchmarks
==========================================


**NUMA balancing can lead to performance degradation on                    NUMA-based arm64 systems when tasks migrate,  
                    and their memory accesses now suffer additional latency.**
# Platform
  

|System|Information|
| :--- | :--- |
|Architecture|aarch64|
|Processor version|Kunpeng 920-6426|
|CPUs|128|
|NUMA nodes|4|
|Kernel release|5.6.0+|
|Node name|ARMv2-3|

# Test results

## PerfBenchSchedPipe
  
~~~  
perf bench -f simple sched pipe  
~~~  

|Test|Result|
| :--- | :--- |
|numa_balancing-ON|10.012 (usecs/op)|
|numa_balancing-OFF|10.509 (usecs/op)|
  

## PerfBenchSchedMessaging
  
~~~  
perf bench -f simple sched messaging -l 10000  
~~~  

|Test|Result|
| :--- | :--- |
|numa_balancing-ON|6.417 (Sec)|
|numa_balancing-OFF|6.494 (Sec)|
  

## PerfBenchMemMemset
  
~~~  
perf bench -f simple  mem memset -s 4GB -l 5 -f default  
~~~  

|Test|Result|
| :--- | :--- |
|numa_balancing-ON|17.438783330964565 (GB/sec)|
|numa_balancing-OFF|17.63163114627642 (GB/sec)|
  

## PerfBenchFutexWake
  
~~~  
perf bench -f simple futex wake -s -t 1024 -w 1  
~~~  

|Test|Result|
| :--- | :--- |
|numa_balancing-ON| 9.2742  (ms)|
|numa_balancing-OFF| 9.2178  (ms)|
  

## SysBenchCpu
  
~~~  
sysbench cpu --time=10 --threads=64 --cpu-max-prime=10000 run  
~~~  

|Test|Result|
| :--- | :--- |
|numa_balancing-ON|214960.28 (Events/sec)|
|numa_balancing-OFF|214965.55 (Events/sec)|
  

## SysBenchMemory
  
~~~  
sysbench memory --memory-access-mode=rnd --threads=64 run  
~~~  

|Test|Result|
| :--- | :--- |
|numa_balancing-ON|1645 (MB/s)|
|numa_balancing-OFF|1959 (MB/s)|
  

## SysBenchThreads
  
~~~  
sysbench threads --threads=64 run  
~~~  

|Test|Result|
| :--- | :--- |
|numa_balancing-ON|4604 (Events/sec)|
|numa_balancing-OFF|5390 (Events/sec)|
  

## SysBenchMutex
  
~~~  
sysbench mutex --mutex-num=1 --threads=512 run  
~~~  

|Test|Result|
| :--- | :--- |
|numa_balancing-ON|33.2165 (Sec)|
|numa_balancing-OFF|32.1088 (Sec)|
  
