# 安全目录 Category 中心化多表示检索方案

## 1. 目标

当前索引把 318 个 category 和 972 个 example 混在同一个 kNN 文档集合中。短字段检索时，同一 category 的多个 example 会占用多个候选名额，正确 category 常在候选池中却排不进 top20。

本方案将 `category` 作为唯一的候选和分类单位，同时保留多种检索表示，解决两个问题：

1. category 长文本 embedding 可能稀释“姓名”“角色代码”等短 query；
2. example 文档重复占位，导致 category 排序和 topK 截断失真。

## 2. 核心原则

- 一个 category 只产生一个最终候选，不让 example 作为独立候选参与最终 topK。
- 一个 category 可以有多种 embedding 表示；“一个 category”不等于“一个向量”。
- 短字段优先使用名称和示例表示，表级画像优先使用定义和完整语义表示。
- 各路分数先在本路由内校准，再在 category 层合并，禁止直接比较不同路由的原始 cosine 分数。
- 检索阶段只负责扩大和排序候选，最终敏感定级仍继承被选 category 的目录属性。
- 新索引与旧索引并行评估，旧 C0 作为固定对照，不直接替换线上版本。

## 3. Category 文档模型

安全目录中的每个叶子分类保留一个稳定的 `category_id` 和完整路径：

```text
category_id
level1 / level2 / level3 / level4
category_path
level3_definition
level4_definition_or_content
examples[]
aliases[]
security_level
regulatory_level
important_data_info
```

例如“个人基本概况信息”不再拆成 11 个独立的最终候选，而是保存为一个 category，并在其内部保留：

```text
名称：个人基本概况信息、个人基本资料、个人身份信息
示例：姓名、性别、国籍、民族、婚姻状况、证件号码、家庭住址
定义：个人自然属性、身份属性和基本情况数据
完整语义：名称 + 定义 + 说明 + 示例
```

原始目录文本、人工确认的别名和示例应保留来源标记。模型生成的扩展词只能作为低权重实验特征，不能覆盖正式目录内容。

## 4. 多表示 embedding

每个 category 建议生成四种表示。它们共享 `category_id`，但可以分别进入不同向量字段或同一索引的 `representation_type` 字段。

| 表示 | 内容 | 主要用途 |
|---|---|---|
| `name_embedding` | level4、路径末级名称、已确认别名 | 处理短字段和名称近似 |
| `example_embedding` | 该 category 的全部示例字段，分隔拼接 | 处理“姓名”“证件号码”等字段级命中 |
| `definition_embedding` | level3 定义、level4 说明、目录正文 | 处理业务语义和宽泛分类 |
| `full_embedding` | 名称 + 定义 + 说明 + 示例 | 通用检索和表画像检索 |

`example_embedding` 不能只拼成无上下文的超长列表。建议加入 category 名称作为前缀，例如：

```text
个人基本概况信息。字段示例：姓名、性别、国籍、民族、婚姻状况、证件号码、家庭住址。
```

这样既保留短字段的精确语义，又避免 example 文档独立占用 kNN slot。

## 5. 索引结构选择

第一阶段建议建立独立实验索引，避免影响现有索引：

```text
catalog_category_repr
文档数约：318 × 4 = 1272（没有表示的 category 不强行补造）
主键：category_id + representation_type
向量：name_embedding / example_embedding / definition_embedding / full_embedding
```

有两种实现方式：

### 5.1 多向量字段

每个 category 只有一条文档，文档内有多个 dense_vector 字段。优点是 category 天然唯一；缺点是查询不同表示时需要分别指定向量字段，索引映射较复杂。

### 5.2 表示文档

每个 category 的每种表示是一条文档，共享 `category_id`。优点是容易复用现有 kNN 代码，且可按 `representation_type` 过滤；缺点是检索后必须按 category 聚合。

建议先采用 5.2 做实验，因为它改动较小，便于和旧 1290 文档索引对照。无论采用哪种实现，最终候选都必须按 `category_id` 去重。

## 6. 查询路由

### 6.1 短字段 query

对字段中文名先做清洗，保留原文和清洗后文本两份。短字段（长度较短、包含 ID/代码/编号/标识/序号等技术后缀）执行：

```text
name_embedding 检索
example_embedding 检索
字段关键词检索
```

短字段不要只查 `full_embedding`，因为长定义可能稀释字段本身的信号。

### 6.2 表名 + 字段 query

执行：

```text
name_embedding：表名 + 字段名
definition_embedding：表业务语义 + 字段语义
full_embedding：表名 + 字段名 + 表级画像
```

### 6.3 整表画像 query

执行：

```text
definition_embedding
full_embedding
```

整表画像只提供辅助证据，不能单独覆盖明确的字段级命中。

## 7. Category 层合并

每一路先保留原始证据：最佳 rank、最佳 cosine、命中的 representation、命中的 example 数量。然后按 `category_id` 聚合：

```text
category_evidence = {
  best_name_score,
  best_example_score,
  best_definition_score,
  best_full_score,
  route_count,
  matched_examples,
  best_rank_by_route
}
```

排序建议分两阶段：

1. 主排序使用各路归一化后的向量信号；
2. 只有主分数差距小于决胜阈值时，才使用 example 命中、路由数量和表画像一致性决胜。

不要把 `matched_examples` 或 `route_count` 直接按固定大 bonus 加到所有候选上。否则会重现当前 D/E 实验中“多路命中但语义不正确的 category 被提升”的问题。

## 8. 分数校准

不同表示的 cosine 分布不一致，必须分路校准：

```text
score_norm = normalize(score, route-specific distribution)
```

首轮可使用每次查询结果内部的 min-max 或 rank percentile；若结果稳定，再用开发集统计每一路的均值、标准差和分位点进行固定校准。

新 category 索引不应直接沿用旧索引的 `min_score=0.5`。先关闭或放宽阈值，观察各表示的分数分布，再设置阈值。候选不足时保留低分结果并交给后续风险规则处理，不能因为统一阈值提前删掉正确 category。

## 9. 检索和分类流程

```text
数据字典行
  ↓
读取或生成整表画像
  ↓
构造短字段、表名+字段、画像三个 query
  ↓
按 query 类型检索 name/example/definition/full 表示
  ↓
按 category_id 聚合全部证据
  ↓
分路校准 + category 主排序
  ↓
近分候选使用辅助证据决胜
  ↓
保留 top50 作为保护池，输出 top20 给分类模型
  ↓
高敏感风险且 top20 无敏感候选时，使用 top50 或完整目录兜底
  ↓
程序校验完整路径和安全属性
```

分类模型看到的是 category 候选卡，而不是 example 文档。候选卡中应同时展示 category 名称、完整路径、定义、示例、监管定级和升级条件。

## 10. 最小实验矩阵

不应同时改变索引、排序和分类提示词。建议按以下顺序：

| 组别 | 索引 | 表示 | 目的 |
|---|---|---|---|
| X0 | 旧索引 | 现有 C0 | 固定基线 |
| X1 | 318 category | full | 验证单 category 长文本是否稀释短 query |
| X2 | category 表示文档 | name + example | 验证短字段检索 |
| X3 | category 表示文档 | name + example + definition + full | 验证完整多表示方案 |
| X4 | X3 | 增加 category 聚合和分路校准 | 验证排序改进 |

每组固定同一批 query、同一 embedding 模型、同一测试集和同一 `top20` 评估脚本。先完成检索评估，再接入 LLM 分类，避免把召回和分类误差混在一起。

## 11. 验收指标

必须同时报告总体和分类型指标：

- `strict_all`：gold 是否进入完整候选池；
- `strict_top20`：gold 是否进入实际提供给 LLM 的 top20；
- 短字段 top20 召回率；
- 模糊字段（客户编号、角色代码、附件编号等）top20 召回率；
- 敏感字段进入候选池比例；
- 敏感字段进入 top20 比例；
- 敏感最终漏判率；
- 非敏感字段 top20 召回率；
- 平均检索耗时和 P95 耗时。

重点分析“gold 在池中但不在 top20”的样本，判断是 category 表示不足、路由权重错误，还是最终截断问题。

## 12. 实施边界

本方案暂不要求：

- 更换 embedding 模型；
- 立即放弃 int8 HNSW；
- 引入复杂 cross-encoder；
- 修改安全目录定义；
- 将历史盘点结果作为硬过滤条件。

如果 X3/X4 仍不能改善短字段和模糊字段，下一步才考虑为高难字段增加专门的语义重写或 reranker，而不是继续扩大 topK。

## 13. 推荐落地顺序

```text
1. 保留 C0，建立 318 category 表示实验索引
2. 先测试 name/example 两种表示
3. 再加入 definition/full 表示
4. 按 category_id 聚合并保留完整证据
5. 做分路分数校准和近分决胜
6. 用固定测试集比较 X0-X4
7. 选出最佳检索方案后再接敏感兜底
```

最终推荐的方向是：**category 是候选单位，多种 embedding 是检索证据，category 层负责聚合和排序。**
