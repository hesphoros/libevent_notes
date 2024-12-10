# event_loop flag

## <font color="#8064a2">EVLOOP_ONCE</font> <font color="#8064a2">EVLOOP_NONBLOCK</font> <font color="#8064a2">EVLOOP_NO_EXIT_ON_EMPTY</font>

这三个也是 <font color="#4bacc6">event_base_loop()</font>的flag参数
```c
/** @name Loop flags

  

    These flags control the behavior of event_base_loop().

 */

/**@{*/

/** Block until we have an active event, then exit once all active events

 * have had their callbacks run. */

#define EVLOOP_ONCE  0x01

/** Do not block: see which events are ready now, run the callbacks

 * of the highest-priority ones, then exit. */

#define EVLOOP_NONBLOCK 0x02

/** Do not exit the loop because we have no pending events.  Instead, keep

 * running until event_base_loopexit() or event_base_loopbreak() makes us

 * stop.

 */

#define EVLOOP_NO_EXIT_ON_EMPTY 0x04

/**@}*/
```

- **`EVLOOP_ONCE` (0x01)**:
    
    - **含义**: 运行事件循环一次。即，阻塞直到有活动事件发生，然后在处理完所有活动事件的回调后退出循环。
    - **行为**: 当设置了这个标志时，事件循环将会一直阻塞，直到有一个或多个事件变为活动状态。一旦事件的回调函数被处理完，循环将会退出。适用于你希望事件循环处理完所有活动事件后立即退出的场景。
- **`EVLOOP_NONBLOCK` (0x02)**:
    
    - **含义**: 不进行阻塞检查。即，不等待新的事件发生，而是检查当前事件的状态，处理那些已经就绪的事件，然后退出循环。
    - **行为**: 当设置了这个标志时，事件循环不会阻塞。如果没有活动事件，事件循环将会快速退出。这适用于你希望在检查事件状态后立即退出的场景，例如周期性检查或非阻塞操作。
- **`EVLOOP_NO_EXIT_ON_EMPTY` (0x04)**:
    
    - **含义**: 即使没有待处理的事件，也不要退出循环。事件循环将会继续运行，直到调用 `event_base_loopexit()` 或 `event_base_loopbreak()` 使其停止。
    - **行为**: 当设置了这个标志时，事件循环即使在没有活动事件的情况下也不会退出。它将继续运行，直到显式地要求它退出。这适用于需要长期运行的事件循环，直到收到特定的退出命令。
# <font color="#8064a2">evutil_gettimeofday</font>

```c
/** Replacement for gettimeofday on platforms that lack it. */

#ifdef EVENT__HAVE_GETTIMEOFDAY

#define evutil_gettimeofday(tv, tz) gettimeofday((tv), (tz))

#else

struct timezone;

EVENT2_EXPORT_SYMBOL

int evutil_gettimeofday(struct timeval *tv, struct timezone *tz);

#endif
```

这段代码定义了一个函数`evutil_gettimeofday`，它是用于获取当前时间的函数，用于替代一些平台中没有`gettimeofday`函数的情况。

首先，代码中使用了`#ifdef`和`#define`来判断编译器是否支持`gettimeofday`函数。如果支持，则直接使用`gettimeofday`函数。如果不支持，则定义了一个名为`evutil_gettimeofday`的函数，它接受两个参数：`tv`和`tz`，分别表示要设置的时间值和时区信息。

# event definition flag


<font color="#8064a2">EV_TIMEOUT</font>
<font color="#8064a2">EV_READ</font>
<font color="#8064a2">EV_WRITE</font>
<font color="#8064a2">EV_SIGNAL</font>
<font color="#8064a2">EV_PERSIST</font>
<font color="#8064a2">EV_ET</font>

| 宏定义        | 含义                                  |
| ---------- | ----------------------------------- |
| EV_SIGNAL  | 信号事件                                |
| EV_READ    | IO读事件                               |
| EV_WRITE   | IO写事件                               |
| EV_PERSIST | 持续事件，事件触发一次后不会被删除，下次触发了还会继续调用回调函数处理 |
| EV_TIMEOUT | 超时事件                                |
~~~c
/**

 * @name event flags

 * Flags to pass to event_new(), event_assign(), event_pending(), and

 * anything else with an argument of the form "short events"

 */

/**@{*/

/** Indicates that a timeout has occurred.  It's not necessary to pass

 * this flag to event_for new()/event_assign() to get a timeout. */

#define EV_TIMEOUT   0x01

/** Wait for a socket or FD to become readable */

#define EV_READ      0x02

/** Wait for a socket or FD to become writeable */

#define EV_WRITE  0x04

/** Wait for a POSIX signal to be raised*/

#define EV_SIGNAL 0x08

/**

 * Persistent event: won't get removed automatically when activated.

 *

 * When a persistent event with a timeout becomes activated, its timeout

 * is reset to 0.

 */

#define EV_PERSIST   0x10

/** Select edge-triggered behavior, if supported by the backend. */

#define EV_ET     0x20

/**
 * If this option is provided, then event_del() will not block in one thread

 * while waiting for the event callback to complete in another thread.

 * To use this option safely, you may need to use event_finalize() or

 * event_free_finalize() in order to safely tear down an event in a

 * multithreaded application.  See those functions for more information.

 **/
 
/**
	*如果提供了这个选项，那么event_del()将不会在一个线程中阻塞
	
	当在另一个线程中等待事件回调完成时。
	
	*为了安全地使用这个选项，你可能需要使用event_finalize()或
	
	event_free_finalize()，以便安全地删除事件
	
	*多线程应用程序。有关更多信息，请参阅这些函数。
**/

#define EV_FINALIZE     0x40

/**
 * Detects connection close events.  You can use this to detect when a
 * connection has been closed, without having to read all the pending data
 * from a connection.
 * Not all backends support EV_CLOSED.  To detect or require it, use the
 * feature flag EV_FEATURE_EARLY_CLOSE.
 **/
#define EV_CLOSED 0x80

/**@}*/

/**
   @name evtimer_* macros
   Aliases for working with one-shot timer events
   If you need EV_PERSIST timer use event_*() functions.
 */
/**@{*/
~~~
## <font color="#8064a2">EV_TIMEOUT</font>
这个标志表示某超时时间流逝后事件成为激活的。构造事件的时候，<font color="#8064a2">EV_TIMEOUT</font>标志是被忽略的：可以在添加事件的时候设置超时，也可以不设置。超时发生时，回调函数的<font color="#4bacc6">what</font>参数将带有这个标志

## <font color="#8064a2">EV_READ</font>

表示指定的文件描述符已经就绪，可以读取的时候，事件将成为激活的。

## <font color="#8064a2">EV_WRITE</font>
表示指定的文件描述符已经就绪，可以写入的时候，事件将成为激活的。

## <font color="#8064a2">EV_SIGNAL</font>
用于实现信号检测。见[[]]

## <font color="#8064a2">EV_PERSIST</font>
表示事件是“持久的”， 

---

默认情况下，每当未决事件成为激活的（因为fd已经准备好读取或者写入，或者因为超时），事件将在其回调被执行前成为非未决的。如果想让事件再次成为未决的，可以在回调函数中再次对其调用<font color="#4bacc6">event_add（）</font>。

然而，如果设置了EV_PERSIST标志，事件就是持久的。这意味着即使其回调被激活，事件还是会保持为<font color="#c0504d">未决状态</font>。如果想在回调中让事件成为非未决的，可以对其调用<font color="#c0504d">event_del（）</font>。

每次执行事件回调的时候，持久事件的超时值会被复位。因此，如果具有<font color="#8064a2">EV_READ|EV_PERSIST</font>标志，以及5秒的超时值，则事件将在以下情况下成为激活的：

-  套接字已经准备好被读取的时候

-  从最后一次成为激活的开始，已经逝去5秒

## <font color="#8064a2">EV_ET</font>
表示如果底层的<font color="#4bacc6">event_base</font>后端支持边沿触发事件，则事件应该是边沿触发的。这个标志影响EV_READ和EV_WRITE的语义。

从2.0.1-alpha版本开始，可以有任意多个事件因为同样的条件而未决。比如说，可以有两个事件因为某个给定的fd已经就绪，可以读取而成为激活的。这种情况下，多个事件回调被执行的次序是不确定的。

这些标志定义在<event2/event.h>中。除了<font color="#8064a2">EV_ET</font>在2.0.1-alpha版本中引入外，所有标志从1.0版本开始就存在了。

# timeout event

 
~~~c
#define evtimer_assign(ev, b, cb, arg) \

   event_assign((ev), (b), -1, 0, (cb), (arg))

#define evtimer_new(b, cb, arg)     event_new((b), -1, 0, (cb), (arg))

#define evtimer_add(ev, tv)      event_add((ev), (tv))

#define evtimer_del(ev)       event_del(ev)

#define evtimer_pending(ev, tv)     event_pending((ev), EV_TIMEOUT, (tv))

#define evtimer_initialized(ev)     event_initialized(ev)
~~~

# event signal 
~~~c
#define evsignal_add(ev, tv)     event_add((ev), (tv))

#define evsignal_assign(ev, b, x, cb, arg)         \

   event_assign((ev), (b), (x), EV_SIGNAL|EV_PERSIST, cb, (arg))

#define evsignal_new(b, x, cb, arg)          \

   event_new((b), (x), EV_SIGNAL|EV_PERSIST, (cb), (arg))

#define evsignal_del(ev)      event_del(ev)

#define evsignal_pending(ev, tv) event_pending((ev), EV_SIGNAL, (tv))

#define evsignal_initialized(ev) event_initialized(ev)
~~~


# evutil_socket_t
~~~c
/**

 * A type wide enough to hold the output of "socket()" or "accept()".  On

 * Windows, this is an intptr_t; elsewhere, it is an int. */

#ifdef _WIN32

#define evutil_socket_t intptr_t

#else

#define evutil_socket_t int

#endif
~~~

# <font color="#8064a2">BEV_LOCK</font> <font color="#8064a2">BEV_UNLOCK</font>

These definitions are used when locking is not enabled. `EVUTIL_NIL_STMT_` is typically a no-op or empty statement, meaning no locking is applied.

~~~c
#define BEV_LOCK(b) EVUTIL_NIL_STMT_
#define BEV_UNLOCK(b) EVUTIL_NIL_STMT_
~~~


---
~~~c
#define BEV_LOCK(b) do {                        \
     struct bufferevent_private *locking =  BEV_UPCAST(b);   \
     EVLOCK_LOCK(locking->lock, 0);              \
 } while (0)

#define BEV_UNLOCK(b) do {                      \
     struct bufferevent_private *locking =  BEV_UPCAST(b);   \
     EVLOCK_UNLOCK(locking->lock, 0);            \
 } while (0)

~~~

- **`BEV_LOCK(b)`**:
    
    - `b` 是一个指向 `bufferevent` 结构体的指针。
    - `BEV_UPCAST(b)` 宏将 `b` 转换为 `bufferevent_private` 结构体的指针，并赋值给 `locking`。
    - `EVLOCK_LOCK(locking->lock, 0)` 用于对 `locking` 中的 `lock` 进行加锁操作。这里的 `0` 可能表示锁的某种标志或者选项，具体含义取决于 `EVLOCK_LOCK` 的实现。
- **`BEV_UNLOCK(b)`**:
    
    - `b` 也是一个指向 `bufferevent` 结构体的指针。
    - 同样，`BEV_UPCAST(b)` 将 `b` 转换为 `bufferevent_private` 结构体的指针，并赋值给 `locking`。
    - `EVLOCK_UNLOCK(locking->lock, 0)` 用于对 `locking` 中的 `lock` 进行解锁操作。这里的 `0` 与加锁时的标志或选项相同。

# <font color="#8064a2">BEV_UPCAST</font>

	/** 给定一个 bufferevent，返回其对应的 bufferevent_private。*/
~~~c
/** Internal: Given a bufferevent, return its corresponding

 * bufferevent_private. */

#define BEV_UPCAST(b) EVUTIL_UPCAST((b), struct bufferevent_private, bev)
~~~

~~~c  
  

/** Helper macro.  If we know that a given pointer points to a field in a

    structure, return a pointer to the structure itself.  Used to implement

    our half-baked C OO.  Example:

  

    struct subtype {

    int x;

    struct supertype common;

    int y;

    };

    ...

    void fn(struct supertype *super) {

    struct subtype *sub = EVUTIL_UPCAST(super, struct subtype, common);

    ...

    }

 */

#define EVUTIL_UPCAST(ptr, type, field)             \

    ((type *)(((char*)(ptr)) - evutil_offsetof(type, field)))
~~~    
