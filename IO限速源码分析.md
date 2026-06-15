
# 数据结构
![816](images/QQ_1781535532673.png)
write_maximum最大写速率；
msec_per_tick时间；
tick内最大写入数据量=write_maximum * msec_per_tick；
tick内最大读入数据量=read_maximum * msec_per_tick;

# 关键流程

## 初始化流程
![463](images/QQ_1781535587960.png)
## 用户设置流程
![458](images/QQ_1781535620313.png)
## 更新发送和接收数据的量
![712](images/QQ_1781535656902.png)
# 读速录限制
![819](images/QQ_1781535688544.png)

# 写速率限制

![828](images/QQ_1781535711295.png)