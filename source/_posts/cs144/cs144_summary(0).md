---
tags:
  - cs144
category: cs144
title: cs144 summary(0)
---
**cs144 实验最终要实现一个tcp/ip协议栈，对于理解计算机网络有很大帮助。** 

实验环境推荐使用ubuntu环境，需安装以下包
```bash
 sudo apt update && sudo apt install git cmake gdb build-essential clang \
            clang-tidy clang-format gcc-doc pkg-config glibc-doc tcpdump tshark
```

然后就可以快乐编码了

## webget
首先要实现一个基于tcp socket的webget客户端程序。
socket实现的流程是

![](img/Pasted%20image%2020231208191140.png)

因此我们只需要实现client端，
首先声明一个socket，然后和服务器建立连接，接着向socket写请求报文，最后等待服务器回复报文。


```cpp

void get_URL( const string& host, const string& path )
{
  TCPSocket sock {};
  sock.connect( Address( host, "http" ) );
  const string data_send = "GET " + path + " HTTP/1.1\r\nHost: " + host + "\r\n" + "Connection: close\r\n\r\n";
  sock.write( data_send );
  string buf;
  while ( !sock.eof() ) {
    sock.read( buf );
    std::cout << buf;
    buf.clear();
  }

  sock.close();
}
```

这里只需要注意socket需要关闭，有两种关闭方式
* 请求报文中写入 `Connection: close\r\n`
* 在写入完请求报文后调用`shutdown()`

这样就完成了该实验

## 内存中可靠的字节流

然后要求实现一个可靠的字节流

这里已经给好了要实现的api，所以需要我们自己选的合适的数据结构，完成测试。

先看要实现的函数，发现有string的写入和读出，很容易就想到了`deque`这个容器，支持数据的双向写入写出。
因此数据结构的选择如下
```cpp
 uint64_t capacity_;
  // Please add any additional state to the ByteStream here, and not to the Writer and Reader interfaces.
  bool error_;
  bool end_;
  uint64_t poped_num_;
  std::deque<char> buf_;
```

使用两个bool变量表示是否结束或者出错，使用poped_num_表示已经读出的索引，用buf_缓存未排好序的数据，

这里的实现都比较简单，因此直接给出代码
```cpp
ByteStream::ByteStream( uint64_t capacity )
  : capacity_( capacity ), error_( false ), end_( false ), poped_num_( 0 ), buf_()
{}

void Writer::push( string data )
{
  // Your code here.
  uint64_t len = min( (uint64_t)data.length(), this->available_capacity() );
  for ( uint64_t i = 0; i < len; i++ ) {
    buf_.push_back( data[i] );
  }
}

void Writer::close()
{
  // Your code here.
  this->end_ = true;
}

void Writer::set_error()
{
  // Your code here.
  this->error_ = true;
}

bool Writer::is_closed() const
{
  // Your code here.
  return this->end_;
}

uint64_t Writer::available_capacity() const
{
  // Your code here.
  return capacity_ - this->buf_.size();
}

uint64_t Writer::bytes_pushed() const
{
  // Your code here.
  return poped_num_ + this->buf_.size();
}

string_view Reader::peek() const
{
  // Your code here.
  return string_view( { &buf_.front(), 1 } );
}

bool Reader::is_finished() const
{
  // Your code here.
  return this->end_ && this->buf_.size() == 0;
}

bool Reader::has_error() const
{
  // Your code here.
  return this->error_;
}

void Reader::pop( uint64_t len )
{
  // Your code here.
  len = min( len, this->bytes_buffered() );
  for ( uint64_t i = 0; i < len; i++ ) {
    buf_.pop_front();
  }
  poped_num_ += len;
}

uint64_t Reader::bytes_buffered() const
{
  // Your code here.
  return this->buf_.size();
}

uint64_t Reader::bytes_popped() const
{
  // Your code here.
  return poped_num_;
}

```

![](img/Pasted%20image%2020231208151310.png)