# C++构造函数

## 定义
C++是一种面向对象编程语言，在设计时，将具有共同特征的对象抽象成类，在使用时，通过将类实例化得到不同的对象。创建一个实例对象的时候，就需要用到构造函数。
创建一个实例有两种方法：
(1) 通过初始化成员变量的来构造实例对象，这时候调用普通构造函数。
(2) 通过复制已有的实例化对象来构造新的实例对象，这时候调用拷贝构造函数。

构造函数的函数名与类名相同，没有返回值，可以通过不同的参数值实现重载。定义Object类如下：
```C++
class Object
{
public:
    int d;
    Object() { std::cout << "default constructor" << std::endl; }  // 默认构造函数，调用时可不提供参数的构造函数，即Object obj
    Object(int k) { d = k; std::cout << "defined constructor" << std::endl; }  // 普通构造函数
    Object(const Object& obj) { d = obj.d; std::cout << "copy constructor" << std::endl; }  // 拷贝构造函数，定义时需要以当前类对象的const引用作为入参，即const Object& obj
};
```
定义Object对象如下：
```C++
Object obj1 = Object();  // default constructor
Object obj2 = Object(3);  // defined constructor
Object obj3 = obj1;  // copy constructor，注意不是重载=的赋值函数
```

## 合成
当程序中没有显示定义默认和拷贝构造函数时，逻辑上编译器会自己定义默认和拷贝构造函数。编译器自己定义的构造函数可以分为无用的(trivial)和有用的(non-trivial)。编译器通过实现优化，仅在有限的几种情况下，编译器才会将定义的有用的构造函数真正合成出来。

### 默认构造函数
C++默认构造函数有两个重要作用：

一是提供成员变量初始化的方法，默认构造函数并不会申请成员变量所需的内存空间，也不会主动给成员变量赋初值，而只是提供了给成员变量赋初值的一种途径。

二是在virtual场景下，完成虚函数表指针和虚基类指针的设置，使得在运行时可以通过指针或引用访问到正确的函数实现，呈现出多态的特性。

鉴于此，C++编译器会在以下场景合成non-trivial的默认构造函数：

1、virtual场景，即**定义了虚函数或虚继承**的情形，此时需要合成默认构造函数，以正确设置虚函数表指针和虚基类指针。

2、组合或继承场景下，**被组合的类或被继承的类中显式定义了默认构造函数**，需要在合成的默认构造函数中递归调用被组合类或被继承类的构造函数。

**注：**
非virtual场景，C++构造函数类似于Python的__init__方法，区别在于C++编译器会在构造函数中自动插入父类的构造函数，而Python需要通过super显式调用父类的构造函数。

### 拷贝构造函数
C语言中的拷贝是bitwise拷贝（内存上存的是啥就拷贝啥），C++如果用bitwise拷贝，可能会导致程序出问题，例如：

1、在virtual场景下，如果用派生类对象初始化父类对象（Base base_obj = derived_obj 对象模型会发生切割），bitwise拷贝使得base_obj的vptr指向Derived的虚函数表，导致程序异常。

2、在非virtual场景下，如果类中有指针类型的成员对象，bitwise拷贝使得两个对象的成员变量指向同一处内存（浅拷贝），析构时由于内存被重复释放导致程序崩溃，此时需要定义拷贝构造函数使得拷贝结果位于一处新内存上（深拷贝）。

因此，C++编译器会在以下场景合成non-trivial的拷贝构造函数：

1、virtual场景，即**定义了虚函数或虚继承**的情形，此时需要合成拷贝构造函数，保证在复制过程（特别是用派生类对象初始化基类对象）中正确设置虚函数表指针和虚基类指针。

2、组合或继承场景下，**被组合的类或被继承的类中显式定义了拷贝构造函数**，需要在合成的默认构造函数中递归调用被组合类或被继承类的拷贝构造函数。

## 应用
设计模式中有一种单例模式，意思就是在程序中有且仅有唯一的实例。要实现单例模式，需要解决两个问题：

(1) 创建一个实例：解决存在性问题

(2) 禁止多个实例：解决唯一性问题

定义SingleInstance类如下：
```C++
class SingleInstance;
```

先看问题(2)，要避免出现多个实例，首先要禁止通过已有实例复制得到新实例，其次要防止反复调用普通或默认构造函数创建多个对象。通过将SingleInstance类的默认构造函数、拷贝构造函数都声明为private，可以实现这两个目标。
```C++
class SingleInstance
{
private:
    SingleInstance() { std::cout << "default constructor" << std::endl; };
    SingleInstance(const SingleInstance& obj) { std::cout << "copy constructor" << std::endl; };
};
```

再看问题(1)，为解决唯一性问题，默认构造函数被设为private，无法通过auto ins = SingleInstance()的方式来创造实例。为解决这个问题，需要在SingleInstance类中增加一个public方法，在这个方法中调用默认构造函数来完成实例的创建。
```C++
class SingleInstance
{
private:
    SingleInstance() { std::cout << "default constructor" << std::endl; };
    SingleInstance(const SingleInstance& obj) { std::cout << "copy constructor" << std::endl; };
public:
    static SingleInstance& GetInstance() {
        static auto ins = SingleInstance();
        return ins;
    }
};
```

上述代码有两个问题：

一是GetInstance是普通成员方法，需要在实例化后才能调用，而这个方法需要在实例创建之前调用（这个方法就是要创建一个实例），出现矛盾，因此需要声明为静态成员方法；

二是多次调用GetInstance将返回多个ins实例，破坏了唯一性，为保证多次调用只产生一次，需要将ins声明为静态变量。

代码修改后如下：
```C++
class SingleInstance
{
private:
    SingleInstance() {};
    SingleInstance(const SingleInstance&) {};
public:
    static SingleInstance& GetInstance() {
        static auto ins = SingleInstance();
        return ins;
    }
};
```

测试单例模式如下：
```C++
int main()
{
    auto& ins = SingleInstance::GetInstance();  // 这里必须定义为引用，&号不能去掉，否则会报错
    auto& ins1 = SingleInstance::GetInstance();
//    auto ins2 = SingleInstance::GetInstance();  // 这句话的含义是要通过GetInstance()得到实例ins_tmp，然后调用拷贝构造函数创建ins，因此会报错
}
```

可以看到，最后只打印了一条记录：
> default constructor
