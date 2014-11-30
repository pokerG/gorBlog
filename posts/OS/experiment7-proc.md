---
date: 2014-11-30
layout:  post
title:  "HITOSExperiment7-proc文件系统的实现"
tagline:  ""
categories:
-  OS
tags:
- proc
---

实验内容
======

在Linux 0.11上实现procfs（proc文件系统）内的psinfo结点。当读取此结点的内容时，可得到系统当前所有进程的状态信息。例如，用cat命令显示/proc/psinfo的内容，可得到：

<pre><code>
\# cat /proc/psinfo
pid    state	father	counter	start_time
0	1	-1	0	0
1	1	0	28	1
4	1	1	1	73
3	1	1	27	63
6	0	4	12	817
\# cat /proc/hdinfo
total_blocks:62000; 
free_blocks:39037; 
used_blocks:22963; 
total_inodes:20666; 
... 
\#cat /prco/inodeinfo
inr:1;zone[0]:659 
inr:2;zone[0]:xxxx 
... 
</code></pre>
procfs及其结点要在内核启动时自动创建。相关功能实现在fs/proc.c文件内。


跟着实验指导书一步步来,第一个psinfo应该没啥问题,重点是后面两个


###hdinfo

首先要获得super_block,使用get_super(),这里的参数可以用proc_read加一个参数idev 使用inode->i_dev;或者也可以直接使用ROOT_DEV;

super_block的结构在include/linux/fs.h里定义,这里就不细说了,Linux内核完全剖析里也有.

其中有个s_zamp[i] 是buffer_head结构,也在fs.h里定义, total_blocks和total_inodes都好说直接在super_block里就有,而free_blocks等必须通过位运算累加获得.

这里还要注意判断操作过的位数是否和total_blocks相等,相等则跳出.另外位图中的第一位是不算的,这里在剖析书里有介绍,*强烈建议做这个之前好好看看书*.

###inodeinfo

这个老师的要求是打印出一部分就行了,所以不难,但是如果超过二三十就可能报错,不过不影响结论,你打印10来个就行了,要是想打印更多的话就得想办法,这里就不细讲了,自己看代码吧.

具体代码请戳[https://github.com/pokerG/HIT-OS-Experiment/tree/proc](https://github.com/pokerG/HIT-OS-Experiment/tree/proc)
