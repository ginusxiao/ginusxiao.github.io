# 提纲
[toc]

## 直接在reactor中返回响应(直接在CQLConnectionContext::HandleCall中返回响应，响应中不带数据)
### Cassandra read测试
reactor数目|connection数目|threads数目|OPS|时延
---|---|---|---|---
1|1|1|1.2w - 1.7w|0.06 - 0.09ms
1|1|2|2.5w|0.08ms
1|1|4|4.8w|0.08ms
1|1|8|7.3w|0.11ms
1|1|16|9.5w - 11.5w|0.14 - 0.17ms
1|1|32|11w - 14w|0.22 - 0.28ms
1|1|64|14w|0.46ms
1|1|128|14w - 17w|0.75ms - 0.89ms
1|1|256|14.8w - 17.8w|1.49ms - 1.67ms
1|1|384|14.1w - 18.7w|2.05ms - 2.71ms
1|2|8|7.5w|0.11ms
1|2|16|11w|0.15ms
1|2|32|12.5w|0.25ms
1|2|64|16w|0.4ms
1|2|128|18w|0.7ms
1|2|256|20w|1.24ms
1|2|384|22w - 25w|1.5 - 1.7ms
1|4|64|16w - 21w|0.31ms - 0.37ms
1|4|128|20w - 28w|0.45ms - 0.63ms
1|4|256|17.2w - 29w|0.89ms - 1.48ms
1|8|64|10w - 30w(测试时间比较长)|0.21ms - 0.62ms
1|8|128|15w - 26w|0.48ms - 0.84ms
1|8|256|20w - 33w|0.78ms - 1.29ms
1|16|64|17w - 20w|0.31ms - 0.38ms
1|16|128|22w|0.6ms
1|16|256|16w - 31w|0.82ms - 1.6ms

从结果来看：
- 1个reactor，1个connection，1个测试线程的情况下，OPS约为1.5w左右，时延约为0.08ms左右；
- 1个reactor，1个connection的情况下，随着测试线程数目的增加，OPS在局部范围内呈现近线性增长，但是随着测试线程数目的增加，请求时延也会增加；
- 1个reactor的情况下，对于相同的测试线程数目，随着connection数目的增加，OPS也会增加；
- 1个reactor的情况下，最大OPS可达33w左右；

reactor数目|connection数目|threads数目|OPS|时延|备注
---|---|---|---|---|---
2|2|64|20w - 23w|0.27ms - 0.32ms|只有1个CQLServer reactor busy
2|2|128|23.5w - 26w|0.49ms - 0.54ms|只有1个CQLServer reactor busy
2|2|256|20w - 22w|1.16ms - 1.27ms|只有1个CQLServer reactor busy
2|4|64|22w - 25w|0.25ms - 0.29ms|2个CQLServer reactor busy，但是负载不均衡
2|4|128|25.7w - 29w|0.44ms - 0.5ms|2个CQLServer reactor busy，但是负载不均衡
2|4|256|29w - 31.5w|0.81ms - 0.88ms|2个CQLServer reactor busy，但是负载不均衡
2|4|384|30w - 33.4ww|1.15ms - 1.31ms|2个CQLServer reactor busy，但是负载不均衡
2|8|64|26w - 29.4w|0.22ms - 0.25ms|2个CQLServer reactor busy，负载基本均衡
2|8|128|26w - 30.5w|0.41ms - 0.49ms|2个CQLServer reactor busy，负载基本均衡
2|8|256|28w - 32.5w|0.79ms - 0.91ms|2个CQLServer reactor busy，负载基本均衡
2|16|64|22.3w - 26.8w|0.24ms - 0.29ms|2个CQLServer reactor busy，负载基本均衡
2|16|128|24.6w - 30w|0.46ms - 0.52ms|2个CQLServer reactor busy，负载基本均衡
2|16|256|26w - 30.4w|0.84ms - 0.99ms|2个CQLServer reactor busy，负载基本均衡

从测试结果来看：
- 2个reactor的情况下，对于相同的测试线程数目，随着connection数目的增加，OPS也会增加；
- 2个reactor的情况下，最大OPS可达33w左右；
- 相较于1个reactor的情况：
    - 相同connection数目 + 相同threads数目的情况下，2个reactor的OPS更佳，时延也更低； 

reactor数目|connection数目|threads数目|OPS|时延|备注
---|---|---|---|---|---
4|4|64|28w - 30w|0.21ms - 0.23ms|3个CQLServer reactor busy，但是负载不均衡
4|4|128|36.5w - 38.7w|0.33ms - 0.35ms|4个CQLServer reactor busy，负载基本均衡
4|4|256|40w - 42.4w|0.6ms - 0.65ms|4个CQLServer reactor busy，负载基本均衡
4|4|384|41w - 43.6w|0.88ms - 0.93ms|4个CQLServer reactor busy，负载基本均衡
4|8|64|29w - 33w|0.19ms - 0.22ms|4个CQLServer reactor busy，但是负载不均衡，其中2个负载较高
4|8|128|36.4w - 40w|0.32ms - 0.35ms|4个CQLServer reactor busy，但是负载不均衡，其中3个负载较高
4|8|256|37.4w - 42.7w|0.6ms - 0.68ms|4个CQLServer reactor busy，但是负载不均衡，其中3个负载较高
4|16|64|31w|0.2m|4个CQLServer reactor busy，负载基本均衡
4|16|128|38.4w - 41w|0.31ms - 0.33ms|4个CQLServer reactor busy，负载基本均衡
4|16|256|39.2w - 42.3w|0.58ms - 0.65ms|4个CQLServer reactor busy，负载基本均衡
4|32|64|25w - 27.3w|0.23ms - 0.25ms|4个CQLServer reactor busy，负载基本均衡
4|32|128|33.2w - 35.6w|0.36ms - 0.38ms|4个CQLServer reactor busy，负载基本均衡
4|32|256|39w - 42w|0.61ms - 0.66ms|4个CQLServer reactor busy，负载基本均衡

从测试结果来看：
- 4个reactor的情况下，对于相同的threads数目，随着connection数目的增加(除connection数目为32之外)，OPS也会增加；
- 4个reactor的情况下，最大OPS可达43.6w左右；
- 相较于2个reactor的情况：
    - 相同connection数目 + 相同threads数目的情况下，4个reactor的OPS更佳，时延也更低； 
    

reactor数目|connection数目|threads数目|OPS|时延|备注
---|---|---|---|---|---
8|8|64|41w|0.15ms|7个CQLServer reactor busy，但是负载不均衡
8|8|128|40.7w - 45.3w|0.28ms - 0.32ms|6个CQLServer reactor busy，但是负载不均衡
8|8|256|49.2w - 54.3w|0.47ms - 0.51ms|6个CQLServer reactor busy，但是负载不均衡
8|8|384|49.3w - 54.5w|0.72ms - 0.78ms|6个CQLServer reactor busy，但是负载不均衡
8|16|64|36w - 39.6w|0.16ms - 0.18ms|8个CQLServer reactor busy，但是负载不均衡
8|16|128|46.3w - 52.5w|0.24ms - 0.28ms|8个CQLServer reactor busy，但是负载不均衡
8|16|256|60w - 62w|0.41ms - 0.42ms|8个CQLServer reactor busy，但是负载不均衡
8|32|64|33.5w|0.19ms|8个CQLServer reactor busy，但是负载不均衡
8|32|128|44.4w - 46.6w|0.27ms - 0.29ms|8个CQLServer reactor busy，但是负载不均衡
8|32|256|54.1w - 59.2w|0.43ms - 0.48ms|8个CQLServer reactor busy，但是负载不均衡

从测试结果来看：
- 8个reactor的情况下，对于相同的threads数目，随着connection数目的增加(除connection数目为32之外)，OPS也会增加；
- 8个reactor的情况下，最大OPS可达62w左右；
- 8个reactor的情况下，无论connection数目是8，16还是32，CQLServer reactor都无法实现负载均衡；
- 相较于4个reactor的情况：
    - 相同connection数目 + 相同threads数目的情况下，8个reactor的OPS更佳，时延也更低； 


reactor数目|connection数目|threads数目|OPS|时延|备注
---|---|---|---|---|---
16|16|64|42w|0.15ms|12个CQLServer reactor busy，但是负载不均衡
16|16|128|53.8w - 61.3w|0.21ms - 0.24ms|11个CQLServer reactor busy，但是负载不均衡
16|16|256|61w - 67.2w|0.38ms - 0.42ms|10个CQLServer reactor busy，但是负载不均衡
16|16|384|66.1w - 72.8w|0.52ms - 0.58ms|11个CQLServer reactor busy，但是负载不均衡
16|16|512|64.6w - 74w|0.58ms - 0.78ms|10个CQLServer reactor busy，但是负载不均衡
16|32|64|32w - 38w|0.17ms - 0.2ms|15个CQLServer reactor busy，但是负载不均衡
16|32|128|46w - 51.4w|0.25ms - 0.28ms|14个CQLServer reactor busy，但是负载不均衡
16|32|256|66w|0.38ms|15个CQLServer reactor busy，但是负载不均衡
16|32|384|70w|0.54ms|14个CQLServer reactor busy，但是负载不均衡
16|32|512|70w|0.72ms|15个CQLServer reactor busy，但是负载不均衡
16|16|16|17w|0.09ms|9个CQLServer reactor busy，但是负载不均衡
16|32|32|25w|0.13ms|13个CQLServer reactor busy，但是负载不均衡
16|64|64|31w|0.2ms|16个CQLServer reactor busy，但是负载不均衡
16|64|128|39w|0.33ms|16个CQLServer reactor busy，但是负载不均衡
16|64|256|54w|0.47ms|16个CQLServer reactor busy，但是负载不均衡
16|64|384|62w|0.62ms|16个CQLServer reactor busy，但是负载不均衡
16|64|512|65w|0.79ms|16个CQLServer reactor busy，但是负载不均衡
16|128|128|38w|0.34ms|16个CQLServer reactor busy，但是负载不均衡


从测试结果来看：
- 16个reactor的情况下，对于相同的threads数目，随着connection数目的增加(除connection数目为64之外)，OPS也会增加；
- 16个reactor的情况下，最大OPS可达74w左右；
- 16个reactor的情况下，无论connection数目是16,32还是64，CQLServer reactor都无法实现负载均衡；
- 相较于8个reactor的情况：
    - 相同connection数目 + 相同threads数目的情况下，16个reactor的OPS更佳，时延也更低； 

## Redis不同测试工具测试性能对比
### Redis-benchmark set测试(正常流程)
测试工具为Redis官方的Redis-benchmark，除了-n和-c参数以外，其它参数采用默认值。

reactor数目|Redis-benchmark实例数目|每个Redis-benchmark实例中connection数目|OPS
---|---|---|---|---
1|1|1|0.14w
1|1|2|0.46w
1|1|4|1.1w
1|1|8|2.8w
1|1|16|4.1w
1|1|32|4.6w
1|1|64|5w
1|1|128|5.4w
1|1|256|6w
1|1|384|5.2w
1|2|1|0.31w
1|2|2|1.08w
1|2|4|2.69w
1|2|8|3.99w
1|2|16|4.68w
1|2|32|5.12w
1|2|64|5.55w
1|2|128|5.42w
1|2|256|5.76w
1|4|1|0.94w
1|4|2|2.62w
1|4|4|3.9w
1|4|8|4.58w
1|4|16|5.06w
1|4|32|5.43w
1|4|64|5.77w
1|4|128|5.74w
1|4|256|5.57w


### Jedis benchamark测试(正常流程)
测试工具jedis benchmark，见[这里](https://github.com/sheki/jedis-benchmark)。

reactor数目|jedis测试线程数目|connection数目|sadd OPS|set OPS
---|---|---|---|---
1|1|1|0.14w|0.11w
1|2|1|0.15w|0.11w
1|2|2|0.32w|0.29w
1|4|1|0.14w|0.11w
1|4|2|0.32w|0.3w
1|4|4|0.72w|0.7w
1|8|1|0.12w|0.11w
1|8|2|0.26w|0.29w
1|8|4|0.72w|0.69w
1|8|8|1.3w|1.29w
1|16|1|0.1w|0.11w
1|16|2|0.25w|0.29w
1|16|4|0.6w|0.68w
1|16|8|1.3w|1.3w
1|16|16|1.6w|1.7w
1|32|1|0.11w|0.12w
1|32|2|0.25w|0.28w
1|32|4|0.62w|0.68w
1|32|8|1.2w|1.24w
1|32|16|1.87w|1.99w
1|32|32|2.05w|2.07w
1|64|1|0.11w|0.10w
1|64|2|0.25w|0.27w
1|64|4|0.6w|0.68w
1|64|8|1.22w|1.35w
1|64|16|1.66w|2.08w
1|64|32|1.75w|2.22w
1|64|64|2w|2.07w
1|128|1|0.1w|0.10w
1|128|2|0.24w|0.28w
1|128|4|0.62w|0.69w
1|128|8|1.31w|1.39w
1|128|16|2.02w|2.03w
1|128|32|2.12w|1.71w
1|128|64|1.96w|2.07w
1|128|128|1.75w|2.08w
1|256|1|0.1w|0.10w
1|256|2|0.255w|0.28w
1|256|4|0.61w|0.69w
1|256|8|1.28w|1.33w
1|256|16|1.81w|2.04w
1|256|32|2.05w|2.11w
1|256|64|2w|2.02w
1|256|128|1.83w|2.02w
1|256|256|1.87w|2.01w

## 测试总结
### reactor直接返回响应的情况下，性能跟reactor数目的关系
在reactor直接返回响应的情况下，随着reactor数目的增加：
- 最高OPS是逐渐增加的，即更多的reactor数目意味着更高的OPS极限性能；
- 相同connection数目 + 相同threads数目的情况下，OPS是逐渐增加的，但不是线性增加的，时延则是逐渐降低的；