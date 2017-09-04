[/status](http://brpc.baidu.com:8765/status)可以访问服务的主要统计信息。这些信息和/vars是同源的，但按服务重新组织方便查看。

![img](http://wiki.baidu.com/download/attachments/165876293/image2016-12-22%2011%3A19%3A56.png?version=1&modificationDate=1482376797000&api=v2)

上图中字段的含义分别是：

- **non_service_error**: "non"修饰的是“service_error"，后者即是分列在各个服务下的error，此外的error都计入non_service_error。服务处理过程中client断开连接导致无法成功写回response就算non_service_error。而服务内部对后端的连接断开属于服务内部逻辑，只要最终服务成功地返回了response，即使错误也是计入该服务的error，而不是non_service_error。
- **connection_count**: 向该server发起请求的连接个数，不包含[对外连接](http://brpc.baidu.com:8765/vars/rpc_channel_connection_count)的个数。
- **example.EchoService**: 服务的完整名称，包含名字空间。
- **Echo (EchoRequest****) returns (EchoResponse****)**: 方法签名，一个服务可包含多个方法，点击request/response上的链接可查看对应的protobuf结构体。
- **count**`:` 成功处理的请求总个数。
- `**error**`: 失败的请求总个数。
- `**latency**`: 在web界面下从右到左分别是过去60秒，60分钟，24小时，30天的平均延时。在文本界面下是10秒内([-bvar_dump_interval](http://brpc.baidu.com:8765/flags/bvar_dump_interval)控制）的平均延时。
- `**latency_percentiles**`: 是延时的50%, 90%, 99%, 99.9%分位值，统计窗口默认10秒([-bvar_dump_interval](http://brpc.baidu.com:8765/flags/bvar_dump_interval)控制），web界面下有曲线。
- `**latency_cdf**`: 是分位值的另一种展现形式，类似histogram，只能在web界面下查看。
- `**max_latency**`: 在web界面下从右到左分别是过去60秒，60分钟，24小时，30天的最大延时。在文本界面下是10秒内([-bvar_dump_interval](http://brpc.baidu.com:8765/flags/bvar_dump_interval)控制）的最大延时。
- `**qps**`: 在web界面下从右到左分别是过去60秒，60分钟，24小时，30天的平均qps。在文本界面下是10秒内([-bvar_dump_interval](http://brpc.baidu.com:8765/flags/bvar_dump_interval)控制）的平均qps。
- `**processing**`: 正在处理的请求个数。如果持续不为0（特别是在压力归0后），应考虑程序是否有bug。

 

用户可通过让对应Service实现[baidu::rpc::Describable](https://svn.baidu.com/public/trunk/baidu-rpc/src/baidu/rpc/describable.h)自定义在/status页面上的描述.

比如:

![img](http://wiki.baidu.com/download/attachments/165876293/image2017-1-14%2023%3A58%3A20.png?version=1&modificationDate=1484409504000&api=v2)

[public/bvar](https://svn.baidu.com/public/trunk/bvar/)是多线程环境下的计数器类库，方便记录和查看用户程序中的各类数值，它利用了thread local存储避免了cache bouncing，相比UbMonitor几乎不会给程序增加性能开销，也快于竞争频繁的原子操作。baidu-rpc集成了bvar，[/vars](http://brpc.baidu.com:8765/vars)可查看所有曝光的bvar，[/vars/VARNAME](http://brpc.baidu.com:8765/vars/rpc_socket_count)可查阅某个bvar，增加计数器的方法请查看[bvar](http://wiki.baidu.com/display/RPC/bvar)。baidu-rpc大量使用了bvar提供统计数值，当你需要在多线程环境中计数并展现时，应该第一时间想到bvar。但bvar不能代替所有的计数器，它的本质是把写时的竞争转移到了读：读得合并所有写过的线程中的数据，而不可避免地变慢了。当你读写都很频繁并得基于数值做一些逻辑判断时，你不应该用bvar。

## 查询方法

[/vars](http://brpc.baidu.com:8765/vars) : 列出所有的bvar

[/vars/NAME](http://brpc.baidu.com:8765/vars/rpc_socket_count)：查询名字为NAME的bvar

[/vars/NAME1,NAME2,NAME3](http://brpc.baidu.com:8765/vars/pid;process_cpu_usage;rpc_controller_count)：查询名字为NAME1或NAME2或NAME3的bvar

[/vars/foo*,b$r](http://brpc.baidu.com:8765/vars/rpc_server*_count;iobuf_blo$k_*)：查询名字与某一统配符匹配的bvar，注意用$代替?匹配单个字符，因为?在url中有特殊含义。

以下动画演示了如何使用过滤功能。你可以把包含过滤表达式的url复制粘贴给他人，他们点开后将看到你看到的内容。

![img](http://wiki.baidu.com/download/attachments/37774685/filter_bvar.gif?version=1&modificationDate=1494403262000&api=v2)

1.0.123.31011及之后的版本加入了一个搜索框加快了寻找特定bvar的速度，在这个搜索框你只需键入bvar名称的一部分，框架将补上*进行模糊查找。不同的名称间可以逗号、分号或空格分隔。

![img](http://wiki.baidu.com/download/attachments/37774685/search_var.gif?version=1&modificationDate=1494403229000&api=v2)

你也可以在命令行中访问vars：

```
$ curl brpc.baidu.com:8765/vars/bthread*
bthread_creation_count : 125134
bthread_creation_latency : 3
bthread_creation_latency_50 : 3
bthread_creation_latency_90 : 5
bthread_creation_latency_99 : 7
bthread_creation_latency_999 : 12
bthread_creation_latency_9999 : 12
bthread_creation_latency_cdf : "click to view"
bthread_creation_latency_percentiles : "[3,5,7,12]"
bthread_creation_max_latency : 7
bthread_creation_qps : 100
bthread_group_status : "0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 "
bthread_num_workers : 24
bthread_worker_usage : 1.01056
```

## 查看历史趋势

点击大部分数值型的bvar会显示其历史趋势。每个可点击的bvar记录了过去60秒，60分钟，24小时，30天总计174个值。当有1000个可点击bvar时大约会占1M内存。

![img](http://wiki.baidu.com/download/attachments/37774685/plot_bvar.gif?version=1&modificationDate=1494403276000&api=v2)

## 统计和查看分位值

x%分位值（percentile）是指把一段时间内的N个统计值排序，排在第N * x%位的值就是x%分位值。比如一段时间内有1000个值，排在第500位(1000 * 50%)的值是50%分位值（即中位数），排在第990位的是99%分位值(1000 * 99%)，排在第999位的是99.9%分位值。分位值能比平均值更准确的刻画数值分布，对衡量系统SLA有重要意义。对于最常见的延时统计，平均值很难反映出实质性的内容，99.9%分位值往往更加关键，它决定了系统能做什么。

分位值可以绘制为CDF曲线和按时间变化时间。

![img](http://wiki.baidu.com/download/attachments/71337189/image2015-9-21%2022%3A34%3A14.png?version=1&modificationDate=1442846055000&api=v2)

上图是CDF曲线。纵轴是延时。横轴是小于纵轴数值的数据比例。很明显地，这个图就是由从10%到99.99%的所有分位值组成。比如横轴=50%处对应的纵轴值便是50%分位值。那为什么要叫它CDF？CDF是[Cumulative Distribution Function](https://en.wikipedia.org/wiki/Cumulative_distribution_function)的缩写。当我们选定一个纵轴值x时，对应横轴的含义是"数值 <= x的比例”，如果数值是来自随机采样，那么含义即为“数值 <= x的概率”，这不就是概率的定义么？CDF的导数是[概率密度函数](https://en.wikipedia.org/wiki/Probability_density_function)，换句话说如果我们把CDF的纵轴分为很多小段，对每个小段计算两端对应的横轴值之差，并把这个差作为新的横轴，那么我们便绘制了PDF曲线，就像（横着的）正态分布，泊松分布那样。但密度会放大差距，中位数的密度往往很高，在PDF中很醒目，这使得边上的长尾相对很扁而不易查看，所以大部分系统测试结果选择CDF曲线而不是PDF曲线。

可以用一些简单规则衡量CDF曲线好坏：

- 越平越好。一条水平线是最理想的，这意味着所有的数值都相等，没有任何等待，拥塞，停顿。当然这是不可能的。
- 99%之后越窄越好：99%之后是长尾的聚集地，对大部分系统的SLA有重要影响，越少越好。如果存储系统给出的性能指标是"99.9%的读请求在xx毫秒内完成“，那么你就得看下99.9%那儿的值；如果检索系统给出的性能指标是”99.99%的请求在xx毫秒内返回“，那么你得关注99.99%分位值。

一条真实的好CDF曲线的特征是”斜率很小，尾部很窄“。

 

![img](http://wiki.baidu.com/download/attachments/71337189/image2015-9-21%2022%3A57%3A1.png?version=1&modificationDate=1442847422000&api=v2)

上图是按时间变化曲线。包含了4条曲线，横轴是时间，纵轴从上到下分别对应99.9%，99%，90%，50%分位值。颜色从上到下也越来越浅（从橘红到土黄）。滑动鼠标可以阅读对应数据点的值，上图中显示是”39秒种前的99%分位值是330微秒”。这幅图中不包含99.99%的曲线，因为99.99%分位值常明显大于99.9%及以下的分位值，画在一起的话会使得其他曲线变得很”矮“，难以辨认。你可以点击以"_latency_9999"结尾的bvar独立查看99.99%曲线，当然，你也可以独立查看50%,90%,99%,99.9%等曲线。按时间变化曲线可以看到分位值的变化趋势，对分析系统的性能变化很实用。

 

baidu-rpc的服务都会自动统计延时分布，用户不用自己加了。如下图所示：

![img](http://wiki.baidu.com/download/attachments/71337189/image2015-9-21%2023%3A5%3A41.png?version=1&modificationDate=1442847942000&api=v2)

你可以用bvar::LatencyRecorder统计非baidu-rpc服务的延时，这么做（更具体的使用方法请查看[bvar-c++](http://wiki.baidu.com/pages/viewpage.action?pageId=133624370)）：

```c++
#include <bvar/bvar.h>
 
...
bvar::LatencyRecorder g_latency_recorder("client");  // expose this recorder
... 
void foo() {
    ...
    g_latency_recorder << my_latency;
    ...
}
```

如果这个程序使用了baidu-rpc server，那么你应该已经可以在/vars看到client_latency, client_latency_cdf等变量，点击便可查看动态曲线。如下图所示：

![img](http://wiki.baidu.com/download/thumbnails/71337189/image2015-9-21%2023%3A33%3A16.png?version=1&modificationDate=1442849597000&api=v2)

## 非baidu-rpc server

如果这个程序只是一个baidu-rpc client或根本没有使用baidu-rpc，并且你也想看到动态曲线，看[这里](http://wiki.baidu.com/pages/viewpage.action?pageId=213843633)。