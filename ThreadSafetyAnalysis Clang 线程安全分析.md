# 介绍
Clang 线程安全分析是一个 C++ 语言扩展，它对 代码中的潜在争用条件。分析是完全静态的（即编译时没有运行时开销分析仍然是正在积极开发中，但它已经足够成熟，可以部署在工业环境。它由 Google 与 CERT/SEI，并在 Google 的内部代码库中广泛使用。

线程安全分析的工作方式与多线程的类型系统非常相似 程序。除了声明数据类型（例如 int、flout、 等），程序员可以（可选地）声明对该数据的访问方式 在多线程环境中进行控制。例如，如果由 mutex 保护，则分析将在 一段代码在没有首先锁定的情况下读取或写入 。 同样，如果存在只能由 GUI 线程，则分析将在其他线程调用这些 例程。

## 一个简单的例子

~~~c
#include <mutex>
class Mutex {
public:
  void Lock() { }
  void Unlock() { }
};

#define GUARDED_BY(x)  //GUARDED_BY 属性声明线程必须先锁定 mu 才能读取或写入 balance，从而确保增量和减量操作是原子的
#define REQUIRES(x)    //REQUIRES  声明调用线程在调用 withdrawImpl 之前必须锁定 mu。


class BankAccount {
private:
  Mutex mu;
  int   balance GUARDED_BY(mu);

    // brief : 存款
  void depositImpl(int amount) {
    balance += amount;       // WARNING! Cannot write balance without locking mu. 因为REQUIRES
  }


    // brief : 取钱
  void withdrawImpl(int amount) REQUIRES(mu) {
    balance -= amount;       // OK. Caller must have locked mu.
  }

public:
  void withdraw(int amount) {
    mu.Lock();
    withdrawImpl(amount);    // OK.  We've locked mu.
  }                          // WARNING!  Failed to unlock mu.

  //钱款来源
  void transferFrom(BankAccount& b, int amount) {
    mu.Lock();
    b.withdrawImpl(amount);  // WARNING!  Calling withdrawImpl() requires locking b.mu.
    depositImpl(amount);     // OK.  depositImpl() has no requirements.
    mu.Unlock();
  }
};
~~~

此示例演示了分析背后的基本概念。该属性声明线程必须先锁定，然后才能锁定 读取或写入 ，从而确保递增和递减操作是原子的。同样，声明调用线程必须在调用之前锁定。 由于假定调用方已锁定，因此在方法体内进行修改是安全的。

# <font color="#8064a2">GUARDED_BY</font>
<font color="#8064a2">GUARDED_BY</font> 是数据成员的一个属性，声明数据成员受给定功能的保护。对数据的读取操作需要共享访问，而写入操作需要独占访问。

## <font color="#8064a2">PT_GUARDED_BY</font>
<font color="#8064a2">PT_GUARDED_BY</font> 类似，但旨在用于指针和智能指针。数据成员本身没有约束，但它指向的数据受到给定功能的保护。


# Test1
~~~c
#include "mutex.h"
#include <memory>

Mutex mu;
int *p1                  
GUARDED_BY(mu);
int *p2                  PT_GUARDED_BY(mu);
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
<font color="#8064a2">ACQUIRE</font> 和 <font color="#8064a2">ACQUIRE_SHARED</font> 是函数或方法上的属性，用于声明函数获取功能但不释放该功能。(获取锁但不释放锁)，给定的功能不得在进入时保留，并且将在退出时保留（对于 <font color="#8064a2">ACQUIRE</font> 是独有的，对于 <font color="#8064a2">ACQUIRE_SHARED</font> 是共享的）。
## <font color="#8064a2">REQUIRES REQUIRES_SHARED </font>
<font color="#8064a2">REQUIRES</font> 是函数或方法的一个属性，它声明调用线程必须具有对给定功能的独占访问权限。可以指定多个功能。这些功能必须在函数入口处保留，并且在退出时仍必须保留。

<font color="#8064a2">REQUIRES_SHARED</font> 类似，但仅要求共享访问权限。
## <font color="#8064a2">RELEASE RELEASE_GENERIC </font>
<font color="#8064a2">RELEASE</font>  和 <font color="#8064a2">RELEASE_GENERIC</font> 声明函数释放给定的功能。功能必须在进入时保留（<font color="#8064a2">RELEASE</font> 独占、<font color="#8064a2">RELEASE_SHARED</font> 共享、<font color="#8064a2">RELEASE_GENERIC</font> 独占或共享），退出时将不再保留。

## <font color="#8064a2">ACQUIRE ACQUIRE</font>
在资源管理的上下文中，ACQUIRE 和 ACQUIRE 通常是成对出现的，ACQUIRE 是获取资源，RELEASE 是释放资源。例如，线程在访问共享资源之前需要执行 ACQUIRE 操作，而访问完资源之后则需要执行 RELEASE 操作。
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




| <font color="#8064a2">GUARDED_BY</font> | <font color="#8064a2">GUARDED_BY</font> 是数据成员的一个属性，声明数据成员受给定功能的保护。对数据的读取操作需要共享访问，而写入操作需要独占访问。 |
| --------------------------------------- | --------------------------------------------------------------------------------------------- |
|                                         |                                                                                               |
