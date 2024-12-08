# Memory allocation related macro definition
 
```c
#define mm_malloc(sz) 			event_mm_malloc_(sz)
#define mm_calloc(count, size) 	event_mm_calloc_((count), (size))
#define mm_strdup(s) 			event_mm_strdup_(s)
#define mm_realloc(p, sz) 		event_mm_realloc_((p), (sz))
#define mm_free(p) 				event_mm_free_(p)
//#ifdef EVENT__DISABLE_MM_REPLACEMENT 下
#define mm_malloc(sz) malloc(sz)
#define mm_calloc(n, sz) calloc((n), (sz))
#define mm_strdup(s) strdup(s)
#define mm_realloc(p, sz) realloc((p), (sz))
#define mm_free(p) free(p)
```


| 宏                                       | default definition                  | <font color="#8064a2">EVENT__DISABLE_MM_REPLACEMENT</font> not definition |
| --------------------------------------- | ----------------------------------- | ------------------------------------------------------------------------- |
| <font color="#4bacc6">mm_malloc</font>  | `event_mm_malloc_(sz)`              | `malloc(sz)`                                                              |
| <font color="#4bacc6">mm_calloc</font>  | `event_mm_calloc_((count), (size))` | `calloc((n), (sz))`                                                       |
| <font color="#4bacc6">mm_strdup</font>  | `event_mm_strdup_(s)`               | `strdup(s)`                                                               |
| <font color="#4bacc6">mm_realloc</font> | `event_mm_realloc_((p),(sz))`       | `realloc((p), (sz))`                                                      |
| <font color="#4bacc6">mm_free</font>    | `event_mm_free_(p)`                 | `free(p)`                                                                 |
## memory allocation function
---
```c
static void *(*mm_malloc_fn_)(size_t sz) = NULL;//定义一个接受参数为seize_t类型的函数指针mm_malloc_fn_
static void *(*mm_realloc_fn_)(void *p, size_t sz) = NULL;
static void (*mm_free_fn_)(void *p) = NULL;
```

- **`mm_malloc_fn_`**:
    
    - **类型**: `void *(*mm_malloc_fn_)(size_t sz)`
    - **用途**: 指向一个接受 `size_t` 类型大小并返回 `void*` 的内存分配函数，通常类似于 `malloc`。这个指针允许 `libevent` 使用自定义的内存分配实现。
- **`mm_realloc_fn_`**:
    
    - **类型**: `void *(*mm_realloc_fn_)(void *p, size_t sz)`
    - **用途**: 指向一个接受一个内存块指针和新的大小，并返回调整后的内存块指针的函数，类似于 `realloc`。通过这个指针，`libevent` 可以使用自定义的内存重新分配函数。
- **`mm_free_fn_`**:
    
    - **类型**: `void (*mm_free_fn_)(void *p)`
    - **用途**: 指向一个接受内存块指针并释放内存的函数，类似于 `free`。这个指针允许 `libevent` 使用自定义的内存释放实现。
## <font color="#4bacc6">mm_realloc()</font>

```c
void *  event_mm_realloc_(void *ptr, size_t sz)  
{  
    if (mm_realloc_fn_)  
        return mm_realloc_fn_(ptr, sz);  
    else  
        return realloc(ptr, sz);  
}
```

## <font color="#4bacc6">mm_strdup()</font>
```c
char * event_mm_strdup_(const char *str)
{
	if (!str) {
		errno = EINVAL;
		return NULL;
	}

	if (mm_malloc_fn_) {
		size_t ln = strlen(str);
		void *p = NULL;
		if (ln == EV_SIZE_MAX)
			goto error;
		p = mm_malloc_fn_(ln+1);
		if (p)
			return memcpy(p, str, ln+1);
	} else
        //不同平台下的函数不一样
#ifdef _WIN32
		return _strdup(str);
#else
		return strdup(str);
#endif

error:
	errno = ENOMEM;
	return NULL;
}
 
```

## <font color="#4bacc6">mm_malloc()</font>
```c
void * event_mm_malloc_(size_t sz)
{
	if (sz == 0)
		return NULL;

	if (mm_malloc_fn_)
		return mm_malloc_fn_(sz);
	else
		return malloc(sz);
}

```

## <font color="#4bacc6">mm_free()</font>
```c
#define mm_free(p) event_mm_free_(p)

static void (*mm_free_fn_)(void *p) = NULL;//指向释放内存的函数
```

```c
void event_mm_free_(void *ptr)
{
	if (mm_free_fn_)
		mm_free_fn_(ptr);
	else
		free(ptr);
}
```

## <font color="#4bacc6">mm_calloc()</font>
```c
 

#ifndef EVENT__DISABLE_MM_REPLACEMENT
#define mm_malloc(sz) event_mm_malloc_(sz)
#define mm_calloc(count, size) event_mm_calloc_((count), (size))
#define mm_strdup(s) event_mm_strdup_(s)
#define mm_realloc(p, sz) event_mm_realloc_((p), (sz))
#define mm_free(p) event_mm_free_(p)
#else
#define mm_malloc(sz) malloc(sz)
#define mm_calloc(n, sz) calloc((n), (sz))
#define mm_strdup(s) strdup(s)
#define mm_realloc(p, sz) realloc((p), (sz))
#define mm_free(p) free(p)
#endif //EVENT__DISABLE_MM_REPLACEMENT

static void *(*mm_malloc_fn_)(size_t sz) = NULL;//定义一个接受参数为seize_t类型的函数指针mm_malloc_fn_
void * event_mm_calloc_(size_t count, size_t size)
{
	if (count == 0 || size == 0)
		return NULL;

	if (mm_malloc_fn_) { 
        //非空
		size_t sz = count * size;
		void *p = NULL;
		if (count > EV_SIZE_MAX / size)
			goto error;
		p = mm_malloc_fn_(sz);
		if (p)
			return memset(p, 0, sz);
	} else {
        //为空
		void *p = calloc(count, size);
#ifdef _WIN32
		/* Windows calloc doesn't reliably set ENOMEM */
		if (p == NULL)
			goto error;
#endif
		return p;
	}

error:
	errno = ENOMEM;
	return NULL;
}
```
