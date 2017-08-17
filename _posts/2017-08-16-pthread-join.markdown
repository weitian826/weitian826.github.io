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

1. 捕获ctrl+c	 `signal(SIGINT, my_handler);`
2. 线程通过标记be_continue自动退出，pthread_join等待线程退出

代码

	#include <pthread.h>
	#include <stdio.h>
	#include <stdlib.h>
	#include "unistd.h"
	
	int be_continue = true;
	
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


