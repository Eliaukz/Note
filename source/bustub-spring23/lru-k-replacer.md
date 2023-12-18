---
category: "15445"
title: lru-k-replacer
tags:
  - db
  - cmu15445
---
本文记录一下lru-k-replacer的实现历程

在操作系统课程上，学习过[lru算法](https://leetcode.cn/problems/lru-cache/description/)，但是我没有接触过lru-k因此上网搜了一下，最后发现只需要使用两个list+unordered_map就可以实现了。这是我茅塞顿开。

因此数据结构的设计上，对于lrunode，我保存命中的次数，以及frame_id。具体如下。

```cpp

class LRUKNode {
 public:
  LRUKNode(frame_id_t frame, size_t k) : k_(k), hit_(1), fid_(frame), is_evictable_(false) {}
  auto is_evictable() -> bool { return is_evictable_; }
  auto set_evictable(bool evictable) { is_evictable_ = evictable; }
  auto access() { hit_++; }
  auto should_buffer() -> bool { return hit_ >= k_; }
  auto get_frameid() -> frame_id_t { return fid_; }

 private:
  /** History of last seen K timestamps of this page. Least recent timestamp stored in front. */
  // Remove maybe_unused if you start using them. Feel free to change the member variables as you want.

  // [[maybe_unused]] std::list<size_t> history_;
  // [[maybe_unused]] size_t k_;
  // [[maybe_unused]] frame_id_t fid_;
  // [[maybe_unused]] bool is_evictable_{false};
  size_t k_;
  size_t hit_;
  frame_id_t fid_;
  bool is_evictable_{false};
};
```

lrukreplacer的数据结构设计上，我的设计如下。
锁的使用上，我都是一把大锁完成实验。

```cpp
  std::list<LRUKNode> history_list_;
  std::unordered_map<frame_id_t, std::list<LRUKNode>::iterator> history_map_;
  std::list<LRUKNode> buffer_list_;
  std::unordered_map<frame_id_t, std::list<LRUKNode>::iterator> buffer_map_;
  size_t replacer_size_;
  size_t k_;
  size_t curr_size_{0};
  std::mutex latch_;
```

## Evict

使用两个队列，因此命中小于k次的位于history_list_，驱逐时遵循先进先出策略，每次命中后，判断是否应该移植buffer_list
命中大于等于k次的位于buffer_list，每次命中后，将其移植队列尾部，
至于为什么移至队列尾部而不是头部，我认为驱逐时并非驱逐第一个元素，而是需要判断是否可驱逐，因此需要从头向后遍历。而移至尾部，需要从后向前遍历，设计比较不人性化。

对于evict函数的实现，首先遍历history_list如果有可以驱逐的，则驱逐该队列的元素，否则遍历buffer_list，如果都不能驱逐，则返回false。

注意驱逐之后，需要对size执行--操作。

```cpp
auto LRUKReplacer::Evict(frame_id_t *frame_id) -> bool {
  std::lock_guard<std::mutex> lock(latch_);
  bool flag = false;  // 表示成功删除元素与否
  if (history_map_.size()) {
    // 小于k次  fifo
    auto it = history_list_.begin();
    for (; it != history_list_.end(); it++) {
      if (it->is_evictable()) {
        flag = true;
        *frame_id = it->get_frameid();
        history_list_.erase(it);
        history_map_.erase(*frame_id);

        curr_size_--;
        break;
      }
    }
  }
  if (!flag && buffer_map_.size()) {
    auto it = buffer_list_.begin();
    for (; it != buffer_list_.end(); it++) {
      if (it->is_evictable()) {
        flag = true;
        *frame_id = it->get_frameid();
        buffer_list_.erase(it);
        buffer_map_.erase(*frame_id);

        curr_size_--;
        break;
      }
    }
  }
  return flag;
}
```


## RecordAccess

对于RecordAccess函数，判断是否缓存，缓存命中的操作不再赘述。如果未缓存，需要将其插入history_list队列，如果容量达到限制，需要驱逐一个页面。在进行插入操作。

```cpp

void LRUKReplacer::RecordAccess(frame_id_t frame_id, [[maybe_unused]] AccessType access_type) {
  std::lock_guard<std::mutex> lock(latch_);

  if (history_map_.count(frame_id)) {
    auto it = history_map_[frame_id];
    it->access();
    if (it->should_buffer()) {
      LRUKNode node = *it;
      history_list_.erase(it);
      history_map_.erase(frame_id);

      buffer_list_.emplace_back(node);
      buffer_map_[frame_id] = std::prev(buffer_list_.end());  // 指向最后一个迭代器
    }
    return;
  } else if (buffer_map_.count(frame_id)) {
    auto it = buffer_map_[frame_id];
    it->access();
    auto node = *it;
    buffer_list_.erase(it);
    buffer_map_.erase(frame_id);

    buffer_list_.emplace_back(node);
    buffer_map_[frame_id] = std::prev(buffer_list_.end());
    return;
  } else {
    auto node = LRUKNode(frame_id, k_);
    if (history_map_.size() + buffer_map_.size() >= replacer_size_) {
      // 需要删除一个节点
      frame_id_t res;
      Evict(&res);
    }
    history_list_.emplace_back(node);
    history_map_[frame_id] = std::prev(history_list_.end());
    return;
  }
}
```

## SetEvictable

这个函数比较简单，但是为了方便统计size，因此分四种情况讨论了。

```cpp
void LRUKReplacer::SetEvictable(frame_id_t frame_id, bool set_evictable) {
  std::lock_guard<std::mutex> lock(latch_);

  if (history_map_.count(frame_id)) {
    auto it = history_map_[frame_id];
    if (it->is_evictable() && !set_evictable) {
      it->set_evictable(set_evictable);
      curr_size_--;
    } else if (!it->is_evictable() && set_evictable) {
      it->set_evictable(set_evictable);
      curr_size_++;
    }
  } else if (buffer_map_.count(frame_id)) {
    auto it = buffer_map_[frame_id];
    if (it->is_evictable() && !set_evictable) {
      it->set_evictable(set_evictable);
      curr_size_--;
    } else if (!it->is_evictable() && set_evictable) {
      it->set_evictable(set_evictable);
      curr_size_++;
    }
  }
}

```


## Remove

remove只需判断是否缓存，是否可驱逐即可。


```cpp
void LRUKReplacer::Remove(frame_id_t frame_id) {
  std::lock_guard<std::mutex> lock(latch_);

  if (history_map_.count(frame_id)) {
    auto it = history_map_[frame_id];
    if (it->is_evictable()) {
      history_map_.erase(frame_id);
      history_list_.erase(it);
      curr_size_--;
    }
  } else if (buffer_map_.count(frame_id)) {
    auto it = buffer_map_[frame_id];
    if (it->is_evictable()) {
      buffer_map_.erase(frame_id);
      buffer_list_.erase(it);
      curr_size_--;
    }
  }
}
```


## Size

size函数因为我在操作的过程中记录了size的大小，因此直接返回就OK

```cpp
auto LRUKReplacer::Size() -> size_t {
  std::lock_guard<std::mutex> lock(latch_);
  return curr_size_;
}
```





