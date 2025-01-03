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

# Capabilities参考指南

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

- 当一个函数需要操作共享数据时，如果这个操作可能会同时被多个线程访问，就需要对该函数进行加锁操作，确保在任意时刻只有一个线程能访问这部分共享数据。
- `ACQUIRE` 通常用来标记这种需要获取独占锁的函数。


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
Negative Capabilities是一项实验性功能，可通过以下方式启用：
- `-Wthread-safety-negative`: Negative capabilities. Off by default.

# Negative Capabilities
线程安全分析旨在防止竞争条件和死锁。<font color="#8064a2">GUARDED_BY</font> 和 REQUIRES 属性可确保在读取或写入受保护数据之前保留某种能力，从而防止竞争条件；
<font color="#8064a2">EXCLUDES</font> 属性可确保不保留互斥锁，从而防止死锁。

然而，EXCLUDES 是可选属性，并不提供与 <font color="#8064a2">REQUIRES</font> 相同的安全保证。具体来说：
- A function which acquires a capability does not have to exclude it.
    
- A function which calls a function that excludes a capability does not have transitively exclude that capability.、

- 获得某项能力的函数不必排除该能力。

- 调用排除某项能力的函数的函数不必间接排除该能力。

作为结果，EXCLUDES很容易产生 false negatives:
~~~c

class Foo {
  Mutex mu;

  void foo() {
    mu.Lock();
    bar();           // No warning.
    baz();           // No warning.
    mu.Unlock();
  }

  void bar() {       // No warning.  (Should have EXCLUDES(mu)).
    mu.Lock();
    // ...
    mu.Unlock();
  }

  void baz() {
    bif();           // No warning.  (Should have EXCLUDES(mu)).
  }

  void bif() EXCLUDES(mu);
};
~~~
Negative requirements 要求是替代的 EXCLUDES，可提供更强的安全保障。否定要求使用 <font color="#8064a2">REQUIRES</font> 属性与 ! 运算符结合使用，以指示不应保留某项功能。

例如，使用 REQUIRES(!mu) 而不是 EXCLUDES(mu) 将产生相应的警告：

~~~c

class FooNeg{

	Mutex mu;
	
	void foo() REQUIRES(!mu){ //foo() now require !mu
		mu.Lock();
		bar();
		baz();
		mu.Unlock();
	}
	void bar(){
		mu.Lock();        //WARNING! Missing REQUIRES(!mu)
		//...
		mu.Unlock();
	}
	void baz(){
		bif();           //WARNING! Missing REQUIRES(!mu)
	}

	void bif()  REQUIRES(!mu);
	
};
~~~

## 常见问题
---

Q:   我应该把属性放在头文件中，还是放在 .cc/.cpp/.cxx 文件中？
(A): 属性是函数正式接口的一部分，应始终放在标头中，这样它们对包含标头的任何程序都可见。.cpp 文件中的属性在直接翻译单元之外不可见，这会导致误判和误判。

Q:   “这里的每条路径上的互斥锁都没有被锁定？”这是什么意思？
(A):  请参阅下文“No conditionally held locks.”。

# Known Limitations
---

## Lexical scope

线程安全属性包含普通的 C++ 表达式，因此遵循普通的 C++ 作用域规则。具体而言，这意味着互斥体和其他功能必须先声明，然后才能在属性中使用。在单个类中，声明前使用是可以的，因为属性与方法体同时解析。（C++ 将方法体的解析延迟到类的末尾。）但是，不允许在类之间使用声明前使用，如下所示

~~~c
class Foo;

class Bar {
  void bar(Foo* f) REQUIRES(f->mu);  // Error: mu undeclared.
};

class Foo {
  Mutex mu;
};
~~~

## Private Mutexes
良好的软件工程实践要求互斥锁应为私有成员，因为线程安全类使用的锁定机制是其内部实现的一部分。但是，私有互斥锁有时会泄漏到类的公共接口中。线程安全属性遵循正常的 C++ 访问限制，因此如果 mu 是 c 的私有成员，那么在属性中写入 c.mu 就是错误的。

一种解决方法是（滥用）使用 <font color="#8064a2">RETURN_CAPABILITY</font> 属性来为私有互斥锁提供公共名称，而不实际公开底层互斥锁。例如：

~~~c
class MyClass {
private:
  Mutex mu;

public:
  // For thread safety analysis only.  Does not need to be defined.
  Mutex* getMu() RETURN_CAPABILITY(mu);

  void doSomething() REQUIRES(mu);
};

void doSomethingTwice(MyClass& c) REQUIRES(c.getMu()) {
  // The analysis thinks that c.getMu() == c.mu
  c.doSomething();
  c.doSomething();
}
~~~

在上面的例子中，doSomethingTwice() 是一个外部例程，需要锁定 c.mu，由于 mu 是私有的，因此无法直接声明。不建议使用这种模式，因为它违反了封装，但有时它是必要的，尤其是在向现有代码库添加注释时。解决方法是将 getMu() 定义为伪 getter 方法，它仅用于线程安全分析。

## No conditionally held locks.

分析必须能够确定在每个程序点上是否持有锁。因此，可能持有锁的代码部分将生成虚假警告（误报）。例如：

~~~c
void foo() {
  bool b = needsToLock();
  if (b) mu.Lock();
  ...  // Warning!  Mutex 'mu' is not held on every path through here.
  if (b) mu.Unlock();
}
~~~

## No checking inside constructors and destructors.
目前，分析不会对构造函数或析构函数进行任何检查。换句话说，每个构造函数和析构函数都被视为已使用 NO_THREAD_SAFETY_ANALYSIS 注释。原因是在初始化期间，通常只有一个线程可以访问正在初始化的对象，因此初始化受保护成员而不获取任何锁是安全的（也是常见的做法）。析构函数也是如此。

理想情况下，分析将允许初始化正在初始化或销毁的对象内的受保护成员，同时仍然对其他所有内容实施通常的访问限制。然而，这在实践中很难实施，因为在基于指针的复杂数据结构中，很难确定封闭对象拥有哪些数据。

## No inlining.
线程安全分析严格地在过程内进行，就像普通的类型检查一样。它仅依赖于函数声明的属性，并且不会尝试内联任何方法调用。因此，以下代码将不起作用：

~~~c
class MutexUnlocker {
  Mutex* mu;

public:
  MutexUnlocker(Mutex* m) RELEASE(m) : mu(m)  { mu->Unlock(); }
  ~MutexUnlocker() ACQUIRE(mu) { mu->Lock(); }
};

Mutex mutex;
void test() REQUIRES(mutex) {
  {
    MutexUnlocker munl(&mutex);  // unlocks mutex
    doSomeIO();
  }                              // Warning: locks munl.mu
}
~~~

MutexUnlocker 类旨在与 mutexLocker 类相对应，后者在 mutex.h 中定义。但是，它不起作用，因为分析不知道 munl.mu == mutex。<font color="#8064a2">SCOPED_CAPABILITY</font> 属性处理 MutexLocker 的别名，但只针对该特定模式。

ACQUIRED_BEFORE(…) 和 ACQUIRED_AFTER(…) 支持仍处于试验阶段。
ACQUIRED_BEFORE(…) 和 ACQUIRED_AFTER(…) 目前正在 -Wthread-safety-beta 标志下进行开发。


## mutex.h
线程安全分析可以与任何线程库一起使用，但它要求将线程 API 包装在具有适当注释的类和方法中。以下代码提供了 mutex.h 作为示例；应填写这些方法以调用适当的底层实现。

~~~c
#ifndef THREAD_SAFETY_ANALYSIS_MUTEX_H
#define THREAD_SAFETY_ANALYSIS_MUTEX_H

// Enable thread safety attributes only with clang.
// The attributes can be safely erased when compiling with other compilers.
#if defined(__clang__) && (!defined(SWIG))
#define THREAD_ANNOTATION_ATTRIBUTE__(x)   __attribute__((x))
#else
#define THREAD_ANNOTATION_ATTRIBUTE__(x)   // no-op
#endif

#define CAPABILITY(x) \
  THREAD_ANNOTATION_ATTRIBUTE__(capability(x))

#define SCOPED_CAPABILITY \
  THREAD_ANNOTATION_ATTRIBUTE__(scoped_lockable)

#define GUARDED_BY(x) \
  THREAD_ANNOTATION_ATTRIBUTE__(guarded_by(x))

#define PT_GUARDED_BY(x) \
  THREAD_ANNOTATION_ATTRIBUTE__(pt_guarded_by(x))

#define ACQUIRED_BEFORE(...) \
  THREAD_ANNOTATION_ATTRIBUTE__(acquired_before(__VA_ARGS__))

#define ACQUIRED_AFTER(...) \
  THREAD_ANNOTATION_ATTRIBUTE__(acquired_after(__VA_ARGS__))

#define REQUIRES(...) \
  THREAD_ANNOTATION_ATTRIBUTE__(requires_capability(__VA_ARGS__))

#define REQUIRES_SHARED(...) \
  THREAD_ANNOTATION_ATTRIBUTE__(requires_shared_capability(__VA_ARGS__))

#define ACQUIRE(...) \
  THREAD_ANNOTATION_ATTRIBUTE__(acquire_capability(__VA_ARGS__))

#define ACQUIRE_SHARED(...) \
  THREAD_ANNOTATION_ATTRIBUTE__(acquire_shared_capability(__VA_ARGS__))

#define RELEASE(...) \
  THREAD_ANNOTATION_ATTRIBUTE__(release_capability(__VA_ARGS__))

#define RELEASE_SHARED(...) \
  THREAD_ANNOTATION_ATTRIBUTE__(release_shared_capability(__VA_ARGS__))

#define RELEASE_GENERIC(...) \
  THREAD_ANNOTATION_ATTRIBUTE__(release_generic_capability(__VA_ARGS__))

#define TRY_ACQUIRE(...) \
  THREAD_ANNOTATION_ATTRIBUTE__(try_acquire_capability(__VA_ARGS__))

#define TRY_ACQUIRE_SHARED(...) \
  THREAD_ANNOTATION_ATTRIBUTE__(try_acquire_shared_capability(__VA_ARGS__))

#define EXCLUDES(...) \
  THREAD_ANNOTATION_ATTRIBUTE__(locks_excluded(__VA_ARGS__))

#define ASSERT_CAPABILITY(x) \
  THREAD_ANNOTATION_ATTRIBUTE__(assert_capability(x))

#define ASSERT_SHARED_CAPABILITY(x) \
  THREAD_ANNOTATION_ATTRIBUTE__(assert_shared_capability(x))

#define RETURN_CAPABILITY(x) \
  THREAD_ANNOTATION_ATTRIBUTE__(lock_returned(x))

#define NO_THREAD_SAFETY_ANALYSIS \
  THREAD_ANNOTATION_ATTRIBUTE__(no_thread_safety_analysis)

// Defines an annotated interface for mutexes.
// These methods can be implemented to use any internal mutex implementation.
class CAPABILITY("mutex") Mutex {
public:
  // Acquire/lock this mutex exclusively.  Only one thread can have exclusive
  // access at any one time.  Write operations to guarded data require an
  // exclusive lock.
  void Lock() ACQUIRE();

  // Acquire/lock this mutex for read operations, which require only a shared
  // lock.  This assumes a multiple-reader, single writer semantics.  Multiple
  // threads may acquire the mutex simultaneously as readers, but a writer
  // must wait for all of them to release the mutex before it can acquire it
  // exclusively.
  void ReaderLock() ACQUIRE_SHARED();

  // Release/unlock an exclusive mutex.
  void Unlock() RELEASE();

  // Release/unlock a shared mutex.
  void ReaderUnlock() RELEASE_SHARED();

  // Generic unlock, can unlock exclusive and shared mutexes.
  void GenericUnlock() RELEASE_GENERIC();

  // Try to acquire the mutex.  Returns true on success, and false on failure.
  bool TryLock() TRY_ACQUIRE(true);

  // Try to acquire the mutex for read operations.
  bool ReaderTryLock() TRY_ACQUIRE_SHARED(true);

  // Assert that this mutex is currently held by the calling thread.
  void AssertHeld() ASSERT_CAPABILITY(this);

  // Assert that is mutex is currently held for read operations.
  void AssertReaderHeld() ASSERT_SHARED_CAPABILITY(this);

  // For negative capabilities.
  const Mutex& operator!() const { return *this; }
};

// Tag types for selecting a constructor.
struct adopt_lock_t {} inline constexpr adopt_lock = {};
struct defer_lock_t {} inline constexpr defer_lock = {};
struct shared_lock_t {} inline constexpr shared_lock = {};

// MutexLocker is an RAII class that acquires a mutex in its constructor, and
// releases it in its destructor.
class SCOPED_CAPABILITY MutexLocker {
private:
  Mutex* mut;
  bool locked;

public:
  // Acquire mu, implicitly acquire *this and associate it with mu.
  MutexLocker(Mutex *mu) ACQUIRE(mu) : mut(mu), locked(true) {
    mu->Lock();
  }

  // Assume mu is held, implicitly acquire *this and associate it with mu.
  MutexLocker(Mutex *mu, adopt_lock_t) REQUIRES(mu) : mut(mu), locked(true) {}

  // Acquire mu in shared mode, implicitly acquire *this and associate it with mu.
  MutexLocker(Mutex *mu, shared_lock_t) ACQUIRE_SHARED(mu) : mut(mu), locked(true) {
    mu->ReaderLock();
  }

  // Assume mu is held in shared mode, implicitly acquire *this and associate it with mu.
  MutexLocker(Mutex *mu, adopt_lock_t, shared_lock_t) REQUIRES_SHARED(mu)
    : mut(mu), locked(true) {}

  // Assume mu is not held, implicitly acquire *this and associate it with mu.
  MutexLocker(Mutex *mu, defer_lock_t) EXCLUDES(mu) : mut(mu), locked(false) {}

  // Same as constructors, but without tag types. (Requires C++17 copy elision.)
  static MutexLocker Lock(Mutex *mu) ACQUIRE(mu) {
    return MutexLocker(mu);
  }

  static MutexLocker Adopt(Mutex *mu) REQUIRES(mu) {
    return MutexLocker(mu, adopt_lock);
  }

  static MutexLocker ReaderLock(Mutex *mu) ACQUIRE_SHARED(mu) {
    return MutexLocker(mu, shared_lock);
  }

  static MutexLocker AdoptReaderLock(Mutex *mu) REQUIRES_SHARED(mu) {
    return MutexLocker(mu, adopt_lock, shared_lock);
  }

  static MutexLocker DeferLock(Mutex *mu) EXCLUDES(mu) {
    return MutexLocker(mu, defer_lock);
  }

  // Release *this and all associated mutexes, if they are still held.
  // There is no warning if the scope was already unlocked before.
  ~MutexLocker() RELEASE() {
    if (locked)
      mut->GenericUnlock();
  }

  // Acquire all associated mutexes exclusively.
  void Lock() ACQUIRE() {
    mut->Lock();
    locked = true;
  }

  // Try to acquire all associated mutexes exclusively.
  bool TryLock() TRY_ACQUIRE(true) {
    return locked = mut->TryLock();
  }

  // Acquire all associated mutexes in shared mode.
  void ReaderLock() ACQUIRE_SHARED() {
    mut->ReaderLock();
    locked = true;
  }

  // Try to acquire all associated mutexes in shared mode.
  bool ReaderTryLock() TRY_ACQUIRE_SHARED(true) {
    return locked = mut->ReaderTryLock();
  }

  // Release all associated mutexes. Warn on double unlock.
  void Unlock() RELEASE() {
    mut->Unlock();
    locked = false;
  }

  // Release all associated mutexes. Warn on double unlock.
  void ReaderUnlock() RELEASE() {
    mut->ReaderUnlock();
    locked = false;
  }
};

#ifdef USE_LOCK_STYLE_THREAD_SAFETY_ATTRIBUTES
// The original version of thread safety analysis the following attribute
// definitions.  These use a lock-based terminology.  They are still in use
// by existing thread safety code, and will continue to be supported.

// Deprecated.
#define PT_GUARDED_VAR \
  THREAD_ANNOTATION_ATTRIBUTE__(pt_guarded_var)

// Deprecated.
#define GUARDED_VAR \
  THREAD_ANNOTATION_ATTRIBUTE__(guarded_var)

// Replaced by REQUIRES
#define EXCLUSIVE_LOCKS_REQUIRED(...) \
  THREAD_ANNOTATION_ATTRIBUTE__(exclusive_locks_required(__VA_ARGS__))

// Replaced by REQUIRES_SHARED
#define SHARED_LOCKS_REQUIRED(...) \
  THREAD_ANNOTATION_ATTRIBUTE__(shared_locks_required(__VA_ARGS__))

// Replaced by CAPABILITY
#define LOCKABLE \
  THREAD_ANNOTATION_ATTRIBUTE__(lockable)

// Replaced by SCOPED_CAPABILITY
#define SCOPED_LOCKABLE \
  THREAD_ANNOTATION_ATTRIBUTE__(scoped_lockable)

// Replaced by ACQUIRE
#define EXCLUSIVE_LOCK_FUNCTION(...) \
  THREAD_ANNOTATION_ATTRIBUTE__(exclusive_lock_function(__VA_ARGS__))

// Replaced by ACQUIRE_SHARED
#define SHARED_LOCK_FUNCTION(...) \
  THREAD_ANNOTATION_ATTRIBUTE__(shared_lock_function(__VA_ARGS__))

// Replaced by RELEASE and RELEASE_SHARED
#define UNLOCK_FUNCTION(...) \
  THREAD_ANNOTATION_ATTRIBUTE__(unlock_function(__VA_ARGS__))

// Replaced by TRY_ACQUIRE
#define EXCLUSIVE_TRYLOCK_FUNCTION(...) \
  THREAD_ANNOTATION_ATTRIBUTE__(exclusive_trylock_function(__VA_ARGS__))

// Replaced by TRY_ACQUIRE_SHARED
#define SHARED_TRYLOCK_FUNCTION(...) \
  THREAD_ANNOTATION_ATTRIBUTE__(shared_trylock_function(__VA_ARGS__))

// Replaced by ASSERT_CAPABILITY
#define ASSERT_EXCLUSIVE_LOCK(...) \
  THREAD_ANNOTATION_ATTRIBUTE__(assert_exclusive_lock(__VA_ARGS__))

// Replaced by ASSERT_SHARED_CAPABILITY
#define ASSERT_SHARED_LOCK(...) \
  THREAD_ANNOTATION_ATTRIBUTE__(assert_shared_lock(__VA_ARGS__))

// Replaced by EXCLUDE_CAPABILITY.
#define LOCKS_EXCLUDED(...) \
  THREAD_ANNOTATION_ATTRIBUTE__(locks_excluded(__VA_ARGS__))

// Replaced by RETURN_CAPABILITY
#define LOCK_RETURNED(x) \
  THREAD_ANNOTATION_ATTRIBUTE__(lock_returned(x))

#endif  // USE_LOCK_STYLE_THREAD_SAFETY_ATTRIBUTES

#endif  // THREAD_SAFETY_ANALYSIS_MUTEX_H
~~~