<event2/util.h>定义了很多在实现可移植应用时有用的函数，libevent内部也使用这些类型和函数。

# Base type
## <font color="#4bacc6">evutil_socket_t</font>
在除Windows之外的大多数地方，套接字是个整数，操作系统按照数值次序进行处理。然而，使用Windows套接字API时，socket具有类型SOCKET，它实际上是个类似指针的句柄，收到这个句柄的次序是未定义的。在Windows中，libevent定义evutil_socket_t类型为整型指针，可以处理socket()或者accept()的输出，而没有指针截断的风险。

~~~c
/**

 * A type wide enough to hold the output of "socket()" or "accept()".  On

 * Windows, this is an intptr_t; elsewhere, it is an int. */

#ifdef _WIN32

#define evutil_socket_t intptr_t

#else

#define evutil_socket_t int

#endif
~~~


# Standard int types

落后于21世纪的C系统常常没有实现C99标准规定的stdint.h头文件。考虑到这种情况，
libevent定义了来自于stdint.h的、位宽度确定（bit-width-specific）的整数类型：


| Type                                     | Width | Signed | Maximum                                    | Minimum      |
| ---------------------------------------- | ----- | ------ | ------------------------------------------ | ------------ |
| <font color="#4bacc6">ev_uint64_t</font> | 64    | No     | <font color="#8064a2">EV_UINT64_MAX</font> | 0            |
| <font color="#4bacc6">ev_int64_t</font>  | 64    | Yes    | <font color="#8064a2">EV_INT64_MAX</font>  | EV_INT64_MIN |
| <font color="#4bacc6">ev_uint32_t</font> | 32    | No     | <font color="#8064a2">EV_UINT32_MAX</font> | 0            |
| <font color="#4bacc6">ev_int32_t</font>  | 32    | Yes    | <font color="#8064a2">EV_INT32_MAX</font>  | EV_INT32_MIN |
| <font color="#4bacc6">ev_uint16_t</font> | 16    | No     | <font color="#8064a2">EV_UINT16_MAX</font> | 0            |
| <font color="#4bacc6">ev_int16_t</font>  | 16    | Yes    | <font color="#8064a2">EV_INT16_MAX</font>  | EV_INT16_MIN |
| <font color="#4bacc6">ev_uint8_t</font>  | 8     | No     | <font color="#8064a2">EV_UINT8_MAX</font>  | 0            |
| <font color="#4bacc6">ev_int8_t</font>   | 8     | Yes    | <font color="#8064a2">EV_INT8_MAX</font>   | EV_INT8_MIN  |
|                                          |       |        |                                            |              |
跟C99标准一样，这些类型都有明确的位宽度。

这些类型由1.4.0-alpha版本引入。MAX/MIN常量首次出现在2.0.4-alpha版本。
~~~c
  

/**

 * @name Standard integer types.

 *

 * Integer type definitions for types that are supposed to be defined in the

 * C99-specified stdint.h.  Shamefully, some platforms do not include

 * stdint.h, so we need to replace it.  (If you are on a platform like this,

 * your C headers are now over 10 years out of date.  You should bug them to

 * do something about this.)

 *

 * We define:

 *

 * <dl>

 *   <dt>ev_uint64_t, ev_uint32_t, ev_uint16_t, ev_uint8_t</dt>

 *      <dd>unsigned integer types of exactly 64, 32, 16, and 8 bits

 *          respectively.</dd>

 *    <dt>ev_int64_t, ev_int32_t, ev_int16_t, ev_int8_t</dt>

 *      <dd>signed integer types of exactly 64, 32, 16, and 8 bits

 *          respectively.</dd>

 *    <dt>ev_uintptr_t, ev_intptr_t</dt>

 *      <dd>unsigned/signed integers large enough

 *      to hold a pointer without loss of bits.</dd>

 *    <dt>ev_ssize_t</dt>

 *      <dd>A signed type of the same size as size_t</dd>

 *    <dt>ev_off_t</dt>

 *      <dd>A signed type typically used to represent offsets within a

 *      (potentially large) file</dd>

 *

 * @{

 */

#ifdef EVENT__HAVE_UINT64_T

#define ev_uint64_t uint64_t

#define ev_int64_t int64_t

#elif defined(_WIN32)

#define ev_uint64_t unsigned __int64

#define ev_int64_t signed __int64

#elif EVENT__SIZEOF_LONG_LONG == 8

#define ev_uint64_t unsigned long long

#define ev_int64_t long long

#elif EVENT__SIZEOF_LONG == 8

#define ev_uint64_t unsigned long

#define ev_int64_t long

#elif defined(EVENT_IN_DOXYGEN_)

#define ev_uint64_t ...

#define ev_int64_t ...

#else

#error "No way to define ev_uint64_t"

#endif

  

#ifdef EVENT__HAVE_UINT32_T

#define ev_uint32_t uint32_t

#define ev_int32_t int32_t

#elif defined(_WIN32)

#define ev_uint32_t unsigned int

#define ev_int32_t signed int

#elif EVENT__SIZEOF_LONG == 4

#define ev_uint32_t unsigned long

#define ev_int32_t signed long

#elif EVENT__SIZEOF_INT == 4

#define ev_uint32_t unsigned int

#define ev_int32_t signed int

#elif defined(EVENT_IN_DOXYGEN_)

#define ev_uint32_t ...

#define ev_int32_t ...

#else

#error "No way to define ev_uint32_t"

#endif

  

#ifdef EVENT__HAVE_UINT16_T

#define ev_uint16_t uint16_t

#define ev_int16_t  int16_t

#elif defined(_WIN32)

#define ev_uint16_t unsigned short

#define ev_int16_t  signed short

#elif EVENT__SIZEOF_INT == 2

#define ev_uint16_t unsigned int

#define ev_int16_t  signed int

#elif EVENT__SIZEOF_SHORT == 2

#define ev_uint16_t unsigned short

#define ev_int16_t  signed short

#elif defined(EVENT_IN_DOXYGEN_)

#define ev_uint16_t ...

#define ev_int16_t ...

#else

#error "No way to define ev_uint16_t"

#endif

  

#ifdef EVENT__HAVE_UINT8_T

#define ev_uint8_t uint8_t

#define ev_int8_t int8_t

#elif defined(EVENT_IN_DOXYGEN_)

#define ev_uint8_t ...

#define ev_int8_t ...

#else

#define ev_uint8_t unsigned char

#define ev_int8_t signed char

#endif

  

#ifdef EVENT__HAVE_UINTPTR_T

#define ev_uintptr_t uintptr_t

#define ev_intptr_t intptr_t

#elif EVENT__SIZEOF_VOID_P <= 4

#define ev_uintptr_t ev_uint32_t

#define ev_intptr_t ev_int32_t

#elif EVENT__SIZEOF_VOID_P <= 8

#define ev_uintptr_t ev_uint64_t

#define ev_intptr_t ev_int64_t

#elif defined(EVENT_IN_DOXYGEN_)

#define ev_uintptr_t ...

#define ev_intptr_t ...

#else

#error "No way to define ev_uintptr_t"

#endif
~~~

## Definition of maximum and minimum values
~~~c
#ifndef EVENT__HAVE_STDINT_H

#define EV_UINT64_MAX ((((ev_uint64_t)0xffffffffUL) << 32) | 0xffffffffUL)

#define EV_INT64_MAX  ((((ev_int64_t) 0x7fffffffL) << 32) | 0xffffffffL)

#define EV_INT64_MIN  ((-EV_INT64_MAX) - 1)

#define EV_UINT32_MAX ((ev_uint32_t)0xffffffffUL)

#define EV_INT32_MAX  ((ev_int32_t) 0x7fffffffL)

#define EV_INT32_MIN  ((-EV_INT32_MAX) - 1)

#define EV_UINT16_MAX ((ev_uint16_t)0xffffUL)

#define EV_INT16_MAX  ((ev_int16_t) 0x7fffL)

#define EV_INT16_MIN  ((-EV_INT16_MAX) - 1)

#define EV_UINT8_MAX  255

#define EV_INT8_MAX   127

#define EV_INT8_MIN   ((-EV_INT8_MAX) - 1)

#else

#define EV_UINT64_MAX UINT64_MAX

#define EV_INT64_MAX  INT64_MAX

#define EV_INT64_MIN  INT64_MIN

#define EV_UINT32_MAX UINT32_MAX

#define EV_INT32_MAX  INT32_MAX

#define EV_INT32_MIN  INT32_MIN

#define EV_UINT16_MAX UINT16_MAX

#define EV_INT16_MIN  INT16_MIN

#define EV_INT16_MAX  INT16_MAX

#define EV_UINT8_MAX  UINT8_MAX

#define EV_INT8_MAX   INT8_MAX

#define EV_INT8_MIN   INT8_MIN

/** @} */

#endif
~~~


# Various compatibility types

在有ssize_t（有符号的size_t）类型的平台上，ev_ssize_t定义为ssize_t；而在没有的平台上，则定义为某合理的默认类型。
ev_ssize_t类型的最大可能值是EV_SSIZE_MAX；最小可能值是EV_SSIZE_MIN。（在平台没有定义SIZE_MAX的时候，size_t类型的最大可能值是EV_SIZE_MAX）.

ev_off_t用于代表文件或者内存块中的偏移量。在有合理off_t类型定义的平台，它被定义为off_t；在Windows上则定义为ev_int64_t。

某些套接字API定义了socklen_t长度类型，有些则没有定义。在有这个类型定义的平台中，ev_socklen_t定义为socklen_t，在没有的平台上则定义为合理的默认类型。

ev_intptr_t是一个有符号整数类型，足够容纳指针类型而不会产生截断；而ev_uintptr_t则是相应的无符号类型。


ev_ssize_t类型由2.0.2-alpha版本加入。ev_socklen_t类型由2.0.3-alpha版本加入。ev_intptr_t与ev_uintptr_t类型，以及EV_SSIZE_MAX/MIN宏定义由2.0.4-alpha版本加入。ev_off_t类型首次出现在2.0.9-rc版本。

# Timer portable functions
不是每个平台都定义了标准timeval操作函数，所以libevent也提供了自己的实现。

## evutil_timeradd() evutil_timersub()
~~~
#define evutil_timeradd(tvp, uvp, vvp) timeradd((tvp), (uvp), (vvp))

#define evutil_timersub(tvp, uvp, vvp) timersub((tvp), (uvp), (vvp))
~~~


~~~c
#ifdef EVENT__HAVE_TIMERADD

#define evutil_timeradd(tvp, uvp, vvp) timeradd((tvp), (uvp), (vvp))

#define evutil_timersub(tvp, uvp, vvp) timersub((tvp), (uvp), (vvp))

#else

#define evutil_timeradd(tvp, uvp, vvp)                  \

    do {                                \

        (vvp)->tv_sec = (tvp)->tv_sec + (uvp)->tv_sec;      \

        (vvp)->tv_usec = (tvp)->tv_usec + (uvp)->tv_usec;       \

        if ((vvp)->tv_usec >= 1000000) {            \

            (vvp)->tv_sec++;                \

            (vvp)->tv_usec -= 1000000;          \

        }                           \

    } while (0)

#define evutil_timersub(tvp, uvp, vvp)                  \

    do {                                \

        (vvp)->tv_sec = (tvp)->tv_sec - (uvp)->tv_sec;      \

        (vvp)->tv_usec = (tvp)->tv_usec - (uvp)->tv_usec;   \

        if ((vvp)->tv_usec < 0) {               \

            (vvp)->tv_sec--;                \

            (vvp)->tv_usec += 1000000;          \

        }                           \

    } while (0)

#endif /* !EVENT__HAVE_TIMERADD */
~~~

这些宏分别对前两个参数进行加或者减运算，将结果存放到第三个参数中。
~~~c
#ifdef __USE_MISC

/* Convenience macros for operations on timevals.

   NOTE: `timercmp' does not work for >= or <=.  */

# define timerisset(tvp)   ((tvp)->tv_sec || (tvp)->tv_usec)

# define timerclear(tvp)   ((tvp)->tv_sec = (tvp)->tv_usec = 0)

# define timercmp(a, b, CMP)                       \

  (((a)->tv_sec == (b)->tv_sec)                    \

   ? ((a)->tv_usec CMP (b)->tv_usec)                     \

   : ((a)->tv_sec CMP (b)->tv_sec))

# define timeradd(a, b, result)                       \

  do {                                 \

    (result)->tv_sec = (a)->tv_sec + (b)->tv_sec;              \

    (result)->tv_usec = (a)->tv_usec + (b)->tv_usec;              \

    if ((result)->tv_usec >= 1000000)                    \

      {                                \

   ++(result)->tv_sec;                       \

   (result)->tv_usec -= 1000000;                   \

      }                                \

  } while (0)

# define timersub(a, b, result)                       \

  do {                                 \

    (result)->tv_sec = (a)->tv_sec - (b)->tv_sec;              \

    (result)->tv_usec = (a)->tv_usec - (b)->tv_usec;              \

    if ((result)->tv_usec < 0) {                   \

      --(result)->tv_sec;                       \

      (result)->tv_usec += 1000000;                   \

    }                               \

  } while (0)

#endif   /* Misc.  */
~~~

使用do{}while(0) 包装宏
## evutil_timerisse()
~~~c
# define timerisset(tvp)   ((tvp)->tv_sec || (tvp)->tv_usec)

# define timerclear(tvp)   ((tvp)->tv_sec = (tvp)->tv_usec = 0)
~~~

清除timeval会将其值设置为0。evutil_timerisset宏检查timeval是否已经设置，如果已经设置为非零值，返回ture，否则返回false。

## timercmp()
~~~c
# define timercmp(a, b, CMP)                       \

  (((a)->tv_sec == (b)->tv_sec)                    \

   ? ((a)->tv_usec CMP (b)->tv_usec)                     \

   : ((a)->tv_sec CMP (b)->tv_sec))

~~~
evutil_timercmp宏比较两个timeval，如果其关系满足cmp关系运算符，返回true。比如说，evutil_timercmp(t1,t2,<=)的意思是“是否t1<=t2？”。注意：与某些操作系统版本不同的是，libevent的时间比较支持所有C关系运算符（也就是<、>、\==、!=、<=和>=）。

## evutil_gettimeofday
~~~
#define evutil_gettimeofday(tv, tz) gettimeofday((tv), (tz))
~~~

~~~c
extern int gettimeofday (struct timeval *__restrict __tv,
	void *__restrict __tz) __THROW __nonnull ((1));
~~~
evutil_gettimeofdy（）函数设置tv为当前时间，tz参数未使用。           
~~~c
#include <bits/types/struct_timeval.h>

#include <event2/event.h>  

#include <event2/util.h>

#include <ctime>

#include <unistd.h>

  

void func()

{

    struct timeval tv1,tv2,tv3;

    tv1.tv_sec = 5; tv1.tv_usec = 500*1000;

    evutil_gettimeofday(&tv2,NULL);

    evutil_timeradd(&tv1,&tv2,&tv3);

    if(evutil_timercmp(&tv1,&tv2,==))

        puts("tv1 == tv2");

    if(evutil_timercmp(&tv1,&tv2,>=))

        puts("tv1 >= tv2");

    if(evutil_timercmp(&tv1,&tv2,<))

        puts("tv1 < tv2");

}
~~~

除evutil_gettimeofday（）由2.0版本引入外，这些函数由1.4.0-beta版本引入。

注意：在1.4.4之前的版本中使用<=或者>=是不安全的。

`evutil_gettimeofday()` 是 `libevent` 库中的一个函数，用于获取当前的时间。它的功能是填充一个 `struct timeval` 结构体，该结构体包含当前的秒数和微秒数。这个函数通常用于需要精确时间戳或时间间隔计算的场景。

# Socket API compatibility

本节由于历史原因而存在：Windows从来没有以良好兼容的方式实现Berkeley套接字API。

~~~c
int evutil_closesocket(evutil_socket_t sock)
~~~

这个接口用于关闭套接字。在Unix中，它是close()的别名；在Windows中，它调用closesocket()。（在Windows中不能将close()用于套接字，也没有其他系统定义了closesocket()）

evutil_closesocket()函数在2.0.5-alpha版本引入。在此之前，需要使用<font color="#8064a2">EVUTIL_CLOSESOCKE</font>T宏。

~~~c
int evutil_closesocket(evutil_socket_t sock)

{

#ifndef _WIN32

    return close(sock);

#else

    return closesocket(sock);

#endif

}

#define EVUTIL_CLOSESOCKET(s) evutil_closesocket(s)
~~~

## EVUTIL_*
~~~C
EVENT2_EXPORT_SYMBOL

int evutil_make_tcp_listen_socket_deferred(evutil_socket_t sock);

  

#ifdef _WIN32

/** Return the most recent socket error.  Not idempotent on all platforms. */

#define EVUTIL_SOCKET_ERROR() WSAGetLastError()

/** Replace the most recent socket error with errcode */

#define EVUTIL_SET_SOCKET_ERROR(errcode)        \

    do { WSASetLastError(errcode); } while (0)

/** Return the most recent socket error to occur on sock. */

EVENT2_EXPORT_SYMBOL

int evutil_socket_geterror(evutil_socket_t sock);

/** Convert a socket error to a string. */

EVENT2_EXPORT_SYMBOL

const char *evutil_socket_error_to_string(int errcode);

#define EVUTIL_INVALID_SOCKET INVALID_SOCKET

#elif defined(EVENT_IN_DOXYGEN_)

/**

   @name Socket error functions

  

   These functions are needed for making programs compatible between

   Windows and Unix-like platforms.

  

   You see, Winsock handles socket errors differently from the rest of

   the world.  Elsewhere, a socket error is like any other error and is

   stored in errno.  But winsock functions require you to retrieve the

   error with a special function, and don't let you use strerror for

   the error codes.  And handling EWOULDBLOCK is ... different.

  

   @{

*/

/** Return the most recent socket error.  Not idempotent on all platforms. */

#define EVUTIL_SOCKET_ERROR() ...

/** Replace the most recent socket error with errcode */

#define EVUTIL_SET_SOCKET_ERROR(errcode) ...

/** Return the most recent socket error to occur on sock. */

#define evutil_socket_geterror(sock) ...

/** Convert a socket error to a string. */

#define evutil_socket_error_to_string(errcode) ...

#define EVUTIL_INVALID_SOCKET -1

/**@}*/

#else /** !EVENT_IN_DOXYGEN_ && !_WIN32 */

#define EVUTIL_SOCKET_ERROR() (errno)

#define EVUTIL_SET_SOCKET_ERROR(errcode)        \

        do { errno = (errcode); } while (0)

#define evutil_socket_geterror(sock) (errno)

#define evutil_socket_error_to_string(errcode) (strerror(errcode))

#define EVUTIL_INVALID_SOCKET -1

#endif /** !_WIN32 */
~~~

这些宏访问和操作套接字错误代码。<font color="#8064a2">EVUTIL_SOCKET_ERROR（）</font>返回本线程最后一次套接字操作的全局错误号，evutil_socket_geterror（）则返回某特定套接字的错误号。（在类Unix系统中都是errno）<font color="#8064a2">EVUTIL_SET_SOCKET_ERROR</font>()修改当前套接字错误号（与设置Unix中的errno类似），<font color="#8064a2">evutil_socket_error_to_string</font>（）返回代表某给定套接字错误号的字符串（与Unix中的strerror()类似）。

（因为对于来自套接字函数的错误，Windows不使用errno，而是使用WSAGetLastError()，所以需要这些函数。）

**注意**：Windows套接字错误与从errno看到的标准C错误是不同的。

## evutil_make_socket_nonblocking()
~~~c
int evutil_make_socket_nonblocking(evutil_socket_t fd)
~~~
用于对套接字进行非阻塞IO的调用也不能移植到Windows中。<font color="#00b050">evutil_make_socket_nonblocking</font>()函数要求一个套接字（来自socket()或者accept()）作为参数，将其设置为非阻塞的。（设置Unix中的O_NONBLOCK标志和Windows中的<font color="#8064a2">FIONBIO</font>标志）

## evutil_make_listen_socket_reuseable()
~~~c
int evutil_make_listen_socket_reuseable(evutil_socket_t sock)
~~~
这个函数确保关闭监听套接字后，它使用的地址可以立即被另一个套接字使用。（在Unix中它设置<font color="#8064a2">SO_REUSEADDR</font>标志，在Windows中则不做任何操作。不能在Windows中使用<font color="#8064a2">SO_REUSEADDR</font>标志：它有另外不同的含义（译者注：多个套接字绑定到相同地址））

## evutil_make_socket_closeonexec()
~~~c
int evutil_make_socket_closeonexec(evutil_socket_t fd)
~~~

这个函数告诉操作系统，如果调用了exec()，应该关闭指定的套接字。在Unix中函数设置FD_CLOEXEC标志，在Windows上则没有操作。

## evutil_socketpair()
~~~c
int evutil_socketpair(int family, int type, int protocol, evutil_socket_t fd[2])
~~~
这个函数的行为跟Unix的socketpair（）调用相同：创建两个相互连接起来的套接字，可对其使用普通套接字IO调用。函数将两个套接字存储在sv[0]和sv[1]中，成功时返回0，失败时返回-1。

在Windows中，这个函数仅能支持<font color="#8064a2">AF_INET</font>协议族、SOCK_STREAM类型和0协议的套接字。注意：在防火墙软件明确阻止127.0.0.1，禁止主机与自身通话的情况下，函数可能失败。

除了evutil_make_socket_closeonexec（）由2.0.4-alpha版本引入外，这些函数都由1.4.0-alpha版本引入。
### source code
#### evutil_make_socket_nonblocking()
~~~c
  

int

evutil_make_socket_nonblocking(evutil_socket_t fd)

{

#ifdef _WIN32

    {

        unsigned long nonblocking = 1;

        if (ioctlsocket(fd, FIONBIO, &nonblocking) == SOCKET_ERROR) {

            event_sock_warn(fd, "fcntl(%d, F_GETFL)", (int)fd);

            return -1;

        }

    }

#else

    {

        int flags;

        if ((flags = fcntl(fd, F_GETFL, NULL)) < 0) {

            event_warn("fcntl(%d, F_GETFL)", fd);

            return -1;

        }

        if (!(flags & O_NONBLOCK)) {

            if (fcntl(fd, F_SETFL, flags | O_NONBLOCK) == -1) {

                event_warn("fcntl(%d, F_SETFL)", fd);

                return -1;

            }

        }

    }

#endif

    return 0;

}
~~~

#### evutil_make_listen_socket_reuseable()
~~~c
int

evutil_make_listen_socket_reuseable(evutil_socket_t sock)

{

#if defined(SO_REUSEADDR) && !defined(_WIN32)

    int one = 1;

    /* REUSEADDR on Unix means, "don't hang on to this address after the

     * listener is closed."  On Windows, though, it means "don't keep other

     * processes from binding to this address while we're using it. */

    return setsockopt(sock, SOL_SOCKET, SO_REUSEADDR, (void*) &one,

        (ev_socklen_t)sizeof(one));

#else

    return 0;

#endif

}
~~~

#### evutil_make_socket_closeonexec()
~~~c
int

evutil_make_socket_closeonexec(evutil_socket_t fd)

{

#if !defined(_WIN32) && defined(EVENT__HAVE_SETFD)

    int flags;

    if ((flags = fcntl(fd, F_GETFD, NULL)) < 0) {

        event_warn("fcntl(%d, F_GETFD)", fd);

        return -1;

    }

    if (!(flags & FD_CLOEXEC)) {

        if (fcntl(fd, F_SETFD, flags | FD_CLOEXEC) == -1) {

            event_warn("fcntl(%d, F_SETFD)", fd);

            return -1;

        }

    }

#endif

    return 0;

}
~~~


#### evutil_socketpair()
~~~c
int

evutil_socketpair(int family, int type, int protocol, evutil_socket_t fd[2])

{

#ifndef _WIN32

    return socketpair(family, type, protocol, fd);

#else

    return evutil_ersatz_socketpair_(family, type, protocol, fd);

#endif

}
~~~


# Portable string manipulation functions
## evutil_strtoll

~~~c
ev_int64_t evutil_strtoll(const char *s, char **endptr, int base)
~~~

这个函数与strtol行为相同，只是用于64位整数。在某些平台上，仅支持十进制。
## evutil_snprintf evutil_vsnprintf

~~~c
 int evutil_snprintf(char *buf, size_t buflen, const char *format, ...);

 int evutil_vsnprintf(char *buf, size_t buflen, const char *format, va_list ap);
~~~


这些<font color="#8064a2">snprintf</font>替代函数的行为与标准snprintf和vsnprintf接口相同。函数返回在缓冲区足够长的情况下将写入的字节数，不包括结尾的NULL字节。（这个行为遵循C99的snprintf()标准，但与Windows的_snprintf()相反：如果字符串无法放入缓冲区，_snprintf()会返回负数）

evutil_strtoll（）从1.4.2-rc版本就存在了，其他函数首次出现在1.4.5版本中。

## source code
### evutil_strtoll()
~~~c
ev_int64_t

evutil_strtoll(const char *s, char **endptr, int base)

{

#ifdef EVENT__HAVE_STRTOLL

    return (ev_int64_t)strtoll(s, endptr, base);

#elif EVENT__SIZEOF_LONG == 8

    return (ev_int64_t)strtol(s, endptr, base);

#elif defined(_WIN32) && defined(_MSC_VER) && _MSC_VER < 1300

    /* XXXX on old versions of MS APIs, we only support base

     * 10. */

    ev_int64_t r;

    if (base != 10)

        return 0;

    r = (ev_int64_t) _atoi64(s);

    while (isspace(*s))

        ++s;

    if (*s == '-')

        ++s;

    while (isdigit(*s))

        ++s;

    if (endptr)

        *endptr = (char*) s;

    return r;

#elif defined(_WIN32)

    return (ev_int64_t) _strtoi64(s, endptr, base);

#elif defined(EVENT__SIZEOF_LONG_LONG) && EVENT__SIZEOF_LONG_LONG == 8

    long long r;

    int n;

    if (base != 10 && base != 16)

        return 0;

    if (base == 10) {

        n = sscanf(s, "%lld", &r);

    } else {

        unsigned long long ru=0;

        n = sscanf(s, "%llx", &ru);

        if (ru > EV_INT64_MAX)

            return 0;

        r = (long long) ru;

    }

    if (n != 1)

        return 0;

    while (EVUTIL_ISSPACE_(*s))

        ++s;

    if (*s == '-')

        ++s;

    if (base == 10) {

        while (EVUTIL_ISDIGIT_(*s))

            ++s;

    } else {

        while (EVUTIL_ISXDIGIT_(*s))

            ++s;

    }

    if (endptr)

        *endptr = (char*) s;

    return r;

#else

#error "I don't know how to parse 64-bit integers."

#endif

}
~~~

### evutil_snprintf()
~~~c
int

evutil_snprintf(char *buf, size_t buflen, const char *format, ...)

{

    int r;

    va_list ap;

    va_start(ap, format);

    r = evutil_vsnprintf(buf, buflen, format, ap);

    va_end(ap);

    return r;

}
~~~

### evutil_vsnprintf()
~~~c
int

evutil_vsnprintf(char *buf, size_t buflen, const char *format, va_list ap)

{

    int r;

    if (!buflen)

        return 0;

#if defined(_MSC_VER) || defined(_WIN32)

    r = _vsnprintf(buf, buflen, format, ap);

    if (r < 0)

        r = _vscprintf(format, ap);

#elif defined(sgi)

    /* Make sure we always use the correct vsnprintf on IRIX */

    extern int      _xpg5_vsnprintf(char * __restrict,

        __SGI_LIBC_NAMESPACE_QUALIFIER size_t,

        const char * __restrict, /* va_list */ char *);

  

    r = _xpg5_vsnprintf(buf, buflen, format, ap);

#else

    r = vsnprintf(buf, buflen, format, ap);

#endif

    buf[buflen-1] = '\0';

    return r;

}
~~~

# Locale-independent string manipulation functions
## evutil_ascii_strcasecmp evutil_ascii_strncasecmp
实现基于ASCII的协议时，可能想要根据字符类型的ASCII记号来操作字符串，而不管当前的区域设置。libevent为此提供了一些函数：
~~~c  
int evutil_ascii_strcasecmp(const char *s1, const char *s2);

int evutil_ascii_strncasecmp(const char *s1, const char *s2, size_t n);
~~~

这些函数与strcasecmp()和strncasecmp()的行为类似，只是它们总是使用ASCII字符集进行比较，而不管当前的区域设置。这两个函数首次在2.0.3-alpha版本出现。

### source code
#### evutil_ascii_strcasecmp()

~~~c
int

evutil_ascii_strcasecmp(const char *s1, const char *s2)

{

    char c1, c2;

    while (1) {

        c1 = EVUTIL_TOLOWER_(*s1++);

        c2 = EVUTIL_TOLOWER_(*s2++);

        if (c1 < c2)

            return -1;

        else if (c1 > c2)

            return 1;

        else if (c1 == 0)

            return 0;

    }

}


~~~

#### evutil_ascii_strncasecmp()
~~~c
int evutil_ascii_strncasecmp(const char *s1, const char *s2, size_t n)

{

    char c1, c2;

    while (n--) {

        c1 = EVUTIL_TOLOWER_(*s1++);

        c2 = EVUTIL_TOLOWER_(*s2++);

        if (c1 < c2)

            return -1;

        else if (c1 > c2)

            return 1;

        else if (c1 == 0)

            return 0;

    }

    return 0;

}
~~~

# IPv6 helper and compatibility functions
## evutil_inet_ntop evutil_inet_pton
~~~c  
const char * evutil_inet_ntop(int af, const void *src, char *dst, size_t len);

int evutil_inet_pton(int af, const char *src, void *dst);
~~~

这些函数根据RFC 3493的规定解析和格式化IPv4与IPv6地址，与标准inet_ntop()和inet_pton()函数行为相同。要格式化IPv4地址，调用evutil_inet_ntop()，设置af为AF_INET，src指向in_addr结构体，dst指向大小为len的字符缓冲区。对于IPv6地址，af应该是AF_INET6，src则指向in6_addr结构体。要解析IP地址，调用evutil_inet_pton()，设置af为AF_INET或者AF_INET6，src指向要解析的字符串，dst指向一个in_addr或者in_addr6结构体。

失败时evutil_inet_ntop()返回NULL，成功时返回到dst的指针。成功时evutil_inet_pton()返回0，失败时返回-1。
## evutil_parse_sockaddr_port()
~~~c
int evutil_parse_sockaddr_port(const char *ip_as_string, struct sockaddr *out, int *outlen);
~~~
这个接口解析来自str的地址，将结果写入到out中。outlen参数应该指向一个表示out中可用字节数的整数；函数返回时这个整数将表示实际使用了的字节数。成功时函数返回0，失败时返回-1。函数识别下列地址格式：
-  [ipv6]:端口号（如[ffff::]:80）

-  ipv6（如ffff::）

-  [ipv6]（如[ffff::]）

-  ipv4:端口号（如1.2.3.4:80）

-  ipv4（如1.2.3.4）

如果没有给出端口号，结果中的端口号将被设置为0。

~~~c
int

evutil_sockaddr_cmp(const struct sockaddr *sa1, const struct sockaddr *sa2,

    int include_port)
~~~


evutil_sockaddr_cmp()函数比较两个地址，如果sa1在sa2前面，返回负数；如果二者相等，则返回0；如果sa2在sa1前面，则返回正数。函数可用于AF_INET和AF_INET6地址；对于其他地址，返回值未定义。函数确保考虑地址的完整次序，但是不同版本中的次序可能不同。

如果include_port参数为false，而两个地址只有端口号不同，则它们被认为是相等的。否则，具有不同端口号的地址被认为是不等的。

除evutil_sockaddr_cmp()在2.0.3-alpha版本引入外，这些函数在2.0.1-alpha版本中引入。
## source code
### evutil_inet_ntop
~~~c
  

const char *

evutil_inet_ntop(int af, const void *src, char *dst, size_t len)

{

#if defined(EVENT__HAVE_INET_NTOP) && !defined(USE_INTERNAL_NTOP)

    return inet_ntop(af, src, dst, len);

#else

    if (af == AF_INET) {

        const struct in_addr *in = src;

        const ev_uint32_t a = ntohl(in->s_addr);

        int r;

        r = evutil_snprintf(dst, len, "%d.%d.%d.%d",

            (int)(ev_uint8_t)((a>>24)&0xff),

            (int)(ev_uint8_t)((a>>16)&0xff),

            (int)(ev_uint8_t)((a>>8 )&0xff),

            (int)(ev_uint8_t)((a    )&0xff));

        if (r<0||(size_t)r>=len)

            return NULL;

        else

            return dst;

#ifdef AF_INET6

    } else if (af == AF_INET6) {

        const struct in6_addr *addr = src;

        char buf[64], *cp;

        int longestGapLen = 0, longestGapPos = -1, i,

            curGapPos = -1, curGapLen = 0;

        ev_uint16_t words[8];

        for (i = 0; i < 8; ++i) {

            words[i] =

                (((ev_uint16_t)addr->s6_addr[2*i])<<8) + addr->s6_addr[2*i+1];

        }

        if (words[0] == 0 && words[1] == 0 && words[2] == 0 && words[3] == 0 &&

            words[4] == 0 && ((words[5] == 0 && words[6] && words[7]) ||

            (words[5] == 0xffff))) {

            /* This is an IPv4 address. */

            if (words[5] == 0) {

                evutil_snprintf(buf, sizeof(buf), "::%d.%d.%d.%d",

                    addr->s6_addr[12], addr->s6_addr[13],

                    addr->s6_addr[14], addr->s6_addr[15]);

            } else {

                evutil_snprintf(buf, sizeof(buf), "::%x:%d.%d.%d.%d", words[5],

                    addr->s6_addr[12], addr->s6_addr[13],

                    addr->s6_addr[14], addr->s6_addr[15]);

            }

            if (strlen(buf) > len)

                return NULL;

            strlcpy(dst, buf, len);

            return dst;

        }

        i = 0;

        while (i < 8) {

            if (words[i] == 0) {

                curGapPos = i++;

                curGapLen = 1;

                while (i<8 && words[i] == 0) {

                    ++i; ++curGapLen;

                }

                if (curGapLen > longestGapLen) {

                    longestGapPos = curGapPos;

                    longestGapLen = curGapLen;

                }

            } else {

                ++i;

            }

        }

        if (longestGapLen<=1)

            longestGapPos = -1;

  

        cp = buf;

        for (i = 0; i < 8; ++i) {

            if (words[i] == 0 && longestGapPos == i) {

                if (i == 0)

                    *cp++ = ':';

                *cp++ = ':';

                while (i < 8 && words[i] == 0)

                    ++i;

                --i; /* to compensate for loop increment. */

            } else {

                evutil_snprintf(cp,

                                sizeof(buf)-(cp-buf), "%x", (unsigned)words[i]);

                cp += strlen(cp);

                if (i != 7)

                    *cp++ = ':';

            }

        }

        *cp = '\0';

        if (strlen(buf) > len)

            return NULL;

        strlcpy(dst, buf, len);

        return dst;

#endif

    } else {

        return NULL;

    }

#endif

}
~~~
### evutil_inet_pton
~~~c
  

int

evutil_inet_pton(int af, const char *src, void *dst)

{

#if defined(EVENT__HAVE_INET_PTON) && !defined(USE_INTERNAL_PTON)

    return inet_pton(af, src, dst);

#else

    if (af == AF_INET) {

        unsigned a,b,c,d;

        char more;

        struct in_addr *addr = dst;

        if (sscanf(src, "%u.%u.%u.%u%c", &a,&b,&c,&d,&more) != 4)

            return 0;

        if (a > 255) return 0;

        if (b > 255) return 0;

        if (c > 255) return 0;

        if (d > 255) return 0;

        addr->s_addr = htonl((a<<24) | (b<<16) | (c<<8) | d);

        return 1;

#ifdef AF_INET6

    } else if (af == AF_INET6) {

        struct in6_addr *out = dst;

        ev_uint16_t words[8];

        int gapPos = -1, i, setWords=0;

        const char *dot = strchr(src, '.');

        const char *eow; /* end of words. */

        if (dot == src)

            return 0;

        else if (!dot)

            eow = src+strlen(src);

        else {

            unsigned byte1,byte2,byte3,byte4;

            char more;

            for (eow = dot-1; eow >= src && EVUTIL_ISDIGIT_(*eow); --eow)

                ;

            ++eow;

  

            /* We use "scanf" because some platform inet_aton()s are too lax

             * about IPv4 addresses of the form "1.2.3" */

            if (sscanf(eow, "%u.%u.%u.%u%c",

                       &byte1,&byte2,&byte3,&byte4,&more) != 4)

                return 0;

  

            if (byte1 > 255 ||

                byte2 > 255 ||

                byte3 > 255 ||

                byte4 > 255)

                return 0;

  

            words[6] = (byte1<<8) | byte2;

            words[7] = (byte3<<8) | byte4;

            setWords += 2;

        }

  

        i = 0;

        while (src < eow) {

            if (i > 7)

                return 0;

            if (EVUTIL_ISXDIGIT_(*src)) {

                char *next;

                long r = strtol(src, &next, 16);

                if (next > 4+src)

                    return 0;

                if (next == src)

                    return 0;

                if (r<0 || r>65536)

                    return 0;

  

                words[i++] = (ev_uint16_t)r;

                setWords++;

                src = next;

                if (*src != ':' && src != eow)

                    return 0;

                ++src;

            } else if (*src == ':' && i > 0 && gapPos==-1) {

                gapPos = i;

                ++src;

            } else if (*src == ':' && i == 0 && src[1] == ':' && gapPos==-1) {

                gapPos = i;

                src += 2;

            } else {

                return 0;

            }

        }

  

        if (setWords > 8 ||

            (setWords == 8 && gapPos != -1) ||

            (setWords < 8 && gapPos == -1))

            return 0;

  

        if (gapPos >= 0) {

            int nToMove = setWords - (dot ? 2 : 0) - gapPos;

            int gapLen = 8 - setWords;

            /* assert(nToMove >= 0); */

            if (nToMove < 0)

                return -1; /* should be impossible */

            memmove(&words[gapPos+gapLen], &words[gapPos],

                    sizeof(ev_uint16_t)*nToMove);

            memset(&words[gapPos], 0, sizeof(ev_uint16_t)*gapLen);

        }

        for (i = 0; i < 8; ++i) {

            out->s6_addr[2*i  ] = words[i] >> 8;

            out->s6_addr[2*i+1] = words[i] & 0xff;

        }

  

        return 1;

#endif

    } else {

        return -1;

    }

#endif

}
~~~

### evutil_parse_sockaddr_port()
~~~c
  

int

evutil_parse_sockaddr_port(const char *ip_as_string, struct sockaddr *out, int *outlen)

{

    int port;

    unsigned int if_index;

    char buf[128];

    const char *cp, *addr_part, *port_part;

    int is_ipv6;

    /* recognized formats are:

     * [ipv6]:port

     * ipv6

     * [ipv6]

     * ipv4:port

     * ipv4

     */

  

    cp = strchr(ip_as_string, ':');

    if (*ip_as_string == '[') {

        size_t len;

        if (!(cp = strchr(ip_as_string, ']'))) {

            return -1;

        }

        len = ( cp-(ip_as_string + 1) );

        if (len > sizeof(buf)-1) {

            return -1;

        }

        memcpy(buf, ip_as_string+1, len);

        buf[len] = '\0';

        addr_part = buf;

        if (cp[1] == ':')

            port_part = cp+2;

        else

            port_part = NULL;

        is_ipv6 = 1;

    } else if (cp && strchr(cp+1, ':')) {

        is_ipv6 = 1;

        addr_part = ip_as_string;

        port_part = NULL;

    } else if (cp) {

        is_ipv6 = 0;

        if (cp - ip_as_string > (int)sizeof(buf)-1) {

            return -1;

        }

        memcpy(buf, ip_as_string, cp-ip_as_string);

        buf[cp-ip_as_string] = '\0';

        addr_part = buf;

        port_part = cp+1;

    } else {

        addr_part = ip_as_string;

        port_part = NULL;

        is_ipv6 = 0;

    }

  

    if (port_part == NULL) {

        port = 0;

    } else {

        port = atoi(port_part);

        if (port <= 0 || port > 65535) {

            return -1;

        }

    }

  

    if (!addr_part)

        return -1; /* Should be impossible. */

#ifdef AF_INET6

    if (is_ipv6)

    {

        struct sockaddr_in6 sin6;

        memset(&sin6, 0, sizeof(sin6));

#ifdef EVENT__HAVE_STRUCT_SOCKADDR_IN6_SIN6_LEN

        sin6.sin6_len = sizeof(sin6);

#endif

        sin6.sin6_family = AF_INET6;

        sin6.sin6_port = htons(port);

        if (1 != evutil_inet_pton_scope(

            AF_INET6, addr_part, &sin6.sin6_addr, &if_index)) {

            return -1;

        }

        if ((int)sizeof(sin6) > *outlen)

            return -1;

        sin6.sin6_scope_id = if_index;

        memset(out, 0, *outlen);

        memcpy(out, &sin6, sizeof(sin6));

        *outlen = sizeof(sin6);

        return 0;

    }

    else

#endif

    {

        struct sockaddr_in sin;

        memset(&sin, 0, sizeof(sin));

#ifdef EVENT__HAVE_STRUCT_SOCKADDR_IN_SIN_LEN

        sin.sin_len = sizeof(sin);

#endif

        sin.sin_family = AF_INET;

        sin.sin_port = htons(port);

        if (1 != evutil_inet_pton(AF_INET, addr_part, &sin.sin_addr))

            return -1;

        if ((int)sizeof(sin) > *outlen)

            return -1;

        memset(out, 0, *outlen);

        memcpy(out, &sin, sizeof(sin));

        *outlen = sizeof(sin);

        return 0;

    }

}
~~~
### evutil_sockaddr_cmp()


~~~c
int

evutil_sockaddr_cmp(const struct sockaddr *sa1, const struct sockaddr *sa2,

    int include_port)

{

    int r;

    if (0 != (r = (sa1->sa_family - sa2->sa_family)))

        return r;

  

    if (sa1->sa_family == AF_INET) {

        const struct sockaddr_in *sin1, *sin2;

        sin1 = (const struct sockaddr_in *)sa1;

        sin2 = (const struct sockaddr_in *)sa2;

        if (sin1->sin_addr.s_addr < sin2->sin_addr.s_addr)

            return -1;

        else if (sin1->sin_addr.s_addr > sin2->sin_addr.s_addr)

            return 1;

        else if (include_port &&

            (r = ((int)sin1->sin_port - (int)sin2->sin_port)))

            return r;

        else

            return 0;

    }

#ifdef AF_INET6

    else if (sa1->sa_family == AF_INET6) {

        const struct sockaddr_in6 *sin1, *sin2;

        sin1 = (const struct sockaddr_in6 *)sa1;

        sin2 = (const struct sockaddr_in6 *)sa2;

        if ((r = memcmp(sin1->sin6_addr.s6_addr, sin2->sin6_addr.s6_addr, 16)))

            return r;

        else if (include_port &&

            (r = ((int)sin1->sin6_port - (int)sin2->sin6_port)))

            return r;

        else

            return 0;

    }

#endif

    return 1;

}
~~~

# Struct Portability Functions
## evutil_offsetof
~~~c
/** Replacement for offsetof on platforms that don't define it. */

#ifdef offsetof

#define evutil_offsetof(type, field) offsetof(type, field)

#else

#define evutil_offsetof(type, field) ((off_t)(&((type *)0)->field))

#endif

~~~

~~~c
#define offsetof(t, d) __builtin_offsetof(t, d)
~~~


用于计算结构体成员相对于结构体起始位置的字节偏移量
`evutil_offsetof` 是 `libevent` 库中的一个宏，用于计算结构体成员相对于结构体起始位置的字节偏移量。它的作用类似于 C 标准库中的 `offsetof` 宏，但在 `libevent` 中可能有一些额外的检查或调整。
 

`#define evutil_offsetof(type, member) ((size_t)(&((type *)0)->member))`

这个宏通过创建一个指向 `type` 类型的空指针，并访问它的 `member` 成员，然后将其转换为 `size_t` 类型，计算成员的偏移量。由于我们从一个零地址的指针开始，这样可以获得成员在结构体中的实际偏移量。

# Secure Random Number Generator
很多应用（包括evdns）为了安全考虑需要很难预测的随机数。

~~~c
void

evutil_secure_rng_get_bytes(void *buf, size_t n)

{

    ev_arc4random_buf(buf, n);

}
~~~

这个函数用随机数据填充buf处的n个字节。

如果所在平台提供了arc4random()，libevent会使用这个函数。否则，libevent会使用自己的arc4random()实现，种子则来自操作系统的熵池（entropy pool）（Windows中的CryptGenRandom，其他平台中的/dev/urandom）

~~~c
int

evutil_secure_rng_init(void)

{

    int val;

  

    ARC4_LOCK_();

    val = (!arc4_stir()) ? 0 : -1;

    ARC4_UNLOCK_();

    return val;

}
~~~

~~~c
  

#if !defined(EVENT__HAVE_ARC4RANDOM) || defined(EVENT__HAVE_ARC4RANDOM_ADDRANDOM)

void

evutil_secure_rng_add_bytes(const char *buf, size_t n)

{

    arc4random_addrandom((unsigned char*)buf,

        n>(size_t)INT_MAX ? INT_MAX : (int)n);

}

#endif
~~~

不需要手动初始化安全随机数发生器，但是如果要确认已经成功初始化，可以调用evutil_secure_rng_init()。函数会播种RNG（如果没有播种过），并在成功时返回0。函数返回-1则表示libevent无法在操作系统中找到合适的熵源（source of entropy），如果不自己初始化RNG，就无法安全使用RNG了。

如果程序运行在可能会放弃权限的环境中（比如说，通过执行chroot()），在放弃权限前应该调用evutil_secure_rng_init()。

可以调用evutil_secure_rng_add_bytes()向熵池加入更多随机字节，但通常不需要这么做。

这些函数是2.0.4-alpha版本引入的。