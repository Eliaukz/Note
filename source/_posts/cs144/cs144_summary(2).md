---
title: cs144 summary(2)
tags:
  - cs
  - cs144
category: cs144
---
## tcp receiver

这个实验要求实现一个tcp receiver 感觉有难度，

首先要实现一个索引之间的转换，
在发送方和接收方，索引都是用64位整数表示，但是在发送的报文中，索引使用32位整数表示表示，这是为了节省空间
因此需要实现一个32位整数到64位整数之间的转换，

1. wrap函数实现很简单，将64位的absolute seqnum转换成32位的seqnum，只需对`2**32`取余
 ```cpp
Wrap32 Wrap32::wrap( uint64_t n, Wrap32 zero_point )
{
  // Your code here.
  // Convert absolute seqno → seqno
  return zero_point + n;
}
```
2. 将seqnum转换成absolute seqnum，
     首先需要意识到最终的结果应该位于`checkpoint - 2**31` 到 `checkpoint + 2**32`之间，因此可以先计算出seqnum到isn之间的偏移，然后如果偏移大于`2**31` 就将result减去`2**32` 
     ```cpp    
uint64_t Wrap32::unwrap( Wrap32 zero_point, uint64_t checkpoint ) const
{
  // Your code here.
  // Convert seqno → absolute seqno
 
  //求出当前索引距离zero_point的偏移 （32位）
  uint32_t offset = this->raw_value_ - wrap( checkpoint, zero_point ).raw_value_;
  uint64_t result = checkpoint + offset;
  
  //最终结果应该介于checkout-2**31   到  checkpoin + 2**32  之间
  if ( offset > ( 1u << 31 ) && result >= ( 1ul << 32 ) )
    result -= ( 1ul << 32 );
  return result;
}
```

然后就是实现tcp receiver，

这里可以理顺一下接收方的逻辑，首先receiver收到流数据之后，因为数据可能是错误的，重复的，乱序到达的，所以要将数据交给reassembler对数据重排，然后交给writer最终排序好的数据。

不妨查看一下tcpSenderMessage，可以看到是已经简化的报文。

```cpp

/*
 * The TCPSenderMessage structure contains the information sent from a TCP sender to its receiver.
 *
 * It contains four fields:
 *
 * 1) The sequence number (seqno) of the beginning of the segment. If the SYN flag is set, this is the
 *    sequence number of the SYN flag. Otherwise, it's the sequence number of the beginning of the payload.
 *
 * 2) The SYN flag. If set, it means this segment is the beginning of the byte stream, and that
 *    the seqno field contains the Initial Sequence Number (ISN) -- the zero point.
 *
 * 3) The payload: a substring (possibly empty) of the byte stream.
 *
 * 4) The FIN flag. If set, it means the payload represents the ending of the byte stream.
 */

struct TCPSenderMessage
{
  Wrap32 seqno { 0 };
  bool SYN { false };
  Buffer payload {};
  bool FIN { false };

  // How many sequence numbers does this segment use?
  size_t sequence_length() const { return SYN + payload.size() + FIN; }
};

```

当收到数据时，首先查看isn，如果没有收到第一个报文，则不会接受任何数据，在接受完一个报文后，查看syn，如果`syn==true && `  reassembler已经没有缓存数据了，说明已经接受完全部数据，此时可以关闭output。

因此数据部分选择如下
```cpp
  Wrap32 zero_point { 0 };
  uint64_t reassembler_in { 0 };//缓存的字符数
  bool FIN_set { false };
  bool SYN_set { false };
```

```cpp

void TCPReceiver::receive( TCPSenderMessage message, Reassembler& reassembler, Writer& inbound_stream )
{
  // Your code here.
  if ( message.SYN ) {
    SYN_set = true;
    zero_point = message.seqno;
  }
  if ( SYN_set && message.FIN ) {
    FIN_set = true;
  }

  if ( !SYN_set )
    return;

  uint64_t first_index = message.seqno.unwrap( zero_point, inbound_stream.bytes_pushed() ) + message.SYN;
  // cout << "first_index  " << first_index << " payload  " <<  message.payload.length()<< endl;
  reassembler.insert( first_index - 1, message.payload, FIN_set, inbound_stream );
  reassembler_in = reassembler.bytes_pending();
  if ( FIN_set && reassembler_in == 0 )
    inbound_stream.close();
}
```

这里要注意的是，syn和fin也要占据一个字符的位置，因此在计算first_index时要考虑在内。

tcpReceiverMessage 则更加简单，只包含一个可选的ackno，和窗口长度。

```cpp
/*
 * The TCPReceiverMessage structure contains the information sent from a TCP receiver to its sender.
 *
 * It contains two fields:
 *
 * 1) The acknowledgment number (ackno): the *next* sequence number needed by the TCP Receiver.
 *    This is an optional field that is empty if the TCPReceiver hasn't yet received the Initial Sequence Number.
 *
 * 2) The window size. This is the number of sequence numbers that the TCP receiver is interested
 *    to receive, starting from the ackno if present. The maximum value is 65,535 (UINT16_MAX from
 *    the <cstdint> header).
 */

struct TCPReceiverMessage
{
  std::optional<Wrap32> ackno {};
  uint16_t window_size {};
};

```

在接受到数据时，要返回一个ackno和窗口长度， 如果没有收到数据，仅返回窗口长度就足够了。
如果收到数据了，ackno就是期待收到的下一个字符的索引。

```cpp
TCPReceiverMessage TCPReceiver::send( const Writer& inbound_stream ) const
{
  // Your code here.
  uint16_t window_size
    = inbound_stream.available_capacity() > 0xffff ? 0xffff : inbound_stream.available_capacity();

  if ( !SYN_set ) {
    return { optional<Wrap32> {}, window_size };
  }

  bool def_end = false;
  if ( FIN_set && reassembler_in == 0 ) {
    def_end = true;
  }
  // cout << "zero_point  " << zero_point.unwrap(Wrap32(0),0) << endl;
  // cout << "bytes_pushed  " << inbound_stream.bytes_pushed() << "  window_size : " << window_size << endl;
  return TCPReceiverMessage { optional<Wrap32>( zero_point + inbound_stream.bytes_pushed() + SYN_set + def_end ),
                              window_size };
}
```


![](images/Pasted%20image%2020231208190408.png)