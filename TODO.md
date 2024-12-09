
1. 在lu_event_operate_t中进行后端的选择以及封装;在event_base_new_with_config进行初始化
2. 需要全局变量static const struct eventop *eventops[]；配置支持的后端
3. 