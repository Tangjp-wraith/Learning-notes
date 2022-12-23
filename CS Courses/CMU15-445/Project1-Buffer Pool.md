参考资料：
- https://blog.eleven.wiki/posts/cmu15-445-project1-buffer-pool-manager/
- https://ym9omojhd5.feishu.cn/docx/Fk5MdLNJuopJGaxcDHncC9wZnDg


## Extendible Hash Table
参考资料：
- https://en.wikipedia.org/wiki/Extendible_hashing

这部分的难点在于：实现Insert操作
```cpp
template <typename K, typename V>
void ExtendibleHashTable<K, V>::Insert(const K &key, const V &value);
```

首先使用IndexOf函数计算出key所在目录中的位置
```cpp
size_t index = IndexOf(key);
```

使用dir[index]的Find函数检查key是否已经存在

- 如果存在就使用Bucket类的Insert函数更新对应的value值
```cpp
if (dir_[index]->Find(key, val)) {  
  dir_[index]->Insert(key, value);  
  return;  
}
```

- 如果key不存在，则检查所在的桶是否已满。如果已满，则进行分裂操作

分裂操作步骤如下：

如果桶的深度等于哈希表全局的深度，则需要对目录进行扩展，并将整个全局深度加1
```cpp
if (local_depth == global_depth_) {
  size_t dir_size = dir_.size();
  dir_.reserve(2 * dir_size);
  std::copy_n(dir_.begin(), dir_size, std::back_inserter(dir_));
  ++global_depth_;
}
```

创建两个新的桶，并将它们的深度设置为原桶的深度加1
```cpp
auto b0 = std::make_shared<Bucket>(bucket_size_, local_depth + 1);
auto b1 = std::make_shared<Bucket>(bucket_size_, local_depth + 1);
```

遍历原桶中的所有项，根据项的哈希值的最高位为0还是1，将它们分别插入新桶0和新桶1中
```cpp
for (auto &[k, v] : dir_[index]->GetItems()) {
  size_t item_hash = std::hash<K>()(k);
  if (static_cast<bool>(local_mask & item_hash)) {
	b1->Insert(k, v);
  } else {
	b0->Insert(k, v);
  }
}
```

使用 key 的哈希值更新目录中对应的桶的指针
```cpp
for (size_t i = (std::hash<K>()(key) & (local_mask - 1)); i < dir_.size(); i += local_mask) {
  if (static_cast<bool>(i & local_mask)) {
	dir_[i] = b1;
  } else {
	dir_[i] = b0;
  }
}
```
对于这一个 for 循环的条件：遍历所有可能与新 key 相关的桶的位置，每个位置的索引都是通过将新 key 的哈希值与 local_mask 做位运算得到的，这样就可以保证只遍历与新 key 相关的位置。在 for 循环中，对于每个位置 i，都会检查 i 与local_mask的位运算结果是否为 1。如果为 1，就将 dir[i] 更新为 b1；否则，将 dir[i] 更新为 b0。

在这里有个注意点：为什么不像下面一样直接指定而是用循环呢？
```cpp
dir_[(std::hash<K>()(key) & (local_mask - 1))]=b0;
dir_[(std::hash<K>()(key) & (local_mask - 1))+local_mask]=b1;
```
参考：[ 重新安排指针实际上是重新安排指向需要 split 的 bucket 的兄弟指针。需要注意的是，兄弟指针不一定只有两个，而可以有 2^n 次个。例如下面这种情况](https://blog.eleven.wiki/posts/cmu15-445-project1-buffer-pool-manager/#:~:text=%E5%A6%82%E4%BD%95%E9%87%8D%E6%96%B0%E5%AE%89%E6%8E%92%E6%8C%87%E9%92%88%EF%BC%9F%20%E9%87%8D%E6%96%B0%E5%AE%89%E6%8E%92%E6%8C%87%E9%92%88%E5%AE%9E%E9%99%85%E4%B8%8A%E6%98%AF%E9%87%8D%E6%96%B0%E5%AE%89%E6%8E%92%E6%8C%87%E5%90%91%E9%9C%80%E8%A6%81%20split%20%E7%9A%84%20bucket%20%E7%9A%84%E5%85%84%E5%BC%9F%E6%8C%87%E9%92%88%E3%80%82%E9%9C%80%E8%A6%81%E6%B3%A8%E6%84%8F%E7%9A%84%E6%98%AF%EF%BC%8C%E5%85%84%E5%BC%9F%E6%8C%87%E9%92%88%E4%B8%8D%E4%B8%80%E5%AE%9A%E5%8F%AA%E6%9C%89%E4%B8%A4%E4%B8%AA%EF%BC%8C%E8%80%8C%E5%8F%AF%E4%BB%A5%E6%9C%89%202%5En%20%E6%AC%A1%E4%B8%AA%E3%80%82%E4%BE%8B%E5%A6%82%E4%B8%8B%E9%9D%A2%E8%BF%99%E7%A7%8D%E6%83%85%E5%86%B5%EF%BC%9A)

最后，更新index，使用 dir_[index] 的 Insert 函数将 key 和 value 插入到所在的桶中

## LRU-K Replacement Policy


