# C++知识点
##  push_back() 和emplace_back()有什么区别？
- `emplace_back`避免额外的移动和拷贝操作
> The element uses placement-new to construct the element in-place at the location provided by the container.

[C++性能之战（3）--emplace_back VS push_back](https://blog.csdn.net/u013834525/article/details/104047635)
[cppreference](https://en.cppreference.com/w/cpp/container/vector/emplace_back)示例输出：
> emplace_back:
> I am being constructed.
> 
> push_back:
> I am being constructed.
> I am being moved.


## 智能指针 `std::unique_ptr<T>()`
- C++ prime  智能指针 12.1.5
- `unique_ptr` 独占指针，只能使用`std::move()`转移拥有权

![[Pasted image 20221206215457.png]]
![[Pasted image 20221206215523.png]]


## 移动语义 std::move()
-   C++ Primer 13.6移动语义以及16.2.6
-   右值引用：只能绑定到一个将要销毁的对象
-   右值引用也不过是某个对象的另一个名字而已。
-   `std::move()`显式地将左值转换为右值引用，头文件`utility`中
-   从一个左值`static_cast`到一个右值是允许的
```cpp
std::move(ptr);
```

## 类型转换 static_cast()
-   `static_cast` 使用最多
-   `reinterpret_cast` 任何类型
-   `const_cast` 去掉const
-   `dynamic_cast` 继承关系间，指针和引用的互相转换

`dynamic_cast` 可用于将父类的类型转换成子类，如下：
```cpp
auto value_node = dynamic_cast<TrieNodeWithValue<T> *>(node->get());
```

## 完美转发 std::forward
-   C++ Primer 16.2.7 转发
-   `std::forward`将参数连同类型转发给其他函数，保留const和左值，右值属性。
```cpp
foo(std::forward<Type>(args));
```

## 示例
```cpp
#include <memory>
#include <iostream>
#include <string>

class Entity{
public:
    Entity(){
        std::cout<<"constructor\n";
    }
    void print(){
        std::cout<<"smart boy!\n";
    }
    ~Entity(){
         std::cout<<"destructor\n";
    }
};
void foo(std::unique_ptr<Entity> temp){
    std::cout<<"foo \n";
    temp->print();
}
int main() {
    
    // 1. p.get()慎用，返回p中保存的指针
    // 2. p->mem 等价于 (*p).mem
    // 3. *p 解引用，获得它指向的对象
    std::cout<<"main\n";
    {
       // 创建只支持new
       // 不支持p2(p1)
       // 不支持p2 = p1
       std::unique_ptr<Entity> p(new Entity());
       foo(std::move(p));
    }
    std::cout<<"gg\n";
    return 0;
}
```

## readability-redundant-smartptr-get

在实现Insert函数时，下面代码中`*last_node()->get()`虽然不错，但是并不能通过项目的`.clang-tidy`要求
```cpp
auto new_node = new TrieNodeWithValue<T>(std::move(*last_node()->get()), value);
```

查阅资料之后发现，原因是：`*last_node()->get()`造成了**智能指针的get()方法的可读性冗余**，
改为`**last_node()`即可

Find and remove redundant calls to smart pointer’s `.get()` method.
```cpp
ptr.get()->Foo()  ==>  ptr->Foo()
*ptr.get()  ==>  *ptr
*ptr->get()  ==>  **ptr
if (ptr.get() == nullptr) ... => if (ptr == nullptr) ...
```

# 字典树Trie
-   前瞻：[208. 实现 Trie (前缀树)](https://www.bilibili.com/video/BV1Mr4y167kx?spm_id_from=333.999.0.0&vd_source=e9f1ced96b267a4bc02ec41ca31d850a)

## 原理

要求：Project0中要求实现并发的字典树，给定了三个类：TrieNode、TrieNodeWithValue、Trie
![[Pasted image 20221206221127.png]]

字典树原理如下：trie 中的每个节点都存储一个键的单个字符，并且可以有多个子节点代表不同的可能的下一个字符。当到达键的末尾时，设置一个标志以指示其对应的节点是结束节点。
对于本Project中中间的普通节点由Trie类维护，对于每个单词的最后一个尾字符由TrieNodeWithValue类维护

![[Pasted image 20221206221334.png]]

例如下图中存储着字符串"ab"和"ac"，而字符"b"是个TrieNodeWithValue类，存有value：1
字符"c"是个TrieNodeWithValue类，存有value："val"；字符a只是一个TrieNode类

![[Pasted image 20221206221359.png]]

## 实现
Project0的难点在于实现Trie类中的Insert、Remove、GetValue三个函数

### Insert
沿 Trie 树搜索到键的最后一个字符，过程中如果节点不存在就创建。对于最后一个键的字符分三种情况：
- 如果相应节点不存在，创建一个终结节点 TrieNodeWithValue，插入成功；
- 如果相应节点存在但不是终结节点（通过 is_end_ 判断），将其转化为 TrieNodeWithValue 并把值赋给该节点，该操作不破坏以该键为前缀的后续其它节点（children_ 不变），插入成功；
- 如果相应节点存在且是终结节点，说明该键在 Trie 树存在，规定不能覆盖已存在的值，返回插入失败。

```cpp
template <typename T>
bool Insert(const std::string &key, T value) {
  if (key.empty()) {
    return false;
  }
  latch_.WLock();
  auto node = &root_;
  for (char ch : key.substr(0, key.size() - 1)) {
    auto next = node->get()->GetChildNode(ch);
    if (next == nullptr) {
      next = node->get()->InsertChildNode(ch, std::make_unique<TrieNode>(ch));
    }
    node = next;
  }
  char ch = key[key.size() - 1];
  auto last_node = node->get()->GetChildNode(ch);
  bool result = false;
  if (last_node == nullptr) {
    node->get()->InsertChildNode(ch, std::make_unique<TrieNodeWithValue<T>>(ch, value));
    result = true;
  } else if (!last_node->get()->IsEndNode()) {
    auto new_node = new TrieNodeWithValue<T>(std::move(**last_node), value);
    last_node->reset(new_node);
    result = true;
  } else {
    result = false;
  }
  latch_.WUnlock();
  return result;
}
```

### Remove()
先沿 Trie 树逐字符向下搜索给定键，如果中途发现不存在直接返回 false。找到后，将该节点的 is_end_ 标为 false（此时虽然其值仍存在，但会被我们视为非终结节点）。如果该节点没有子节点，则可以删除。相应地还要向上回溯，如果其 parent 节点在移除该子节点后没有其它子节点，也删除。显然该函数可以有递归和非递归两种实现方式，我这里用非递归方式，在向下搜索时记录路径，回溯时反向遍历。因为使用智能指针，无需手动 delete 删除的 TrieNode。

```cpp
bool Remove(const std::string &key) {
  if (key.empty()) {
    return false;
  }
  latch_.WLock();
  auto node = &root_;
  std::vector<std::unique_ptr<TrieNode> *> path;
  for (char ch : key) {
    path.emplace_back(node);
    auto next = node->get()->GetChildNode(ch);
    if (next == nullptr) {
      latch_.WUnlock();
      return false;
    }
    node = next;
  }
  if (node->get()->HasChildren()) {
    node->get()->SetEndNode(false);
  } else {
    for (int i = path.size() - 1; i >= 0; --i) {
      auto suffix = path[i];
      if ((node->get()->IsEndNode() || node->get()->HasChildren()) && (i < static_cast<int>(key.size() - 1))) {
        break;
      }
      suffix->get()->RemoveChildNode(key[i]);
      node = suffix;
    }
  }
  latch_.WUnlock();
  return true;
}
```

### GetValue()
沿 Trie 树查找，如果键不存在，或者节点中存储的值类型与函数调用的类型 `T` 不一致，将 `*success` 标识设为 false。类型判断的方式是使用 `dynamic_cast`。

```cpp
template <typename T>
T GetValue(const std::string &key, bool *success) {
  if (key.empty()) {
    *success = false;
    return {};
  }
  latch_.RLock();
  auto node = &root_;
  for (char ch : key) {
    auto next = node->get()->GetChildNode(ch);
    if (next == nullptr) {
      *success = false;
      latch_.RUnlock();
      return {};
    }
    node = next;
  }
  auto value_node = dynamic_cast<TrieNodeWithValue<T> *>(node->get());
  T result;
  if (value_node != nullptr) {
    *success = true;
    result = value_node->GetValue();
  } else {
    *success = false;
    result = {};
  }
  latch_.RUnlock();
  return result;
}
```

## 并发
在Project0中 trie 将需要确保插入、删除和获取操作在多线程环境中工作。使用`RwLatch`（BusTub 的读写锁实现）本项目只需要获取根节点的读写锁，即可实现简单的并发控制。`GetValue`函数应该获取根节点上的读锁（通过调用`RwLatch`的`RLock`方法），而`Insert`操作`Remove`应该获取根节点上的写锁（通过调用`RwLatch`的`Wlock`方法），在上面代码中已经实现。