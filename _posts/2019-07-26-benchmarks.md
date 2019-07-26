---
title:      "常见组件性能指标"
date:       2019-07-26 10:03:50 +0800
categories: 技术
tags:
- 架构
- 性能
- 开源组件
---

收集了一些常见开源组件的大概性能技术指标数据，供架构设计时参考：

# 负载均衡
- nginx 大概5万/秒
- LVS 性能是10万级，据说可以达到80万
- F5 的性能是百万级，从200万到800万都有

# 缓存
- MemCache 的读取性能5万左右
- Redis 只使用单核，而 Memcached 可以使用多核，所以平均每一个核上 Redis 在存储小数据时比 Memcached 性能更高。而在 100k 以上的数据中，Memcached 性能要高于 Redis。

# 消息队列
- Kafka 号称百万级 QPS，延迟 ms 以内。
- Kafka 吞吐受 topic 数量的影响特别明显；对比来看，虽然 topic 比较小的时候，RocketMQ 吞吐较小，但是基本非常稳定。

参考[这里](https://blog.csdn.net/liuzhixiong_521/article/details/84849184)和[这里](https://blog.csdn.net/belvine/article/details/80842240)

# 数据库
- etcd v3 写性能万级别，读取性能十万级别。etcd v3可以存储百万到千万级别的key
- etcd v2 只能存储数十万级别的key，主要原因是一致性系统都采用了基于log的复制，log不能无限增长
- zookeeper 写入读取2万以上

# rpc框架
- ice，thrift的 tps 最高，ice 是 thrift 的1.6倍，是 dubbo 的4.4倍，是 grpc 的6倍。详见[这里](https://blog.csdn.net/whzhaochao/article/details/51406539)
