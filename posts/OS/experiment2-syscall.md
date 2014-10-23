---
date: 2014-09-28
layout:  post
title:  "HITOSExperiment2-系统调用"
tagline:  ""
categories:
-  OS
tags:
- syscall
---

##实验内容##

此次实验的基本内容是：在Linux 0.11上添加两个系统调用，并编写两个简单的应用程序测试它们。

####iam()####

第一个系统调用是iam()，其原型为：

int iam(const char * name);
完成的功能是将字符串参数name的内容拷贝到内核中保存下来。要求name的长度不能超过23个字符。返回值是拷贝的字符数。如果name的字符个数超过了23，则返回“-1”，并置errno为EINVAL。

在kernal/who.c中实现此系统调用。

####whoami()####

第二个系统调用是whoami()，其原型为：

int whoami(char* name, unsigned int size);
它将内核中由iam()保存的名字拷贝到name指向的用户地址空间中，同时确保不会对name越界访存（name的大小由size说明）。返回值是拷贝的字符数。如果size小于需要的空间，则返回“-1”，并置errno为EINVAL。

也是在kernal/who.c中实现。

测试程序

运行添加过新系统调用的Linux 0.11，在其环境下编写两个测试程序iam.c和whoami.c。最终的运行结果是：

$ ./iam lizhijun
$ ./whoami
lizhijun 


##原理分析##

关于*_syscalln* ， *int 0x80* 以及 *用户态和内核态之间的数据传递* 指导书上已经写的很清楚了，我只说说我们需要做的工作.

1. 在*include/unistd.h* 定义iam 和 whoami 的功能号

<pre><code>
#define __NR_ssetmask	69
#define __NR_setreuid	70
#define __NR_setregid	71
#define __NR_iam		72 //我们增加的功能
#define __NR_whoami		73
</code></pre>

这里需要注意在0.11环境下编译C程序，包含的头文件都在/usr/include目录下。该目录下的unistd.h是标准头文件。我们可以使用*sudo ./mount-hdc*挂载虚拟硬盘，具体请看 实验指导书-实验环境的搭建与使用-ubuntu与linux0.11之间的文件交换。

2. 在*include/linux/sys.h* 声明函数并加入到*sys_call_table[]*中

<pre><code>
extern int sys_setregid();
extern int sys_iam();
extern int sys_whoami();

fn_ptr sys_call_table[] = { sys_setup, sys_exit, sys_fork, sys_read,
sys_write, sys_open, sys_close, sys_waitpid, sys_creat, sys_link,
sys_unlink, sys_execve, sys_chdir, sys_time, sys_mknod, sys_chmod,
sys_chown, sys_break, sys_stat, sys_lseek, sys_getpid, sys_mount,
sys_umount, sys_setuid, sys_getuid, sys_stime, sys_ptrace, sys_alarm,
sys_fstat, sys_pause, sys_utime, sys_stty, sys_gtty, sys_access,
sys_nice, sys_ftime, sys_sync, sys_kill, sys_rename, sys_mkdir,
sys_rmdir, sys_dup, sys_pipe, sys_times, sys_prof, sys_brk, sys_setgid,
sys_getgid, sys_signal, sys_geteuid, sys_getegid, sys_acct, sys_phys,
sys_lock, sys_ioctl, sys_fcntl, sys_mpx, sys_setpgid, sys_ulimit,
sys_uname, sys_umask, sys_chroot, sys_ustat, sys_dup2, sys_getppid,
sys_getpgrp, sys_setsid, sys_sigaction, sys_sgetmask, sys_ssetmask,
sys_setreuid,sys_setregid,sys_iam,sys_whoami };
</code></pre>

3. 修改*kernel/sys_call.s*中的*nr_system_calls* 之前为72，改为74

<pre><code>
# offsets within sigaction
sa_handler = 0
sa_mask = 4
sa_flags = 8
sa_restorer = 12

nr_system_calls = 74
</code></pre>

4. 编写*kernel/who.c*,其中用两个系统调用iam()和whoami(), 这个按着要求来就行了，注意   #include __LIBRARY__  和 使用 get_fs_byte() put_fs_byte() 读取写入数据


5. 修改makefile，使得who.c 会被编入内核


6. 编写应用程序 iam.c whoami.c,这个也通过虚拟硬盘传到linux0.11

<pre><code>
#define __LIBRARY__
#include <unistd.h>
#include <errno.h>
#include <stdio.h>

_syscall1(int,iam,const char*,name)

int main(int argc,char * args[]){
	if(argc > 1){
		if(iam(args[1]) < 0){
			printf("SystemCall Exception!\n");
			return -1;
		}
	}else{
		printf("Input Exception!\n");
		return -1;
	}
	return 0;
}
</code></pre>

<pre><code>
#define __LIBRARY__
#include <unistd.h>
#include <string.h>
#include <errno.h>
#include <stdio.h>

_syscall2(int,whoami,char*,name,unsigned int,size)

int main(){
	int count;
	char name[30] = {0};
	count = whoami(name,30);
	if(count < 0){
		printf("SystemCall Exception!\n");
		return -1;
	}else{
		printf("%s\n",name);
	}
	return 0;
}
</code></pre>

7. 使用提供的测试脚本进行测试

##总结##

好好看看实验指导书，只要理解了原理，做起来还是很简单的

具体代码请戳[https://github.com/pokerG/HIT-OS-Experiment/tree/syscall](https://github.com/pokerG/HIT-OS-Experiment/tree/syscall)





