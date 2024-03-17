---
tags:
  - cs144
  - 计算机网络
category: cs144
title: cs144 summary(1)
---
## 重排器

此实验要求我们实现一个字符流的重排器，其实还是比较简单，但是我有一个地方理解错了，因此调试了很久，最后还请教了别人，才做出来。

首先看两个要实现的api

```cpp
// Insert a new substring to be reassembled into a ByteStream.
   void insert( uint64_t first_index, std::string data,
                bool is_last_substring, Writer& output );
   // How many bytes are stored in the Reassembler itself?
   uint64_t bytes_pending() const;
```

第二个比较好理解，就是返回缓存的字符数量

第一个是将到来的字符流插入到我们的数据结构中，然后将已经排好序的字符流写入到output中。

依次介绍一下四个参数
* first_index 字符流的第一个字符在流中的索引
* data 到来的字符流
* is_last_substring 是否最后一个字符流
* output 排好序的字符流写入的数据结构


而判断是否已经完全接受完数据而关闭output需要两方面的要求
当is_last_substring为true时，说明这个是发送方发送的最后一个子串，但是因为我们的缓存有容量限制，所以这个子串不一定能完全存入我们的缓存中，因此当这个标志到来时，我们只能得到最后一个字符的下标，只有我们的缓存的字符串下标大于等于这个数据时，才可以关闭output

这里提到了容量的问题，这就是我遇到的坑，我的理解是，重排器有一个容量，写入的output有一个容量，导致我的测试无论如何都无法通过，最后在别人的帮助下，我认识到所谓的容量时output的容量，或者说outout的available_capacipy，才是重排器的容量。

这里需要缓存，所以我选择了一个非常简单的数据结构--哈希表(索引为key，字符为value)，这也导致我后面的速度测试无法通过，不知道是不是这里的原因。

```cpp
  uint64_t unpop_index_;
  uint64_t last_index_;
  bool end_;
  std::unordered_map<uint64_t, std::string> buf_;
```
构造函数如下
```cpp
  Reassembler() : unpop_index_( 0 ), last_index_( 0 ), end_( false ), buf_() {}
```

只有当 `end_ == true && unpop_index >= last_index_ ` 时才可以关闭output。

如果没有犯我说的容量问题的错误的话，这个应该是比较好实现的。

要注意的一点是，一部分排好序的字符串需要立即push进output中，

首先需要判断是否只最后一个子串，如果是的话需要将end_置为true，将last_index_更新为新数据。

然后获得缓冲的可用容量，这个也是当前重排器的容量。

然后获得将要插入的数据的起始索引，和终止索引。虽然字符串与buf_的相对位置关系很多，但是可以使用一个循环，就可以实现。

插入数据之后就是将排好序的数据写入到output中

这里使用了一个while循环，同时得到了字符串和更新了unpop_index_的值。最后将得到的字符串写入output中，就实现了该函数


```cpp

void Reassembler::insert( uint64_t first_index, string data, bool is_last_substring, Writer& output )
{

  // Your code here.
  if ( is_last_substring ) {
    last_index_ = first_index + data.length();
    end_ = true;
  }
  uint64_t avail_cap_ = output.available_capacity();

  for ( uint64_t index = max( first_index, unpop_index_ );
        index < first_index + data.length() && index < unpop_index_ + avail_cap_;
        index++ ) {
    buf_[index] = data[index - first_index];
  }

  uint64_t len = 0;
  string s = "";
  while ( buf_.count( unpop_index_ ) && avail_cap_ > 0 ) {
    s += buf_[unpop_index_];
    buf_.erase( unpop_index_ );
    len++;
    unpop_index_++;
    avail_cap_--;
  }

  output.push( s );
  if ( end_ && unpop_index_ >= last_index_ ) {
    output.close();
  }
}
```

字符串的相对位置大概如下所示

![](/img/cs144/Pasted%20image%2020231208155130.png)


第二个函数要求返回buf_的已经缓存的字符数量

```cpp
uint64_t Reassembler::bytes_pending() const
{
  // Your code here.
  return buf_.size();
}

```

![](/img/cs144/Pasted%20image%2020231208155204.png)