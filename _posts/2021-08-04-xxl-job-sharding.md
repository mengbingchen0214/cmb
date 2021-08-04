---
layout: post
title: Cambly（和外国人英语口语交流App）使用一个月后的感受
categories: English
description: 使用Cambly,30天内和17个外国人付费聊天
keywords: Cambly, English
---






2019.7.24-8.24

## 使用Cambly

30天内和17个外国人付费聊天

为什么选择Cambly

起初在看英语学习博主视频里知道了Cambly，那时并没有在意。可是英语口语始终是自己的一个心结---缺乏英语交流环境，发音正但是口语不流畅，见的外国人少导致不自信等。之前有找过线上网友互相用英文交流，可是缺点很明显，双方都是中国人，表达很Chinglish之外，说错了也无法互相指出纠正，而且若是没有明显的主题，只是闲聊，坚持一个月都很困难。外国人的话，我以前会尝试过用英文发信息，可是写下来和直接说还是不同的，写下来中间的思考时间以及改正时间正是口语中需要减少的地方。免费的东西不容易重视，而且在各种地方找免费资源浪费时间，性价比不高，所以我就有了付费和外国人交流的想法。选择Cambly之前，我并没有比较市面上其他的类似产品，这边仅对Cambly进行体验一个月之后的分享。

### 注册订阅套餐之前

我去知乎看了知友对其的感受，并随便找了一个邀请码开始注册。邀请码是每个注册Cambly的人都会有的，把邀请码分享出去，用你的邀请码进行注册的人会有15min的免费体验，如果那个人体验后还订阅套餐了，你将会有60min的免费学习时间。（我的邀请码 CMB666 ）。我在手机端进行了15min的体验后，觉得还不错，网络可以，系统随机给我的老师还帮我指出了一个错误，15min太少，还意犹未尽呢，所以之后，我决定订阅套餐。

### 订阅套餐

我是在电脑端订阅付费的，有很多套餐，每周安排是15/30/60min/day， 2/3/5/7d/week， 套餐周期有月，度，年。我订阅的是30min/day，7d/w，那会儿（2019.07.25）人民币汇率还没破7，花了1202￥，这会儿得更高一点。电脑端的价格显示的是美元，用支付宝付款的时候就折合成人民币了，手机端的价格显示的是人民币。关于套餐频率，因人而异。如果是有需要考IELTS等国外考试需要辅导的话，可能60min/d比较适合，但是如果你像我一样，只是想锻炼自己的口语，见识见识外国人的话，30min/d足够了，至于一周几天的话，按个人预算来定，个人建议是5d/w，因为自己体验下来发现，一周总有那么两天心情状态不好，导致上课的时候心不在焉，或者临时有事，与预约时间冲突。
现在一张数据表里有大量数据需要某个服务端应用来处理，要求：

1. 能够并行处理；
2. 能够较灵活地控制并行任务数量。
3. 压力较均衡地分散到不同的服务器节点；

## 思路

因为需要并行处理同一张数据表里的数据，所以比较自然地想到了分片查询数据，可以利用对 id 取模的方法进行分片，避免同一条数据被重复处理。

根据第 1、2 点要求，本来想通过对线程池的动态配置来实现，但结合第 3 点来考虑，服务器节点数量有可能会变化，节点之间相互无感知无通信，自己在应用内实现一套调度机制可能会很复杂。

如果有现成的独立于这些服务器节点之外的调度器就好了——顺着这个思路，就想到了已经接入的分布式任务调度平台 XXL-JOB，而在阅读其 [官方文档][1] 后发现「分片广播 & 动态分片」很贴合这种场景。

![](/images/posts/java/xxl-job-sharding-broadcast.png)

## 方案

1. 利用 XXL-JOB 的路由策略「分片广播」来调度定时任务；
2. 通过任务参数传入执行任务节点数量；
3. 定时任务逻辑里，根据获取到的分片参数、执行任务节点数量，决策当前节点是否需要执行，分片查询数据并处理：
    - 如果 *分片序号 > (执行任务节点数量 - 1)*，则当前节点不执行任务，直接返回；
    - 否则，取 *分片序号* 和 *执行任务节点数量* 作为分片参数，查询数据并处理。

这样，我们可以实现灵活调度 [1, N] 个节点并行执行任务处理数据。

## 主要代码示例

JobHandler 示例：

```java
@XxlJob("demoJobHandler")
public void execute() {
    String param = XxlJobHelper.getJobParam();
    if (StringUtils.isBlank(param)) {
        XxlJobHelper.log("任务参数为空");
        XxlJobHelper.handleFail();
        return;
    }

    // 执行任务节点数量
    int executeNodeNum = Integer.valueOf(param);
    // 分片序号
    int shardIndex = XxlJobHelper.getShardIndex();
    // 分片总数
    int shardTotal = XxlJobHelper.getShardTotal();

    if (executeNodeNum <= 0 || executeNodeNum > shardTotal) {
        XxlJobHelper.log("执行任务节点数量取值范围[1,节点总数]");
        XxlJobHelper.handleFail();
        return;
    }

    if (shardIndex > (executeNodeNum - 1)) {
        XxlJobHelper.log("当前分片 {} 无需执行", shardIndex);
        XxlJobHelper.handleSuccess();
        return;
    }

    shardTotal = executeNodeNum;

    // 分片查询数据并处理
    process(shardIndex, shardTotal);

    XxlJobHelper.handleSuccess();
}
```

分片查询数据示例：

```sql
select field1, field2 
from table_name 
where ... 
    and mod(id, #{shardTotal}) = #{shardIndex} 
order by id limit #{rows};
```

## 进一步思考

1. 如果需要更大的并发量，需要有大于应用节点数量的任务并行，如何处理？

    两种思路：
    
    - 通过任务参数传入一个并发数，单个节点在处理任务时，将查询到的数据按这个数字进行再分片，交由线程池并行处理；
    - 配置 M 个定时任务，指定相同的 JobHandler，给它们编号 0、1、2...M，并将定时任务编号和 M 这两个数，由任务参数传入，定时任务逻辑里，先根据分片参数、定时任务编号、M，重新计算出新的分片参数，如 *分片序号 = (分片序号 * M) + 定时任务编号*，*分片总数 = 分片总数 \* M*，再查询数据并处理。

2. 如果有可能频繁调整任务执行逻辑，包括可能要新增任务参数等，而不想重启服务器，如何解决？

    可以考虑使用 XXL-JOB 的「GLUE模式」任务，能够在线编辑和更新定时任务执行逻辑。

## 参考

- [分布式任务调度平台XXL-JOB][1]

[1]: https://www.xuxueli.com/xxl-job/
