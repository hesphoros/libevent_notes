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
