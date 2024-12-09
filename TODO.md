
1. 在lu_event_operate_t中进行后端的选择以及封装;在event_base_new_with_config进行初始化
2. 需要全局变量static const struct eventop *eventops[]；配置支持的后端
3. 完善lu_event_base_t的初始化 以及lu_event_config_t 
	1. lu_event_base_new
	2. lu_event_config_new
	3. lu_event_base_new_with_config
	4. lu_event_config_free
4. 封装mm-inernal.h memory_manager.h memory_manager.c