libevent库为开发人员提供网络事件的处理框架，封装好事件处理的API，方便开发人员编写网络应用相关的程序，而不必从0开始写一个完整的网络事件库。

libevent使用事件驱动程序，libevent会把用户需要关注的事件保存，每个事件的对应的回调函数会在事件触发的时候调用，这样用户就能感知到事件发生，采取相应的措施。

libevent主要工作流程是提供一个event-loop的机制，保持程序一直运行，检测相应事件的触发；其大致伪流程如下：

检测是否有活动的事件

while 有活动的事件处理:

    获取当前需要的活动事件

 如果有事件对应的回调函数：

      调用回调函数
![](images/Pasted%20image%2020241207233733.png)
# 常见使用流程
![](images/Pasted%20image%2020241207234221.png)
- event_base_new
	- 申请libevent最关键的数据结构event_base;该数据结构主要保存了如下几个关键数据：
		- 网络IO事件
		- 时间事件
		- 信号事件
		- 活动的事件队列
		- 信号操作函数接口
		- IO操作函数接口
- event_new
	- 申请一个新的event结构，初始化其事件类型以及事件触发的回调函数接口；事件初始化
- event_add
	- 把新申请的event添加到event_base里面，后续调用event_base_dispatch函数等待事件触发；事件等待发生。
- event_base_dispatch
	- 等待用户注册的事件触发，处理活动事件列表，调用其对应的用户回调接口

# 整体状态以及关键函数
![](images/Pasted%20image%2020241207235137.png)
在应用程序中，主要分为3类型事件
- 网络IO事件
	- 网络通信与其他机器建立了TCP、UDP连接，在有数据到来，或者需要发送数据而产生的读写事件
- 时间事件
	- 用户期望周期处理任务，某个时间点定时任务
- 信号事件
	- 用户设置系统产生信号，用户程序需要做的处理；比如有收到SIGTERM信号，SIGPIPE信号
- event事件
	- 主要用来保存用户程序关心的事件以及事件对应的回调函数；以及事件对应的状态转换处理

- event事件主要对应如下三种状态
	- 初始化状态
		-  用户调用event_new新申请事件，并且调用event_assign进行事件初始化后，可以认为该事件是一个初始化状态
	- 等待激活
		- 用户通过调用event_add函数把事件加入；如果是IO事件，libevent在底层通过调用epoll/select相应的add函数把事件加入linux内核，后续该事件对应的套接字，有数据到来或者数据发送，产生相应的读写事件；时间事件，就是超时触发；信号事件，就是内核检测到对应的信号产生，递交给用户态，然后libevent通过本身的机制，产生相应的事件。
	-  激活状态
		- 如果相应的事件发生，就把事件加入到event_base对应的active queue；然后在event_base_loop函数遍历该队列，调用用户的回调函数进行处理；同时，如果该事件不是持久化事件，则直接调用event_del函数将事件处理；如果是持久化事件，则进入等待状态，等待下一次事件触发。
- event_base主要提供select/epoll添加、删除IO事件；event_process_active处理相应的active queue


| 数据结构                   | 作用                                       |
| ---------------------- | ---------------------------------------- |
| struct event_base      | Libevent关键的数据结构，管理整个事件的添加，输出和响应          |
| struct event           | IO事件，时间事件，信号事件                           |
| event_signal_map       | 信号和IO事件共用的管理结构                           |
| event_io_map           | IO事件hash表管理结构                            |
| min_heap_t             | 时间事件最小堆管理结构                              |
| struct evcallback_list | event_base 结构体里面的活动队列，主要是一个链表，在queue.h实现 |
![](images/Pasted%20image%2020241208231220.png)