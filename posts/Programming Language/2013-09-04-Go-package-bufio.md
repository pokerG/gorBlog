---
date: 2013-08-28
layout:  post
title:  Go标准库-bufio
tagline:  ""
categories:
-  Programming Language
tags:
- Go
- Package
---

&emsp;&emsp;bufio包实现了带缓存的I/O操作。封装了一个io.Reader或io.Writer对象，返回一个具有缓存和文本读写的对象。

bufio.go
===
    
    //实现了带缓存的
    type Reader struct{
        	buf          []byte
        	rd           io.Reader
        	r, w         int
        	err          error
        	lastByte     int
        	lastRuneSize int
    }
  
    //NewReaderSize将rd封装成一个具有size大小的bufio.Reader对象
    //如果rd的类型就是bufio.Reader且size > minReadBufferSize = 16 直接返回
    //size的大小要大于 minReadBufferSize 否则 返回大小为minReadBufferSize的bufio.Reader
    func NewReaderSize(rd io.Reader, size int) *Reader

    
