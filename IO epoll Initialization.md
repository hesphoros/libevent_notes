# epoll初始化
epoll模型关键的epollop数据结构
~~~c
struct epollop
{
    struct epoll_event *events;
    int nevents;
    int epfd;
#ifdef USING_TIMERFD
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
关键API

| 函数接口                       | 函数说明                                               |
| -------------------------- | -------------------------------------------------- |
| event_base_priority_init   | 初始化event_base接口的优先级队列，必须在event_base_dispatch接口之前使用 |
| event_base_get_npriorities | 获取event_base优先级队列总个数                               |
| event_priority_set         | 设置某个事件的优先级                                         |
