![libevent](images/libevent.png)
 
## 前言

此笔记采用obsidian编写。如遇到无法打开笔记、照片等问题，请使用obsidan。

Libevent是用于编写高速可移植非阻塞IO应用的库，其设计目标是：

- 可移植性：使用libevent编写的程序应该可以在libevent支持的所有平台上工作。即使没有好的方式进行非阻塞IO，libevent也应该支持一般的方式，让程序可以在受限的环境中运行
    
- 速度：libevent尝试使用每个平台上最高速的非阻塞IO实现，并且不引入太多的额外开销。
    
- 可扩展性：libevent被设计为程序即使需要上万个活动套接字的时候也可以良好工作。
    
- 方便：无论何时，最自然的使用libevent编写程序的方式应该是稳定的、可移植的。(_Libevent_ should compile on Linux, *BSD, Mac OS X, Solaris, Windows, and more.)
---


libevent由下列组件构成：

- evutil：用于抽象不同平台网络实现差异的通用功能。
    
- event和event_base：libevent的核心，为各种平台特定的、基于事件的非阻塞IO后端提供抽象API，让程序可以知道套接字何时已经准备好，可以读或者写，并且处理基本的超时功能，检测OS信号。
    
- bufferevent：为libevent基于事件的核心提供使用更方便的封装。除了通知程序套接字已经准备好读写之外，还让程序可以请求缓冲的读写操作，可以知道何时IO已经真正发生。（bufferevent接口有多个后端，可以采用系统能够提供的更快的非阻塞IO方式，如Windows中的IOCP。）
    
- evbuffer：在bufferevent层之下实现了缓冲功能，并且提供了方便有效的访问函数。
    
- evhttp：一个简单的HTTP客户端/服务器实现。
    
- evdns：一个简单的DNS客户端/服务器实现。
    
- evrpc：一个简单的RPC实现。

---

---
- libevent API提供了一种机制，可以在文件描述符上发生特定事件或达到超时后执行回调函数。此外，libevent还支持由于信号或常规超时而产生的回调。
    
- Libevent旨在取代事件驱动网络服务器中的事件循环。应用程序只需要调用event_dispatch()，然后动态地添加或删除事件，而不必更改事件循环。
    
- Libevent还为缓冲网络IO提供了一个复杂的框架，支持套接字、过滤器、速率限制、SSL、零拷贝文件传输和IOCP。Libevent支持几种有用的协议，包括DNS、HTTP和一个最小的RPC框架。

![](images/Pasted%20image%2020240904111332.png) 


## 相关的库
libevent公用头文件都安装在event2目录中，分为三类：

- API头文件：定义libevent公用接口。这类头文件没有特定后缀。
    
- 兼容头文件：为已废弃的函数提供兼容的头部包含定义。不应该使用这类头文件，除非是在移植使用较老版本libevent的程序时。
    
- 结构头文件：这类头文件以相对不稳定的布局定义各种结构体。这些结构体中的一些是为了提供快速访问而暴露；一些是因为历史原因而暴露。直接依赖这类头文件中的任何结构体都会破坏程序对其他版本libevent的二进制兼容性，有时候是以非常难以调试的方式出现。这类头文件具有后缀“_struct.h”。

## 如何使用libevent

### 创建event_base
 [event_base](event_base.md)
 
## 使用事件循环
 [Event loop](Event%20loop.md)

## 创建事件
[Create event](Create%20event.md)

## 相关的辅助函数和类型
[Helper types and functions](Helper%20types%20and%20functions.md)

## DNS
[DNS with libevent](DNS%20with%20libevent.md)

 