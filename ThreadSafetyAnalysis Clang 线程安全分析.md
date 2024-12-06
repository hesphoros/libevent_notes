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

    // brief : 存款实现
  void depositImpl(int amount) {
    balance += amount;       // WARNING! Cannot write balance without locking mu. 因为REQUIRES
  }


    // brief : 取钱实现
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

此示例演示了分析背后的基本概念。<font color="#8064a2">GUARDED_BY</font> 属性声明线程必须先锁定 mu 才能读取或写入 balance，从而确保增量和减量操作是原子的。同样，<font color="#8064a2">REQUIRES</font> 声明调用线程必须在调用 withdrawImpl 之前锁定 mu。由于假定调用者已锁定 mu，因此在方法主体内修改 balance 是安全的。

depositImpl() 方法没有 <font color="#8064a2">REQUIRES</font>，因此分析会发出警告。线程安全分析不是过程间的，因此必须明确声明调用者要求。transferFrom() 中也有一个警告，因为尽管该方法锁定了 this->mu，但它没有锁定 b.mu。分析会理解这是两个不同的对象中的两个单独的互斥锁。

最后，withdraw() 方法中有一个警告，因为它无法解锁 mu。每个锁都必须有相应的解锁，分析将检测双重锁定和双重解锁。允许函数获取锁而不释放它（反之亦然），但必须这样注释（使用 ACQUIRE/RELEASE）。

 

 


# Capabilities的基本概念

线程安全分析提供了一种通过能力保护资源的方法。（资源可以是数据成员，也可以是提供对某些底层资源的访问的函数/方法）该分析可确保调用线程无法访问资源（即调用函数或读取/写入数据），除非它具有这样做的能力。
功能与命名的 C++ 对象相关联，这些对象声明了获取和释放功能的特定方法。对象的名称用于标识功能。最常见的示例是互斥锁。例如，如果 mu 是互斥锁，则调用 mu.Lock() 会导致调用线程获得访问受 mu 保护的数据的能力。同样，调用 mu.Unlock() 会释放该能力。

线程可以独占或共享地持有能力。独占能力一次只能由一个线程持有，而共享能力可以同时由多个线程持有。此机制强制执行多读取器、单写入器模式。对受保护数据的写入操作需要独占访问，而读取操作仅需要共享访问。

在程序执行期间的任何给定时刻，线程都持有一组特定的能力（例如，它已锁定的互斥锁集）。这些能力就像允许线程访问给定资源的密钥或令牌。就像物理安全密钥一样，线程不能复制能力，也不能销毁能力。线程只能将能力释放给另一个线程，或从另一个线程获取能力。注释故意不考虑用于获取和释放能力的确切机制；它假设底层实现（例如互斥锁实现）以适当的方式进行交接。

在程序执行的给定点，给定线程实际持有的能力集是一个运行时概念。静态分析通过计算该集合的近似值（称为能力环境）来工作。能力环境针对每个程序点进行计算，并描述静态地知道在该特定点保留或不保留的能力集。此环境是线程在运行时实际保留的全套能力的保守近似值。

# 参考指南

线程安全分析使用属性来声明线程约束。属性必须附加到命名声明，例如类、方法和数据成员。强烈建议用户为各种属性定义宏；示例定义可以在下面的 mutex.h 中找到。以下文档假定使用宏。
这些属性仅控制线程安全分析做出的假设及其发出的警告。它们不会影响生成的代码或运行时行为。
由于历史原因，先前版本的线程安全使用了非常以锁为中心的宏名称。这些宏后来被重命名以适应更通用的功能模型。先前的名称仍在使用，并将在适当的位置在之前的标签下提及。

## <font color="#8064a2">GUARDED_BY(c)  PT_GUARDED_BY(c)</font>

<font color="#8064a2">GUARDED_BY</font>    是数据成员的一个属性，声明数据成员受给定功能的保护。对数据的读取操作需要共享访问，而写入操作需要独占访问。
<font color="#8064a2">PT_GUARDED_BY</font> 类似，但旨在用于指针和智能指针。数据成员本身没有约束，但它指向的数据受到给定功能的保护。


~~~c
#include "mutex.h"
#include <memory>

Mutex mu;
int *p1                  GUARDED_BY(mu);
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



## <font color="#8064a2">REQUIRES(…)    REQUIRES_SHARED(…)</font>

_Previously_: `EXCLUSIVE_LOCKS_REQUIRED`, `SHARED_LOCKS_REQUIRED`

<font color="#8064a2">REQUIRES</font> 是函数或方法上的一个属性，它声明调用线程必须对给定的功能具有独占访问权限。可以指定多个功能。这些功能必须在函数入口处保留，并且在退出时仍必须保留。

<font color="#8064a2">REQUIRES_SHARED</font> 类似，但仅需要共享访问权限。

~~~c

Mutex mu1, mu2;
int a GUARDED_BY(mu1);
int b GUARDED_BY(mu2);

void foo() REQUIRES(mu1, mu2) {
  a = 0;
  b = 0;
}

void test() {
  mu1.Lock();
  foo();         // Warning!  Requires mu2.
  mu1.Unlock();
}

~~~

![](images/Pasted%20image%2020241201161933.png)

## <font color="#8064a2">ACQUIRE(…)  ACQUIRE_SHARED(…) RELEASE(…)  RELEASE_SHARED(…)  RELEASE_GENERIC(…)</font>
 
_Previously_: `EXCLUSIVE_LOCK_FUNCTION`, `SHARED_LOCK_FUNCTION`, `UNLOCK_FUNCTION`

<font color="#8064a2">ACQUIRE</font> 和 <font color="#8064a2">ACQUIRE_SHARED</font> 是函数或方法上的属性，用于声明函数获取功能但不释放该功能。给定的功能不得在进入时保留，并且将在退出时保留（对于 ACQUIRE 是独有的，对于 ACQUIRE_SHARED 是共享的）。

<font color="#8064a2">RELEASE、RELEASE_SHARED</font> 和 <font color="#8064a2">RELEASE_GENERIC </font>声明函数释放给定的功能。功能必须在进入时保留（RELEASE 独占、RELEASE_SHARED 共享、RELEASE_GENERIC 独占或共享），退出时将不再保留。


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

## <font color="#8064a2">EXCLUDES(…)</font>
_Previously_: `LOCKS_EXCLUDED`

<font color="#8064a2">EXCLUDES</font> 是函数或方法上的一个属性，声明调用者不得拥有给定的功能。此注释用于防止死锁。许多互斥锁实现都不是可重入的，因此如果函数再次获取互斥锁，则可能会发生死锁。
~~~c
#include "mutex.h"

Mutex mu;
int a GUARDED_BY(mu);

void clear()EXCLUDES(mu)
{
    mu.Lock();
    a = 0;
    mu.Unlock();
}

void reset()
{
    mu.Lock();
    clear();// warning: cannot call function 'clear' while mutex 'mu' is held [-Wthread-safety-analysis]
    mu.Unlock();
}

 ~~~
 与 <font color="#8064a2">REQUIRES</font> 不同，<font color="#8064a2">EXCLUDES</font> 是可选的。如果属性缺失，分析不会发出警告，这在某些情况下可能会导致误报。此问题在<font color="#8064a2">负面功能</font>中进一步讨论。

## <font color="#8064a2">NO_THREAD_SAFETY_ANALYSIS</font>
<font color="#8064a2">NO_THREAD_SAFETY_ANALYSIS</font> 是函数或方法上的一个属性，用于关闭该方法的线程安全检查。
它为以下函数提供了一个逃生出口：
	(1) 故意线程不安全，或 (2) 线程安全，但过于复杂，无法通过分析理解。(2) 的原因将在下文的已知限制中描述。
~~~c
class Counter {
  Mutex mu;
  int a GUARDED_BY(mu);

  void unsafeIncrement() NO_THREAD_SAFETY_ANALYSIS { a++; }
}
~~~
与其他属性不同的是，<font color="#8064a2">NO_THREAD_SAFETY_ANALYSIS</font> 不是函数接口的一部分，因此应该放在函数定义中（在 .cc 或 .cpp 文件中），而不是函数声明中（在头文件种）。

## <font color="#8064a2">RETURN_CAPABILITY(c)</font>
_Previously_: `LOCK_RETURNED`

<font color="#8064a2">RETURN_CAPABILITY</font> 是函数或方法上的一个属性，用于声明函数返回对给定功能的引用。它用于注释返回互斥锁的 <font color="#4bacc6">getter</font> 方法。
~~~c
#include "mutex.h"

class MyClass {
private:
  Mutex mu;
  int a GUARDED_BY(mu);

public:
  Mutex* getMu() RETURN_CAPABILITY(mu) { return &mu; }

  // analysis knows that getMu() == mu
  void clear() REQUIRES(getMu()) { a = 0; }
};
~~~

##  <font color="#8064a2">ACQUIRED_BEFORE(…), ACQUIRED_AFTER(…)</font>

<font color="#8064a2">ACQUIRED_BEFORE</font> 和 <font color="#8064a2">ACQUIRED_AFTER</font> 是成员声明的属性，特别是互斥锁或其他功能的声明。这些声明强制执行必须按照特定顺序获取互斥锁，以防止死锁。

~~~c
#include "mutex.h"

Mutex m1;
Mutex m2 ACQUIRED_AFTER(m1);

// Alternative declaration
// Mutex m2;
// Mutex m1 ACQUIRED_BEFORE(m2);

void foo() {
  m2.Lock();
  m1.Lock();  // Warning!  m2 must be acquired after m1.
  m1.Unlock();
  m2.Unlock();
}
~~~


## <font color="#8064a2">CAPABILITY</font>(<<font color="#00b050">string</font>>)
_Previously_: `LOCKABLE`
<font color="#8064a2">CAPABILITY</font> 是类的一个属性，它指定类的对象可用作功能。字符串参数指定错误消息中的功能类型，例如“mutex”。

请参阅上面给出的 Container 示例或 mutex.h 中的 Mutex 类。

## <font color="#8064a2">SCOPED_CAPABILITY</font>
<font color="#8064a2">SCOPED_CAPABILITY</font> 是实现 <font color="#8064a2">RAII</font> 样式锁定的类的一个属性，其中功能在构造函数中获取，并在析构函数中释放。此类类需要特殊处理，因为构造函数和析构函数通过不同的名称引用功能；请参阅下面 mutex.h 中的 MutexLocker 类。

作用域功能被视为在构造时隐式获取并在析构时释放的功能。它们与构造函数或函数上线程安全属性中命名的一组（常规）功能相关联，这些功能通过值返回（使用 C++17 保证的复制省略）。其他成员函数上的获取类型属性被视为适用于该组关联功能，而 RELEASE 则意味着函数会以任何模式释放所有关联功能。

## TRY_ACQUIRE(<<font color="#00b050">bool</font>>, …) TRY_ACQUIRE_SHARED(<<font color="#00b050">bool</font>>, …)

Previously:`EXCLUSIVE_TRYLOCK_FUNCTION`, `SHARED_TRYLOCK_FUNCTION`

这些是尝试获取给定能力的函数或方法的属性，并返回表示成功或失败的布尔值。第一个参数必须为 true 或 false，以指定哪个返回值表示成功，其余参数的解释方式与ACQUIRE相同。有关示例用法，请参阅下面的 mutex.h。
由于分析不支持条件锁定，因此在 try-acquire 函数的返回值上的第一个分支之后，能力被视为已获得。

~~~c
Mutex mu;
int a GUARDED_BY(mu);

void foo() {
  bool success = mu.TryLock();
  a = 0;         // Warning, mu is not locked.
  if (success) {
    a = 0;       // Ok.
    mu.Unlock();
  } else {
    a = 0;       // Warning, mu is not locked.
  }
}
~~~

## <font color="#8064a2">ASSERT_CAPABILITY(…)   ASSERT_SHARED_CAPABILITY(…)</font>
_Previously:_ `ASSERT_EXCLUSIVE_LOCK`, `ASSERT_SHARED_LOCK`
这些是函数或方法的属性，用于断言调用线程已拥有给定的功能，例如通过执行运行时测试并在未拥有该功能时终止。
此注释的存在会导致分析假设在调用注释函数后拥有该功能。有关示例用法，请参阅下面的 mutex.h。

## <font color="#8064a2">GUARDED_VAR  PT_GUARDED_VAR</font>
这些属性的使用已被弃用。

### Warning flags
- `-Wthread-safety`: Umbrella flag which turns on the following:
    
    - `-Wthread-safety-attributes`: Semantic checks for thread safety attributes. //线程安全属性的语义检查。
        
    - `-Wthread-safety-analysis`: The core analysis.
        
    - `-Wthread-safety-precise`: Requires that mutex expressions match precisely. //要求互斥表达式精确匹配。
        
        This warning can be disabled for code which has a lot of aliases.      //对于具有大量别名的代码可以禁用此警告。
        
    - `-Wthread-safety-reference`: Checks when guarded members are passed by reference. //检查受保护成员何时通过引用传递。
