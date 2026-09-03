# AI SQL 优化器：问题细化与研究切入点

> 时间：2026-09
>
> 《AI SQL 中的查询优化：调研笔记》1.4 节回答“优化器面临哪些问题”；本文进一步回答“这些问题在具体查询中怎样发生、已有方法处理到哪里、可以从哪个最小问题开始研究”。

## 1. 从挑战清单到研究切入点

1.4 节的 C1—C10 是对问题范围的分类。研究选题还需要再向下收敛一层。一个可以实际推进的切入点，至少应回答以下问题：

1. **具体对象是什么**：研究哪类查询、哪类 AI 算子和哪一项优化决策；
2. **现有方法为什么失效**：能否构造一个查询，使传统规则或已有方法选出明显较差的计划；
3. **未知量和决策变量是什么**：例如选择率、调用成本、错误率，以及 placement、join order、模型和级联阈值；
4. **已有工作做到哪里**：缺口必须是相对于 Horrila、Larch、Abacus、Cortex AISQL 等工作的具体差异，而不是笼统地说“研究较少”；
5. **准备解决哪一小块**：第一阶段不应同时解决 `T × π × ι × ρ` 的全部搜索；
6. **如何验证**：需要有反例、算法基线、数据集和可量化指标。

下面以一个电商场景为例，分析 Join 如何将 placement、join order、物理实现、重写和物化等问题连接起来，并在此基础上提炼候选研究问题。

## 2. 一个主场景：Join 如何放大 AI 算子的优化问题

假设有两张表：

```sql
products(pid, title, description)       -- 10 万件商品
reviews(rid, pid, text, stars)          -- 100 万条评论，平均每件商品 10 条
```

查询目标是：找出描述中宣称“静音”，但评论中有人抱怨噪音的商品。

```sql
SELECT DISTINCT p.pid, p.title
FROM products p
JOIN reviews r ON p.pid = r.pid
WHERE AI_FILTER('商品描述宣称静音', p.description)
  AND AI_FILTER('评论在抱怨噪音', r.text);
```

### 2.1 Placement：Join 扇出会改变 AI 调用次数

设“宣称静音”的商品占 10%，暂时假定两个 AI 谓词单次成本相同，并且优化器先判断商品描述谓词，再短路判断评论谓词。

**计划 A：先 Join，再执行两个 AI 谓词。**

```text
数据流向：↑（从叶子节点向根节点执行）

③ AI_FILTER(review.text)
             |
② AI_FILTER(product.description)
             |
① products ⋈ reviews                 -- 约 100 万行
```

在没有函数缓存时：

- 商品描述谓词在 Join 结果上执行约 100 万次；
- 约 10% 的行通过后，评论谓词再执行约 10 万次；
- 总调用量约 110 万次。如果两个谓词不能短路或被并行求值，上界为 200 万次。

**计划 B：先过滤商品，再 Join，再判断相关评论。**

```text
数据流向：↑（从叶子节点向根节点执行）

              ③ AI_FILTER(review.text)   -- 约 10 万条候选评论
                         |
                       ② JOIN
                       /    \
① AI_FILTER(product.description)      reviews
             |
          products                    -- 10 万件商品
```

- 商品描述谓词执行 10 万次，保留约 1 万件商品；
- 与这些商品关联的评论约 10 万条；
- 评论谓词再执行约 10 万次；
- 总调用量约 20 万次。

在这一组假设下，两种 placement 的调用量相差约 5.5 倍；如果计划 A 不做短路，相差可到 10 倍。差异不是来自 AI 算子自身，而是来自它位于 1:N Join 的哪一侧。

但这里还有一个重要条件：**是否存在 function caching**。计划 A 中同一件商品的 `description` 平均重复 10 次。如果系统按“模型、prompt、输入值”缓存确定的判断结果，商品描述谓词也可能只产生约 10 万次实际模型调用。此时两个计划的 LLM 调用差距会明显缩小，但它们处理的关系中间结果、缓存查找次数和内存占用仍然不同。

因此，“谓词应该放在 Join 上方还是下方”不能只看行数，还取决于：

- Join 的扇出与过滤选择率；
- AI 谓词之间能否短路；
- 重复输入能否通过函数缓存复用（缓存是否启用、重复输入占比和实际命中率）；
- Join 的物理实现和访问路径。

这与 [Horrila](https://arxiv.org/abs/2604.09944) 的核心观察一致：在 function caching 下，LLM 成本更接近不同输入值的数量，而不是计划节点的总行数；但把 AI 谓词延后又可能增加关系算子的处理成本。这个例子也说明，placement 和执行层机制并不完全独立。

### 2.2 不是所有谓词都能安全下推

把评论谓词改为：

```sql
AI_FILTER(
  '这条评论是否与商品描述相矛盾',
  p.description,
  r.text
)
```

它同时依赖 `products` 和 `reviews`，在 Join 之前任何一侧都缺少必要输入，因此不能直接下推。

还有一种更隐蔽的情况：

```text
版本 1：只输入评论，判断“这条评论是否严重负面”
版本 2：同时输入商品标题、价格和评论，再判断“这条评论是否严重负面”
```

版本 1 可以下推到 `reviews`，版本 2 只能在 Join 后求值。但两者已经不是同一个确定性谓词的不同位置，而是输入上下文不同的两个 AI 实现，其结果可能不同。传统的谓词下推要求结果等价；这里发生的是一个**可能降低成本、但改变结果质量的近似改写**。

因此需要区分：

- **精确 placement**：移动前后输入和语义不变，只改变执行成本；
- **上下文裁剪式 placement**：为了提前执行而减少输入上下文，需要进入质量约束；
- **不可移动谓词**：跨表语义依赖是硬约束，只能在相关属性全部可见后执行。

这对应 1.4 节的 C1、C2 和 C7，也给重写规则提出了具体要求：规则不仅要声明“能否下推”，还要声明所需属性、上下文条件以及是否保持语义。

### 2.3 选择率未知会使 Join Order 和求值顺序失稳

考虑三张表：

```sql
customers(cid, notes)                  -- 100 万客户
orders(oid, cid, pid, amount, item_desc) -- 1000 万订单
products(pid, category)                -- 10 万商品
```

查询如下：

```sql
SELECT o.oid
FROM orders o
JOIN customers c ON o.cid = c.cid
JOIN products p ON o.pid = p.pid
WHERE o.amount > 1000
  AND AI_FILTER('客户备注暗示其有流失风险', c.notes)
  AND AI_FILTER('订单商品属于高退货风险品类', o.item_desc);
```

假设普通谓词 `amount > 1000` 选择率约 1%，过滤后涉及约 8 万个不同客户。

- 如果“流失风险”选择率是 0.1%，先在 `customers` 上执行 AI 谓词只留下约 1000 个客户，随后使用索引或 Hash Join 可能非常划算；
- 如果它的真实选择率是 50%，先对全部 100 万客户调用模型却只能过滤一半，先执行普通谓词并对涉及的 8 万客户做 AI 判断可能更便宜；
- 如果“高退货风险品类”和客户流失风险相关，将两者按独立性相乘还会继续放大基数误差。

传统优化器可以从直方图估计 `amount > 1000`，却没有现成统计量回答自然语言谓词的选择率。更麻烦的是，估计选择率本身需要付费调用模型。

因此问题不只是“怎样估计得更准”，还包括：

1. 哪个 AI 谓词值得采样；
2. 采样多少行才值得；
3. 估计误差是否足以改变计划排名；
4. 无法区分两个候选计划时，是继续采样、选择鲁棒计划，还是执行后重优化。

[Larch](https://arxiv.org/abs/2606.07923) 已经研究了语义 filter 的选择率预测和求值顺序，因此“估计 LLM 谓词选择率”本身不能再作为足够具体的新问题。仍值得进一步确认的空间是：**多表 Join 中面向计划决策的采样、多个 AI 谓词的相关性，以及估计成本与计划收益的联合权衡**。

### 2.4 Join 的输出组织会影响批处理和 KV-cache

考虑为每条评论生成带商品标题上下文的翻译：

```sql
SELECT p.pid,
       AI_COMPLETE(
         CONCAT('商品标题：', p.title,
                '\n请把评论翻译成英文：', r.text)
       )
FROM products p
JOIN reviews r ON p.pid = r.pid;
```

如果同一商品的 10 条评论连续进入模型服务，请求可以共享相同的 prompt 前缀，也更容易组成同形批次；如果 Join 输出把不同商品交错排列，共享前缀和批处理收益可能下降。

于是传统物理计划中通常不被保留的属性开始影响 AI 成本：

- 是否按 `pid` 聚集输出；
- 是否具有相同的 prompt 模板和共享前缀；
- 每个组的大小能否形成有效 batch；
- 排序或更换 Join 实现的关系计算成本，能否换来足够的推理收益。

一个可能的研究方式，是把“prompt 前缀分组”“可批处理性”视为类似 interesting order 的物理属性，并让优化器比较额外 Sort、不同 Join 实现与推理节省。这对应 C3、C6 和 C9。

### 2.5 Semantic Join 为什么需要专门的重写和物理实现

查询目标：将 5 万条买家咨询匹配到 200 条标准 FAQ。

```sql
SELECT q.qid, f.fid
FROM questions q
JOIN faq f
  ON AI_FILTER('该咨询可以由这条 FAQ 回答', q.text, f.answer);
```

朴素 Cross Join 需要判断 `5 万 × 200 = 1000 万` 对输入。至少有三条优化路线：

1. **改写为分类**：如果 200 条 FAQ 构成互斥或可多选的稳定标签集，可以每条咨询分类一次，约 5 万次调用；
2. **检索后精排**：先用 embedding 为每条咨询召回 Top-K FAQ，再用 LLM 逐对判断；当 `K=5` 时，精排约 25 万对，但召回可能漏掉正确答案；
3. **批量比较**：一次 prompt 放入多个候选 FAQ，减少请求次数，但 token 成本、上下文长度和判断质量未必同比下降。

分类改写并不普遍成立。如果一条咨询可能匹配多个 FAQ、没有合适标签，或者匹配条件需要比较两侧细节，分类可能改变结果集合。优化器需要知道：

- 什么语义条件下可以分类归约；
- 什么情况下只能做候选召回；
- 重写带来的 recall、precision 和成本怎样估计；
- 质量约束应落在单个算子还是最终查询结果上。

Cortex AISQL 已展示 semantic join 到多标签分类的改写及明显的性能收益；[DocETL](https://arxiv.org/abs/2410.12189)、[MOAR](https://arxiv.org/abs/2512.02289) 和 [Nirvana](https://arxiv.org/abs/2511.19830) 也已经覆盖不同形式的 agentic rewrite。因此可研究的问题不能只写成“用 LLM 自动改写计划”，而应收敛到**某类改写的适用条件、质量边界和选择方法**。

### 2.6 AI 输出物化带来新的有效性问题

如果许多查询都使用“商品是否宣称静音”，可以把结果物化为派生列：

```text
products.claims_quiet BOOLEAN
```

它可以把后续模型调用退化为普通布尔过滤，但传统物化视图很少需要处理以下问题：

- 模型从一个版本升级到另一个版本后，旧结果是否仍有效；
- prompt 修改后，旧列能否部分复用；
- 同一模型重复执行得到不同结果时，以物化值还是重算值为准；
- 数据、prompt、模型、推理参数和输出之间如何记录血缘；
- 旧结果能否作为低质量候选集，而不是简单地“有效/失效”。

这说明 Semantic Cache 或 AI 派生列可能不只是执行层缓存，还可能成为优化器可选择的一种 access path。它主要对应 C5、C7 和 C10。

## 3. 从主场景提炼出的候选研究切入点

上面的六个现象并不等于六个互相独立的新挑战。它们是 C1—C10 在具体查询中的表现，并在若干交叉位置形成可研究的问题。

| 候选切入点 | 来源 | 主要对应挑战 | 初步判断 |
|---|---|---|---|
| R1. Join 中面向计划决策的 AI 选择率采样与自适应优化 | 2.1、2.3 | C3、C5、C6、C8 | 问题具体，与 Join 场景直接对应；需与 Larch 和传统自适应优化区分 |
| R2. Placement 与 Physical Selection 联合优化 | 2.1、2.2 | C3、C6、C7 | 跨层耦合清楚，数据库优化器特征强；搜索空间和质量建模难度较高 |
| R3. Semantic Join 近似重写的适用条件与质量契约 | 2.2、2.5 | C1、C2、C7 | 容易形成规则与理论贡献；通用质量定义较难 |
| R4. 面向批处理和 KV-cache 的物理属性与计划选择 | 2.4 | C3、C6、C9 | 切口新且可测量；结论可能依赖具体推理引擎 |
| R5. AI 派生结果作为可版本化访问路径 | 2.6 | C5、C7、C10 | 系统价值明确；容易扩展成较大的缓存、血缘和一致性课题 |

### 3.1 R1：Join 中面向计划决策的选择率采样

#### 问题定义

AI 谓词的选择率未知，获取估计又需要模型调用。给定若干候选 placement 和 join order，优化器应当决定：对哪些 AI 谓词采样、采样多少，以及何时停止估计并选择计划。

可以把问题写为：

```text
未知量：s1, ..., sn                    -- AI 谓词选择率及其相关性
候选计划：P1, ..., Pk
采样动作：a = (谓词、数据分层、样本量)

目标：最小化
      采样成本 + 执行成本 + 选错计划的期望损失
```

这里的关键不是追求统一的“最高估计精度”，而是追求**足以区分候选计划**的估计。即使选择率置信区间较宽，只要区间内同一个计划始终最优，就没有必要继续花钱采样。

#### 与已有工作的边界

- Larch 已研究 semantic filter 的选择率预测和逐行求值顺序；
- Cortex AISQL 和 [Sema](https://arxiv.org/abs/2603.11622) 已使用运行时反馈调整算子顺序或执行路径；
- 仍需核实的缺口是：这些方法是否联合考虑了多表 Join、采样成本、计划排名的不确定性和重优化触发条件。

因此这里不能声称“首次估计 AI 选择率”，更稳妥的暂定定位是：

> 面向 Join 计划选择的 decision-aware sampling：只为会影响 placement 或 join order 的不确定性付费，并将编译期采样与运行期重优化连接起来。

#### 可以从哪里开始

第一阶段只考虑：

- 固定模型和 prompt；
- 1—2 个 AI_FILTER；
- 一个星型或链式 Join；
- 候选计划仅改变 AI 谓词 placement 和一个局部 join order；
- 先忽略质量变化，把 AI 谓词看作结果稳定的昂贵黑盒。

初步方法可以包括：

1. 对每个选择率维护置信区间或后验分布；
2. 计算不确定性对候选计划成本排序的敏感度；
3. 用 expected value of information 选择下一次采样；
4. 当计划排名足够稳定或采样收益低于成本时停止；
5. 在执行中设置物化检查点，真实基数超出区间时重优化剩余计划。

#### 实验需要回答什么

- 相比固定样本量，能否减少用于估计的 LLM 调用；
- 相比不采样或只用点估计，能否降低 plan regret；
- 选择率分布漂移、谓词相关和数据倾斜时是否稳定；
- 编译期采样与运行期重优化各自适合什么条件。

### 3.2 R2：Placement 与 Physical Selection 联合优化

#### 问题定义

同一个逻辑 AI 谓词可能有多种实现：

- oracle 大模型直接判断；
- proxy 小模型直接判断；
- proxy—oracle 级联及不同阈值；
- embedding 预过滤后由 LLM 复核；
- 使用或不使用 function cache、batching。

实现选择会同时改变单行成本、有效选择率、错误率和批处理特征，进而改变最优 placement 和顺序。因此，“先选逻辑位置，再为每个算子选择实现”的两阶段方法可能不是全局最优。

可以把第一阶段问题限定为：关系 Join 骨架 `T` 固定，近似重写 `ρ` 暂不考虑，只联合搜索：

```text
π：AI 谓词在固定 Join 树中的合法位置和执行顺序
ι：每个谓词的模型、级联策略和阈值

min    E[Cost(T, π, ι)]
s.t.   Qual(T, π, ι) >= τ
```

每个候选实现至少要提供：

```text
c(o, m, θ)       单次或单批成本
s(o, m, θ)       有效选择率
e(o, m, θ)       错误率或混淆矩阵
b(o, m, θ)       批处理/缓存特征
```

#### 一个可以证明两阶段方法失效的例子

设两个 AI 谓词 `P1`、`P2`：

- `P1` 使用 oracle 时选择率低、质量高，适合优先执行；
- `P1` 换成便宜 proxy 后假阳性较多，有效选择率升高，无法有效减少 `P2` 的输入；
- `P2` 的 oracle 成本很高。

如果优化器先根据 oracle 统计选择 `P1 → P2`，再把 `P1` 独立替换成 proxy，得到的计划可能比 `P2 → P1` 更贵，甚至不满足最终质量约束。这个反例能够明确说明联合搜索的必要性。

#### 与已有工作的边界

- Horrila 重点解决混合关系计划中的 semantic operator placement；
- [Abacus](https://arxiv.org/abs/2505.14661) 重点为 semantic operator system 选择物理实现，并支持成本、延迟和质量约束；
- Nirvana 同时包含 agentic logical optimizer 和 cost-aware physical optimizer；
- Cortex AISQL 分别实现 AI-aware placement、模型级联和 semantic join rewrite。

因此，是否具有论文空间不能仅凭“现有工作没有统一框架”判断。后续必须精读这些方法的搜索状态和接口，确认它们是否把**实现引起的选择率和误差变化反馈到关系计划位置**。当前最值得验证的假设是：已有方法多为分阶段处理，或者只在语义算子流水线内选择实现，尚未系统处理混合 Join 计划中的 placement—implementation 耦合。

#### 可以从哪里开始

1. 固定 Join Order，避免一开始搜索整个 `T × π × ι`；
2. 只支持 context-independent AI_FILTER，保证 placement 本身等价；
3. 每个 filter 只提供 oracle 和一种 cascade；
4. 用相对 oracle 的 precision/recall 或 F1 表示质量；
5. 先设计小规模精确搜索作为 oracle，再研究剪枝或动态规划。

验证联合优化有效以后，再逐步加入 Join Order、上下文裁剪式 placement 和运行时反馈。

### 3.3 R3：Semantic Join 近似重写的适用条件与质量契约

#### 问题定义

把逐对 semantic join 改写成分类或 retrieval + rerank，降低了复杂度，但不一定保持结果集合。研究重点可以不是生成更多 rewrite，而是给 rewrite 建立数据库式契约：

```text
Rewrite Rule
  前置条件：标签集封闭性、匹配关系基数、所需上下文
  输出保证：precision / recall / top-k coverage 的统计边界
  成本模型：分类、召回、精排和验证的成本
  失效条件：数据漂移、标签更新、模型版本变化
```

#### 最小切口

只研究一种改写，例如：

> semantic join → embedding Top-K retrieval + LLM verification。

优化器选择 `K` 和 verifier 模型，在 recall 下限下最小化成本。与普通向量检索不同，最终目标是对原 semantic join 结果的近似，需要估计候选生成阶段的漏配率，并把它传播到最终结果质量。

#### 与已有工作的边界

Cortex AISQL 已经验证了 semantic join 改写为多标签分类可以显著降低执行成本，并报告了改写后的结果质量；DocETL、MOAR 和 Nirvana 也已经分别研究了语义管道重写、成本—质量搜索和逻辑计划改写。因此，“近似改写会改变质量”以及“需要比较成本和质量”本身不是新的研究发现。

这里需要验证的更具体问题是：优化器能否在执行前，根据标签集合、匹配关系、语义重叠和所需上下文等可观测条件，判断某条 rewrite 是否适用；并为改写后的最终查询结果给出可检查的质量边界。DEMO 中的成本—质量对比首先用于复现已有现象，进一步分析“质量损失能否由这些条件预测”才是候选研究缺口。

#### 风险

该方向容易把问题扩大为“所有近似重写的统一质量理论”。第一阶段必须固定算子、任务和质量指标，否则很难形成可验证模型。

### 3.4 R4：把推理侧特征纳入物理计划属性

#### 问题定义

传统优化器会保留 interesting order，因为某个 Sort 的成本可能被后续 Merge Join、Group By 或 Order By 摊销。AI SQL 中可能出现新的“有用物理属性”：

- 相同 prompt 前缀连续；
- 同一模型和输入长度区间连续；
- 可形成稳定 batch；
- 相同语义输入可缓存或去重。

优化器需要比较：

```text
额外 Sort / Repartition / 更换 Join 实现的成本
                        vs.
批处理、prefix cache 和模型吞吐带来的收益
```

#### 最小切口

先只研究一种属性：按 Join key 聚集同前缀请求。比较 Hash Join、Sort-Merge Join 和 `Hash Join + Sort` 在不同扇出、前缀长度和模型服务配置下的总延迟/token 成本，建立可供优化器使用的分段成本模型。

#### 风险

推理引擎的 batching 和 prefix cache 实现差异较大，需要区分可以跨引擎成立的优化原则与某个服务栈的工程结论。

### 3.5 R5：AI 派生结果作为可版本化访问路径

#### 问题定义

将已计算的 AI 结果表示为带血缘和质量元数据的物化对象，优化器在以下方案中选择：

- 直接复用；
- 对增量数据补算；
- 用新模型全量刷新；
- 把旧结果作为候选集，再用新模型验证；
- 放弃缓存并重新计算。

这个问题类似物化视图选择和维护，但“等价匹配”变成了带模型、prompt、参数、时间和质量的兼容性判断。

#### 最小切口

先限制为布尔 AI_FILTER 派生列，定义 `(data_version, prompt_version, model_version, decoding_config)` 血缘键，并研究模型升级时“全部重算”和“抽样校验后继续复用”的成本—质量权衡。

## 4. 现有研究边界：哪些宽泛表述已经不够准确

截至 2026 年 9 月，几个宽泛方向已经出现直接相关工作：

| 宽泛方向 | 已有代表工作 | 仍需寻找的具体差异 |
|---|---|---|
| AI 算子 placement | Horrila；Cortex AISQL | Join Order 耦合、不确定性下的鲁棒/自适应决策、与物理实现联合 |
| AI filter 选择率和顺序 | Larch；Cortex AISQL 运行时调整 | 多表 Join、相关谓词、decision-aware sampling、跨查询统计 |
| 物理实现/模型选择 | Abacus；Nirvana；Cortex AISQL 级联 | 实现选择对 placement、基数和关系计划的反馈 |
| Agentic rewriting | DocETL；MOAR；Nirvana | 特定规则的适用条件、等价性或质量契约，而非泛化“自动改写” |
| 运行时自适应 | Sema；Cortex AISQL | 编译期采样与运行期重优化的边界、触发规则和反馈复用 |

这意味着选题时应避免直接使用以下表述：

- “LLM 谓词没有选择率估计方法”；
- “还没有人研究 AI 算子 placement”；
- “现有系统不能选择不同模型”；
- “让 LLM 自动改写查询计划是空白”。

更可靠的做法是先构造现有方法失败的查询，再检查论文的搜索变量、假设和实验是否覆盖该反例。

## 5. 研究主线与优先切入点

### 5.1 研究主线

可以把总体研究主线暂定为：

> **不确定且依赖物理实现的 AI 统计信息下，混合关系—语义查询的计划优化。**

Join 是这条主线最合适的场景载体，因为它会放大选择率误差，使 placement、join order、缓存、batching 和物理实现发生直接耦合。但 Join 不是唯一研究对象，也不预设最终一定只做 semantic join。

### 5.2 起点 A：先解决“是否值得为计划选择做采样”

R1 可以作为一个相对独立、范围较小的起点：

> 给定包含 Join 和 AI_FILTER 的少量候选计划，优化器如何以尽可能少的付费样本，判断哪个 placement 或 join order 更优；如果编译期无法判断，何时转为运行时重优化？

这里的研究目标不只是提高选择率预测精度，而是让采样直接服务于计划决策：仅在选择率不确定性可能改变最优计划时，才为估计付出模型调用成本。

### 5.3 起点 B：验证“逻辑—物理两阶段优化是否会错”

R2 可以作为另一条重点验证的假设：

> 当 proxy、cascade 和 oracle 具有不同的有效选择率、成本和误差时，先 placement、后 physical selection 的两阶段方法是否会系统性错过更优计划？

第一步不必立即设计复杂算法，只需要：

1. 明确两阶段基线；
2. 构造能够逆转最优顺序或 placement 的反例；
3. 用穷举得到小规模联合最优；
4. 测量两阶段计划的成本差距和质量违约率；
5. 判断差距是否足以支撑后续算法研究。

如果实验显示两阶段方法的损失普遍很小，应及时放弃或缩小该方向；如果差距稳定存在，再研究联合搜索和质量模型。这样可以避免先搭一个很大的统一框架，最后才发现缺少足够强的研究动机。

### 5.4 两个起点的关系

两个起点不是互斥选题：

```text
起点 A 处理“统计信息不知道怎么办”
起点 B 处理“知道后要在什么联合空间里决策”
```

可以先各自做小规模反例和可行性实验，再决定：

- 哪一个现象更稳定、与现有工作差异更明确，就作为首篇工作的核心；
- 另一个作为扩展变量或后续工作；
- 不在第一阶段同时加入 semantic join rewrite、KV-cache、物化和完整 Join Order 搜索。

## 6. 下一步需要完成的验证

本文给出的是研究假设，不等于已经确认的创新点。下一步应按以下顺序收敛：

1. **精读边界**：逐项核对 Horrila、Larch、Abacus、Nirvana、Cortex AISQL 和 Sema 的搜索变量、成本模型、质量定义与实现；
2. **建立反例集**：至少为 R1、R2、R3 分别准备可执行查询，并明确数据规模、选择率、候选计划或改写形式；
3. **做小规模穷举实验**：用真实或模拟的模型成本、选择率和混淆矩阵判断问题是否具有稳定收益；
4. **再确定题目**：根据收益幅度、已有工作重合度和实现成本，在 R1、R2 与 R3 中选择主问题；
5. **最后扩展范围**：主问题成立后，再考虑 KV-cache、更多 rewrite 类型或物化复用。

## 7. 初版 DEMO 计划

### 7.1 DEMO 的目标

第一版 DEMO 不以实现完整优化器为目标，而是作为问题验证工具，回答三个问题：

1. 在什么条件下，AI 谓词选择率未知会导致 placement、谓词顺序或 join order 选错；
2. 当 AI 谓词可以选择不同模型或执行方式时，分阶段优化是否会稳定地错过联合最优计划；
3. 当优化器使用非等价或近似重写降低成本时，结果质量会怎样变化，以及什么条件下改写仍能满足查询级质量约束。

这三个宽泛问题都已有相关研究，不能整体视为尚未验证的空白。DEMO 包含两个不同层次的目标：

1. **已有现象复现**：复现论文已经观察到的成本或质量变化，用于校验数据、执行平台、指标和基线实现；这部分不作为创新点；
2. **候选缺口验证**：进入已有方法没有覆盖或只部分覆盖的条件，判断问题是否真实、稳定且具有足够收益；只有这部分可能进一步形成研究选题。

| 方向 | 已有工作已经覆盖的基本现象 | DEMO 需要验证的候选缺口 |
|---|---|---|
| R1 | AI 谓词选择率会影响求值顺序和成本；已有随机采样、固定选择率和学习式估计方法 | 多表 Join 中，能否只为可能改变 placement 或 join order 的不确定性付费采样，并联合决定停止采样和运行时重优化 |
| R2 | 已有工作分别研究 placement、模型选择和级联，Nirvana 还采用先逻辑优化、后物理优化的流程 | 当物理实现改变有效选择率和错误率时，分阶段优化是否在混合 Join 计划中稳定次优 |
| R3 | 已有工作已经展示 semantic join 分类改写、agentic rewrite 及其成本—质量变化 | 能否用可检查的标签、匹配关系和上下文条件判断 rewrite 是否适用，并预测查询级质量是否满足约束 |

DEMO 应先复现已有工作的基本现象，再构造候选缺口出现的条件，最后决定研究 R1、R2、R3，或其中更小的交叉问题。它需要同时支持：

- 控制数据规模、Join 扇出、谓词选择率和谓词相关性；
- 枚举并强制执行不同计划；
- 模拟不同 AI 实现的成本、选择率和错误率；
- 比较原始计划与近似改写计划的结果差异；
- 使用真实模型复核模拟实验中的关键反例；
- 记录关系执行、模型调用和结果质量三类指标。

### 7.2 技术底座：DuckDB + Flock + 外部实验控制器

[Flock](https://github.com/dais-polymtl/flock) 是 FlockMTL 的后续开源项目，以 DuckDB 扩展形式提供 `llm_filter`、`llm_complete`、`llm_embedding`、`llm_reduce` 和 `llm_rerank` 等函数，支持 OpenAI、Azure、Ollama 和 Anthropic。它还允许配置模型、prompt、批大小和异步执行，并通过 `flock_get_metrics()` 返回 API 调用数、token 数、API 时间和执行时间。

它适合作为 AI SQL 的**执行底座**，但目前不能直接作为完整的实验优化器。根据当前源码，`llm_filter` 仍以 DuckDB scalar function 注册，而不是带有专门 cost、selectivity 和 quality 属性的逻辑/物理算子；Flock 负责模型调用和批处理，DuckDB 的传统优化器看不到完整的 AI 成本模型。因此第一版采用以下结构：

```text
外部实验控制器（Python）
    ├── 生成数据和实验配置
    ├── 枚举候选计划
    ├── 生成用于强制计划的 SQL
    ├── 调用 DuckDB + Flock 执行
    └── 汇总成本、延迟和质量
                    |
                    v
DuckDB：关系算子、Join、物化、EXPLAIN ANALYZE
Flock：真实 AI 函数、模型接入、batching、调用指标
```

第一阶段不修改 DuckDB 优化器。待反例和收益得到确认后，再决定把搜索算法保留在外部控制器，还是实现为 DuckDB/Flock 的优化器扩展。

这里需要区分“执行引擎”“被测优化器”和“最优参照”：

| 角色 | 作用 | 由谁提供 |
|---|---|---|
| 执行引擎 | 按指定计划执行 Join 和 AI 函数，返回基数、调用量、token 和延迟 | DuckDB + Flock |
| 基线优化器 | 根据规则或估计统计信息，从候选计划中选择一个计划 | 实验控制器实现；算法来自传统启发式或已有论文 |
| 穷举 oracle | 执行或计算所有候选计划的真实成本，给出该实验配置下的最优计划 | 实验控制器实现，仅用于离线评测 |
| 待研究方法 | 根据有限采样选择计划，或联合选择 placement 和物理实现 | 问题验证成立后再实现 |

因此，第一版不是把查询交给某个现成 AI 优化器并接受其结果。DuckDB 和 Flock 负责执行；外部控制器负责生成有限的候选计划，并实现若干简单选择策略。穷举 oracle 由于需要尝试全部计划，不能用于线上优化，但可以为 DEMO 提供“真正最优”的参照。

### 7.3 分阶段实验设计

#### 阶段一：参数化模拟

先实现一个可配置、确定性的模拟 AI_FILTER，不调用真实模型。每个模拟算子具有：

```text
selectivity                 选择率
per_tuple_cost              单元组成本
false_positive_rate         假阳性率
false_negative_rate         假阴性率
batch_size                  批大小
cache_enabled               是否允许结果复用
```

模拟阶段的作用是快速扫描参数空间，寻找最优计划发生变化的边界。例如：

- 选择率从 0.1% 增加到 80% 时，最优 placement 在哪里发生变化；
- Join 扇出从 1 增加到 100 时，Join 前后求值的成本差距如何变化；
- proxy 的假阳性率增加到什么程度时，原来的最优谓词顺序被逆转；
- 重复输入比例和缓存命中率达到什么程度时，placement 收益消失；
- 两个 AI 谓词相关时，独立性假设会产生多大的 plan regret。

这一阶段可以穷举大量配置，且不存在真实模型费用和随机性干扰。

#### 阶段二：Flock 真实执行验证

从模拟实验中选择少量具有代表性的反例，使用 Flock 接入本地 Ollama 模型或外部模型 API，验证：

- 真实 AI 谓词是否表现出相似的选择率和相关性；
- proxy 和 oracle 是否会产生不同的有效选择率；
- batching、token 长度和异步执行是否改变计划排序；
- 模拟实验中的计划逆转能否在真实执行中复现。

为了保证不同计划可比，应尽量固定模型版本和生成参数，例如设置 `temperature=0`。对核心实验，可以预先物化每个 `(model, prompt, input)` 的输出，让不同计划复用同一批模型判断结果；否则模型自身的随机性可能被误认为计划质量差异。

#### 阶段三：优化方法原型

只有当前两个阶段确认问题稳定存在以后，再实现相应优化方法：

- 若主要问题来自估计成本，研究 decision-aware sampling；
- 若主要问题来自模型实现改变选择率和质量，研究 placement—implementation 联合搜索；
- 若近似改写具有显著成本收益但质量不稳定，研究改写的适用条件和查询级质量约束；
- 若缓存或 batching 经常逆转最优计划，再将其加入物理属性和代价模型。

### 7.4 第一组工作负载

第一组工作负载沿用第 2 节的两表电商场景：

```sql
products(pid, title, description)
reviews(rid, pid, text, stars)
```

主查询为：

```sql
SELECT DISTINCT p.pid, p.title
FROM products p
JOIN reviews r ON p.pid = r.pid
WHERE AI_FILTER('商品描述宣称静音', p.description)
  AND AI_FILTER('评论在抱怨噪音', r.text);
```

初始数据优先使用可控的合成文本。生成数据时显式控制：

| 参数 | 初始取值 |
|---|---|
| 商品数 | 100、1,000、10,000 |
| 每件商品评论数 | 1、5、10、50 |
| “宣称静音”选择率 | 1%、10%、50% |
| “抱怨噪音”选择率 | 1%、10%、50% |
| 两个语义条件的相关性 | 独立、正相关、负相关 |
| 重复商品描述比例 | 0%、50%、90% |

合成数据保留真实标签，因此可以直接计算 precision、recall 和 F1。真实模型验证阶段再选择 Amazon Reviews 等公开数据的子集，人工标注或使用固定 oracle 结果作为参照。

第二组工作负载使用第 2.3 节的三表查询，引入一个局部 Join Order 决策：

```sql
customers(cid, notes)
orders(oid, cid, pid, amount, item_desc)
products(pid, category)
```

它用于验证选择率估计误差是否会改变“先过滤 customers”还是“先过滤 orders 并缩小相关客户集合”的选择。第一阶段只允许两个明确的 Join 顺序，不直接枚举所有 Join 树。

第三组工作负载用于验证近似重写的成本—质量权衡。查询目标是将买家咨询匹配到标准 FAQ：

```sql
questions(qid, text)
faq(fid, label, answer)

SELECT q.qid, f.fid
FROM questions q
JOIN faq f
  ON AI_FILTER('该咨询可以由这条 FAQ 回答', q.text, f.answer);
```

首先使用较小规模的逐对判断结果或人工标注作为参照，再比较两类近似改写：

1. **分类改写**：把 FAQ 视为封闭标签集，每条咨询调用一次分类模型；
2. **检索后验证**：embedding 召回 Top-K FAQ，再由 LLM 逐对验证候选。

实验显式改变 FAQ 标签是否互斥、每条咨询的匹配数、标签集合是否封闭、FAQ 语义重叠程度和 Top-K。这样可以观察分类或召回改写在什么条件下降低成本，又在什么条件下产生不可接受的漏配。

此外增加一组上下文裁剪实验，对比：

```text
Join 后判断：输入商品属性和评论，判断“这条评论是否严重负面”
Join 前判断：只输入评论，判断“这条评论是否严重负面”
```

后者看似只是提前 placement，实际删除了跨表上下文，属于可能改变结果的近似 rewrite。该实验用于区分等价 placement 与上下文裁剪式改写。

### 7.5 第一版计划空间

两表查询先枚举以下维度：

| 决策维度 | 候选值 |
|---|---|
| 商品 AI 谓词 placement | Join 前、Join 后 |
| 评论 AI 谓词 placement | Join 前、Join 后 |
| 同一节点上的谓词顺序 | 商品优先、评论优先 |
| 物理实现 | oracle、proxy |
| batch size | 1、16 |
| 结果复用 | 不复用、物化复用 |

不是所有笛卡尔积组合都有不同语义或合法执行方式，实验控制器应在生成计划时去重并排除非法组合。预计第一版保留几十个候选计划，能够直接穷举得到真实最优值。

对每组数据和算子参数，穷举器首先计算：

```text
p* = argmin 实际成本(p)              -- 穷举得到的真实最优计划
```

被测优化器只根据规则或估计选择率选择：

```text
p_hat = argmin 估计成本(p)
```

两者的差距用 plan regret 衡量：

```text
plan regret = 实际成本(p_hat) / 实际成本(p*)
```

例如，被测优化器选择的计划实际成本为 100，穷举最优计划成本为 20，则 plan regret 为 5。这里的“实际成本”在模拟阶段由真实参数计算，在 Flock 验证阶段由真实执行指标得到。

R1 主要比较以下策略：

1. **DuckDB 默认计划**：作为工程基线，但它不知道完整的 AI 成本和选择率，不能作为唯一研究基线；
2. **固定下推**：语义允许时，把 AI 谓词放在尽可能靠近输入的位置；
3. **固定延后**：把 AI 谓词放在尽可能靠近输出的位置；
4. **固定选择率优化**：为所有 AI_FILTER 设置同一默认选择率，再根据估计成本选择 placement；可先实现 Horrila-like 的简化版本；
5. **固定采样后优化**：每个 AI 谓词统一采样 100 或 1,000 行，再根据点估计选择计划；
6. **真实统计优化**：将真实选择率交给同一个成本模型，用于区分“统计估计错误”和“成本模型错误”；
7. **联合穷举**：尝试所有候选计划，作为实验 oracle。

其中 Horrila 当前采用固定的 semantic filter 选择率进行 placement 搜索；Larch 已研究基于学习的 semantic filter 选择率和求值顺序。第一版可以根据论文描述实现与当前问题范围匹配的简化基线，不依赖它们的完整源码。

R2 主要比较以下策略：

1. **仅 placement**：固定物理实现，只选择位置和顺序；
2. **仅 physical selection**：固定位置，只选择模型；
3. **两阶段 P→I**：先确定 placement，再选择实现；
4. **两阶段 I→P**：先选择实现，再确定 placement；
5. **联合穷举**：同时选择 placement、顺序和实现，作为小规模 oracle。

联合穷举不是最终算法，而是用来衡量其他策略距离全局最优有多远。

R3 主要比较以下执行形式：

1. **逐对 semantic join**：对所有候选对执行 AI 判断，作为结果参照；
2. **分类改写**：每条左侧记录在右侧标签集合中分类；
3. **retrieval + verification**：embedding 召回 Top-K，再用 LLM 验证；
4. **上下文完整谓词**：在 Join 后使用两侧属性判断；
5. **上下文裁剪谓词**：只使用一侧属性提前执行。

R3 不只比较哪种形式成本最低，还需要在给定质量下限 `τ` 时选择可行计划：

```text
min    实际成本(p)
s.t.   Quality(p) >= τ
```

逐对判断也不应被直接假设为绝对真值。小规模实验优先使用人工标注作为 ground truth，同时报告近似改写相对逐对计划和相对人工标注的质量。

### 7.6 如何强制不同计划

DuckDB 会自行进行谓词下推、CTE 内联和 Join Order 优化。为了确保执行的确是指定计划，第一版优先使用临时表形成明确的物化边界，而不只依赖 SQL 文本顺序。

**计划 A：先 Join，再执行 AI 谓词。**

```sql
CREATE OR REPLACE TEMP TABLE joined AS
SELECT p.pid, p.title, p.description, r.text
FROM products p
JOIN reviews r ON p.pid = r.pid;

SELECT DISTINCT pid, title
FROM joined
WHERE llm_filter(/* 商品描述谓词 */)
  AND llm_filter(/* 评论谓词 */);
```

**计划 B：先过滤商品，再 Join，再过滤评论。**

```sql
CREATE OR REPLACE TEMP TABLE quiet_products AS
SELECT *
FROM products
WHERE llm_filter(/* 商品描述谓词 */);

CREATE OR REPLACE TEMP TABLE candidates AS
SELECT p.pid, p.title, r.text
FROM quiet_products p
JOIN reviews r ON p.pid = r.pid;

SELECT DISTINCT pid, title
FROM candidates
WHERE llm_filter(/* 评论谓词 */);
```

局部 Join Order 实验可以使用 DuckDB 提供的控制项：

```sql
SET disabled_optimizers = 'join_order,build_side_probe_side';
```

关闭相关优化后，DuckDB 按 SQL 中的 Join 顺序生成左深树。每次执行都需要保存 `EXPLAIN` 和 `EXPLAIN ANALYZE`，确认实际计划与预期一致。实验结束后恢复：

```sql
SET disabled_optimizers = '';
```

### 7.7 观测指标

每次计划执行前调用：

```sql
SELECT flock_reset_metrics();
```

执行后通过 `flock_get_metrics()` 获取模型侧指标，并通过 DuckDB profiling 获取关系侧指标。第一版统一记录：

| 类别 | 指标 | 来源 |
|---|---|---|
| 计划 | placement、join order、模型、batch size、缓存方式 | 实验配置 |
| 关系执行 | 各节点估计基数、实际基数、中间结果大小、算子时间 | `EXPLAIN ANALYZE` |
| AI 输入 | 输入元组数、distinct input 数、重复率 | 实验控制器或补充埋点 |
| 模型服务 | API calls、input/output tokens、API duration | `flock_get_metrics()` |
| 总体效率 | 端到端延迟、估算货币成本 | 控制器 + 模型价格表 |
| 结果质量 | precision、recall、F1、相对 oracle 保持率 | 标注结果或物化 oracle 输出 |
| 优化效果 | plan regret、质量约束违约率、优化耗时 | 实验控制器 |

其中需要明确区分：

```text
AI 输入元组数 ≠ distinct input 数 ≠ API calls
```

Flock 会把多个元组批量放入一次请求，因此 API calls 不能代替逻辑 AI 求值次数；而存在结果复用时，输入元组数也不能直接代表实际模型计算量。当前 Flock 指标未直接提供前两项，第一版可以由计划节点基数和物化表统计得到，必要时再给扩展增加轻量埋点。

### 7.8 需要验证的假设

R1、R2 和 R3 应先分开验证，避免把统计估计错误、决策空间耦合和近似改写造成的质量变化混为同一个原因。

**R1：固定实现，只改变统计信息。**

- 固定模型、prompt、单次成本和模型输出；
- 改变真实选择率、Join 扇出和谓词相关性；
- 向被测优化器提供不同精度的估计选择率；
- 比较其选择的计划 `p_hat` 与穷举计划 `p*`。

R1 首先回答：选择率变化是否会使最优 placement、谓词顺序或局部 join order 发生变化；估计错误造成的 plan regret 有多大；该差距是否在非极端参数下仍然存在。确认这些问题以后，才比较固定采样与 decision-aware sampling。

**R2：提供真实统计信息，只改变物理实现。**

- 把每种实现的真实成本、有效选择率和错误率直接提供给被测优化器，排除 R1 的估计误差；
- 改变 oracle、proxy 及其 batch 和缓存配置；
- 比较 P→I、I→P 与联合穷举；
- 观察最优顺序或 placement 是否因实现变化而逆转。

如果在统计信息完全准确时，两阶段方法仍有明显 plan regret 或质量违约，才能说明 R2 是独立的联合优化问题，而不只是 R1 的附属现象。

**R3：固定查询意图，改变计划的语义实现。**

- 使用同一批咨询、FAQ 和人工标注，分别执行逐对判断、分类和 retrieval + verification；
- 改变标签封闭性、语义重叠程度、每条记录的真实匹配数和 Top-K；
- 记录成本—质量曲线，而不是只比较端到端延迟；
- 在多个质量下限 `τ` 下，检查优化器能否选择最低成本的可行计划；
- 对上下文裁剪实验，测量提前执行节省的调用量以及删除跨表上下文造成的结果变化。

R3 首先回答：哪些数据和谓词条件会使特定 rewrite 近似保持查询意图；质量下降是否可预测；算子级 precision/recall 能否用于判断查询级约束是否满足。只有这些关系具有一定稳定性，才进一步研究可自动判定的 rewrite rule 或质量契约。

**已有现象复现**用于校验实验平台和基线，不作为创新点：

- **B1**：AI 谓词选择率变化能够改变最优求值顺序或 placement，错误估计会产生额外成本；
- **B2**：proxy、oracle 等实现具有不同的成本和结果质量，并可能表现出不同的有效选择率；
- **B3**：分类或 retrieval + verification 可以降低 semantic join 成本，但可能改变 precision、recall 和 F1；
- **B4**：删除跨表上下文以提前执行 AI 谓词，可能使结果与上下文完整的计划不同。

如果 B1—B4 无法复现，应先检查工作负载、指标和基线实现，而不是据此声称已有结论不成立。

**候选缺口验证**决定是否形成后续研究问题：

- **G1（R1）**：在多表 Join 中，只为可能改变计划排名的不确定性继续采样，可以比固定样本量获得更低的“采样成本 + 执行成本”，并能给出明确的停止条件；
- **G2（R2）**：即使已知真实统计信息，先 placement、后 physical selection 的两阶段方法仍会在非极端参数范围内产生显著 plan regret 或质量违约；
- **G3（R3）**：近似 rewrite 的质量损失与标签封闭性、语义重叠、匹配基数或上下文依赖等可观测条件存在稳定关系，优化器可以据此选择满足 `Quality(p) >= τ` 的最低成本计划；
- **G4（扩展）**：重复输入比例、缓存和 batch size 能够逆转最优计划，因此需要作为优化器的显式决策或物理属性。

G1—G4 都是可证伪假设，不要求全部成立。DEMO 的价值也包括尽早否定收益较小、只在极端条件下出现，或已被现有机制处理的问题。

### 7.9 初版产物与收敛条件

第一版 DEMO 的产物包括：

1. 可重复生成两表、三表和 FAQ 匹配数据的脚本；
2. 参数化模拟 AI 谓词；
3. 候选计划生成器与穷举执行器；
4. DuckDB/Flock 真实执行适配层；
5. 统一的实验结果表和可视化；
6. B1—B4 的复现实验，以及与已有工作结论的对照；
7. 至少一组支持 G1、G2、G3 或 G4 的稳定反例或边界实验；
8. semantic join 三种执行形式的成本—质量曲线，以及上下文完整/裁剪谓词的结果对比。

是否继续形成优化算法，可以依据以下条件判断：

- 反例是否只依赖人为构造的极端参数；
- 在真实模型和真实数据子集上能否复现；
- 两阶段方法相对联合最优的差距是否稳定且足够大；
- 近似改写的质量变化是否与可观测的数据或谓词条件存在稳定关系；
- 问题是否已被已有系统的某个未注意到的机制解决；
- 为解决问题引入的优化开销是否低于计划收益。

若 R1 的差距更稳定，下一步实现 decision-aware sampling；若 R2 的差距更稳定，下一步研究联合搜索；若 R3 展现出可预测的成本—质量边界，下一步研究 rewrite rule 的适用条件和质量契约。若三者收益都不明显，再根据实验结果转向 batching、缓存或物化复用，而不是继续扩大原问题。

## 参考材料

- [Cortex AISQL: A Production SQL Engine for Unstructured Data](https://arxiv.org/abs/2511.07663)
- [Horrila: Cost-Based Placement of Semantic Operators in Hybrid Query Plans](https://arxiv.org/abs/2604.09944)
- [Larch: Learned Query Optimization for Semantic Predicates](https://arxiv.org/abs/2606.07923)
- [Abacus: A Cost-Based Optimizer for Semantic Operator Systems](https://arxiv.org/abs/2505.14661)
- [Beyond Relational: Semantic-Aware Multi-Modal Analytics with LLM-Native Query Optimization（Nirvana）](https://arxiv.org/abs/2511.19830)
- [Sema: A High-performance System for LLM-based Semantic Query Processing](https://arxiv.org/abs/2603.11622)
- [DocETL: Agentic Query Rewriting and Evaluation for Complex Document Processing](https://arxiv.org/abs/2410.12189)
- [Multi-Objective Agentic Rewrites for Unstructured Data Processing（MOAR）](https://arxiv.org/abs/2512.02289)
- [Beyond Quacking: Deep Integration of Language Models and RAG into DuckDB（FlockMTL）](https://arxiv.org/abs/2504.01157)
- [Flock 项目仓库](https://github.com/dais-polymtl/flock)
- [DuckDB：强制 Join Order](https://duckdb.org/docs/current/guides/performance/join_operations)
- [DuckDB：EXPLAIN ANALYZE](https://duckdb.org/docs/current/guides/meta/explain_analyze)
