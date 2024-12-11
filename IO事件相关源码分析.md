## event_base事件管理结构初始化
struct event_io_map io；

struct event_io_map有如下两种定义
~~~c
#ifdef EVMAP_USE_HT
#define HT_NO_CACHE_HASH_VALUES
#include "ht-internal.h"
struct event_map_entry;
HT_HEAD(event_io_map, event_map_entry);
#else
#define event_io_map event_signal_map
#endif
~~~
主要有两种定义方式：
1. hash表
2. 数组表
查看数组表event_signal_map定义
~~~c
struct event_signal_map 
{
  void **entries;
  int nentries;
};
~~~
其对应的数据结构如下所示：
![](images/Pasted%20image%2020241211202902.png)每个fd对应一个数组元素，存储对应的fd需要关注的读事件、写事件、错误事件。

event_base数据结构的成员变量io的初始化流程：
![](images/Pasted%20image%2020241211202927.png)
evmap_signal_initmap_ 函数主要把该数据结构的成员赋值为空，后续添加事件的时候，在申请内存。


