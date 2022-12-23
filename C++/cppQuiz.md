# Question-1
According to the C++17 standard, what is the output of this program?
```cpp
template<class T>
void f(T &i) { std::cout << 1; }

template<>
void f(const int &i) { std::cout << 2; }

int main() {
    int i = 42;
    f(i);
}
```

The program is guaranteed to output: 1

Reason:
对于call f(i)，因为i的类型是int，模板参数推导得到T=int
在`template<> void f(const int &i)`模板显式特化中，T=const int，因此没有匹配
在`template <class T> void f(T &i)`中创建了隐式实例化`void f<int>(int&)`

# Question-2
According to the C++17 standard, what is the output of this program?
```cpp
void f(const std::string &) { std::cout << 1; }

void f(const void *) { std::cout << 2; }

int main() {
  f("foo");
  const char *bar = "bar";
  f(bar);
}
```

The program is guaranteed to output: 22

Reason:
一个string literal（本题中的"foo"）是const char[]类型，而非std::string类
如果编译器选择f(const std::string &)，它必须经过用户定义的转换创建一个临时的std::string，相反，它更喜欢f(const void *)，它不需要用户定义的转换

# Question-3
According to the C++17 standard, what is the output of this program?
```cpp
void f(int) { std::cout << 1; }
void f(unsigned) { std::cout << 2; }

int main() {
  f(-2.5);
}
```

The program has a compilation error

Reason:
这种重载是有歧义的. 
call 有两个可行的函数了，让编译器选择一个，其中一个需要比另一个更好，否则程序格式错误。
在这个例子中，它们同样好。 int是有符号的，为什么-2.5 => int 和-2.5 => unsigned一样好?
所有的转换都有一个等级，double => int 和 double => unsigned int都是"floating-integral conversion"类型, 参阅[conv.fpint](https://timsong-cpp.github.io/cppwp/n4659/conv.fpint)

# Question-4
According to the C++17 standard, what is the output of this program?
```cpp
void f(float) { std::cout << 1; }
void f(double) { std::cout << 2; }

int main() {
  f(2.5);
  f(2.5f);
}
```

The program is guaranteed to output: 21

Reason：
floating point literal 2.5的类型是double,  2.5f的类型是float. 

# Question-5
According to the C++17 standard, what is the output of this program?
```cpp
struct A {
    A() { std::cout << "A"; }
};

struct B {
    B() { std::cout << "B"; }
};

class C {
public:
    C() : a(), b() {}

private:
    B b;
    A a;
};

int main() {
    C();
}
```

The program is guaranteed to output: BA

Reason：
成员变量的初始化顺序是由它们的声明顺序决定的，而不是它们在初始化列表中的顺序

# Question-6
According to the C++17 standard, what is the output of this program?
```cpp
int main() {
    for (int i = 0; i < 3; i++)
        std::cout << i;
    for (int i = 0; i < 3; ++i)
        std::cout << i;
}
```

The program is guaranteed to output:012012

Reason：
无论是后增量还是前增量 i，它的值都不会改变，直到循环体执行之后.  

# Question-7
According to the C++17 standard, what is the output of this program?
```cpp
class A {
public:
  void f() { std::cout << "A"; }
};

class B : public A {
public:
  void f() { std::cout << "B"; }
};

void g(A &a) { a.f(); }

int main() {
  B b;
  g(b);
}
```

The program is guaranteed to output: A

Reason：
想要实现子类多态必须将成员函数设置为virtual.
只要A::f()不是virtual的，就会调用A::f()，即使引用或指针指向类型为B的对象.
使用虚函数继承时，当继承类被强转成基类后调用虚函数，调用的还是继承类的虚函数。
而重载方式的继承类被强转成基类再调用重载函数，则调用的是基类的函数。

# Question-8
According to the C++17 standard, what is the output of this program?
```cpp
class A {
public:
  virtual void f() { std::cout << "A"; }
};

class B : public A {
public:
  void f() { std::cout << "B"; }
};

void g(A a) { a.f(); }

int main() {
  B b;
  g(b);
}
```
The program is guaranteed to output: A

Reason：
g(A a)按值获取类型的对象A，而不是通过引用或指针。
这意味着当b通过g(b)调用传递给void g(A a)时,会调用A的拷贝构造函数(即使我们传递的对象b的类型是B).
同时我们会在函数中得到一个全新的类型为A的对象,这通常被称为切片slicing.

# Question-9
According to the C++17 standard, what is the output of this program?
```cpp
int f(int &a, int &b) {
  a = 3;
  b = 4;
  return a + b;
}
int main() {
  int a = 1;
  int b = 2;
  int c = f(a, a);
  std::cout << a << b << c;
}
```
The program is guaranteed to output: 428

Reason：
当f()的两个实参都是a，两个参数都引用同一个变量。这称为别名aliasing。首先将 a 设置为 3，然后将 a 设置为 4，然后返回 4+4。 b 从未被修改

# Question-11
According to the C++17 standard, what is the output of this program?
```cpp
int a;
int main () {
    std::cout << a;
}
```
The program is guaranteed to output: 0

Reason:
因为a具有"静态存储周期",并且没有初始化,因此可以保证零初始化.
如果 a 在 main() 中定义为局部非静态变量，则不会发生这种情况。
Note: int a 具有"静态存储周期",是因为它被定义在命名空间内.而且它不需要在static前缀,static 前缀只会表示internal linkage
而不是"静态存储周期"

# Question-12
According to the C++17 standard, what is the output of this program?
```cpp
int main() {
  static int a;
  std::cout << a;
}
```
The program is guaranteed to output: 0

Reason:
a是静态局部变量，具有"静态存储周期"。它被自动0初始化。
如果删除关键字static，a 会是non-static local variable.不会默认初始化

# Question-13
According to the C++17 standard, what is the output of this program?
```cpp
class A {
public:
  A() { std::cout << "a"; }
  ~A() { std::cout << "A"; }
};

class B {
public:
  B() { std::cout << "b"; }
  ~B() { std::cout << "B"; }
};

class C {
public:
  C() { std::cout << "c"; }
  ~C() { std::cout << "C"; }
};

A a;
int main() {
  C c;
  B b;
}
```

The program is guaranteed to output: acbBCA

Reason:
a具有静态存储舟曲, 由于A()不是constexpr，根据[basic.start.dynamic](https://timsong-cpp.github.io/cppwp/n4659/basic.start.dynamic#5)标准 a的初始化是动态的。有两种可能：
- a在main()被调用之前被初始化，即在c或b被初始化之前。
- a不在main()之前初始化。但是，它保证在同一翻译单元中定义的任何函数被使用之前初始化，也就是在b和c的构造函数被调用之前


