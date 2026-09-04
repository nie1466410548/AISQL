# AI SQL 查询优化：问题背景与研究范围

本文讨论查询中已经包含 `AI_FILTER`、`AI_JOIN`、`AI_AGG`、`AI_COMPLETE` 等 AI 算子以后，数据库如何优化执行计划。

## 1. AI 对传统查询优化流程的影响

传统查询优化大致包含以下环节：

```text
SQL / 逻辑计划
       ↓
规则优化（Rule-Based Optimization）
       ↓
统计信息与基数估计
       ↓
代价估计
       ↓
物理实现选择与计划空间搜索
       ↓
执行、反馈与必要的重新优化
```

AI 算子进入查询计划后，查询优化仍然包括上述基本环节。变化在于，AI 算子的语义、统计、成本和质量属性难以直接由现有优化器表示和获取，同时引入了新的物理实现与计划决策。

| 优化器职责 | AI 算子带来的变化 |
|---|---|
| 规则优化 | 移动、融合、分解或推导 AI 算子时，规则成立所需的确定性、输入依赖和可组合性难以检查，部分改写只能保证近似等价 |
| 基数估计 | AI 谓词的输出不是已有数据列，而是由模型和 prompt 在运行时生成；统计量还会随模型、prompt 和实现变化 |
| 代价估计 | 成本受 token、batch、缓存、并发和服务状态影响，不能简单表示为“输入行数 × 单行成本” |
| 优化目标 | 不同模型和近似改写可能改变结果，优化器需要在成本、延迟和结果质量之间选择 |
| 物理实现 | 同一个逻辑 AI 算子可以使用大模型、小模型、模型级联、embedding 检索或专用模型执行 |
| 计划搜索 | 除 Join Order 和访问路径外，还要考虑 AI 算子的位置、实现、执行参数和语义改写，计划空间进一步扩大 |
| 执行反馈 | 模型、数据、缓存和服务状态会变化，编译期估计和静态计划更容易失效 |

## 2. 术语澄清

### 2.1 本文研究的不是 Text-to-SQL

```text
自然语言问题 ──Text-to-SQL──> SQL
                                  │
                                  │ 查询优化
                                  ▼
                         关系算子 + AI 算子的执行计划
```

| 方向 | 研究对象 |
|---|---|
| Text-to-SQL | 如何把自然语言问题翻译成 SQL |
| AI4DB | 如何用机器学习改进基数估计、参数调优、索引推荐等数据库内部任务 |
| DB4AI | 如何用数据库管理和执行 AI 工作负载 |
| 本文的 AI SQL 查询优化 | 当 AI 算子已经出现在查询中时，如何生成和执行较优计划 |

区别在于：AI4DB 通常把 AI 当作帮助优化器决策的工具；本文把 AI 算子当作优化器需要安排和执行的对象。

### 2.2 AI 算子不等于 LLM 调用

AI 算子的后端可以是字符串或向量计算、embedding 模型、专用任务模型，也可以是大语言或多模态模型。即使逻辑上都是 `AI_FILTER`，也可能有以下实现：

```text
大模型直接判断
小模型判断，不确定样本交给大模型
embedding 预筛选后再由大模型判断
领域分类器执行，大模型兜底
```

因此，优化器既要决定 AI 算子在计划中的位置，也要决定它采用哪种物理实现。

## 3. 当前研究主要回答的三类决策

现有 AI SQL 查询优化工作可以归纳为三类：

| 决策 | 需要回答的问题 | 典型做法 | 代表工作 |
|---|---|---|---|
| **Placement** | AI 算子放在计划的什么位置，多个谓词按什么顺序执行 | AI 谓词相对 Join 的上拉或下推、谓词排序、运行时重排 | Cortex AISQL、PLOP、Sema |
| **Physical Selection** | 每个逻辑 AI 算子采用什么实现 | 模型选择、小模型—大模型级联、embedding 代理、batch 和缓存 | Cortex AISQL、LOTUS、ThalamusDB、Palimpzest/Abacus |
| **Rewriting** | 查询或 AI 算子能否改写成成本更低的表达形式 | Semantic Join 改写为分类、AI Filter 融合、AI 聚合分层、pipeline rewrite | Cortex AISQL、Sema、DocETL/MOAR、PLOP |

### 3.1 Placement

Placement 决定 AI 算子相对 Filter、Join、Aggregate 等算子的位置。例如，同一个商品描述在与评论 Join 后可能重复出现多次，因此 `AI_FILTER('商品描述宣称静音', description)` 放在 Join 前还是 Join 后，会改变输入规模和模型调用方式。

Cortex AISQL 的公开案例中，调整普通谓词、文本 AI 谓词和图像 AI 谓词的位置后，两个计划的模型调用量分别约为 11 万次和 330 次。

![Cortex AISQL 中的 Placement 示例](figures/plan_ab.png)

### 3.2 Physical Selection

Physical Selection 决定逻辑 AI 算子怎样执行。例如，`AI_FILTER` 可以直接使用大模型，也可以先由便宜的 proxy 模型判断，只把不确定样本交给 oracle 模型。不同实现的成本、延迟和结果质量可能不同。

![Cortex AISQL 的小模型—大模型级联](figures/aisqlfig2.png)

### 3.3 Rewriting

Rewriting 改变查询或语义任务的表达方式。例如，当 Semantic Join 的一侧是适合作为候选标签的有限集合时，可以尝试把逐对语义判断改写为分类，再执行普通等值 Join，从而把约 `M × N` 次判断降为约 `M` 次模型调用。

这类改写不一定严格等价，优化器还需要判断适用条件和结果质量。

![Semantic Join 改写为分类](figures/join_rewrite.png)

Placement、Physical Selection 和 Rewriting 分别对应传统优化器中的谓词迁移、物理算子选择和查询重写。当前工作已经分别研究了这三类决策，但多数系统只覆盖其中一部分，或者分别优化各个部分。

## 4. 问题的研究形态

相关工作使用的接口和计划载体并不相同，主要可以分为三类：

| 研究形态 | AI 算子位于哪里 | 代表系统 | 主要涉及的优化问题 |
|---|---|---|---|
| 关系查询引擎 | AI 函数或语义算子进入 SQL 关系计划 | Cortex AISQL、BigQuery、ThalamusDB、PLOP、Sema | 与 Join Order、关系算子、访问路径和分布式执行联合优化 |
| 声明式语义处理框架 | AI 算子组成 DataFrame 或 DSL pipeline | LOTUS、DocETL/MOAR、Palimpzest | 算子实现选择、pipeline rewrite 和成本—质量权衡 |
| 任务规划与算子 DAG | 规划器根据任务生成专用算子图 | CAESURA、AOP、CADENZA | 任务分解、算子编排以及专用模型与大模型之间的选择 |

本文重点关注第一类，因为它直接涉及传统数据库优化器；后两类虽然不一定使用 SQL 接口，但其中的 Placement、Physical Selection 和 Rewriting 方法仍可为 AI SQL 查询优化提供参考。
