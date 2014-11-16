---
date: 2014-11-16
layout:  post
title:  "HITOSExperiment5-地址映射与共享"
tagline:  ""
categories:
-  OS
tags:
- shm
---

实验内容
=====
1. 用Bochs调试工具跟踪Linux 0.11的地址翻译（地址映射）过程，了解IA-32和Linux 0.11的内存管理机制；
2. 在Ubuntu上编写多进程的生产者—消费者程序，用共享内存做缓冲区；
3. 在信号量实验的基础上，为Linux 0.11增加共享内存功能，并将生产者—消费者程序移植到Linux 0.11。

这次实验用到了实验4的代码,然后再加上两个系统调用

shmget的参数key和size就够了,不需要shmflag

shmat的参数shmid就可以

因为shm.c是在mm里写的,所以要改mm/makefile,改法和kernel里的一样

shmget的关键是get_free_page

shmat的关键是put_page,用从shmget中得到的物理地址和线性地址建立映射,具体请看虚拟内存的相关内容,

正常来说我们还需要写shmdt来释放共享内存 ,进程结束时，系统会自动检测进程中映射的共享内存并释放该映射，并将mam_map[]减一。 因为实验没要求,所以这里我们要改下mm/memory.c,把释放内存时的panic注释掉.

剩下的就没什么了

具体代码请戳[https://github.com/pokerG/HIT-OS-Experiment/tree/shm](https://github.com/pokerG/HIT-OS-Experiment/tree/shm)