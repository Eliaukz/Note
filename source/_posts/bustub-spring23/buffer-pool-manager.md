---
title: buffer-pool-manager
tags:
  - db
  - cmu15445
category: "15445"
---
这个实验不需要设计数据结构，只需要实现函数功能即可。

本实验是要实现内存页面到磁盘页面的映射，磁盘管理是不要我们是担心的，通过diskmanager来管理，只需要读或写操作即可。

每一个内存页面由一个page表示。

## NewPage

newpage是要从缓冲池中获得一个新的页面，返回pageid。在获取内存页面时，优先从free_list获取。如果没有空页面的话，返回nullptr。

在驱逐页面时，要依次判断是否脏页面，重置内存，最后调用DeallocatePage（？？）。


```cpp

auto BufferPoolManager::NewPage(page_id_t *page_id) -> Page * {
  std::lock_guard<std::mutex> lock(latch_);

  Page *ans = nullptr;
  // if free list or the replacer has place
  if (!free_list_.empty() || replacer_->Size() > 0) {
    frame_id_t fid = 0;
    if (!free_list_.empty()) {
      fid = free_list_.front();
      free_list_.pop_front();
    } else {
      replacer_->Evict(&fid);

      if (pages_[fid].IsDirty()) {
        disk_manager_->WritePage(pages_[fid].GetPageId(), pages_[fid].GetData());
      }
      // erase the page table & reset
      page_table_.erase(pages_[fid].GetPageId());
      pages_[fid].ResetMemory();
      DeallocatePage(pages_[fid].GetPageId());
    }
    // allocate new page id
    page_id_t page_new_id = AllocatePage();
    *page_id = page_new_id;

    pages_[fid].page_id_ = *page_id;
    pages_[fid].pin_count_ = 1;
    pages_[fid].is_dirty_ = false;

    page_table_[*page_id] = fid;

    replacer_->RecordAccess(fid);
    replacer_->SetEvictable(fid, false);

    ans = &pages_[fid];
  }
  return ans;
}
```


## FetchPage

获取给定的内存页面。
如果需要从磁盘中读取的话，需要先获取一个空页面，处理方法与上面相同。

```cpp

auto BufferPoolManager::FetchPage(page_id_t page_id, [[maybe_unused]] AccessType access_type) -> Page * {
  std::lock_guard<std::mutex> lock(latch_);
  Page *res = nullptr;
  if (page_table_.count(page_id)) {
    // buffered
    auto fid = page_table_[page_id];
    pages_[fid].pin_count_++;
    replacer_->RecordAccess(fid);
    replacer_->SetEvictable(fid, false);
    res = &pages_[fid];
  } else if (!free_list_.empty() || replacer_->Size() > 0) {
    frame_id_t fid = 0;
    if (!free_list_.empty()) {
      fid = free_list_.front();
      free_list_.pop_front();
    } else {
      replacer_->Evict(&fid);
      if (pages_[fid].IsDirty()) {
        disk_manager_->WritePage(pages_[fid].GetPageId(), pages_[fid].GetData());
      }
      page_table_.erase(pages_[fid].GetPageId());
      pages_[fid].ResetMemory();
      DeallocatePage(pages_[fid].GetPageId());
    }

    disk_manager_->ReadPage(page_id, pages_[fid].data_);
    pages_[fid].page_id_ = page_id;
    pages_[fid].pin_count_ = 1;
    pages_[fid].is_dirty_ = false;

    page_table_[page_id] = fid;

    replacer_->RecordAccess(fid);
    replacer_->SetEvictable(fid, false);

    res = &pages_[fid];
  }

  return res;
}

```


## UnpinPage

```cpp

auto BufferPoolManager::UnpinPage(page_id_t page_id, bool is_dirty, [[maybe_unused]] AccessType access_type) -> bool {
  std::lock_guard<std::mutex> lock(latch_);
  if (!page_table_.count(page_id)) {
    return false;
  }
  frame_id_t fid = page_table_[page_id];
  if (pages_[fid].pin_count_ <= 0) {
    return false;
  }

  pages_[fid].pin_count_--;
  pages_[fid].is_dirty_ |= is_dirty;
  if (pages_[fid].pin_count_ == 0) {
    replacer_->SetEvictable(fid, true);
  }
  return true;
}
```


## FlushPage

此函数的作用是将内存页面写会物理页面。

```cpp

auto BufferPoolManager::FlushPage(page_id_t page_id) -> bool {
  std::lock_guard<std::mutex> lock(latch_);
  if (!page_table_.count(page_id)) {
    return false;
  }
  frame_id_t fid = page_table_[page_id];
  disk_manager_->WritePage(page_id, pages_[fid].GetData());
  pages_[fid].is_dirty_ = false;
  return true;
}

void BufferPoolManager::FlushAllPages() {
  for (auto [page_id, _] : page_table_) {
    this->FlushPage(page_id);
  }
}

```


## DeletePage

此函数在删除一个内存页面之后要将对应的槽位初始化。

```cpp

auto BufferPoolManager::DeletePage(page_id_t page_id) -> bool {
  std::lock_guard<std::mutex> lock(latch_);

  if (!page_table_.count(page_id)) {
    return true;
  }
  frame_id_t fid = page_table_[page_id];

  if (pages_[fid].pin_count_ > 0) {
    return false;
  }

  replacer_->Remove(fid);
  free_list_.emplace_back(fid);
  // reset the page
  pages_[fid].ResetMemory();
  pages_[fid].page_id_ = INVALID_PAGE_ID;
  pages_[fid].is_dirty_ = false;
  pages_[fid].pin_count_ = 0;
  page_table_.erase(page_id);
  DeallocatePage(page_id);
  return true;
}
```


