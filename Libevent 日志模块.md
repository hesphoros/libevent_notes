# **日志级别和日志记录函数**

代码定义了四个主要的日志级别：

- `EVENT_LOG_DEBUG`：调试信息
- `EVENT_LOG_MSG`：普通消息
- `EVENT_LOG_WARN`：警告信息
- `EVENT_LOG_ERR`：错误信息
~~~c
void event_log(int severity, const char *msg)
{
    if (log_fn) {
        // 如果设置了自定义的回调函数，调用回调函数
        log_fn(severity, msg);
    } else {
        // 否则，输出到标准错误流
        const char *severity_str;
        switch (severity) {
        case EVENT_LOG_DEBUG:
            severity_str = "debug";
            break;
        case EVENT_LOG_MSG:
            severity_str = "msg";
            break;
        case EVENT_LOG_WARN:
            severity_str = "warn";
            break;
        case EVENT_LOG_ERR:
            severity_str = "err";
            break;
        default:
            severity_str = "???";
            break;
        }
        (void)fprintf(stderr, "[%s] %s\n", severity_str, msg);
    }
}
~~~

- 如果 `log_fn` 被设置为用户自定义的回调函数，则调用该回调函数。
- 否则，默认将日志消息输出到标准错误流（`stderr`），并根据 `severity` 值打印不同的日志级别（`debug`, `msg`, `warn`, `err`）。
# 事件级别日志记录函数

`event_err` 和 `event_warn`：记录错误信息和警告信息，并调用 `event_exit` 来处理错误退出或清理。
~~~c
void event_err(int eval, const char *fmt, ...)
{
    va_list ap;
    va_start(ap, fmt);
    event_logv_(EVENT_LOG_ERR, strerror(errno), fmt, ap);  // 记录错误日志
    va_end(ap);
    event_exit(eval);  // 调用退出函数
}

void event_warn(const char *fmt, ...)
{
    va_list ap;
    va_start(ap, fmt);
    event_logv_(EVENT_LOG_WARN, strerror(errno), fmt, ap);  // 记录警告日志
    va_end(ap);
}

~~~
- `event_err` 用于记录错误，并在记录后调用 `event_exit` 函数退出程序。
- `event_warn` 用于记录警告信息，但不会退出程序。
# **事件退出处理**
~~~c
static void event_exit(int errcode)
{
    if (fatal_fn) {
        fatal_fn(errcode);  // 如果设置了自定义的 fatal 回调，调用它
        exit(errcode);  // 不应该到达这里
    } else if (errcode == EVENT_ERR_ABORT_) {
        abort();  // 如果是 `abort` 错误码，调用 `abort`
    } else {
        exit(errcode);  // 否则，正常退出
    }
}
~~~

- 如果设置了 `fatal_fn` 回调，则调用该回调，并传递错误码退出程序。
- 如果错误码是 `EVENT_ERR_ABORT_`，则调用 `abort()` 强制终止程序。
- 否则，程序正常退出。
# 变参日志函数
`event_logv_` 是一个内部函数，用于处理格式化的日志消息：
~~~c
void event_logv_(int severity, const char *errstr, const char *fmt, va_list ap)
{
    char buf[1024];
    size_t len;

    if (severity == EVENT_LOG_DEBUG && !event_debug_get_logging_mask_())
        return;

    if (fmt != NULL)
        evutil_vsnprintf(buf, sizeof(buf), fmt, ap);  // 格式化日志消息
    else
        buf[0] = '\0';

    if (errstr) {
        len = strlen(buf);
        if (len < sizeof(buf) - 3) {
            evutil_snprintf(buf + len, sizeof(buf) - len, ": %s", errstr);  // 添加错误描述
        }
    }

    event_log(severity, buf);  // 调用实际的日志记录函数
}

~~~
- 这个函数将变长参数 `fmt` 和 `ap` 传递给 `evutil_vsnprintf` 来格式化日志消息。
- 如果 `errstr` 非空，则将其附加到日志消息后，形成完整的错误描述。
# 调试日志控制
调试日志的启用由 `event_debug_logging_mask_` 控制，只有当 `event_debug_get_logging_mask_()` 返回非零值时，调试日志才会被记录。

~~~c
#ifdef EVENT_DEBUG_LOGGING_ENABLED
#ifdef USE_DEBUG
#define DEFAULT_MASK EVENT_DBG_ALL
#else
#define DEFAULT_MASK 0
#endif

EVENT2_EXPORT_SYMBOL ev_uint32_t event_debug_logging_mask_ = DEFAULT_MASK;
#endif /* EVENT_DEBUG_LOGGING_ENABLED */

~~~

- 如果定义了 `EVENT_DEBUG_LOGGING_ENABLED`，则根据编译时的宏（`USE_DEBUG`）决定默认的调试日志掩码。如果没有启用调试日志，则掩码为 0。
# **回调机制**

- `event_set_log_callback`：允许用户设置自定义的日志回调函数，用于替代默认的日志输出。
- `event_set_fatal_callback`：允许用户设置一个回调函数，在发生严重错误时调用，用于定制错误处理逻辑。


# some func

`static void event_exit(int errcode) EV_NORETURN;` 是 `libevent` 中的一个静态函数声明，目的是在出现严重错误时退出程序，并且它的 `EV_NORETURN` 属性指示编译器这个函数不会返回。让我们逐步分析这个函数的各个部分。

在 `libevent` 中，`static void event_log(int severity, const char *msg);` 是一个静态的函数声明，用于处理日志记录。它在 `libevent` 的代码中用于根据日志的严重程度输出相应的日志消息。这个函数并不直接暴露给外部，而是作为 `libevent` 内部的一部分用于处理不同级别的日志信息。

# 在event中暴露
~~~c
/** @name Log severities
 */
/**@{*/
#define EVENT_LOG_DEBUG 0
#define EVENT_LOG_MSG   1
#define EVENT_LOG_WARN  2
#define EVENT_LOG_ERR   3
/**@}*/

/* Obsolete names: these are deprecated, but older programs might use them.
 * They violate the reserved-identifier namespace. */
#define _EVENT_LOG_DEBUG EVENT_LOG_DEBUG
#define _EVENT_LOG_MSG EVENT_LOG_MSG
#define _EVENT_LOG_WARN EVENT_LOG_WARN
#define _EVENT_LOG_ERR EVENT_LOG_ERR

/**
  A callback function used to intercept Libevent's log messages.

  @see event_set_log_callback
 */
typedef void (*event_log_cb)(int severity, const char *msg);
/**
  Redirect Libevent's log messages.

  @param cb a function taking two arguments: an integer severity between
     EVENT_LOG_DEBUG and EVENT_LOG_ERR, and a string.  If cb is NULL,
	 then the default log is used.

  NOTE: The function you provide *must not* call any other libevent
  functionality.  Doing so can produce undefined behavior.
  */
EVENT2_EXPORT_SYMBOL
void event_set_log_callback(event_log_cb cb);

/**
   A function to be called if Libevent encounters a fatal internal error.
	致命错误回调函数
   @see event_set_fatal_callback
 */
typedef void (*event_fatal_cb)(int err);

/**
 Override Libevent's behavior in the event of a fatal internal error.

 By default, Libevent will call exit(1) if a programming error makes it
 impossible to continue correct operation.  This function allows you to supply
 another callback instead.  Note that if the function is ever invoked,
 something is wrong with your program, or with Libevent: any subsequent calls
 to Libevent may result in undefined behavior.

 Libevent will (almost) always log an EVENT_LOG_ERR message before calling
 this function; look at the last log message to see why Libevent has died.
 */
EVENT2_EXPORT_SYMBOL
void event_set_fatal_callback(event_fatal_cb cb);

#define EVENT_DBG_ALL 0xffffffffu
#define EVENT_DBG_NONE 0

/**
 Turn on debugging logs and have them sent to the default log handler.

 This is a global setting; if you are going to call it, you must call this
 before any calls that create an event-base.  You must call it before any
 multithreaded use of Libevent.

 Debug logs are verbose.

 @param which Controls which debug messages are turned on.  This option is
   unused for now; for forward compatibility, you must pass in the constant
   "EVENT_DBG_ALL" to turn debugging logs on, or "EVENT_DBG_NONE" to turn
   debugging logs off.
 */
EVENT2_EXPORT_SYMBOL
void event_enable_debug_logging(ev_uint32_t which);
~~~

~~~c

void
event_logv_(int severity, const char *errstr, const char *fmt, va_list ap)
{
	char buf[1024];
	size_t len;
	//`!event_debug_get_logging_mask_()` 是一个逻辑“非”操作，它的作用是检查当前的调试日志掩码是否为(LU_EVENT_DBG_NONE)。
	if (severity == EVENT_LOG_DEBUG && !event_debug_get_logging_mask_())
		return;

	if (fmt != NULL)
		//格式化字符串
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
~~~