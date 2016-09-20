---

layout: post
title: "epoll学习笔记(更新中)"
category: "OS"
draft: false
tags: epoll

--- 

## 目录

1. epoll系统调用
   * epoll_create
   * epoll_ctl
   * epoll_wait
   * 最小示例
2. epoll实现原理
   * 数据结构
   * 核心结构体
   * 关键流程
3. epoll动态追踪
   * 问题描述
   * 解决方案
-----------------------    
## epoll系统调用

### epoll_create

       #include <sys/epoll.h>
       
       int epoll_create(int size);

功能描述：创建一个epoll实例

关键参数说明：size无意义，但必须大于0

返回值：所创实例对应的文件描述符，即epoll_ctl()和epoll_wait()的epfd传参

参考链接：[epoll_create](http://man7.org/linux/man-pages/man2/epoll_create.2.html)

### epoll_ctl

       #include <sys/epoll.h>
       
       int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
       
功能描述：添加、删除、修改某一监听事件

关键参数说明：

* op决定了这次epoll_ctl操作是添加还是删除还是修改
* fd为所要监听事件的文件描述符
* event结构体里存放用户定义的与该监听事件相关的信息，可称为用户自定义事件信息，存放与否且存放何种信息完全是用户行为，结构体定义如下

       typedef union epoll_data {
           void        *ptr;
           int          fd;
           uint32_t     u32;
           uint64_t     u64;
       } epoll_data_t;

       struct epoll_event {
           uint32_t     events;      /* Epoll events */
           epoll_data_t data;        /* User data variable */
       };

返回值：操作成功返回0，操作失败返回-1

参考链接：[epoll_ctl](http://man7.org/linux/man-pages/man2/epoll_ctl.2.html)

### epoll_wait

       #include <sys/epoll.h>

       int epoll_wait(int epfd, struct epoll_event *events,
                      int maxevents, int timeout);
                      
功能描述：等待直到有监听事件到达或等待超时或调用被中断，至多返回maxevents个事件

关键参数说明：

* event指向到达事件对应的用户自定义事件信息，结构体定义如epoll_ctl小节
* timeout设定超时值，为0则不管监听事件到达与否都立即返回，为-1则无限等待直到有监听事件到达

返回值：返回到达的监听事件数

参考链接：[epoll_wait](http://man7.org/linux/man-pages/man2/epoll_wait.2.html)      

### 最小示例

------------------------ 
## epoll实现原理
 
### 数据结构
#### 双向循环列表
#### 红黑树
### 核心结构体
#### eventpoll
#### epitem
### 关键流程
#### epoll实例管理监听事件
#### epoll实例等待、被唤醒
#### epoll实例返回到达事件

------------------------
## epoll动态追踪

### 追踪目标
#### 1. 探测epoll系统调用工作流程
#### 2. 探测epoll_wait系统调用，获取到达事件信息
### 解决方案