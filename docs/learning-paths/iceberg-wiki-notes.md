---
title: Iceberg Wiki 中文学习笔记
type: reference
depth: 入门
tags: [learning-path, iceberg, notes]
aliases: [Iceberg 学习笔记, Iceberg 入门笔记]
related: [iceberg, lake-table, snapshot, manifest, schema-evolution, partition-evolution, compaction, branching-tagging]
status: stable
---

# Iceberg Wiki 中文学习笔记

!!! tip "一句话理解"
    **Iceberg 的本质不是“又一个存储系统”，而是一套让多种计算引擎共享同一张湖表的元数据协议**。学 Iceberg，先抓住“元数据树 + Snapshot 提交 + Schema / Partition 演化 + 维护作业”这四条主线。

!!! abstract "TL;DR"
    - 先把 Iceberg 看成“**表格式协议**”，不要看成单一产品
    - 先理解 `metadata.json → manifest list → manifest → data file` 这条元数据链
    - Snapshot 带来 **time travel / rollback / branch / tag**
    - Schema Evolution、Partition Evolution 的关键是 **尽量不重写历史数据**
    - 真正落地时，难点往往不在“建表”，而在 **Catalog、compaction、snapshot 清理、并发写入**

## 这份笔记怎么用

- **如果你是第一次接触 Iceberg**：按“推荐阅读顺序”读一遍，先建立地图
- **如果你已经会建表**：重点看“核心知识速记”和“常见误区”
- **如果你要做项目选型**：顺着文末“继续深入”跳到 Catalog、对比页和运维页

## 推荐阅读顺序

### 第 1 轮：先建立心智模型

1. [湖表](../lakehouse/lake-table.md) —— 先理解“湖表”和数据库存储引擎不是一回事
2. [Snapshot](../lakehouse/snapshot.md) —— 理解为什么 Iceberg 天然支持 Time Travel
3. [Manifest](../lakehouse/manifest.md) —— 理解查询为什么不需要扫整个对象存储目录
4. [Apache Iceberg](../lakehouse/iceberg.md) —— 把前面几个概念拼成完整系统图

### 第 2 轮：看清“为什么它在工程上好用”

5. [Schema Evolution](../lakehouse/schema-evolution.md) —— 改列不必重写全表
6. [Partition Evolution](../lakehouse/partition-evolution.md) —— 改分区策略不必推倒重来
7. [Time Travel](../lakehouse/time-travel.md) —— 回看历史、回滚、审计
8. [Branching & Tagging](../lakehouse/branching-tagging.md) —— 数据版 Git-like 工作流

### 第 3 轮：看真正会踩坑的地方

9. [Delete Files](../lakehouse/delete-files.md) —— 行级删除为什么会让读路径变复杂
10. [Compaction](../lakehouse/compaction.md) —— 为什么“能写”不等于“能长期跑”
11. [Iceberg REST Catalog](../catalog/iceberg-rest-catalog.md) —— 多引擎共享表时的控制面
12. [湖仓 20 反模式](../ops/anti-patterns.md) —— 用反例记住上线前要避开的坑

## 核心知识速记

### 1. 先记一句：Iceberg 管的是“表”，不是“文件夹”

传统 Hive 时代常把一张表理解成“对象存储里的一个目录 + Metastore 一条记录”。  
Iceberg 则把“表的真相”收敛到**元数据指针**上：

```text
Catalog
  -> 当前 metadata.json
       -> manifest list
            -> manifests
                 -> data files / delete files
```

这套设计带来的直接收益是：

- 查询规划不再依赖全量 `LIST`
- 历史版本可以被明确记录
- 多引擎能围绕同一份表协议协同工作

### 2. Snapshot 是 Iceberg 的灵魂

把每次提交理解成“生成一个新的表快照”，很多能力就自然了：

| 能力 | 为什么能做到 |
| --- | --- |
| Time Travel | 旧 snapshot 还在 |
| Rollback | 指针可以切回旧 snapshot |
| 审计 | 能看到历史提交链 |
| Branch / Tag | 可以给某个 snapshot 命名或分叉 |

所以学习 Iceberg 时，不要只盯着 `CREATE TABLE`，而要反复问自己：

> **这次写入之后，新的 snapshot 是怎么生成、怎么切换、怎么回收的？**

### 3. Iceberg 的强项是“演化”，不是“固定设计一次到位”

最值得记住的两件事：

- **Schema Evolution**：靠列 ID 避免“改列名后历史数据读错位”
- **Partition Evolution**：新旧数据可以按不同分区策略并存

这意味着 Iceberg 很适合真实团队的长期数据产品，而不是只适合一次性离线数仓。

### 4. Catalog 决定了提交控制面

Iceberg 不是只靠对象存储就能优雅跑起来。  
真正的“表指针切换”通常由 **Catalog** 管理，所以 Catalog 选型很关键：

- 想要标准化、多引擎互通：优先看 [Iceberg REST Catalog](../catalog/iceberg-rest-catalog.md)
- 想要 Git-like 数据分支：看 [Nessie](../catalog/nessie.md)
- 已经在某家云生态深度绑定：再看 Glue / Unity / Polaris

一句话记忆：

> **对象存储放数据，Catalog 管提交，计算引擎负责读写执行。**

### 5. 线上是否稳定，常常取决于维护链路

Iceberg 很容易“演示成功”，也很容易“生产退化”。最常见的原因是下面四件事没人持续做：

- 小文件合并
- Delete file 合并
- 过期 snapshot 清理
- 孤儿文件清理

如果你只记一个运维结论，就记这个：

> **Iceberg 不是建完表就结束，而是必须长期维护 metadata 和文件布局。**

## 常见误区

### 误区 1：Iceberg = 一款查询引擎

不是。Iceberg 是表格式协议，需要搭配 Spark / Flink / Trino / DuckDB 等引擎使用。

### 误区 2：有了 Time Travel，就可以无限保留历史

不行。历史越多，metadata 越膨胀，存储成本也会上升。要结合审计需求设计保留策略。

### 误区 3：分区越细越好

不对。高基数分区会让 metadata 和规划成本失控。Iceberg 比 Hive 强，但也不是无限抗压。

### 误区 4：只要支持 Iceberg 的引擎都能完全互通

不完全成立。不同引擎对 spec 版本、delete、branch/tag、v3 新特性的支持成熟度并不一致。

## 30 分钟复盘清单

- [ ] 能向别人解释“为什么 Iceberg 不是一个数据库”
- [ ] 能画出 `metadata.json → manifest list → manifest → data files`
- [ ] 能说明 Snapshot 为什么带来 Time Travel 和 Rollback
- [ ] 能说出 Schema Evolution 和 Partition Evolution 分别解决什么问题
- [ ] 能列出至少 3 个生产维护动作：compaction / expire snapshots / remove orphan files
- [ ] 能说明 Catalog 在 Iceberg 里的角色

## 继续深入

- 想看完整系统图：读 [Apache Iceberg](../lakehouse/iceberg.md)
- 想补协议基础：读 [湖表](../lakehouse/lake-table.md) · [Snapshot](../lakehouse/snapshot.md) · [Manifest](../lakehouse/manifest.md)
- 想看运维实战：读 [Compaction](../lakehouse/compaction.md) · [故障排查](../ops/troubleshooting.md)
- 想做选型对比：读 [四大表格式对比](../compare/iceberg-vs-paimon-vs-hudi-vs-delta.md)
- 想快速查命令：读 [Iceberg 速查](../cheatsheets/iceberg.md)
