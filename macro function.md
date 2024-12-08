
# <font color="#8064a2">TAILQ_INIT</font> 
```c++
/*
 * Tail queue functions.
 * 尾队列的头结点初始化为空队列。
 */
#define	TAILQ_INIT(head) do {						\
	(head)->tqh_first = NULL;					\
	(head)->tqh_last = &(head)->tqh_first;				\
} while (/*CONSTCOND*/0)

```

定义于sys/queue.h 在compat/sys下持有该定义
`TAILQ_INIT` 宏是一个用于初始化尾队列头部的工具，它通过将 `tqh_first` 设置为 `NULL` 并将 `tqh_last` 设置为指向 `tqh_first` 的地址，确保队列可以从空状态正确地过渡到非空状态。这个设计使用了二级指针的概念，使得尾部插入操作更为高效和直接。这种结构广泛应用于需要动态管理链表的系统编程场景中。
	**`do { ... } while (0)`**：

- 这种结构确保了宏在使用时会像一个普通的语句一样执行，而不会引起语法错误。例如，用户可以安全地在 `if` 语句中使用这个宏，而不会因为缺少大括号导致错误
- **`(head)->tqh_first = NULL;`**：
    
    - 初始化 `tqh_first` 为 `NULL`，表示队列当前没有任何元素，是空的。
- **`(head)->tqh_last = &(head)->tqh_first;`**：
    
    - 初始化 `tqh_last` 使其指向 `tqh_first` 的地址。这样做的目的是为了方便在队列末尾进行插入操作。具体来说，如果队列是空的，`tqh_last` 会指向 `tqh_first`，这意味着下一次插入的元素将成为新的第一个元素

# <font color="#8064a2">EVUTIL_ASSERT</font>

```c
#define EVUTIL_ASSERT(cond)						\
	do {								\
		if (EVUTIL_UNLIKELY(!(cond))) {				\
			event_errx(EVENT_ERR_ABORT_,			\
			    "%s:%d: Assertion %s failed in %s",		\
			    __FILE__,__LINE__,#cond,__func__);		\
			/* In case a user-supplied handler tries to */	\
			/* return control to us, log and abort here. */	\
			(void)fprintf(stderr,				\
			    "%s:%d: Assertion %s failed in %s",		\
			    __FILE__,__LINE__,#cond,__func__);		\
			abort();					\
		}							\
	} while (0)
```

<font color="#8064a2">EVUTIL_ASSERT</font> 是一个用于断言条件的宏，在条件不满足时会中止程序运行，并打印相关的调试信息。

**`do { ... } while (0)` 结构**：

- 这是一个常见的宏包装技巧，用于确保宏在使用时能够像普通语句一样执行，不会因为缺少分号或其他语法问题导致错误。
**`if (EVUTIL_UNLIKELY(!(cond))) { ... }`**：

- `EVUTIL_UNLIKELY` 是一个宏或函数，通常用于提示编译器某个条件不太可能发生，这可以帮助编译器优化代码。这里的意思是，如果条件 `cond` 为 `false`（即条件不满足），则执行后续代码。
**`event_errx(EVENT_ERR_ABORT_, "%s:%d: Assertion %s failed in %s", __FILE__, __LINE__, #cond, __func__);`**：

- `event_errx` 是一个函数，用于打印错误信息并终止程序。它会输出文件名（`__FILE__`）、行号（`__LINE__`）、断言失败的条件（`#cond`），以及当前函数的名称（`__func__`）。
- `#cond` 是将 `cond` 转换为字符串的预处理器操作，这样可以打印出具体的条件表达式。
- `EVENT_ERR_ABORT_` 通常是一个预定义的常量，用于指示错误类型，
**`fprintf(stderr, "%s:%d: Assertion %s failed in %s", __FILE__, __LINE__, #cond, __func__);`**：

- 在调用 `event_errx` 后，再次使用 `fprintf` 将相同的错误信息输出到标准错误输出（`stderr`）。这是为了防止用户自定义的错误处理程序可能试图返回控制权，确保错误信息一定会被输出。
最后，调用 `abort()` 函数立即终止程序的执行。`abort()` 会导致程序异常退出，并生成一个核心转储（如果系统配置允许）

```c

void event_errx(int eval, const char *fmt, ...)
{
	va_list ap;
	// 初始化可变参数列表
	va_start(ap, fmt);
	// 使用可变参数列表将格式化的错误信息输出到日志
	event_logv_(EVENT_LOG_ERR, NULL, fmt, ap);
	va_end(ap);
	// 结束可变参数处理
	// 退出程序并返回指定的错误码
	event_exit(eval);
}
```


该函数是可变参函数，相关注解见: [[可变参数]]

```c
void

event_logv_(int severity, const char *errstr, const char *fmt, va_list ap)

{

    char buf[1024];

    size_t len;

  

    if (severity == EVENT_LOG_DEBUG && !event_debug_get_logging_mask_())

        return;

  

    if (fmt != NULL)

        evutil_vsnprintf(buf, sizeof(buf), fmt, ap);

    else

        buf[0] = '\0';

  

    if (errstr) {

        len = strlen(buf);

        if (len < sizeof(buf) - 3) {

            evutil_snprintf(buf + len, sizeof(buf) - len, ": %s", errstr);

        }

    }

  

    event_log(severity, buf);

}
```

# <font color="#8064a2">EVBASE_ACQUIRE_LOCK</font> <font color="#8064a2">N_ACTIVE_CALLBACKS</font>

```c
/** Lock an event_base, if it is set up for locking.  Acquires the lock
    in the base structure whose field is named 'lockvar'. */
#define EVBASE_ACQUIRE_LOCK(base, lockvar) do {				\
		EVLOCK_LOCK((base)->lockvar, 0);			\
	} while (0)

```

```c

#define N_ACTIVE_CALLBACKS(base)					\
	((base)->event_count_active) //最大事件数量

```

```c
/** Largest number of priorities that Libevent can support. */
#define EVENT_MAX_PRIORITIES 256
```

 

```bash
  /**

Set the number of different event priorities

By default Libevent schedules all active events with the same priority.

However, some time it is desirable to process some events with a higher

priority than others.  For that reason, Libevent supports strict priority

queues.  Active events with a lower priority are always processed before

events with a higher priority.

The number of different priorities can be set initially with the

event_base_priority_init() function.  This function should be called

before the first call to event_base_dispatch().  The

event_priority_set() function can be used to assign a priority to an

event.  By default, Libevent assigns the middle priority to all events

unless their priority is explicitly set.

Note that urgent-priority events can starve less-urgent events: after

running all urgent-priority callbacks, Libevent checks for more urgent

events again, before running less-urgent events.  Less-urgent events

will not have their callbacks run until there are no events more urgent

than them that want to be active.

@param eb the event_base structure returned by event_base_new()

@param npriorities the maximum number of priorities

@return 0 if successful, or -1 if an error occurred

@see event_priority_set()

```
# <font color="#8064a2">TAILQ_ENTRY</font>
```c
//通过使用 TAILQ_ENTRY 宏，可以为指定的数据类型创建一个双向链表的入口和出口结构体，方便在链表中进行插入、删除和遍历等操作。
#define	_TAILQ_ENTRY(type, qual)					\
struct {								\
	qual type *tqe_next;		/* next element */		\
	qual type *qual *tqe_prev;	/* address of previous next element */\
}
#define TAILQ_ENTRY(type)	_TAILQ_ENTRY(struct type,)

```
`qual` 用于指定链表结构体成员的修饰符(它可以是 `const`、`volatile` 或其他限定符。)

-  `tqe_next`：指向链表中下一个元素的指针。
  
- `tqe_prev`：指向链表中上一个元素的指针的地址。这里使用了一个指向指针的指针，即二级指针，用于在删除元素时修改前一个元素的 `tqe_next` 指针。

# <font color="#8064a2">TAILQ_HEAD</font>
```c
#define TAILQ_HEAD(name, type)          \
struct name {                          \
    struct type *tqh_first;            /* 指向链表的第一个元素 */ \
    struct type **tqh_last;            /* 指向链表的最后一个元素的指针的指针 */ \
}

```
`TAILQ_HEAD` 是用于定义双向链表头结构的宏，使得管理链表的操作更加简单
- **`tqh_first`**: 指向链表的第一个元素。对于空链表，它是 `NULL`。
    
- **`tqh_last`**: 指向链表的最后一个元素的指针的指针。在链表的最后一个元素中，`tqh_last` 指向链表中最后一个元素的指针（也就是指向该元素的指针的指针）。对于空链表，它指向 `&tqh_first`，即链表头结构中的 `tqh_first`。
# <font color="#4bacc6">EVBASE_NEED_NOTIFY</font>
```c
/** Return true iff we need to notify the base's main thread about changes to

 * its state, because it's currently running the main loop in another

 * thread. Requires lock. */

#define EVBASE_NEED_NOTIFY(base)             \

    (evthread_id_fn_ != NULL &&          \

        (base)->running_loop &&          \

        (base)->th_owner_id != evthread_id_fn_())
```

- **`evthread_id_fn_`**: 一个函数指针，用于获取当前线程的 ID。它用于确定事件循环是否在另一个线程中运行。
    
- **`(base)->running_loop`**: 表示事件基础是否正在运行主事件循环。如果为 `true`，表示主循环正在运行中。
    
- **`(base)->th_owner_id`**: 存储主事件循环线程的 ID。这个 ID 是在事件循环开始时设置的。
    
- **`evthread_id_fn_()`**: 调用 `evthread_id_fn_` 函数来获取当前线程的 ID。
---

- **`evthread_id_fn_ != NULL`**: 确保线程 ID 函数指针有效，这意味着线程 ID 函数已经设置并可用。
    
- **`(base)->running_loop`**: 确保事件循环确实在运行中。
    
- **`(base)->th_owner_id != evthread_id_fn_()`**: 确保当前线程的 ID 与事件循环主线程的 ID 不同，表明主循环在另一个线程中运行。
- ---
 

这个宏的作用是检查是否需要通知主线程关于状态的变化，因为事件基础的主线程可能在不同的线程中运行。如果条件满足（即事件循环在另一个线程中运行且当前线程不是主线程），则返回 `true`，表示需要通知主线程。否则返回 `false`。

# <font color="#8064a2">EVBASE_RELEASE_LOCK</font> <font color="#8064a2">EVLOCK_UNLOCK</font>


```c
/** Unlock an event_base, if it is set up for locking. */

#define EVBASE_RELEASE_LOCK(base, lockvar) do {             \

        EVLOCK_UNLOCK((base)->lockvar, 0);          \

    } while (0)
```

```c
/** Release a lock */

#define EVLOCK_UNLOCK(lockvar,mode)                 \

    do {                                \

        if (lockvar)                        \

            evthread_lock_fns_.unlock(mode, lockvar);   \

    } while (0)
```

- **`lockvar`**: 具体的锁变量。这个变量是锁的实际对象，用于加锁和解锁。
- **`mode`**: 解锁模式，这个参数通常用于指定解锁的方式，但在这段代码中它被设置为 `0`，具体模式取决于锁的实现。
- **`evthread_lock_fns_.unlock`**: 这是一个函数指针，指向实际的解锁函数。`evthread_lock_fns_` 是一个结构体，包含了各种锁操作的函数指针（例如加锁、解锁等）。
- **`EVBASE_RELEASE_LOCK(base, lockvar)`**: 这是一个高层次的宏，用于释放 `event_base` 的锁。它通过调用 `EVLOCK_UNLOCK` 实现具体的解锁操作。
    
- **`EVLOCK_UNLOCK(lockvar, mode)`**: 这是一个底层宏，实际执行解锁操作。它使用 `evthread_lock_fns_.unlock` 函数指针来解锁。`mode` 参数通常用于指定解锁的详细模式，但在这段代码中，它被设置为 `0`。