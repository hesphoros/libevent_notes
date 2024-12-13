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
