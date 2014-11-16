---
date: 2014-11-09
layout:  post
title:  "HITOSExperiment4-信号量的实现和应用"
tagline:  ""
categories:
-  OS
tags:
- sem
---

实验内容
=======

1. 在Ubuntu下编写程序，用信号量解决生产者——消费者问题；
2. 在0.11中实现信号量，用生产者—消费者程序检验之。
          
&emsp;&emsp;这个实验其实没啥好说的,说简单一点不简单,说难其实也就那么回事,(内核代码一遍过,也是让我惊呆了)

&emsp;&emsp;就说说几个需要注意的地方吧,剩下的看指导书和代码就行了

1. 这个实验用到了实验二的东西,所以做之前建议复习下实验二

2. 写pc.c时,如果对缓冲区使用到in,out指针(自定义)的话,要注意out指针是由所有的消费者共享的,因为子进程继承父进程的全局变量是通过复制,所以子进程修改后的只对自己这一支有效,所以我们要想办法让所有进程共享相同的out指针,怎么做呢?我是通过把它写到缓冲区的第10个单元,每次消费者操作时先把它读出来,然后结束时把改变的out再写回去.

3. 内核代码最关键的是sem_t的结构怎么实现,下面是我的实现

    <pre><code>
    /**
    * in unistd.h  
    * sem struct 
    */
    
    #define QUEUE_LIMIT 20
    #define SEM_NAME_LIMIT 20
    typedef struct sem_queue
    {    
	    int front,rear;
	    struct task_struct * task[QUEUE_LIMIT]; /*maybe linkedList better*/
	
    }sem_queue;

    typedef struct sem_t
    {
	    char name[SEM_NAME_LIMIT];
	    int value;
	    sem_queue wait_task; 
    }sem_t;
    </code></pre>
    
4. 确定结构后,就写那四个系统调用就行了,老李上课这部分也讲的挺详细的,就不赘述了,sem_post和sem_wait可以自己通过队列实现,也可以通过kernel本身的sleep_on和wake_up 不过这里要注意不能用if,具体原理看老李ppt.另外注意把一些错误处理什么的加上就好了


具体代码请戳[https://github.com/pokerG/HIT-OS-Experiment/tree/Sem](https://github.com/pokerG/HIT-OS-Experiment/tree/Sem)