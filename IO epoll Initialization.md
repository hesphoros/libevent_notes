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