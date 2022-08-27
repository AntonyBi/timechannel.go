# timechannel

## Introduction

目前分布式ID生成算法的主流依然是snowflake，比较知名的实现有twitter官方版本、sonyflake、美团Leaf。但snowflake在工程实现上，存在一些比较棘手的问题，如时钟回拨、位如何分配等。

故个人重新设计了一个高可靠的`轻量级`实现，命名为`timechannel`，同时也避免了时钟回拨问题、支持更灵活的位分配。在本地4C16G VM中压测，将序列号分配12bit，QPS压测结果达50w/s，可满足绝大多数应用场景。

## Comparison

|     | timechannel | leaf snowflake | leaf segment |
| --- | --- | --- | --- |
| 依赖组件 | SDK<br>Redis集群（推荐sentinel） | SDK<br>Leaf Server<br>Zookeeper集群 | SDK<br>Leaf Server<br>Mysql集群（半同步） |
| 实现复杂度 | 简单，不足500行源码 | 简单，依赖twitter snowflake的实现 | 相对复杂 |
| 性能  | 高   | 高   | 中   |
| 支持worker上限 | 无限，需配置多space | 1024 | 无限，配置不同应用 |
| 支持bit位分配配置 | 是   | 否   | 否   |
| 时钟回拨问题 | 无   | 有   | 无   |
| 潜在风险 | 运行时强依赖Redis | 时钟回拨会造成服务暂停 | 运行时强依赖Mysql |

## Design

请查看：[timechannel.java README](https://github.com/antonybi/timechannel.java#readme)

## Quick Start

### 项目集成

待补充

### 配置介绍

在config.yml中可配置以下参数：

| 配置项 | 含义  | 默认值 | 备注  |
| --- | --- | --- | --- |
| guid.redis.host | redis地址 | -   | 【必填】 |
| guid.redis.port | redis端口 | 6379 |     |
| guid.space | 空间  | 0   | 当实例数接近频道总数，可以按业务场景划分到不同空间 |
| guid.ttl | 每次申请授权续期的时长 | 10m | 此值越大，频道回收时间会越慢，但Redis异常时应用可持续工作越久 |
| guid.group.id | 频道分组编号，从0开始计数 | 0   | 多机房建议独立部署redis集群，并分配不同group |
| guid.bits.group | 频道分组bit位数 | 0   | 与redis集群数量保持一致 |
| guid.bits.channel | 频道占用的bit位数，默认共2048个 | 11  | 同一个space下，该值保持一致 |
| guid.bits.sequence | 序列号占用的bit位数，默认速度1024/ms | 10  | 同一个space下，该值保持一致 |

注： channel和sequence所占用的bit位不能多于22位，避免时间戳的值域空间太小。

### 运行检查

```
# 查看channel的总数
zcard space:0:expiryTime:channel

# 查看channel的租约过期时间
zscore space:0:expiryTime:channel 0

# 查看目前可用的channel总数，时间戳换成当前时间
zcount space:0:expiryTime:channel 0 1661079389000

# 查看正在被占用的channel
zrangebyscore space:0:expiryTime:channel 1661079389000 9999999999999 WITHSCORES

# 根据上条命令查到的频道号查看最后一次申请日志
get space:0:channel:0:log
```

## License

Released under the [MIT License](https://github.com/antonybi/timechannel.go/blob/master/LICENSE)