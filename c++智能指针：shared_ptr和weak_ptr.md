## **引言**

C\+\+中经常需要new一个对象，开辟一个内存空间，返回一个指针来操作这个内存。使用完毕之后，需要通过delete来释放内存空间。如果内存没有释放，那这块内存将无法再利用，导致内存泄漏。为降低人为疏忽，C++ 11的新特性中引入了三种智能指针，来自动化地管理内存资源：

**unique_ptr:** 管理的资源唯一的属于一个对象，但是支持将资源移动给其他unique_ptr对象。当拥有所有权的unique_ptr对象析构时，资源即被释放。

**shared_ptr:** 管理的资源被多个对象共享，内部采用引用计数跟踪所有者的个数。当最后一个所有者被析构时，资源即被释放。

**weak_ptr:** 与shared_ptr配合使用，虽然能访问资源但却不享有资源的所有权，不影响资源的引用计数。有可能资源已被释放，但weak_ptr仍然存在。因此每次访问资源时都需要判断资源是否有效。
本文主要在循环引用的场景下探讨shard_ptr和weak_ptr原理。

## **循环引用**
shared_ptr通过引用计数的方式管理内存，当进行拷贝或赋值操作时，每个shared_ptr都会记录有多少个其他的shared_ptr指向相同的对象，当引用计数为0时，内存将被自动释放。
```c++
auto p = make_shared<int>(10); // 创建一个名为p的shared_ptr，指向一个取值为10的int型对象，这个数值10的引用计数为1，只有p
auto q(p); // 创建一个名为q的shared_ptr，并用p初始化，此时p和q指向同一个对象，此时数值10的引用计数为2
```
当对shared_ptr赋予新值，或被销毁时，引用计数会递减。
```c++
auto r = make_shared<int>(20); // 创建一个名为r的shared_ptr，指向一个取值为20的int型对象，这个数值20的引用计数为1，只有r
r = q; // 对r赋值，让r指向数值10。此时数值10的引用计数加1为3，数值20的引用计数减1位0，数值20的内存将被自动释放
```
通常情况下shared_ptr可以正常运转，但是在循环引用的场景下，shared_ptr无法正确释放内存。循环引用，顾名思义，A指向B，B指向A，在表示双向关系时，是很可能出现这种情况的，例如：
```c++
#include <iostream>
#include <memory>
using namespace std;

class Son;

class Father {
public:
    shared_ptr<Son> son_;
    Father() {
        cout << __FUNCTION__ << endl;
    }
    ~Father() {
        cout << __FUNCTION__ << endl;
    }
};

class Son {
public:
    shared_ptr<Father> father_;
    Son() {
        cout << __FUNCTION__ << endl;
    }
    ~Son() {
        cout << __FUNCTION__ << endl;
    }
};

int main()
{
    auto son = make_shared<Son>();
    auto father = make_shared<Father>();
    son->father_ = father;
    father->son_ = son;
    cout << "son: " << son.use_count() << endl;
    cout << "father: " << father.use_count() << endl;
    return 0;
}
```
程序的执行结果如下：
>Son
>
>Father
>
>son: 2
>
>father: 2

可以看到，程序分别执行了Son和Father的构造函数，但是没有执行析构函数，出现了内存泄漏。

## **shared_ptr原理**
shared_ptr实际上是对裸指针进行了一层封装，成员变量除了裸指针外，还有一个引用计数，它记录裸指针被引用的次数（有多少个shared_ptr指向这同一个裸指针），当引用计数为0时，自动释放裸指针指向的资源。影响引用次数的场景包括：构造、赋值、析构。基于三个最简单的场景，实现一个demo版shared_ptr如下（实现既不严谨也不安全，仅用于阐述原理）：
```c++
#include <iostream>
#include <memory>
using namespace std;

template<typename T>
class SharedPtr {
public:
    int* counter;  // 引用计数，用指针表示，多个SharedPtr之间可以同步修改
    T* resource;  // 裸指针

    SharedPtr(T* resc = nullptr) {  // 构造函数
        cout << __PRETTY_FUNCTION__ << endl;
        counter = new int(1);
        resource = resc;
    }

    SharedPtr(const SharedPtr& rhs) {  // 拷贝构造函数
        cout << __PRETTY_FUNCTION__ << endl;
        resource = rhs.resource;
        counter = rhs.counter;
        ++*counter;
    }

    SharedPtr& operator=(const SharedPtr& rhs) {  // 拷贝赋值函数
        cout << __PRETTY_FUNCTION__ << endl;
        --*counter;  // 原来指向的资源的引用计数减1
        if (*counter == 0) {
            delete counter;
            delete resource;
        }

        resource = rhs.resource;
        counter = rhs.counter;
        ++*counter;  // 新指向的资源的引用计数加1
    }

    ~SharedPtr() {  // 析构函数
        cout << __PRETTY_FUNCTION__ << endl;
        --*counter;
        if (*counter == 0) {
            delete counter;
            delete resource;
        }
    }

    int use_count() {
        return *counter;
    }
};
```
在循环引用示例中，用到了make_shared函数：
```c++
auto son = make_shared<Son>();  // 新建一个Son对象，返回指向这个Son对象的指针
```
此用法等价于：
```c++
auto son_ = new Son();  // 新建一个Son对象，返回指向这个对象的指针son_
shared_ptr<Son> son(son_);  // 创建一个管理son_的shared_ptr
```
代入SharedPtr的实现来分析示例中main函数的执行过程，可以得到：
```c++
auto son = make_shared<Son>();  // 调用构造函数，son.counter=1
auto father = make_shared<Father>();  // 调用构造函数，father.counter=1
son->father_ = father;  // 调用赋值函数，son.counter=2
father->son_ = son;  // 调用赋值函数，father.counter=2
```
当main函数执行完时，执行析构函数，此时由于son.counter=1，father.couter=1，不满足if条件，不会实行delete命令完成资源释放，导致内存泄漏。

## **weak_ptr原理**
为解决循环引用的问题，仅使用shared_ptr是无法实现的。堡垒无法从内部攻破的时候，需要借助外力，于是有了weak_ptr，字面意思是弱指针。为啥叫弱呢？shared_ptr A被赋值给shared_ptr B时，A的引用计数加1；shared_ptr A被赋值给weak_ptr C时，A的引用计数不变。引用力度不够强，不足以改变引用计数，所以就弱了（个人理解，有误请指正）。
weak_ptr在使用时，是与shared_ptr绑定的。基于SharedPtr实现来实现demo版的WeakPtr，并解决循环引用的问题，全部代码如下：
```c++
#include <iostream>
#include <memory>
using namespace std;

template<typename T>
class SharedPtr {
public:
    int* counter;
    int* weakref;
    T* resource;

    SharedPtr(T* resc = nullptr) {
        cout << __PRETTY_FUNCTION__ << endl;
        counter = new int(1);
        weakref = new int(0);
        resource = resc;
    }

    SharedPtr(const SharedPtr& rhs) {
        cout << __PRETTY_FUNCTION__ << endl;
        resource = rhs.resource;
        counter = rhs.counter;
        ++*counter;
    }

    SharedPtr& operator=(const SharedPtr& rhs) {
        cout << __PRETTY_FUNCTION__ << endl;
        --*counter;
        if (*counter == 0) {
            delete counter;
            delete resource;
        }

        resource = rhs.resource;
        counter = rhs.counter;
        ++*counter;
    }

    ~SharedPtr() {
        cout << __PRETTY_FUNCTION__ << endl;
        --*counter;
        if (*counter == 0) {
            delete counter;
            delete resource;
        }
    }

    int use_count() {
        return *counter;
    }
};

template<typename T>
class WeakPtr {
public:
    T* resource;

    WeakPtr(T* resc = nullptr) {
        cout << __PRETTY_FUNCTION__ << endl;
        resource = resc;
    }

    WeakPtr& operator=(SharedPtr<T>& ptr) {
        cout << __PRETTY_FUNCTION__ << endl;
        resource = ptr.resource;
        ++*ptr.weakref;  // 赋值时引用计数counter不变，改变弱引用计数weakref
    }

    ~WeakPtr() {
        cout << __PRETTY_FUNCTION__ << endl;
    }
};

class Son;

class Father {
public:
    SharedPtr<Son> son_;
    Father() {
        cout << __PRETTY_FUNCTION__ << endl;
    }
    ~Father() {
        cout << __PRETTY_FUNCTION__ << endl;
    }
};

class Son {
public:
    WeakPtr<Father> father_;  // 将SharedPtr改为WeakPtr
    Son() {
        cout << __PRETTY_FUNCTION__ << endl;
    }
    ~Son() {
        cout << __PRETTY_FUNCTION__ << endl;
    }
};

int main()
{
    auto son_ = new Son();  // 创建一个Son对象，返回指向Son对象的指针son_
    auto father_ = new Father();  // 创建一个Father对象，返回指向Father对象的指针father_
    SharedPtr<Son> son(son_);  // 调用SharedPtr构造函数：son.counter=1, son.weakref=0
    SharedPtr<Father> father(father_);  // 调用SharedPtr构造函数：father.counter=1, father.weakref=0
    son.resource->father_ = father;  // 调用WeakPtr赋值函数：father.counter=1, father.weakref=1
    father.resource->son_ = son;  // 调用SharedPtr赋值函数：son.counter=2, son.weakref=0
    cout << "son: " << son.use_count() << endl;
    cout << "father: " << father.use_count() << endl;
    return 0;
}
```
代码执行结果如下：
>WeakPtr<T>::WeakPtr(T*) [with T = Father]
>
>Son::Son()
>
>SharedPtr<T>::SharedPtr(T*) [with T = Son]
>
>Father::Father()
>
>SharedPtr<T>::SharedPtr(T*) [with T = Son]
>
>SharedPtr<T>::SharedPtr(T*) [with T = Father]
>
>WeakPtr<T>& WeakPtr<T>::operator=(SharedPtr<T>&) [with T = Father]
>
>SharedPtr<T>& SharedPtr<T>::operator=(const SharedPtr<T>&) [with T = Son]
>
>son: 2
>
>father: 1
>
>SharedPtr<T>::~SharedPtr() [with T = Father]
>
>Father::~Father()
>
>SharedPtr<T>::~SharedPtr() [with T = Son]
>
>SharedPtr<T>::~SharedPtr() [with T = Son]
>
>Son::~Son()
>
>WeakPtr<T>::~WeakPtr() [with T = Father]

可以看到Son对象和Father对象均被析构，内存泄漏的问题得到解决。析构过程解读如下：
>SharedPtr<T>::~SharedPtr() [with T = Father]  # 析构father，由于father.couter=1，减1后执行delete father_
>
>Father::\~Father()  # 析构father_，执行\~Father()，进一步析构成员变量
>
>SharedPtr<T>::~SharedPtr() [with T = Son]  # 析构SharedPtr<Son>，此时son.couter减1，son.counter=1
>
>SharedPtr<T>::~SharedPtr() [with T = Son]  # 析构son，由于son.counter=1，减1后执行delete son_
>
>Son::\~Son()  # 析构son_，执行\~Son()，进一步析构成员变量
>
>WeakPtr<T>::~WeakPtr() [with T = Father]  # 析构WeakPtr<Father>

## **总结**
1. 尽量使用智能指针管理资源申请与释放，减少人为new和delete误操作和考虑不周的问题。
2. 使用make_shared来创建shared_ptr，如果先new一个对象，再用这个对象的裸指针构造一个shared_ptr指针，可能出现问题。shared_ptr会自动释放资源，如果再手动delete，释放两次那就挂了。
