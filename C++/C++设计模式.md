
# 单例模式（Singleton Pattern）

## 什么是单例模式？
单例模式是为确保一个类只有一个实例，并为整个系统提供一个全局访问点的一种模式方法。

## 单例特点：
1 在任何情况下，单例类永远只有一个实例存在。
2 单例需要有能力为整个系统提供这一唯一实例。

示例：打印机，任务管理器等。

## 实现一（单线程使用，多线程不安全）
```cpp
class Singleton{
private:
    Singleton(){}
public:
    static Singleton* instance(){
        if(_instance == 0)
            _instance = new Singleton();
        return _instance;
    }
private:
    static Singleton* _instance;
};
Singleton* Singleton::_instance = 0;
```

上面这种实现在单线程环境下是没有问题的，可是多线程下就有问题了。

分析：

1 线程A进入函数instance执行判断语句，这句执行后就挂起了，这时线程A已经认为_instance为NULL，但是线程A还没有创建singleton对象。
2 又有一个线程B进入函数instance执行判断语句，此时同样认为_instance变量为null，因为A没有创建singleton对象。线程B继续执行，创建了一个singleton对象。
3 稍后，线程A接着执行，也创建了一个新的singleton对象。
4 创建了两个对象！

从上面分析可以看出，需要对_instance变量加上互斥锁：

## 实现二（多线程安全，加锁代价高）
```cpp
std::mutex mt;
class Singleton{
private:
    Singleton(){}
public:
    static Singleton* instance(){
        mt.lock();  // 加锁
        if(_instance == 0)
            _instance = new Singleton();
        mt.unlock();  // 解锁
        return _instance;
    }
private:
    static Singleton* _instance;
};
Singleton* Singleton::_instance = 0;
```
上锁后是解决了线程安全问题，但是有些资源浪费。稍微分析一下：每次instance函数调用时候都需要请求加锁，其实并不需要，instance函数只需第一次调用的时候上锁就行了。这时可以用DCLP解决。

## 实现三（双检查锁，由于内存读写导致不安全）
Double-Checked Locking Pattern
```cpp
std::mutex mt;
class Singleton{
private:
    Singleton(){}
public:
    static Singleton* instance(){
        if(_instance == 0){
            mt.lock();
            if(_instance == 0)
                _instance = new Singleton();
            mt.unlock();
        }
        return _instance;
    }
private:
    static Singleton* _instance;
public:
    int atestvalue;
};
Singleton* Singleton::_instance = 0;
```
这个版本很不错，又叫“双重检查”Double-Check。下面是说明：

第一个条件是说，如果实例创建了，那就不需要同步了，直接返回就好了。
不然，我们就开始同步线程。
第二个条件是说，如果被同步的线程中，有一个线程创建了对象，那么别的线程就不用再创建了。
分析
```cpp
_instance = new Singleton();
```
为了执行这句代码，机器需要做三样事儿：

1.singleton对象分配空间。
2.在分配的空间中构造对象
3.使_instance指向分配的空间
遗憾的是编译器并不是严格按照上面的顺序来执行的。可以交换2和3.

将上面三个步骤标记到代码中就是这样：

```cpp
Singleton* Singleton::instance() {
    if (_instance == 0) {
        mt.lock();
        if (_instance == 0) {
            _instance = // Step 3
            operator new(sizeof(Singleton)); // Step 1
            new (_instance) Singleton; // Step 2
        }
        mt.unlock();
    }
    return _instance;
}
```
线程A进入了instance函数，并且执行了step1和step3，然后挂起。这时的状态是_instance不NULL，而_instance指向的内存区没有对象！
线程B进入了instance函数，发现_instance不为null，就直接return_instance了。

## 实现四（C++ 11版本最简洁的跨平台方案）（推荐版本）
Meyers Singleton

局部静态变量不仅只会初始化一次，而且还是线程安全的。
```cpp
class Singleton{
public:
	// 注意返回的是引用
	static Singleton& getInstance(){
		static Singleton value;  //静态局部变量
		return value;
	}
private:
	Singleton() = default;
	Singleton(const Singleton& other) = delete; //禁止使用拷贝构造函数
	Singleton& operator=(const Singleton&) = delete; //禁止使用拷贝赋值运算符
};
int main(){
	Singleton& s1 = Singleton::getInstance();
	cout << &s1 << endl;
	Singleton& s2 = Singleton::getInstance();
	cout << &s2 << endl;
	return 0;
}
```
这种单例被称为Meyers' Singleton。这种方法很简洁，也很完美，但是注意：

1.gcc 4.0之后的编译器支持这种写法。
2.C++11及以后的版本（如C++14）的多线程下，正确。
3.C++11之前不能这么写。

## 实现五（通过C++11提供的call_once）
在C++11中提供一种方法，使得函数可以线程安全的只调用一次。即使用std::call_once和std::once_flag。实现代码如下：
```cpp
std::once_flag flag;
class Singleton{
public:
	static Singleton& getInstance(){
		std::call_once(flag, []() {instance_.reset(new Singleton()); });
		return *instance_;
	}
private:
	static std::unique_ptr<Singleton> instance_;
private:
	Singleton() = default;
	Singleton(const Singleton& other) = delete;
	Singleton& operator=(const Singleton&) = delete;
};
std::unique_ptr<Singleton> Singleton::instance_;
void do_onceflag(){
	Singleton& s = Singleton::getInstance();
	cout << &s << endl;
}
int main(){
	std::thread t1(do_onceflag);
	std::thread t2(do_onceflag);
	t1.join();
	t2.join();
	return 0;
}
```

## 总结
单例模式看着很简单，但实现上却很复杂，如何做到单例模式下多线程安全是个难点。
需要注意的一点是，上面讨论的线程安全指的是getInstance()是线程安全的，假如多个线程都获取类A的对象，如果只是只读操作，完全OK，但是如果有线程要修改，有线程要读取，那么类A自身的函数需要自己加锁防护，不是说线程安全的单例也能保证修改和读取该对象自身的资源也是线程安全的。

