---
tags: []
parent: 'Less Is More: Trading a Little Bandwidth for {Ultra-Low} Latency in the Data Center'
collections:
    - 中断
version: 9338
libraryID: 1
itemKey: IARDKMC7

---
# Less is More: Trading a little Bandwidth for Ultra-Low Latency in the Data Center

## Abstract

高带宽与低时延的目标。

HULL 使用 Phantom Queues 留出“带宽余量”，Phantom Queues 在网络链路充分利用并在交换机上形成队列之前传递拥塞信号。

为延迟敏感的流量留有空间，以避免缓冲和相关的大延迟。

牺牲 10% 带宽，显著减少平均时延和尾部时延。

## Introduction

提到电路交换到报文交换、TCP 协议，服务质量的方案。

网络的性能指标：吞吐量、延迟

介绍数据中心对于网络评价指标变化的影响

网络延迟的主要构成部分：1. 端点协议栈；2. NIC；3. 交换机

介绍 buffer 对缓存和延迟的影响，接着引出文章的思路，以链路的利用率来替换 buffer
