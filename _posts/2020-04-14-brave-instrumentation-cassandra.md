---
title: brave-instrumentation-cassandra
tags: tracing
key: braveinstrumentationcassandra
---

`brave-cassandra-master/cassandra`该文件夹中包含了对Cassandra的tracing植入。

`brave.cassandra.Tracing`这个类从the custom payload of incoming requests提取trace state（比如，每个请求花费了多少时间、每个子操作花费了多少时间和一些相关tag，例如session ID这些），并汇报给Zipkin。

#### 集成

Cassandra tracing 和 Http tracing很像。Client和Server双方要对如何发送和接受the trace context达成一致，才能正常工作。





