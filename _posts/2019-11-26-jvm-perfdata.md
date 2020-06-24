---
layout:    post
title:     JVM PerfData 源码分析
category:  JVM
tags:  PerfData
---

## 待补充

类：
```
  // frequency of the native high resolution timer
  PerfDataManager::create_constant(SUN_OS, "hrt.frequency",
                                   PerfData::U_Hertz, os::elapsed_frequency(),
                                   CHECK);


PerfTraceTime(){

}

CollectorCounters

// 采样的中心类
StatSampler

StatSamplerTask

// 各个计数器的单位
CounterNS

CollectorCounters
```
