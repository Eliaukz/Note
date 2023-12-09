---
title: cs144 summary(3)
category: cs144
tags:
  - cs144
---
## tcpSender

发送方需要检测发送出去的报文段是否超时，但是该实验要求不使用time库，而是实现一个函数，从而控制时间，因此我们需要自己实现一个time类
tcp超时后会把超时时延扩大为原来的2倍，在收到一个报文后，会把超时时间重置，因此我们需要实现这两个api，此外，为了检测是否超时，需要判断是否在等待报文，以及是否超时，因此我们可以实现这些函数


```cpp
#include <stdint.h>

class Timer
{
private:
  uint64_t initial_RTO_ms;
  uint64_t current_RTO_ms;
  uint64_t time_ms { 0 };
  bool running { false };

public:
  explicit Timer( uint64_t init_ROT ) : initial_RTO_ms( init_ROT ), current_RTO_ms( init_ROT ) {}

  void stop() { running = false; }

  void start()
  {
    time_ms = 0;
    running = true;
  }

  void tick( uint64_t ms_since_last_tick )
  {
    if ( running ) {
      time_ms += ms_since_last_tick;
    }
  }

  bool is_running() { return running; }

  bool is_expired() { return running && time_ms >= current_RTO_ms; }

  void doubleRTO() { current_RTO_ms *= 2; }

  void resetRTO() { current_RTO_ms = initial_RTO_ms; }
};
```

发送方的数据比结构比较复杂

```cpp
  Wrap32 isn_;
  uint64_t initial_RTO_ms_; //超时时长

  bool syn_ { false };
  bool fin_ { false };
  uint64_t retransmit_cnt_ { 0 }; // 连续重复发送的数量

  uint64_t acked_seqno { 0 };
  uint64_t next_seqno { 0 };
  uint64_t window_size { 1 };
  uint64_t outstanding_cnt { 0 }; // 发送中的数量

  std::queue<TCPSenderMessage> outstanding_segments {}; //发送中的队列 发送中未被确认的队列
  std::queue<TCPSenderMessage> queued_segments {};      // 缓存队列 标识还未被发送的队列

  Timer timer { initial_RTO_ms_ };
```

以下是一些必要的且比较简单的函数
```cpp

/* TCPSender constructor (uses a random ISN if none given) */
TCPSender::TCPSender( uint64_t initial_RTO_ms, optional<Wrap32> fixed_isn )
  : isn_( fixed_isn.value_or( Wrap32 { random_device()() } ) ), initial_RTO_ms_( initial_RTO_ms )
{}

uint64_t TCPSender::sequence_numbers_in_flight() const
{
  // Your code here.
  return outstanding_cnt;
}

uint64_t TCPSender::consecutive_retransmissions() const
{
  // Your code here.
  return retransmit_cnt_;
}

optional<TCPSenderMessage> TCPSender::maybe_send()
{
  // Your code here.
  if ( queued_segments.empty() )
    return {};

  if ( !timer.is_running() )
    timer.start();

  TCPSenderMessage msg = queued_segments.front();
  queued_segments.pop();
  return msg;
}

TCPSenderMessage TCPSender::send_empty_message() const
{
  // Your code here.
  return TCPSenderMessage { isn_ + next_seqno, false, {}, false };
}
```




push 函数需要只要当前窗口未满，则从outbounm_stream读取payload_size字节 到msg.payload，然后将其加入到待发送队列。这里细节较多，不再一一赘述

**push函数仅仅向待发送队列中添加数据，maybe_send函数才是真正发送报文的函数 **


```cpp

void TCPSender::push( Reader& outbound_stream )
{
  // Your code here.

  uint64_t currwindow_size = max( window_size, (uint64_t)1 );
  while ( outstanding_cnt < currwindow_size ) {
    TCPSenderMessage msg;
    msg.seqno = Wrap32::wrap( next_seqno, isn_ );

    if ( !syn_ ) {
      msg.SYN = syn_ = true;
      outstanding_cnt += 1;
    }

    uint64_t payload_size = min( TCPConfig::MAX_PAYLOAD_SIZE, (size_t)( currwindow_size - outstanding_cnt ) );

    read( outbound_stream, payload_size, msg.payload ); //从outbounm_stream读取payload_size字节 到msg.payload

    outstanding_cnt += msg.payload.length();

    if ( !fin_ && outbound_stream.is_finished() && outstanding_cnt < currwindow_size ) {
      fin_ = true;
      msg.FIN = true;
      outstanding_cnt += 1;
    }

    if ( msg.sequence_length() == 0 )
      break;

    queued_segments.push( msg );
    outstanding_segments.push( msg );
    next_seqno += msg.sequence_length();

    if ( msg.FIN || outbound_stream.bytes_buffered() == 0 )
      break;
  }
}
```


receive函数需要从收到的报文处读取处window_size，并将其保存，然后如果收到ackno，则检查发送队列，只要满足条件的全部弹出队列，这里每次弹出一个报文时，需要停止定时器，然后如果有新的报文，则开启一个新的定时器。


```cpp

void TCPSender::receive( const TCPReceiverMessage& msg )
{
  // Your code here.
  window_size = msg.window_size;

  if ( msg.ackno.has_value() ) {
    auto ackno = msg.ackno.value().unwrap( isn_, next_seqno );

    if ( ackno > next_seqno )
      return;

    acked_seqno = ackno;

    while ( !outstanding_segments.empty() ) {
      auto front_msg = outstanding_segments.front();
      if ( front_msg.sequence_length() + front_msg.seqno.unwrap( isn_, next_seqno ) <= acked_seqno ) {
        outstanding_segments.pop();
        outstanding_cnt -= front_msg.sequence_length();

        timer.resetRTO();
        if ( !outstanding_segments.empty() ) {
          timer.start();
        }
        retransmit_cnt_ = 0;
      } else {
        break;
      }
    }

    if ( outstanding_segments.empty() )
      timer.stop();
  }
}

```

tick函数是使定时器增加`ms_since_last_tick` 如果报文，就将第一个报文添加到待发送队列中重新发送。同时使计时器的超时时间加倍


```cpp

void TCPSender::tick( const uint64_t ms_since_last_tick )
{
  // Your code here.
  timer.tick( ms_since_last_tick );

  if ( timer.is_expired() ) {
    queued_segments.push( outstanding_segments.front() );
    if ( window_size != 0 ) {
      ++retransmit_cnt_;
      timer.doubleRTO();
    }

    timer.start();
  }
}

```




