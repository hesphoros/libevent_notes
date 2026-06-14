# run loop
一旦有了一个已经注册了某些事件的event_base，就需要让libevent等待事件并且通知事件的发生。
## <font color="#4bacc6">event_base_loop()</font>
```c
int event_base_loop(struct event_base *, int);
```

### source code
相关宏定义见： [Macro definition](Macro%20definition.md)
相关宏函数见： [Macro function](Macro%20function.md)
<font color="#8064a2">EVBASE_ACQUIRE_LOCK</font>
<font color="#8064a2">N_ACTIVE_CALLBACKS</font>
```c

int
event_base_loop(struct event_base *base, int flags)
{
	const struct eventop *evsel = base->evsel;  // 获取事件操作接口
	struct timeval tv;  // 用于存储超时时间
	struct timeval *tv_p;  // 超时时间指针
	int res, done, retval = 0;  // 返回值、循环控制标志和结果状态

	/* 获取锁，以确保线程安全。将在 evsel.dispatch 内部释放锁 */
	EVBASE_ACQUIRE_LOCK(base, th_base_lock);

	if (base->running_loop) {  // 检查是否已有循环在运行
		event_warnx("%s: reentrant invocation.  Only one event_base_loop"
		    " can run on each event_base at once.", __func__);
		EVBASE_RELEASE_LOCK(base, th_base_lock);  // 释放锁
		return -1;  // 返回错误
	}

	base->running_loop = 1;  // 标记循环正在运行

	clear_time_cache(base);  // 清理时间缓存

	if (base->sig.ev_signal_added && base->sig.ev_n_signals_added)  // 如果有信号事件，设置信号基础
		evsig_set_base_(base);

	done = 0;  // 初始化循环终止标志

#ifndef EVENT__DISABLE_THREAD_SUPPORT
	base->th_owner_id = EVTHREAD_GET_ID();  // 设置线程所有者ID
#endif

	base->event_gotterm = base->event_break = 0;  // 初始化事件终止和中断标志

	while (!done) {  // 循环直到 done 为真
		base->event_continue = 0;  // 初始化继续标志
		base->n_deferreds_queued = 0;  // 初始化延迟队列计数

		/* 检查是否需要终止循环 */
		if (base->event_gotterm) {
			break;
		}

		if (base->event_break) {
			break;
		}

		tv_p = &tv;  // 设置超时指针
		if (!N_ACTIVE_CALLBACKS(base) && !(flags & EVLOOP_NONBLOCK)) {  // 如果没有活动回调且非非阻塞标志
			timeout_next(base, &tv_p);  // 获取下一个超时时间
		} else {
			/* 如果有活动事件，直接轮询而不等待 */
			evutil_timerclear(&tv);  // 清除时间
		}

		/* 如果没有事件，并且不允许在空状态下退出，则退出 */
		if (0==(flags&EVLOOP_NO_EXIT_ON_EMPTY) &&
		    !event_haveevents(base) && !N_ACTIVE_CALLBACKS(base)) {
			event_debug(("%s: no events registered.", __func__));
			retval = 1;  // 设置结果状态为1
			goto done;  // 跳转到清理部分
		}

		event_queue_make_later_events_active(base);  // 激活稍后的事件
		clear_time_cache(base);  // 再次清理时间缓存
		res = evsel->dispatch(base, tv_p);  // 调用事件调度函数

		if (res == -1) {  // 检查调度函数是否成功
			event_debug(("%s: dispatch returned unsuccessfully.",
				__func__));
			retval = -1;  // 设置结果状态为-1
			goto done;  // 跳转到清理部分
		}

		update_time_cache(base);  // 更新时间缓存

		timeout_process(base);  // 处理超时事件

		if (N_ACTIVE_CALLBACKS(base)) {  // 如果有活动回调
			int n = event_process_active(base);  // 处理活动回调
			if ((flags & EVLOOP_ONCE)  // 如果是一次性循环
			    && N_ACTIVE_CALLBACKS(base) == 0
			    && n != 0)
				done = 1;  // 设置循环结束标志
		} else if (flags & EVLOOP_NONBLOCK)
			done = 1;  // 如果是非阻塞标志，结束循环
	}
	event_debug(("%s: asked to terminate loop.", __func__));

done:
	clear_time_cache(base);  // 清理时间缓存
	base->running_loop = 0;  // 标记循环已结束

	EVBASE_RELEASE_LOCK(base, th_base_lock);  // 释放锁

	return (retval);  // 返回最终结果
}

```

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
默认情况下，<font color="#4bacc6">event_base_loop（）</font>函数运行<font color="#4f81bd">event_base</font>直到其中没有已经注册的事件为止。执行循环的时候，函数重复地检查是否有任何已经注册的事件被触发（比如说，读事件的文件描述符已经就绪，可以读取了；或者超时事件的超时时间即将到达）。如果有事件被触发，函数标记被触发的事件为“激活的”，并且执行这些事件。

在<font color="#00b050">flags</font>参数中设置一个或者多个标志就可以改变<font color="#4bacc6">event_base_loop（）</font>的行为。如果设置了<font color="#8064a2">EVLOOP_ONCE</font>，循环将等待某些事件成为激活的，执行激活的事件直到没有更多的事件可以执行，然会返回。如果设置了<font color="#8064a2">EVLOOP_NONBLOCK</font>，循环不会等待事件被触发：循环将仅仅检测是否有事件已经就绪，可以立即触发，如果有，则执行事件的回调。

完成工作后，如果正常退出，<font color="#4bacc6">event_base_loop（）</font>返回0；如果因为后端中的某些未处理错误而退出，则返回-1。
## pseudo-code
 
 

![Pasted image 20240831141854](images/Pasted%20image%2020240831141854.png)

## <font color="#4bacc6">event_base_dispatch()</font>

epoll_dispatch select_dispatch 见:[IO 事件响应](IO%20事件响应.md)

```c
int
event_base_dispatch(struct event_base *event_base)
{
    return (event_base_loop(event_base, 0));
}
```
<font color="#4bacc6">event_base_dispatch（）</font>等同于没有设置标志的<font color="#4bacc6">event_base_loop（）</font>。所以，<font color="#4bacc6">event_base_dispatch（）</font>将一直运行，直到没有已经注册的事件了，或者调用了<font color="#4bacc6">event_base_loopbreak（）</font>或者<font color="#4bacc6">event_base_loopexit（）</font>为止。


# stop loop
如果想在移除所有已注册的事件之前停止活动的事件循环，可以调用两个稍有不同的函数。
## API


	注意event_base_loopexit(base,NULL)和event_base_loopbreak(base)在事件循环没有运行时的行为不同：前者安排下一次事件循环在下一轮回调完成后立即停止（就好像带EVLOOP_ONCE标志调用一样）；后者却仅仅停止当前正在运行的循环，如果事件循环没有运行，则没有任何效果。

这两个函数都在成功时返回0，失败时返回-1。

### <font color="#4bacc6">event_base_loopbreak()</font>
event_base_loopbreak（）让event_base立即退出循环。它与event_base_loopexit（base,NULL）的不同在于，如果event_base当前正在执行激活事件的回调，它将在执行完当前正在处理的事件后立即退出。
```c
int event_base_loopbreak(struct event_base *event_base)
```
#### source code
```c
int event_base_loopbreak(struct event_base *event_base)

{

    int r = 0;

    if (event_base == NULL)

        return (-1);

  
	//获取锁
    EVBASE_ACQUIRE_LOCK(event_base, th_base_lock);

    event_base->event_break = 1;

  

    if (EVBASE_NEED_NOTIFY(event_base)) {

        r = evthread_notify_base(event_base);

    } else {

        r = (0);

    }

    EVBASE_RELEASE_LOCK(event_base, th_base_lock);

    return r;

}
```
### <font color="#4bacc6">event_base_loopexit()</font>

注意event_base_loopexit(base,NULL)和event_base_loopbreak(base)在事件循环没有运行时的行为不同：前者安排下一次事件循环在下一轮回调完成后立即停止（就好像带EVLOOP_ONCE标志调用一样）；后者却仅仅停止当前正在运行的循环，如果事件循环没有运行，则没有任何效果。
event_base_loopexit（）让event_base在给定时间之后停止循环。如果tv参数为NULL，event_base会立即停止循环，没有延时。如果event_base当前正在执行任何激活事件的回调，则回调会继续运行，直到运行完所有激活事件的回调之才退出。
```c

int event_base_loopexit(struct event_base *event_base, const struct timeval *tv)
```

#### source code


```c


static void event_loopexit_cb(evutil_socket_t fd, short what, void *arg)

{

    struct event_base *base = arg;

    base->event_gotterm = 1;

}

int event_base_loopexit(struct event_base *event_base, const struct timeval *tv)

{

    return (event_base_once(event_base, -1, EV_TIMEOUT, event_loopexit_cb,

            event_base, tv));

}
```
- **`event_base`**: 事件基础。
- **`-1`**: 事件标识符，`-1` 表示这是一个一次性事件，没有持久化标识。
- **`EV_TIMEOUT`**: 事件类型，这里是超时事件。表示这个事件是基于超时的。
- **`event_loopexit_cb`**: 事件回调函数，当事件触发时会调用这个函数。
- **`event_base`**: 作为回调函数的用户数据传递，这里传递了 `event_base` 结构的指针。
- **`tv`**: 超时时间。如果为 `NULL`，超时时间可能会被视为立即触发

<font color="#4bacc6">event_base_loopexit（）</font>让event_base在给定时间之后停止循环。如果<font color="#00b050">tv</font>参数为<font color="#8064a2">NULL</font>，<font color="#4bacc6">event_base</font>会立即停止循环，没有延时。如果<font color="#4bacc6">event_base</font>当前正在执行任何激活事件的回调，则回调会继续运行，直到运行完所有激活事件的回调之才退出。

<font color="#4bacc6">event_base_loopbreak（）</font>让event_base立即退出循环。它与<font color="#4bacc6">event_base_loopexit（base,NULL）</font>的不同在于，如果event_base当前正在执行激活事件的回调，它将在执行完当前正在处理的事件后立即退出。


## <font color="#4bacc6">event_base_once()</font>
```c
/* Schedules an event once */

int

event_base_once(struct event_base *base, evutil_socket_t fd, short events,

    void (*callback)(evutil_socket_t, short, void *),

    void *arg, const struct timeval *tv)

{

    struct event_once *eonce;

    int res = 0;

    int activate = 0;

  

    if (!base)

        return (-1);

  

    /* We cannot support signals that just fire once, or persistent

     * events. */

    if (events & (EV_SIGNAL|EV_PERSIST))

        return (-1);

  

    if ((eonce = mm_calloc(1, sizeof(struct event_once))) == NULL)

        return (-1);

  

    eonce->cb = callback;

    eonce->arg = arg;

  

    if ((events & (EV_TIMEOUT|EV_SIGNAL|EV_READ|EV_WRITE|EV_CLOSED)) == EV_TIMEOUT) {

        evtimer_assign(&eonce->ev, base, event_once_cb, eonce);

  

        if (tv == NULL || ! evutil_timerisset(tv)) {

            /* If the event is going to become active immediately,

             * don't put it on the timeout queue.  This is one

             * idiom for scheduling a callback, so let's make

             * it fast (and order-preserving). */

            activate = 1;

        }

    } else if (events & (EV_READ|EV_WRITE|EV_CLOSED)) {

        events &= EV_READ|EV_WRITE|EV_CLOSED;

  

        event_assign(&eonce->ev, base, fd, events, event_once_cb, eonce);

    } else {

        /* Bad event combination */

        mm_free(eonce);

        return (-1);

    }

  

    if (res == 0) {

        EVBASE_ACQUIRE_LOCK(base, th_base_lock);

        if (activate)

            event_active_nolock_(&eonce->ev, EV_TIMEOUT, 1);

        else

            res = event_add_nolock_(&eonce->ev, tv, 0);

  

        if (res != 0) {

            mm_free(eonce);

            return (res);

        } else {

            LIST_INSERT_HEAD(&base->once_events, eonce, next_once);

        }

        EVBASE_RELEASE_LOCK(base, th_base_lock);

    }

  

    return (0);

}
```

- **参数说明**:
    
    - **`base`**: 指向 `event_base` 结构的指针，用于指定事件基础。
    - **`fd`**: 文件描述符。用于 I/O 事件时指定哪个文件描述符。
    - **`events`**: 事件类型标志，包括超时 (`EV_TIMEOUT`)、读 (`EV_READ`)、写 (`EV_WRITE`) 和关闭 (`EV_CLOSED`)。
    - **`callback`**: 回调函数指针，在事件发生时调用。
    - **`arg`**: 传递给回调函数的用户数据。
    - **`tv`**: 超时时间的指针。用于设置超时事件的超时时间。
- **主要逻辑**:
    
    - **基础检查**: 检查 `base` 是否为 `NULL`，如果是，则返回 `-1`。
    - **事件类型检查**: 只允许 `EV_TIMEOUT`、`EV_READ`、`EV_WRITE` 和 `EV_CLOSED` 事件类型。不能处理 `EV_SIGNAL` 和 `EV_PERSIST` 类型。
    - **内存分配**: 分配 `struct event_once` 结构的内存，并检查分配是否成功。
    - **事件类型处理**:
        - **超时事件** (`EV_TIMEOUT`): 如果 `tv` 为 `NULL` 或无效，标记事件为立即激活。
        - **I/O 事件** (`EV_READ`、`EV_WRITE`、`EV_CLOSED`): 处理与文件描述符相关的事件。
    - **锁操作**:
        - 获取 `base` 的锁。
        - 如果事件需要立即激活，调用 `event_active_nolock_` 使事件立即激活。
        - 否则，将事件添加到事件队列中，调用 `event_add_nolock_`。
        - 在锁内将事件插入到一次性事件列表 `once_events` 中。
        - 释放锁。
    - **返回值**: 成功时返回 `0`，否则返回错误代码。

# Check the internal time cache
有时候需要在事件回调中获取当前时间的近似视图，但不想调用<font color="#4bacc6">gettimeofday()</font>（可能是因为OS将<font color="#4bacc6">gettimeofday()</font>作为系统调用实现，而你试图避免系统调用的开销）。
## event_base_gettimeofday_cached()
在回调中，可以请求libevent开始本轮回调时的当前时间视图。
```c
int event_base_gettimeofday_cached(struct event_base *base, struct timeval *tv)
```
## source code
```c  
int event_base_gettimeofday_cached(struct event_base *base, struct timeval *tv)

{

    int r;

    if (!base) {

        base = current_base;

        if (!current_base)

            return evutil_gettimeofday(tv, NULL);

    }

  

    EVBASE_ACQUIRE_LOCK(base, th_base_lock);

    if (base->tv_cache.tv_sec == 0) {

        r = evutil_gettimeofday(tv, NULL);

    } else {

        evutil_timeradd(&base->tv_cache, &base->tv_clock_diff, tv);

        r = 0;

    }

    EVBASE_RELEASE_LOCK(base, th_base_lock);

    return r;

}
```
如果当前正在执行回调，<font color="#4bacc6">event_base_gettimeofday_cached()</font>函数设置tv_out参数的值为缓存的时间。否则，函数调用evutil_gettimeofday()获取真正的当前时间。成功时函数返回0，失败时返回负数。

注意，因为libevent在开始执行回调的时候缓存时间值，所以这个值至少是有一点不精确的。如果回调执行很长时间，这个值将非常不精确。

这个函数是libevent 2.0.4-alpha新引入的

# Example Dump <font color="#4bacc6">event_base</font> status
## event_base_dump_events
```c
void event_base_dump_events(struct event_base *base, FILE *output)
```

## source code
~~~c
void event_base_dump_events(struct event_base *base, FILE *output)

{

    EVBASE_ACQUIRE_LOCK(base, th_base_lock);

    fprintf(output, "Inserted events:\n");

    event_base_foreach_event_nolock_(base, dump_inserted_event_fn, output);

  

    fprintf(output, "Active events:\n");

    event_base_foreach_event_nolock_(base, dump_active_event_fn, output);

    EVBASE_RELEASE_LOCK(base, th_base_lock);

}
~~~
为帮助调试程序（或者调试libevent），有时候可能需要加入到event_base的事件及其状态的完整列表。调用event_base_dump_events()可以将这个列表输出到指定的文件中。
这个函数在libevent 2.0.1-alpha版本中引入。

# dep event loop function
前面已经讨论过，老版本的libevent 具有“当前”event_base的概念。

本文讨论的某些事件循环函数具有操作当前event_base的变体。除了没有base参数外，这些函数跟当前新版本函数的行为相同。

| Current function                                    | Obsolete current-base version                     |
| --------------------------------------------------- | ------------------------------------------------- |
| <font color="#4bacc6">event_base_dispatch()</font>  | <font color="#eeece1">event_dispatch()</font><br> |
| <font color="#4bacc6">event_base_loop()</font>      | <font color="#eeece1">event_loop()</font>         |
| <font color="#4bacc6">event_base_loopexit()</font>  | <font color="#eeece1">event_loopexit()</font>     |
| <font color="#4bacc6">event_base_loopbreak()</font> | <font color="#eeece1">event_loopbreak()</font>    |
## <font color="#ffff00">Attention</font>
2.0版本之前的<font color="#8064a2">event_base</font>是不支持锁的，所以这些函数并不是完全线程安全的：不允许在执行事件循环的线程之外的其他线程中调用<font color="#8064a2">_loopbreak()</font>或者<font color="#8064a2">_loopexit()</font>函数。



