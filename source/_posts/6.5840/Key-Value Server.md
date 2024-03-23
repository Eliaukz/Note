---
title: Key/Value Server
category: "6.5840"
tags:
  - 分布式
---
# 本实验要求实现一个本地的分布式kv-server
要求实现无差错以及有差错的kv-server
无差错较简单不再赘述，只需在每次函数调用时依次调用相关函数即可。
## Key/value server with dropped messages 
在本实验中由于rpc在传输的过程中会发生错误，从而丢失，因此在call失败时，需要重新调用。
这就需要在每次调用时做出记录，这样server端在收到请求时，可以区分出是否重复调用。
这里我是用uuid(Universally Unique Identifier，即通用唯一识别码)，作为唯一标识，
~~在框架代码中给出了生成随机数的代码，使用随机数应该也不会出现错误。~~  
### [uuid](https://zh.wikipedia.org/zh-cn/通用唯一识别码)
UUID是由一组32位数的16进制数字所构成，是故UUID理论上的总数为16^32 = 2^128，约等于3.4 x 10^38。UUID的作用是让分布式系统中的所有元素都能有唯一的辨识信息，而不需要通过中央控制端来做辨识信息的指定。
*UUID由以下几部分的组合：*
1. 当前日期和时间，UUID的第一个部分与时间有关，
2. 时钟序列。
3. 全局唯一的IEEE机器识别号，如果有网卡，从网卡MAC地址获得，没有网卡以其他方式获得。

* 在get操作使不需要对server进行修改，因此get操作无需记录是否重复调用，
* 在append以及put操作中，每次调用PutAppend时生成一个uuid然后作为参数传递给server然后由server端判断是否重复请求。
## question
如果c1调用get(key)后，请求失败，此时c2调用putappend(key,...)修改了c1请求的数据，然后c1重新请求数据，此时的数据一致性是否被破坏？？