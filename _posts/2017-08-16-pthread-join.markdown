---
layout:     post
title:      "pthread线程安全优雅的退出"
subtitle:   ""
date:       2017-08-16 18:23:44
author:     "wlq"
header-img: "img/post-bg-hello-blog.jpg"
tags:
    - linux
    - c++
    - pthread
---
  
> linux下pthread最近在用，总结下线程安全退出的方法

最重要的2个内容，

1.捕获ctrl+c

C++ 信号处理库提供了 signal 函数，用来捕获突发事件。以下是 signal() 函数的语法：
	
	
	void (*signal (int sig, void (*func)(int)))(int); 

可以使用函数 raise() 生成信号，该函数带有一个整数信号编号作为参数，语法如下：

	int raise (signal sig);
	
信号包括：
	
	SIGABRT	程序的异常终止，如调用 abort。
	SIGFPE	错误的算术运算，比如除以零或导致溢出的操作。
	SIGILL	检测非法指令。
	SIGINT	接收到交互注意信号。
	SIGSEGV	非法访问内存。
	SIGTERM	发送到程序的终止请求。

2.pthread_join


线程通过标记be_continue自动退出，pthread_join等待线程退出

最终代码

	#include <pthread.h>
	#include <stdio.h>
	#include <stdlib.h>
	#include "unistd.h"
	#include <csignal>
	
	bool be_continue = true;
	
	void *BusyWork(void *t)
	{
	   printf("Thread starting...\n");
	   while (be_continue)
	   {
	      //do something
	      sleep(1);
	   }
	   return NULL;
	}
	
	static void my_handler(int sig)
	{   
	   be_continue = false;  
	}  
	
	int main (int argc, char *argv[])
	{
	   pthread_t threadid;
	   pthread_attr_t attr;
	   int rc;
	   void *status;
	   
	   signal(SIGINT, my_handler);
	   /* Initialize and set thread detached attribute */
	   pthread_attr_init(&attr);
	   pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_JOINABLE);
	
	   rc = pthread_create(&threadid, &attr, BusyWork, NULL); 
	   if (rc) {
	      printf("ERROR; return code from pthread_create() is %d\n", rc);
	      return 1;
	   }
	   
	   while (be_continue)
	   {
	      //do something
	      sleep(1);
	   }
	
	   /* Free attribute and wait for the other threads */
	   pthread_attr_destroy(&attr);
	   rc = pthread_join(threadid, &status);
	   if (rc) {
	      printf("ERROR; return code from pthread_join() is %d\n", rc);
	      return 1;
	   }
	
	   printf("Main: completed join with thread having a status of %ld\n", (long)status);
	 
	   printf("Main: program completed. Exiting.\n");
	   return 0;   
	}

另，顺便记录下linux的通用返回值，代码中的rc值：

	EPERM Operation not permitted 1
	ENOENT No such file or directory 2
	ESRCH No such process 3
	EINTR Interrupted function 4
	EIO I/O error 5
	ENXIO No such device or address 6
	E2BIG Argument list too long 7
	ENOEXEC Exec format error 8
	EBADF Bad file number 9
	ECHILD No spawned processes 10
	EAGAIN No more processes or not enough memory or maximum nesting level reached 11
	ENOMEM Not enough memory 12
	EACCES Permission denied 13
	EFAULT Bad address 14
	EBUSY Device or resource busy 16
	EEXIST File exists 17
	EXDEV Cross-device link 18
	ENODEV No such device 19
	ENOTDIR Not a directory 20
	EISDIR Is a directory 21
	EINVAL Invalid argument 22
	ENFILE Too many files open in system 23
	EMFILE Too many open files 24
	ENOTTY Inappropriate I/O control operation 25
	EFBIG File too large 27
	ENOSPC No space left on device 28
	ESPIPE Invalid seek 29
	EROFS Read-only file system 30
	EMLINK Too many links 31
	EPIPE Broken pipe 32
	EDOM Math argument 33
	ERANGE Result too large 34
	EDEADLK Resource deadlock would occur 36
	EDEADLOCK Same as EDEADLK for compatibility with older Microsoft C versions 36
	ENAMETOOLONG Filename too long 38
	ENOLCK No locks available 39
	ENOSYS Function not supported 40
	ENOTEMPTY Directory not empty 41
	EILSEQ Illegal byte sequence 42
	STRUNCATE String was truncated 80
