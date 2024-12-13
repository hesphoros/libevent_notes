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
~~~
