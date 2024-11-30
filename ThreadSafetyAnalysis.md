## <font color="#8064a2">GUARDED_BY</font>
<font color="#8064a2">GUARDED_BY</font> 是数据成员的一个属性，声明数据成员受给定功能的保护。对数据的读取操作需要共享访问，而写入操作需要独占访问。

## <font color="#8064a2">PT_GUARDED_BY</font>
<font color="#8064a2">PT_GUARDED_BY</font> 类似，但旨在用于指针和智能指针。数据成员本身没有约束，但它指向的数据受到给定功能的保护。


# Test1
~~~c
#include "mutex.h"

#include <memory>

Mutex mu;
int *p1             GUARDED_BY(mu);
int *p2             PT_GUARDED_BY(mu);
std::unique_ptr<int> p3  PT_GUARDED_BY(mu); 

int main() {

  p1 = 0;             // Warning!
  *p2 = 42;           // Warning!
  p2 = new int;       // OK.
  *p3 = 42;           // Warning!
  p3.reset(new int);  // OK.

}
~~~

![](images/Pasted%20image%2020241130175804.png)

## <font color="#8064a2">ACQUIRE</font> <font color="#8064a2">ACQUIRE_SHARED</font>
<font color="#8064a2">ACQUIRE</font> 和 <font color="#8064a2">ACQUIRE_SHARED</font> 是函数或方法上的属性，用于声明函数获取功能但不释放该功能。给定的功能不得在进入时保留，并且将在退出时保留（对于 <font color="#8064a2">ACQUIRE</font> 是独有的，对于 <font color="#8064a2">ACQUIRE_SHARED</font> 是共享的）。
## <font color="#8064a2">REQUIRES REQUIRES_SHARED </font>
<font color="#8064a2">REQUIRES</font> 是函数或方法的一个属性，它声明调用线程必须具有对给定功能的独占访问权限。可以指定多个功能。这些功能必须在函数入口处保留，并且在退出时仍必须保留。

<font color="#8064a2">REQUIRES_SHARED</font> 类似，但仅要求共享访问权限。
## <font color="#8064a2">RELEASE RELEASE_GENERIC </font>
<font color="#8064a2">RELEASE</font>  和 <font color="#8064a2">RELEASE_GENERIC</font> 声明函数释放给定的功能。功能必须在进入时保留（<font color="#8064a2">RELEASE</font> 独占、<font color="#8064a2">RELEASE_SHARED</font> 共享、<font color="#8064a2">RELEASE_GENERIC</font> 独占或共享），退出时将不再保留。


# Test

~~~c

#include "mutex.h"
#include <memory>

class MyClass {
public:
  void init() EXCLUSIVE_LOCK_FUNCTION();
  void cleanup() UNLOCK_FUNCTION();
  void doSomething() EXCLUSIVE_LOCKS_REQUIRED(mu);
};

Mutex mu;
MyClass myObject GUARDED_BY(mu);

void lockAndInit() ACQUIRE(mu) {
  mu.Lock();
  myObject.init();
}

void cleanupAndUnlock() RELEASE(mu) {
  myObject.cleanup();
}                          // Warning!  Need to unlock mu.

void test() {
  lockAndInit();
  myObject.doSomething();
  cleanupAndUnlock();
  myObject.doSomething();  // Warning, mu is not locked.
}
~~~


如果没有参数传递给 <font color="#8064a2">ACQUIRE</font> 或 <font color="#8064a2">RELEASE</font>，则参数被假定为 this，并且分析不会检查函数体。此模式旨在供将锁定细节隐藏在抽象接口后面的类使用。例如
~~~c
template <class T>
class CAPABILITY("mutex") Container 
{
private:
  Mutex mu;
  T* data;

public:
  // Hide mu from public interface.
  void Lock()   ACQUIRE() { mu.Lock(); }
  void Unlock() RELEASE() { mu.Unlock(); }

  T& getElem(int i) { return data[i]; }
};

void test() {
  Container<int> c;
  c.Lock();
  int i = c.getElem(0);
  c.Unlock();
}
~~~