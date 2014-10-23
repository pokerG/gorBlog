---
date: 2014-10-23
layout:  post
title:  "HITOSExperiment3-进程运行轨迹的跟踪与统计"
tagline:  ""
categories:
-  OS
tags:
- thread
---

实验内容
====

&emsp;&emsp;进程从创建（Linux下调用fork()）到结束的整个过程就是进程的生命期，进程在其生命期中的运行轨迹实际上就表现为进程状态的多次切换，如进程创建以后会成为就绪态；当该进程被调度以后会切换到运行态；在运行的过程中如果启动了一个文件读写操作，操作系统会将该进程切换到阻塞态（等待态）从而让出CPU；当文件读写完毕以后，操作系统会在将其切换成就绪态，等待进程调度算法来调度该进程执行……

本次实验包括如下内容：

1. 基于模板“process.c”编写多进程的样本程序，实现如下功能：
所有子进程都并行运行，每个子进程的实际运行时间一般不超过30秒；
父进程向标准输出打印所有子进程的id，并在所有子进程都退出后才退出；

2. 在Linux 0.11上实现进程运行轨迹的跟踪。基本任务是在内核中维护一个日志文件/var/process.log，把从操作系统启动到系统关机过程中所有进程的运行轨迹都记录在这一log文件中。

3. 在修改过的0.11上运行样本程序，通过分析log文件，统计该程序建立的所有进程的等待时间、完成时间（周转时间）和运行时间，然后计算平均等待时间，平均完成时间和吞吐量。可以自己编写统计程序，也可以使用python脚本程序—— stat_log.py ——进行统计。

4. 修改0.11进程调度的时间片，然后再运行同样的样本程序，统计同样的时间数据，和原有的情况对比，体会不同时间片带来的差异。

1
---

这个很简单,cpuio_bound这个函数指导书上已经给你了,只需要在main中创建几个进程去调用它就行,看看fork和wait函数,这个程序就差不多了

<pre><code>
int main(int argc, char * argv[])
{
    pid_t pid1;
    pid_t pid2;
    pid_t pid3;
	pid_t pid4;

	if(0 == (pid1 = fork())){
		printf("In the frist process!\n");
		cpuio_bound(10,1,0);
	}else if(0 == (pid2 = fork())){
		printf("In the second process!\n");
		cpuio_bound(10,0,1);
	}else if(0 == (pid3 = fork())){
		printf("In the third process\n");
		cpuio_bound(10,3,7);
	}else if(0 == (pid4 = fork())){
		printf("In the forth process\n");
		cpuio_bound(10,5,5);
	}else if(pid1 < 0 && pid2 < 0 && pid3 < 0 && pid4 < 0){
		fprintf(stderr, "Fork error!\n");
	}else{
		wait(NULL);
		printf("the frist process's is :%d\n", pid1);
		printf("the second process's is :%d\n", pid2);
		printf("the third process's is :%d\n", pid3);
		printf("the forth process's is :%d\n", pid4);
	}

	
	return 0;
}
</code></pre>

2
---

&emsp;&emsp;这个任务是这次实验的核心,如何打开log文件,写log文件,指导书上写的很明白,修改*init/main.c* 和 *kernel/printk.c* 即可

&emsp;&emsp;接下来的问题就是去记录进程运行跪进,也就是进程的切换点,详细的东西参考指导书就好,  这里就说明几个关键点.

1. 进程的状态有 N(创建),J(就绪),R(运行),W(堵塞),E(退出)五种
2. 如何知道哪里发生了进程状态的变化,很简单找到所有发生进程state变化的地方

    <pre><code>
    p->state = TASK_XXX;
    //我们的工作只需在不同的地方修改下面语句第一个参数和第二个参数
    fprintk(3, "%ld\t%c\t%ld\n", p->pid, 'R', jiffies); //向log文件输出
    </code></pre>

3. 如何判断进程状态发生了何种变化? 新建在fork.c的copy_process

    <pre><code>
    fprintk(3, "%ld\t%c\t%ld\n", p->pid, 'N', jiffies); //向log文件输出
    p->state = TASK_RUNNING;	/* do this last, just in case */
    fprintk(3, "%ld\t%c\t%ld\n", p->pid, 'J', jiffies); //向log文件输出
    </code></pre>

    >就绪与运行间的状态转移是通过schedule()（它亦是调度算法所在）完成的；运行到睡眠依靠的是sleep_on()和interruptible_sleep_on()，还有进程主动睡觉的系统调用sys_pause()和sys_waitpid()；睡眠到就绪的转移依靠的是wake_up()。    
    >退出在exit.c state变为TASK_ZOMBIE的时候
	   
4. 	  

    >为了让生成的log文件更精准，以下几点请注意：

    >进程退出的最后一步是通知父进程自己的退出，目的是唤醒正在等待此事件的父进程。从时序上来说，应该是子进程先退出，父进程才醒来。
    
    >schedule()找到的next进程是接下来要运行的进程（注意，一定要分析清楚next是什么）。如果next恰好是当前正处于运行态的进程，swith_to(next)也会被调用。这种情况下相当于当前进程的状态没变。
    
    >系统无事可做的时候，进程0会不停地调用sys_pause()，以激活调度算法。此时它的状态可以是等待态，等待有其它可运行的进程；也可以叫运行态，因为它是唯一一个在CPU上运行的进程，只不过运行的效果是等待。
 
    第一点没啥说的,第二点用代码说明比较简单
    
    <pre><code>
    //这段代码在sched.c的schedule()
    if(current->state == TASK_RUNNING && current != task[next]){
    	fprintk(3, "%ld\t%c\t%ld\n", current->pid, 'J', jiffies); //向log文件输出
	}
	if(current != task[next]){
		fprintk(3, "%ld\t%c\t%ld\n", task[next]->pid, 'R', jiffies); //向log文件输出
	}
	switch_to(next);
    </code></pre>
     
    第三点
    
    <pre><code
    int sys_pause(void)
    {
    current->state = TASK_INTERRUPTIBLE;
	if(current->pid != 0){
		fprintk(3, "%ld\t%c\t%ld\n", current->pid, 'W', jiffies); //向log文件输出
	}
	schedule();
	return 0;
    }
    </code></pre>

3
---

运行python stat_log.py hdc/var/process.log

4
---

修改include/sched.h的INIT_TASK的第三个值

具体代码请戳[https://github.com/pokerG/HIT-OS-Experiment/tree/ThreadTracker](https://github.com/pokerG/HIT-OS-Experiment/tree/ThreadTracker)