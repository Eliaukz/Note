---
title: cs144 summary(5)
category: cs144
tags:
  - cs144
---
## IP router

最后一个实验是要完成一个路由转发协议

但是路由与转发还是有区别的
* 路由：源主机到目的主机之间的寻址（全局的过程）
* 转发：路由器输入端到输出端的寻址（局部的过程）

而且要实现一个最长前缀匹配协议（要明确 mac地址是与接口一一对应）

因此数据结构的选择需要实现mac地址与接口的对应这里查看构造函数不难想到

```cpp
  struct Item
  {
    uint32_t route_prefix;           // IP地址
    uint8_t prefix_length;           //前缀长度
    std::optional<Address> next_hop; //下一跳地址
    size_t interface_num {};         //接口的索引
  };

  std::vector<Item> routing_table_ {}; //路由表
  std::vector<Item>::iterator longest_prefix_match_( uint32_t dst_ip );
  static int match_length( uint32_t src_ip, uint32_t tgt_ip, uint8_t tgt_len );
```

还需要实现最长前缀匹配算法，这里实验要求并没有说明要用什么方式实现，因此我选择了最简单的方式--遍历

```cpp

std::vector<Router::Item>::iterator Router::longest_prefix_match_( uint32_t dst_ip )
{
  auto res = routing_table_.end();
  auto max_length = -1;
  for ( auto it = routing_table_.begin(); it != routing_table_.end(); ++it ) {
    auto len = match_length( dst_ip, it->route_prefix, it->prefix_length );
    if ( len > max_length ) {
      max_length = len;
      res = it;
    }
  }

  return res;
}

int Router::match_length( uint32_t src_ip, uint32_t tgt_ip, uint8_t tgt_len )
{
  if ( tgt_len == 0 )
    return 0;

  if ( tgt_len > 32 )
    return -1;

  uint8_t const len = 32U - tgt_len;
  src_ip = src_ip >> len;
  tgt_ip = tgt_ip >> len;
  return src_ip == tgt_ip ? tgt_len : -1;
}

```



至于路由函数只需要遍历所有的接口，然后从接口收到帧，然后进行最长前缀匹配，从对应的端口分发出去。需要注意帧的ttl字段，只有还存活的帧才会被分发。


```cpp

void Router::route()
{
  for ( auto& current_interface : interfaces_ ) {
    auto received_dgram = current_interface.maybe_receive();
    if ( received_dgram.has_value() ) {
      auto& dgram = received_dgram.value();
      if ( dgram.header.ttl > 1 ) {
        dgram.header.ttl--;

        dgram.header.compute_checksum();
        auto dst_ip = dgram.header.dst;
        auto it = longest_prefix_match_( dst_ip );
        if ( it != routing_table_.end() ) {
          auto& target_interface = interface( it->interface_num );
          target_interface.send_datagram( dgram, it->next_hop.value_or( Address::from_ipv4_numeric( dst_ip ) ) );
        }
      }
    }
  }
}
```



