# epoll初始化

epoll 相关API见: [epoll API](epoll%20API.md)
epoll模型关键的epollop数据结构
~~~c
struct epollop
{
	/* 指向 epoll_event 结构体数组的指针 用于在调用 epoll_wait 时 接收内核返回的、已经就绪的 I/O 事件列表 */
    struct epoll_event *events;  
    /* 记录 events 数组的最大容量(即最多能同时处理多少个就绪事件) 通常在初始化时分配内存 并作为 epoll_wait 的参数限制返回的事件数量 */
    int                nevents;
    /* epoll 实例的文件描述符.由 epoll_create 或 epoll_create1 创建，后续的 epoll_ctl 和 epoll_wait 都需要用到它 */
    int                   epfd;
#ifdef USING_TIMERFD
	/* 定时器文件描述符（Timer FD） Linux 特有的机制 将定时器转化为文件描述符 
	 * 从而可以像普通 socket 一样 直接放进 epoll 监控，实现高效、统一的定时器事件管理。 */
    int timerfd;
#endif
};
~~~

该结构主要通过epoll_init函数完成初始化；
初始化流程如下所示
![](images/Pasted%20image%2020241210000004.png)
# select初始化
常见的还有select多路复用函数，其初始化如下

select模型关键的selectop数据结构
~~~c
struct selectop 
{
  int event_fds;      /* Highest fd in fd set */
  int event_fdsz;
  int resize_out_sets;
  fd_set *event_readset_in;
  fd_set *event_writeset_in;
  fd_set *event_readset_out;
  fd_set *event_writeset_out;
};
~~~

该结构的初始化流程如下：
![select初始化](images/Pasted%20image%2020241210233830.png)
# **优先级队列初始化以及使用**

| 函数接口                       | 函数说明                                               |
| -------------------------- | -------------------------------------------------- |
| event_base_priority_init   | 初始化event_base接口的优先级队列，必须在event_base_dispatch接口之前使用 |
| event_base_get_npriorities | 获取event_base优先级队列总个数                               |
| event_priority_set         | 设置某个事件的优先级                                         |
