## Create epoll
~~~c
/**
 * @param size 告诉内核监听的数目
 *
 * @returns 返回一个epoll句柄（即一个文件描述符）
 */
int epoll_create(int size);
~~~

~~~c
int epfd = epoll_create(1000);
~~~

![epoll](images/Pasted%20image%2020240913182459.png)
## Control epoll
~~~c
/**
 * @param epfd 用epoll_create所创建的epoll句柄
 * @param op 表示对epoll监控描述符控制的动作
 *
 * EPOLL_CTL_ADD(注册新的fd到epfd)
 * EPOLL_CTL_MOD(修改已经注册的fd的监听事件)
 * EPOLL_CTL_DEL(epfd删除一个fd)
 *
 * @param fd 需要监听的文件描述符
 * @param event 告诉内核需要监听的事件
 *
 * @returns 成功返回0，失败返回-1, errno查看错误信息
 */
int epoll_ctl(int epfd, int op, int fd, 
            struct epoll_event *event);



struct epoll_event {
	 __uint32_t events; /* epoll 事件 */
	 epoll_data_t data; /* 用户传递的数据 */
}

/*
 * events : {EPOLLIN, EPOLLOUT, EPOLLPRI, 
            EPOLLHUP, EPOLLET, EPOLLONESHOT}
 */

typedef union epoll_data {
    void*    ptr;
    int       fd;
    uint32_t u32;
    uint64_t u64;
} epoll_data_t;

~~~

~~~c
struct epoll_event new_event;

new_event.events = EPOLLIN | EPOLLOUT;
new_event.data.fd = 5;

epoll_ctl(epfd, EPOLL_CTL_ADD, 5, &new_event);
~~~

![](images/Pasted%20image%2020240913182621.png)
# epoll framework
~~~c
//创建 epoll
int epfd = epoll_crete(1000);

//将 listen_fd 添加进 epoll 中
epoll_ctl(epfd, EPOLL_CTL_ADD, listen_fd,&listen_event);

while (1) {
    //阻塞等待 epoll 中 的fd 触发
    int active_cnt = epoll_wait(epfd, events, 1000, -1);

    for (i = 0 ; i < active_cnt; i++) {
        if (evnets[i].data.fd == listen_fd) {
            //accept. 并且将新accept 的fd 加进epoll中.
        }
        else if (events[i].events & EPOLLIN) {
            //对此fd 进行读操作
        }
        else if (events[i].events & EPOLLOUT) {
            //对此fd 进行写操作
        }
    }
}
~~~

# A simple framework 
## common.cc
~~~c
﻿#include "globals.h"

double          current_dtime;
time_t          sys_curtime;//long int
struct timeval  current_time;//time template


int             Biggest_FD      =   1024;  /*默认的最大文件描述符数量 1024*/
static int      MAX_POLL_TIME   =   1000;	/* see also comm_quick_poll_required() */
fde             *fd_table       =   NULL;	        

time_t getCurrentTime(void)
{
    /*current_time是一个timeval结构体变量，用于存储当前时间。
    current_dtime是一个double类型的变量，用于存储当前时间的秒数。
    sys_curtime是一个time_t类型的变量，用于存储当前时间的秒数。*/

    // 获取当前时间
    gettimeofday(&current_time, NULL);
    /*
        struct timeval
    {
        __time_t tv_sec;		/// Seconds.  
        __suseconds_t tv_usec;	// Microseconds.  
    }
    */
    //将timeval结构体转换为double类型
    current_dtime = (double) current_time.tv_sec +(double) current_time.tv_usec / 1000000.0;
    return sys_curtime = current_time.tv_sec;//返回当前时间
}

// 清理fd
void comm_close(int fd)
{
	assert(fd>0);//确保fd大于0
	    fde *F = &fd_table[fd];
	if(F) memset((void *)F,'\0',sizeof(fde));
	epollSetEvents(fd, 0, 0);
	close(fd);
}

//初始化异步事件处理框架epoll
void comm_init(int max_fd)
{
	if(max_fd > 0 ) Biggest_FD = max_fd;
	fd_table = calloc(Biggest_FD, sizeof(fde));
    do_epoll_init(Biggest_FD);
}


//清理资源
void comm_select_shutdown(void)
{
    do_epoll_shutdown();
	if(fd_table) free(fd_table);
}


//static int comm_select_handled;


inline void comm_call_handlers(int fd, int read_event, int write_event)
{
    fde *F = &fd_table[fd];
    
    debug(5, 8) ("comm_call_handlers(): got fd=%d read_event=%x write_event=%x F->read_handler=%p F->write_handler=%p\n",fd, read_event, write_event, F->read_handler, F->write_handler);
    
    if (F->read_handler && read_event) {
	    PF *hdl = F->read_handler;
	    void *hdl_data = F->read_data;
	    /* If the descriptor is meant to be deferred, don't handle */

		debug(5, 8) ("comm_call_handlers(): Calling read handler on fd=%d\n", fd);
		//commUpdateReadHandler(fd, NULL, NULL);
		hdl(fd, hdl_data);
    }
	
    if (F->write_handler && write_event) {
	
	    PF *hdl = F->write_handler;
	    void *hdl_data = F->write_data;
	
	    //commUpdateWriteHandler(fd, NULL, NULL);
	    hdl(fd, hdl_data);
    }
}

//设置超时处理函数
int commSetTimeout(int fd, int timeout, PF * handler, void *data)
{
    fde *F;
    debug(5, 3) ("commSetTimeout: FD %d timeout %d\n", fd, timeout);
    assert(fd >= 0);
    assert(fd < Biggest_FD);
    F = &fd_table[fd];

	
    if (timeout < 0) {
        F->timeout_handler = NULL;
        F->timeout_data = NULL;
        return F->timeout = 0;
    }
    assert(handler || F->timeout_handler);
    if (handler || data) {
        F->timeout_handler  =   handler;
        F->timeout_data     =   data;
    }
    return F->timeout       =   sys_curtime + (time_t) timeout;
}

//注册读事件
void commUpdateReadHandler(int fd, PF * handler, void *data)
{
    fd_table[fd].read_handler = handler;
    fd_table[fd].read_data = data;
    
    epollSetEvents(fd,1,0); 
}
//注册写事件
void commUpdateWriteHandler(int fd, PF * handler, void *data)
{
    fd_table[fd].write_handler = handler;
    fd_table[fd].write_data = data;
	
    epollSetEvents(fd,0,1); 
}


//超时 （
static void checkTimeouts(void)
{
    int fd;
    fde *F = NULL;
    PF *callback;

    for (fd = 0; fd <= Biggest_FD; fd++) {
        F = &fd_table[fd];
        /*if (!F->flags.open)
            continue;
        */
        
        if (F->timeout == 0)
            continue;
        if (F->timeout > sys_curtime)
            continue;
        debug(5, 5) ("checkTimeouts: FD %d Expired\n", fd);
        
        if (F->timeout_handler) {
            debug(5, 5) ("checkTimeouts: FD %d: Call timeout handler\n", fd);
            callback = F->timeout_handler;
            F->timeout_handler = NULL;
            //调用超时处理函数

            callback(fd, F->timeout_data);
        } else 
        {
            debug(5, 5) ("checkTimeouts: FD %d: Forcing comm_close()\n", fd);
            comm_close(fd);
        }
    }
}


int comm_select(int msec) //msec 毫秒
{

    static double   last_timeout    = 0.0;//上次操作的时间  
    double          start           = current_dtime;
    int             rc;

    debug(5, 3) ("comm_select: timeout %d\n", msec);

    if (msec > MAX_POLL_TIME)
	    msec = MAX_POLL_TIME;


    //statCounter.select_loops++;
    /* Check timeouts once per second */
    if (last_timeout + 0.999 < current_dtime) {
	    last_timeout = current_dtime;
	    checkTimeouts();//处理超时事件
    } else {
	    int max_timeout = (last_timeout + 1.0 - current_dtime) * 1000;//set max timeout
	    if (max_timeout < msec)
	        msec = max_timeout;
    }
    //comm_select_handled = 0;

    rc = do_epoll_select(msec);


    getCurrentTime();
    //statCounter.select_time += (current_dtime - start);

    if (rc == COMM_TIMEOUT)
	debug(5, 8) ("comm_select: time out\n");

    return rc;
}

//获取当前系统的错误信息 转为字符串
const char * xstrerror(void)
{
    static char xstrerror_buf[BUFSIZ];
    const char *errmsg;

    errmsg = strerror(errno);

    if (!errmsg || !*errmsg)
	errmsg = "Unknown error";

    snprintf(xstrerror_buf, BUFSIZ, "(%d) %s", errno, errmsg);
    return xstrerror_buf;
}

//忽略一些错误
int ignoreErrno(int ierrno)
{
    switch (ierrno) {
    case EINPROGRESS:
    case EWOULDBLOCK:
#if EAGAIN != EWOULDBLOCK
    case EAGAIN:
#endif
    case EALREADY:
    case EINTR:
#ifdef ERESTART
    case ERESTART:
#endif
	return 1;
    default:
	return 0;
    }
    /* NOTREACHED */
}



~~~

## comm_epoll.cc
~~~c
﻿#include "globals.h"
#include <sys/epoll.h>

#define MAX_EVENTS		256	/* 一次处理的最大的事件 */

/* epoll structs */
static 	int 			kdpfd;//epoll的fd
static 	struct 			epoll_event events[MAX_EVENTS];//同时处理最大的事件数
static 	int 			epoll_fds = 0;
static 	unsigned 		*epoll_state;	/* 保存每个epoll 的事件状态 */

static const char * epolltype_atoi(int x)//宏转换为字符串而已
{
    switch (x) {

    case EPOLL_CTL_ADD:
	return "EPOLL_CTL_ADD";

    case EPOLL_CTL_DEL:
	return "EPOLL_CTL_DEL";

    case EPOLL_CTL_MOD:
	return "EPOLL_CTL_MOD";

    default:
	return "UNKNOWN_EPOLLCTL_OP";
    }
}

void do_epoll_init(int max_fd)//初始化epoll mad_fd 最大的并发数量
{

    
    kdpfd = epoll_create(max_fd);
    if (kdpfd < 0)
	  fprintf(stderr,"do_epoll_init: epoll_create(): %s\n", xstrerror());
    //fd_open(kdpfd, FD_UNKNOWN, "epoll ctl");
    //commSetCloseOnExec(kdpfd);

    epoll_state = calloc(max_fd, sizeof(*epoll_state));//分配max_fd个unsigned 空间 （后续直接使用下表进行访问fd）
}


void do_epoll_shutdown()
{
    
    close(kdpfd);
    kdpfd = -1;
    safe_free(epoll_state);
}

//设置事件
void epollSetEvents(int fd, int need_read, int need_write)
{
    int epoll_ctl_type = 0;
    struct epoll_event ev;

    assert(fd >= 0);
    debug(5, 8) ("commSetEvents(fd=%d)\n", fd);

	memset(&ev, 0, sizeof(ev));
    
    ev.events = 0;
    ev.data.fd = fd;

    if (need_read)
		ev.events |= EPOLLIN;

    if (need_write)
		ev.events |= EPOLLOUT;

    if (ev.events)
		ev.events |= EPOLLHUP | EPOLLERR;

    if (ev.events != epoll_state[fd]) {
		/* If the struct is already in epoll MOD or DEL, else ADD */
		if (!ev.events) {
			epoll_ctl_type = EPOLL_CTL_DEL;
		}
		else if (epoll_state[fd]) {
			epoll_ctl_type = EPOLL_CTL_MOD;
		} else {
			epoll_ctl_type = EPOLL_CTL_ADD;
		}

		/* Update the state */
		epoll_state[fd] = ev.events;

		if (epoll_ctl(kdpfd, epoll_ctl_type, fd, &ev) < 0) {
			debug(5, 1) ("commSetEvents: epoll_ctl(%s): failed on fd=%d: %s\n",
			epolltype_atoi(epoll_ctl_type), fd, xstrerror());
		}
		switch (epoll_ctl_type) {
		case EPOLL_CTL_ADD:
			epoll_fds++;
			break;
		case EPOLL_CTL_DEL:
			epoll_fds--;
			break;
		default:
			break;
		}
    }
}

//
int do_epoll_select(int msec)
{
    int i;
    int num;
    int fd;
    struct epoll_event *cevents;

    /*if (epoll_fds == 0) {
	assert(shutting_down);
	return COMM_SHUTDOWN;
    }
    statCounter.syscalls.polls++;
    */
    num = epoll_wait(kdpfd, events, MAX_EVENTS, msec);
    if (num < 0) {
		getCurrentTime();
		if (ignoreErrno(errno))
			return COMM_OK;

		debug(5, 1) ("comm_select: epoll failure: %s\n", xstrerror());
		return COMM_ERROR;
    }
    //statHistCount(&statCounter.select_fds_hist, num);

    if (num == 0)
		return COMM_TIMEOUT;

    for (i = 0, cevents = events; i < num; i++, cevents++) {
		fd = cevents->data.fd;
		//是否具有读写事件
		comm_call_handlers(fd, cevents->events & ~EPOLLOUT, cevents->events & ~EPOLLIN);
    }

    return COMM_OK;
}

~~~
## global.hh
~~~c
﻿#ifndef GLOBALS_H
#define GLOBALS_H

#include <sys/time.h>
#include <sys/resource.h>	
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <sys/stat.h>

#include <unistd.h>
#include <stdlib.h>
#include <stdio.h>
#include <sys/types.h>
#include <errno.h>
#include <fcntl.h>
#include <assert.h>

#define FD_DESC_SZ		64

#define COMM_OK		        (0)
#define COMM_ERROR	        (-1)
#define COMM_NOMESSAGE	    (-3)
#define COMM_TIMEOUT	    (-4)
#define COMM_SHUTDOWN	    (-5)
#define COMM_INPROGRESS     (-6)
#define COMM_ERR_CONNECT    (-7)
#define COMM_ERR_DNS        (-8)
#define COMM_ERR_CLOSING    (-9)


//调试相关
#define DEBUG_LEVEL     0
#define DEBUG_ONLY      8
#define debug(m, n)     if( m >= DEBUG_LEVEL && n <= DEBUG_ONLY  ) printf //一个判断日志级别的宏函数
	
#define safe_free(x)	if (x) { free(x); x = NULL; }

typedef unsigned short u_short;
typedef void PF(int, void *);

typedef struct _fde 
{
    unsigned int        type;//类型
    u_short             local_port;//本地端口
    u_short             remote_port;//远程端口
    struct in_addr      local_addr;//本地地址 
    char                ipaddr[16];		/* dotted decimal address of peer */      
    PF                  *read_handler;//读事件处理函数
    void                *read_data;//读事件处理函数数据
    PF                  *write_handler;//写事件处理函数
    void                *write_data;//写事件处理函数数据
    PF                  *timeout_handler;//超时事件处理函数
    time_t              timeout;//超时时间
    void                *timeout_data;//超时事件处理函数数
}fde;

//typedef struct _fde     fde;

extern fde              *fd_table;		//fd = 1.fd_table[1]
extern int              Biggest_FD;		

/*系统时间相关,设置成全局变量，供所有模块使用*/
extern struct timeval   current_time;
extern double           current_dtime;
extern time_t           sys_curtime;


/* epoll 相关接口实现 */
extern void             do_epoll_init           (int max_fd);
extern void             do_epoll_shutdown       ();
extern void             epollSetEvents          (int fd, int need_read, int need_write);
extern int              do_epoll_select         (int msec);

/*框架外围接口*/
extern int              comm_select             (int msec);
extern inline void      comm_call_handlers      (int fd, int read_event, int write_event);
void                    commUpdateReadHandler   (int fd, PF * handler, void *data);
void                    commUpdateWriteHandler  (int fd, PF * handler, void *data);



extern const char       *xstrerror              (void);
int                     ignoreErrno             (int ierrno);

#endif /* GLOBALS_H */

~~~

