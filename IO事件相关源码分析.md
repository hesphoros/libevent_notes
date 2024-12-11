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
