---
date: 2014-09-27
layout:  post
title:  "HITOSExperiment1-操作系统的引导"
tagline:  ""
categories:
-  OS
tags:
- boot
---

##实验内容##

改写bootsect.s主要完成如下功能：

1. bootsect.s能在屏幕上打印一段提示信息“XXX is booting...”，其中XXX是你给自己的操作系统起的名字，例如LZJos、Sunix等（可以上论坛上秀秀谁的OS名字最帅，也可以显示一个特色logo，以表示自己操作系统的与众不同。）

改写setup.s主要完成如下功能：

1. bootsect.s能完成setup.s的载入，并跳转到setup.s开始地址执行。而setup.s向屏幕输出一行"Now we are in SETUP"。
2. setup.s能获取至少一个基本的硬件参数（如内存参数、显卡参数、硬盘参数等），将其存放在内存的特定地址，并输出到屏幕上。
3. setup.s不再加载Linux内核，保持上述信息显示在屏幕上即可。


####第一个非常简单，而且在实验指导书里写的很清楚了####

<pre><code>
    ! 首先读入光标位置
    mov	ah,#0x03		
    xor	bh,bh
    int	0x10

    ! 显示字符串“LZJos is running...”
    mov	cx,#25			! 要显示的字符串长度
    mov	bx,#0x0007		! page 0, attribute 7 (normal)
    mov	bp,#msg1
    mov	ax,#0x1301		! write string, move cursor
    int	0x10

inf_loop:
    jmp	inf_loop		! 后面都不是正经代码了，得往回跳呀
    ! msg1处放置字符串

msg1:
    .byte 13,10			! 换行+回车
    .ascii "LZJos is running..."
    .byte 13,10,13,10			! 两对换行+回车
    !设置引导扇区标记0xAA55
    .org 510
boot_flag:
    .word 0xAA55			! 必须有它，才能引导
</code></pre>

我们需要改的只有两个地方

<pre><code>
    98 mov cx,#24 !改成msg1的长度
    
    244 msg1:
	   .byte 13,10
	   .ascii "Loading system ..."
	   .byte 13,10,13,10
	   ！改成你希望显示的，但是不能太长
</code></pre>
	   
这是我的结果

<pre><code>
    mov cx,#108
    msg1:
	.byte 13,10
	.ascii "     ___ ___     __   __"
	.byte 13,10
	.ascii "|__|  |   | ___ |  | |__"
	.byte 13,10
	.ascii "|  | _|_  |     |__|  __|"
	.byte 13,10
	.ascii "pokerface is booting ..."
	.byte 13,10,13,10
</code></pre>	
输出 *HIT-OS pokerface is booting ...*

####第二个其实也非常简单，只不过实验指导书上写的有些歧义。就是在*setup.s*的开头加上一段输出的代码

<pre><code>
entry start
start:
!print some message
	mov	ax,#SETUPSEG
	mov	es,ax
	mov	ah,#0x03		! read cursor pos
	xor	bh,bh
	int	0x10
	
	mov	cx,#25
	mov	bx,#0x0007		! page 0, attribute 7 (normal)
	mov	bp,#msg_setup
	mov	ax,#0x1301		! write string, move cursor
	int	0x10

! ok, the read went well so we get current cursor position and save it for
! posterity.
msg_setup:
	.byte 13,10
	.ascii "Now we are in SETUP"
	.byte 13,10,13,10
</code></pre>

注意一点， 这里ES不是*bootsect.s*中的*#INITSEG*而是*SETUPSEG*,如果没有更改将会导致乱码

####关于第三个如何获取硬件参数，指导书上写的很详细了，这里直接上代码

<pre><code>
	mov ax,#SETUPSEG
	mov es,ax	! if not  will be unreadable codes	
!print the cursor position
	mov	ah,#0x03		! read cursor pos
	xor	bh,bh
	int	0x10
	
	mov	cx,#13
	mov	bx,#0x0007		! page 0, attribute 7 (normal)
	mov	bp,#msg_cursor
	mov	ax,#0x1301		! write string, move cursor
	int	0x10

	mov ax,#INITSEG
	mov ds,ax
	mov bp,#0

	call print_hex	

!print the size of memory
	mov	ah,#0x03		! read cursor pos
	xor	bh,bh
	int	0x10
	
	mov	cx,#14
	mov	bx,#0x0007		! page 0, attribute 7 (normal)
	mov	bp,#msg_mem
	mov	ax,#0x1301		! write string, move cursor
	int	0x10

	mov ax,#INITSEG
	mov ds,ax
	mov bp,#2

	call print_hex		

!print the hd0 data:
	mov	ah,#0x03		! read cursor pos
	xor	bh,bh
	int	0x10
	
	mov	cx,#11
	mov	bx,#0x0007		! page 0, attribute 7 (normal)
	mov	bp,#msg_hd0
	mov	ax,#0x1301		! write string, move cursor
	int	0x10

	mov ax,#INITSEG
	mov ds,ax
	mov bp,#0x80
	mov bx,#16
	mov bp,#0x80
loop_hd:
	call print_hex
	add bp,#2
	sub bx,#2
	cmp bx,#0
	ja loop_hd

	jmp print_nl

print_hex:
    mov	cx,#4 		! 4个十六进制数字
    mov	dx,(bp) 	! 将(bp)所指的值放入dx中，如果bp是指向栈顶的话
print_digit:
    rol	dx,#4		! 循环以使低4比特用上 !! 取dx的高4比特移到低4比特处。
    mov	ax,#0xe0f 	! ah = 请求的功能值，al = 半字节(4个比特)掩码。
    and	al,dl 		! 取dl的低4比特值。
    add	al,#0x30 	! 给al数字加上十六进制0x30
    cmp	al,#0x3a
    jl	outp  		!是一个不大于十的数字
    add	al,#0x07  	!是a～f，要多加7
outp: 
    int	0x10
    loop	print_digit
    ret	

print_nl:
    mov	ax,#0xe0d 	! CR
    int	0x10
    mov	al,#0xa 	! LF
    int	0x10
msg_setup:
	.byte 13,10
	.ascii "Now we are in SETUP"
	.byte 13,10,13,10

msg_cursor:
	.byte 13,10
	.ascii "Cursor Pos:"
msg_mem:
	.byte 13,10
	.ascii "Memory Size:"
msg_hd0:
	.byte 13,10
	.ascii "HD0 data:"
</code></pre>
有几点需要注意：

1. 要先把ES设为*#SETUPSEG* 否则会乱码
2. 在*call print_hex* 之前要设置bp，即读取的数据的初始偏移量
3. 因为*print_hex*一次读取2个字节，所以对于多于2个字节的数据要循环读取，如hd0

####还有关于编译的问题

根据指导书所说，我们修改tools/build.c

#####linux 0.11
<pre><code>
	if ((id=open(argv[3],O_RDONLY,0))<0)
		die("Unable to open 'system'");
//	if (read(id,buf,GCC_HEADER) != GCC_HEADER)
//		die("Unable to read header of 'system'");
//	if (((long *) buf)[5] != 0)
//		die("Non-GCC header of 'system'");
	for (i=0 ; (c=read(id,buf,sizeof buf))>0 ; i+=c )
		if (write(1,buf,c)!=c)
			die("Write call failed");
	close(id);
	fprintf(stderr,"System is %d bytes.\n",i);
	if (i > SYS_SIZE*16)
		die("System is too big");
	return(0);
</code></pre>
#####修改后
<pre><code>
	if ((id=open(argv[3],O_RDONLY,0))>=0){		
//	if (read(id,buf,GCC_HEADER) != GCC_HEADER)
//		die("Unable to read header of 'system'");
//	if (((long *) buf)[5] != 0)
//		die("Non-GCC header of 'system'");
		for (i=0 ; (c=read(id,buf,sizeof buf))>0 ; i+=c )
			if (write(1,buf,c)!=c)
				die("Write call failed");
		close(id);
		fprintf(stderr,"System is %d bytes.\n",i);
		if (i > SYS_SIZE*16)
			die("System is too big");
	}
	return(0);
</code></pre>

&emsp;&emsp;总体来说，这个实验还是很简单的，很适合新手刚上来熟悉操作系统的修改，编译，调试

整个修改代码请戳 [https://github.com/pokerG/HIT-OS-Experiment/tree/boot]("https://github.com/pokerG/HIT-OS-Experiment/tree/boot")