---
layout:    post
title:     线程和进程 On Linux
category:  Linux Kernel
tags: Linux
---
1. 线程和进程的区别
	线程是轻量级进程，与进程共享内存、fs 等其他一切资源，底层是 clone
	进程是 fork 产生的，采用 cow 技术，在写的时候，由于产生 page fault 异常而重新加载物理内存；

2. 进程上下文切换
	首先是相关的数据需要保存现场，如寄存器、高速缓存等；
	其次是会影响高速缓存的命中率，即 cache miss 会增加

3. IMBench 可以测试上下文切换的性能影响

4. 时间片是否到了，是用时钟 check 来进行的

5. 设置进程的 CPU 亲和性
	pthread_setaffinity...
	task_set -a -p 01 19999
		-a  表示所有线程
		-p 指定进程
		01 表示二进制的掩码，如 01 表示的是 1，02 表示是 2 个 CPU，03 表示的是1 和 3 两个线程

6. MMU原理

7. 硬实时系统
	cyclelicTest 