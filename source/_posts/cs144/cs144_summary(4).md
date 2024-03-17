---
title: cs144 summary(4)
category: cs144
tags:
  - cs144
  - 计算机网络
---
根据互联网五层协议

|   层次   |  协议举例 |
|  ---      |  ---  |
|  应用层   | http    |
| 运输层 | tcp/udp |
| 网络层 | ip |
| 链路层 | 802.3 |
| 物理层 | 。。。 |

lab0实现的webget属于应用层，lab123实现的tcp发送方和接受方属于运输层，本实验需要实现的arp（地址解析协议）属于网络层，是ip地址到mac地址之间的转换。

 [rfc::826](https://www.rfc-editor.org/rfc/rfc826) 这里可以查看协议的官方文档。

```cpp
  // Construct a network interface with given Ethernet (network-access-layer) and IP (internet-layer)
  // addresses
  NetworkInterface( const EthernetAddress& ethernet_address, const Address& ip_address );

  // Access queue of Ethernet frames awaiting transmission
  std::optional<EthernetFrame> maybe_send();

  // Sends an IPv4 datagram, encapsulated in an Ethernet frame (if it knows the Ethernet destination
  // address). Will need to use [ARP](\ref rfc::rfc826) to look up the Ethernet destination address
  // for the next hop.
  // ("Sending" is accomplished by making sure maybe_send() will release the frame when next called,
  // but please consider the frame sent as soon as it is generated.)
  void send_datagram( const InternetDatagram& dgram, const Address& next_hop );

  // Receives an Ethernet frame and responds appropriately.
  // If type is IPv4, returns the datagram.
  // If type is ARP request, learn a mapping from the "sender" fields, and send an ARP reply.
  // If type is ARP reply, learn a mapping from the "sender" fields.
  std::optional<InternetDatagram> recv_frame( const EthernetFrame& frame );

  // Called periodically when time elapses
  void tick( size_t ms_since_last_tick );
```

从要实现的函数来看，主要要我们实现对到达的数据报的处理（包括tcp报文，arp请求报文，arp回复报文）

因此我们要实现mac地址的寻址（通过发送arp报文实现）

arp协议要求地址缓存一定时间后就被抛弃
（缓存是为了更快的寻址，抛弃是为了更准确的寻址。--科大 郑老师）

因此我们要储存ip地址到mac地址的映射，还要储存邻居ip地址，以及缓存待发送的数据。

```cpp
  // Ethernet (known as hardware, network-access, or link-layer) address of the interface
  EthernetAddress ethernet_address_; // mac地址

  // IP (known as Internet-layer or network-layer) address of the interface
  Address ip_address_; // ip地址

  std::unordered_map<uint32_t, std::pair<EthernetAddress, size_t>>
    ip_and_mac_; // IP地址与mac地址之间的映射 第二个参数表示已经缓存的时间
  std::unordered_map<uint32_t, size_t> arp_timer_;                             // arp ip地址缓存的时间
  std::unordered_map<uint32_t, std::vector<InternetDatagram>> waited_dagrams_; //等待发送的ip数据报
  std::queue<EthernetFrame> out_frames_;                                       //等待发送的帧
```


同理 send_datagram仅向待发送队列中添加数据报，而不负责发送，maybe_send函数才是真正发送

```cpp

// dgram: the IPv4 datagram to be sent
// next_hop: the IP address of the interface to send it to (typically a router or default gateway, but
// may also be another host if directly connected to the same network as the destination)

// Note: the Address type can be converted to a uint32_t (raw 32-bit IP address) by using the
// Address::ipv4_numeric() method.
void NetworkInterface::send_datagram( const InternetDatagram& dgram, const Address& next_hop )
{
  uint32_t target_ip = next_hop.ipv4_numeric();
  if ( ip_and_mac_.count( target_ip ) ) {
    EthernetFrame eframe { { ip_and_mac_[target_ip].first, ethernet_address_, EthernetHeader::TYPE_IPv4 },
                           serialize( dgram ) };
    out_frames_.push( eframe );
  } else {
    if ( !arp_timer_.count( target_ip ) ) {
      ARPMessage arpmsg;
      arpmsg.opcode = ARPMessage::OPCODE_REQUEST;
      arpmsg.sender_ip_address = ip_address_.ipv4_numeric();
      arpmsg.sender_ethernet_address = ethernet_address_;
      arpmsg.target_ip_address = next_hop.ipv4_numeric();

      EthernetFrame eframe { { ETHERNET_BROADCAST, ethernet_address_, EthernetHeader::TYPE_ARP },
                             serialize( arpmsg ) };
      out_frames_.push( eframe );
      arp_timer_.emplace( next_hop.ipv4_numeric(), 0 );
      waited_dagrams_.insert( { target_ip, { dgram } } );
    } else {
      waited_dagrams_[target_ip].push_back( dgram );
    }
  }
}

```


如果这个目标mac地址未被缓存，那么需要发送arp请求报文，查找目标mac地址，这是一个广播报文。 然后将这个报文添加到待发送的队列中

maybe_send就是从待发送的帧中弹出一个发送出去。

```cpp
optional<EthernetFrame> NetworkInterface::maybe_send()
{
  if ( out_frames_.empty() ) {
    return {};
  }
  auto frame = out_frames_.front();
  out_frames_.pop();
  return frame;
}

```


tick函数处理起来也比较简单，如果缓存超过规定时间，就抛弃

```cpp
// ms_since_last_tick: the number of milliseconds since the last call to this method
void NetworkInterface::tick( const size_t ms_since_last_tick )
{
  static const size_t IP_MAP_TTL = 30000;
  static const size_t ARP_TTL = 5000;

  for ( auto it = ip_and_mac_.begin(); it != ip_and_mac_.end(); ) {
    it->second.second += ms_since_last_tick;
    if ( it->second.second >= IP_MAP_TTL ) {
      it = ip_and_mac_.erase( it );
    } else {
      ++it;
    }
  }

  for ( auto it = arp_timer_.begin(); it != arp_timer_.end(); ) {
    it->second += ms_since_last_tick;
    if ( it->second >= ARP_TTL ) {
      it = arp_timer_.erase( it );
    } else {
      ++it;
    }
  }
}
```


最后是收到一个帧，这个帧可能是广播帧，或者携带报文，因此需要分别处理。

* 如果目的地址不是自己，也不是广播帧，只需要直接抛弃
* 如果携带的是ipv4报文，则取出数据，交付给运输层，
* 如果是arp请求报文，则首先缓存发送方的ip地址，和mac地址，然后发送arp回复报文，
* 如果是arp回复报文，则首先缓存发送方的ip地址，和mac地址，然后将发送方对应的ip地址的待发送的报文全部发送。

```cpp

// frame: the incoming Ethernet frame
optional<InternetDatagram> NetworkInterface::recv_frame( const EthernetFrame& frame )
{
  //接收到的帧的目标地址不是自己也不是广播地址
  if ( frame.header.dst != ethernet_address_ && frame.header.dst != ETHERNET_BROADCAST )
    return {};

  if ( frame.header.type == EthernetHeader::TYPE_IPv4 ) {
    InternetDatagram data;
    if ( parse( data, frame.payload ) )
      return data;
  } else if ( frame.header.type == EthernetHeader::TYPE_ARP ) {
    ARPMessage arpmsg;
    if ( parse( arpmsg, frame.payload ) ) {
      ip_and_mac_.insert( { arpmsg.sender_ip_address, { arpmsg.sender_ethernet_address, 0 } } );
      if ( arpmsg.opcode == ARPMessage::OPCODE_REPLY ) {
        vector<InternetDatagram> datas = waited_dagrams_[arpmsg.sender_ip_address];
        for ( InternetDatagram data : datas ) {
          send_datagram( data, Address::from_ipv4_numeric( arpmsg.sender_ip_address ) );
        }

        waited_dagrams_.erase( arpmsg.sender_ip_address );
      } else if ( arpmsg.opcode == ARPMessage::OPCODE_REQUEST ) {
        if ( arpmsg.target_ip_address == ip_address_.ipv4_numeric() ) {
          ARPMessage reply_msg;
          reply_msg.opcode = ARPMessage::OPCODE_REPLY;
          reply_msg.sender_ip_address = ip_address_.ipv4_numeric();
          reply_msg.sender_ethernet_address = ethernet_address_;
          reply_msg.target_ip_address = arpmsg.sender_ip_address;
          reply_msg.target_ethernet_address = arpmsg.sender_ethernet_address;

          EthernetFrame reply_frame {
            { arpmsg.sender_ethernet_address, ethernet_address_, EthernetHeader::TYPE_ARP },
            serialize( reply_msg ) };
          out_frames_.push( reply_frame );
        }
      }
    }
  }

  return {};
}
```




