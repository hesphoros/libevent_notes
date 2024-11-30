# Create and free evconnlistener

# evconnlistener_new()

~~~c

struct evconnlistener * evconnlistener_new(struct event_base *base,
    evconnlistener_cb cb, void *ptr, unsigned flags, int backlog,
    evutil_socket_t fd);
struct evconnlistener*  evconnlistener_new_bind(
								struct event_base *base,
                                evconnlistener_cb cb, void *ptr, unsigned int flags,
                                int backlog, const struct sockaddr *sa, int socklen);

void evconnlistener_free(evconnlistener* evlistener);
    
~~~

两个evconnlistener_new*()函数都分配和返回一个新的连接监听器对象。连接监听器使用event_base来得知什么时候在给定的监听套接字上有新的TCP连接。新连接到达时，监听器调用你给出的回调函数。

两个函数中，base参数都是监听器用于监听连接的event_base。cb是收到新连接时要调用的回调函数；如果cb为NULL，则监听器是禁用的，直到设置了回调函数为止。ptr指针将传递给回调函数。flags参数控制回调函数的行为，下面会更详细论述。backlog是任何时刻网络栈允许处于还未接受状态的最大未决连接数。更多细节请查看系统的listen()函数文档。如果backlog是负的，libevent会试图挑选一个较好的值；如果为0，libevent认为已经对提供的套接字调用了listen()。

两个函数的不同在于如何建立监听套接字。evconnlistener_new()函数假定已经将套接字绑定到要监听的端口，然后通过fd传入这个套接字。如果要libevent分配和绑定套接字，可以调用evconnlistener_new_bind()，传输要绑定到的地址和地址长度。

要释放连接监听器，调用evconnlistener_free()。

## Recognizable signs

可以给evconnlistener_new()函数的flags参数传入一些标志。可以用或(OR)运算任意连接下述标志：

###  **<font color="#8064a2">LEV_OPT_LEAVE_SOCKETS_BLOCKING</font>**

	默认情况下，连接监听器接收新套接字后，会将其设置为非阻塞的，以便将其用于libevent。如果不想要这种行为，可以设置这个标志。

### **<font color="#8064a2">LEV_OPT_CLOSE_ON_FREE</font>**

	如果设置了这个选项，释放连接监听器会关闭底层套接字。

### **<font color="#8064a2">LEV_OPT_CLOSE_ON_EXEC</font>**

	如果设置了这个选项，连接监听器会为底层套接字设置close-on-exec标志。更多信息请查看fcntl和FD_CLOEXEC的平台文档。

### **<font color="#8064a2">LEV_OPT_REUSEABLE</font>**

	某些平台在默认情况下，关闭某监听套接字后，要过一会儿其他套接字才可以绑定到同一个端口。设置这个标志会让libevent标记套接字是可重用的，这样一旦关闭，可以立即打开其他套接字，在相同端口进行监听。

### **<font color="#8064a2">LEV_OPT_THREADSAFE</font>**

为监听器分配锁，这样就可以在多个线程中安全地使用了。这是2.0.8-rc的新功能。
## Connection listener callback
### evconnlistener_cb
~~~c
/**
   A callback that we invoke when a listener has a new connection.

   @param listener The evconnlistener
   @param fd The new file descriptor
   @param addr The source address of the connection
   @param socklen The length of addr
   @param user_arg the pointer passed to evconnlistener_new()
 */
typedef void (*evconnlistener_cb)(struct evconnlistener *, evutil_socket_t, struct sockaddr *,
								  int socklen, void *);
~~~


接收到新连接会调用提供的回调函数。listener参数是接收连接的连接监听器。sock参数是新接收的套接字。addr和len参数是接收连接的地址和地址长度。ptr是调用evconnlistener_new()时用户提供的指针。
## source code

### evconnlistener_new()

~~~c

struct evconnlistener *
evconnlistener_new(struct event_base *base,
    evconnlistener_cb cb, void *ptr, unsigned flags, int backlog,
    evutil_socket_t fd)
{
	struct evconnlistener_event *lev;

#ifdef _WIN32
	if (base && event_base_get_iocp_(base)) {
		const struct win32_extension_fns *ext =
			event_get_win32_extension_fns_();
		if (ext->AcceptEx && ext->GetAcceptExSockaddrs)
			return evconnlistener_new_async(base, cb, ptr, flags,
				backlog, fd);
	}
#endif

	if (backlog > 0) {
		if (listen(fd, backlog) < 0)
			return NULL;
	} else if (backlog < 0) {
		if (listen(fd, 128) < 0)
			return NULL;
	}

	lev = mm_calloc(1, sizeof(struct evconnlistener_event));
	if (!lev)
		return NULL;

	lev->base.ops = &evconnlistener_event_ops;
	lev->base.cb = cb;
	lev->base.user_data = ptr;
	lev->base.flags = flags;
	lev->base.refcnt = 1;

	lev->base.accept4_flags = 0;
	if (!(flags & LEV_OPT_LEAVE_SOCKETS_BLOCKING))
		lev->base.accept4_flags |= EVUTIL_SOCK_NONBLOCK;
	if (flags & LEV_OPT_CLOSE_ON_EXEC)
		lev->base.accept4_flags |= EVUTIL_SOCK_CLOEXEC;

	if (flags & LEV_OPT_THREADSAFE) {
		EVTHREAD_ALLOC_LOCK(lev->base.lock, EVTHREAD_LOCKTYPE_RECURSIVE);
	}

	event_assign(&lev->listener, base, fd, EV_READ|EV_PERSIST,
	    listener_read_cb, lev);

	if (!(flags & LEV_OPT_DISABLED))
	    evconnlistener_enable(&lev->base);

	return &lev->base;
}

~~~
### evconnlistener_new_bind()
~~~c

struct evconnlistener *
evconnlistener_new_bind(struct event_base *base, evconnlistener_cb cb,
    void *ptr, unsigned flags, int backlog, const struct sockaddr *sa,
    int socklen)
{
	struct evconnlistener *listener;
	evutil_socket_t fd;
	int on = 1;
	int family = sa ? sa->sa_family : AF_UNSPEC;
	int socktype = SOCK_STREAM | EVUTIL_SOCK_NONBLOCK;

	if (backlog == 0)
		return NULL;

	if (flags & LEV_OPT_CLOSE_ON_EXEC)
		socktype |= EVUTIL_SOCK_CLOEXEC;

	fd = evutil_socket_(family, socktype, 0);
	if (fd == -1)
		return NULL;

	if (setsockopt(fd, SOL_SOCKET, SO_KEEPALIVE, (void*)&on, sizeof(on))<0)
		goto err;

	if (flags & LEV_OPT_REUSEABLE) {
		if (evutil_make_listen_socket_reuseable(fd) < 0)
			goto err;
	}

	if (flags & LEV_OPT_REUSEABLE_PORT) {
		if (evutil_make_listen_socket_reuseable_port(fd) < 0)
			goto err;
	}

	if (flags & LEV_OPT_DEFERRED_ACCEPT) {
		if (evutil_make_tcp_listen_socket_deferred(fd) < 0)
			goto err;
	}

	if (flags & LEV_OPT_BIND_IPV6ONLY) {
		if (evutil_make_listen_socket_ipv6only(fd) < 0)
			goto err;
	}

	if (sa) {
		if (bind(fd, sa, socklen)<0)
			goto err;
	}

	listener = evconnlistener_new(base, cb, ptr, flags, backlog, fd);
	if (!listener)
		goto err;

	return listener;
err:
	evutil_closesocket(fd);
	return NULL;
}

~~~
### evconnlistener_free()
~~~c
void
evconnlistener_free(struct evconnlistener *lev)
{
	LOCK(lev);
	lev->cb = NULL;
	lev->errorcb = NULL;
	if (lev->ops->shutdown)
		lev->ops->shutdown(lev);
	listener_decref_and_unlock(lev);
}
~~~

# Enable and disable evconnlistener
## evconnlistener_disable() evconnlistener_enable()

~~~c
int evconnlistener_disable(evconnlistener* evlistener);
int evconnlistener_enable(evconnlistener* evlistener);
~~~

这两个函数暂时禁止或者重新允许监听新连接。

## source code

~~~c
int
evconnlistener_disable(struct evconnlistener *lev)
{
	int r;
	LOCK(lev);
	lev->enabled = 0;
	r = lev->ops->disable(lev);
	UNLOCK(lev);
	return r;
}

static int
event_listener_enable(struct evconnlistener *lev)
{
	struct evconnlistener_event *lev_e =
	    EVUTIL_UPCAST(lev, struct evconnlistener_event, base);
	return event_add(&lev_e->listener, NULL);
}

~~~

# Adjust the callback function of evconnlistener
~~~c
void
evconnlistener_set_cb(struct evconnlistener *lev,
    evconnlistener_cb cb, void *arg)
{
	int enable = 0;
	LOCK(lev);
	if (lev->enabled && !lev->cb)
		enable = 1;
	lev->cb = cb;
	lev->user_data = arg;
	if (enable)
		evconnlistener_enable(lev);
	UNLOCK(lev);
}
~~~

函数调整evconnlistener的回调函数和其参数。它是2.0.9-rc版本引入的。

# Check evconnlistener
## evconnlistener_get_fd() evconnlistener_get_fd()
~~~c
 evutil_socket_t evconnlistener_get_fd(struct evconnlistener *lev);
 struct event_base * evconnlistener_get_base(struct evconnlistener *lev);
~~~

~~~c

evutil_socket_t
evconnlistener_get_fd(struct evconnlistener *lev)
{
	evutil_socket_t fd;
	LOCK(lev);
	fd = lev->ops->getfd(lev);
	UNLOCK(lev);
	return fd;
}

struct event_base *
evconnlistener_get_base(struct evconnlistener *lev)
{
	struct event_base *base;
	LOCK(lev);
	base = lev->ops->getbase(lev);
	UNLOCK(lev);
	return base;
}
~~~
这些函数分别返回监听器关联的套接字和event_base。

evconnlistener_get_fd()函数首次出现在2.0.3-alpha版本。
## Detection Error

可以设置一个一旦监听器上的accept()调用失败就被调用的错误回调函数。对于一个不解决就会锁定进程的错误条件，这很重要。
## *evconnlistener_errorcb|  evconnlistener_set_error_cb()
~~~c
/**
   A callback that we invoke when a listener encounters a non-retriable error.

   @param listener The evconnlistener
   @param user_arg the pointer passed to evconnlistener_new()
 */
typedef void (*evconnlistener_errorcb)(struct evconnlistener *, void *);

void evconnlistener_set_error_cb(struct evconnlistener *lev,
    evconnlistener_errorcb errorcb)
{
	LOCK(lev);
	lev->errorcb = errorcb;
	UNLOCK(lev);
}
~~~


如果使用evconnlistener_set_error_cb()为监听器设置了错误回调函数，则监听器发生错误时回调函数就会被调用。第一个参数是监听器，第二个参数是调用evconnlistener_new()时传入的ptr。

这个函数在2.0.8-rc版本引入。


# Echo server

~~~c
#include <event2/listener.h>
#include <event2/bufferevent.h>
#include <event2/buffer.h>

#include <arpa/inet.h>

#include <string.h>
#include <stdlib.h>
#include <stdio.h>
#include <error.h>

static void echo_read_cb(struct bufferevent *bev, void *ctx) {
    struct evbuffer *input = bufferevent_get_input(bev);
    struct evbuffer *output = bufferevent_get_output(bev);
    evbuffer_add_buffer(output, input);

    /**Copy all the data from the input buffer to the output buffer */
    evbuffer_add_buffer(output, input);

}

static void echo_event_cb(struct bufferevent *bev, short events, void *ctx) {
    if (events & BEV_EVENT_ERROR) {
        perror("Error from bufferevent");
    }
    if (events & (BEV_EVENT_EOF | BEV_EVENT_ERROR)) {
        bufferevent_free(bev);
    }
}


static void accept_conn_cb(struct evconnlistener *listener, evutil_socket_t fd,struct sockaddr *address, int socklen, void *ctx) {
    
    /** We get a new connection ! Set up a bufferevent for it */
    struct event_base *base = evconnlistener_get_base(listener);
    struct bufferevent *bev = bufferevent_socket_new(base, fd, BEV_OPT_CLOSE_ON_FREE);
    bufferevent_setcb(bev, echo_read_cb, NULL, echo_event_cb, NULL);
    bufferevent_enable(bev, EV_READ | EV_WRITE);

}

static void accecpt_error_cb(struct evconnlistener *listener, void *ctx) {
    struct event_base *base = evconnlistener_get_base(listener);
    int err = EVUTIL_SOCKET_ERROR();
    fprintf(stderr, "Got an error %d (%s) on the listener. "
            "Shutting down.\n", err, evutil_socket_error_to_string(err));
    event_base_loopbreak(base);
}


int main(int argc, char **argv) {
    struct event_base *base;
    struct evconnlistener *listener;
    struct sockaddr_in sin;

    int port = 9876;
    if (argc > 1) {
        port = atoi(argv[1]);
    }

    if(port <= 0 || port > 65535) {
        fprintf(stderr, "Invalid port number, try in range 0-65535\n");
        return 1;
    }

    base = event_base_new();
    if (!base) {
        perror("Couldn't new event base");
        return 1;
    }

    /** Clear the sockaddr before using it ,in case there are extra
     * platform-specific fields that can mess us up. */
    memset(&sin, 0, sizeof(sin));
    
    /*This is an INET address*/
    sin.sin_family = AF_INET;

    /*Listen on 0.0.0.0, port 9876*/
    sin.sin_port = htons(port);

    listener = evconnlistener_new_bind(base, accept_conn_cb, NULL,LEV_OPT_CLOSE_ON_FREE | LEV_OPT_REUSEABLE, -1, (struct sockaddr *)&sin, sizeof(sin));

    if (!listener) {
        perror("Couldn't create listener");
        return 1;

    }

    evconnlistener_set_error_cb(listener, accecpt_error_cb);

    event_base_dispatch(base);
    return 0;
}
~~~


 