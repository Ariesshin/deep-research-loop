---
name: "deep-research-loop"
description: "目标随材料迭代 + 500+来源规模化检索 + 多视角对抗评审收敛的深度研究流程"
---

# Deep Research Loop 🔬🔁

> 把「定好计划就一路填数据」的研究，升级成**目标会随材料不断修订**、**来源规模到 500+**、**多视角对抗评审直到无异议**的研究流程。

适用：任何需要做出真实决策的研究（硬件选型、技术选型、市场调研、竞品分析、方案评估）。

## 与 deep-research-pro 的核心差别

| 维度 | deep-research-pro（旧） | 本流程（新） |
|---|---|---|
| 研究目标 | 一次定死 | **每轮可修订**，目标是变量 |
| 检索量 | 15–30 源 | **500+ 源**，进 ledger 累积 |
| 修订动力 | 无 | **材料暴露新维度即改写目标** |
| 质量门禁 | 无，写完就交 | **2–3 个对抗评审，全部 PASS + 120** |
| 可追溯 | 引用列表 | 来源 ledger + 目标版本史 + 评审记录 |

## 什么时候用哪个

- 一次性的资料汇总、时间紧 → 用旧流程（deep-research-pro）
- **要做决策、要开跑、预算和风险都要拍板** → 用本流程

## 五阶段流程

### Phase 0 — 探路（Seed Sweep）
- 按**宽泛**的初始问题，并行铺开第一批检索（3+ 并行检索员）
- 每个检索员负责一个子域，产出结构化来源（JSONL）
- **产出**：`sources-*.jsonl`（每子域 ≥120 条）+ 原材料池
- 关键：这一阶段**不做结论**，只求铺面和发现新维度

### Phase 0 执行规则（采集员生命周期，必守）

- **每个检索员的提示词里必须写死「写入哪个文件 + 每行 JSON 的确切 schema + key_facts 必须是数组」**。只给 schema 示例不够，否则有的采集员会把 `key_facts` 写成逗号字符串、或改用自己的字段名——三种 schema 混在一起，汇总时才发现，白跑。发车前把 schema 那一行原样贴进任务。
- **采集型 subagent 不回传完成通知**（collect=true 只写盘、不 announce）。它结束前**必须主动用 `agents_wait(ids=[...])` 收结果**，或用 `subagents(action=list)` 确认终态。**不得在派发后只 `sessions_yield` 等通知**——采集员不通知，yield 后上下文切走，结果就丢。
- **先看盘、再干活**：即使没收成通知，产物多半已写盘。恢复顺序：`ls sources/` → 数行数 → 读首行验 schema。不要因为「没收到总结」就重跑。
- **schema 不一致就标准化，不要重查**：不同采集员用不同字段名（如 `platform`/`price` 而非 `title`/`key_facts`）很常见；写一段归一脚本接受各种变体（`key_facts` 若是字符串则按分号切），合成统一 `ledger.jsonl` 再按 `id`+`url` 去重。只有当某来源**内容本身为空**时才视为废数据重查。

### Phase 1 — 目标修订（RQ Revision）
这是本流程的灵魂。从 Phase 0 材料里主动找：
1. **原目标没覆盖的维度** —— 材料里反复出现但初始问题没问的
2. **矛盾证据** —— 两个来源对同一事实给出不同数字
3. **新出现的子问题** —— 调研中冒出的、比原问题更关键的
4. **反直觉事实** —— 与常识相反、值得单列验证的

若研究对象是产品或竞品全景，在扩展检索前按 [产品全景分层](references/product-landscape.md) 检查分类、纳入边界和比较维度，避免后期才补搜整类产品。

产出 `rq-versions.md`：
```
RQ v1: <初始研究问题>
  → 触发修订的证据: ...
RQ v2: <修订后>
  → 触发修订的证据: ...
```
**每轮修订都必须写明"是什么证据触发了这次修订"。** 无证据不改目标。

### Phase 2 — 检索扩展（Expand）
- 针对 RQ 的新版本，定向补搜缺口
- 来源 ledger 累加，直至 ≥500 条
- 相同来源去重；同一事实必须有 ≥2 个独立来源才能进结论

### Phase 3 — 多视角评审（Loop Gate）
创建 2–3 个**立场互斥**的隔离评审 subagent。默认视角（可按任务调整）：

| 评审 | 立场 | 保护什么 |
|---|---|---|
| **目标守门人** | 我们问对问题了吗 | 防 intent failure |
| **证据检察官** | 凭什么这么说 | 防 resource/quality failure |
| **决策者** | 看完能拍板吗 | 防 execution failure |

**强制**：至少一个评审必须代表**最终使用者/决策人**（用户视角），不能全是技术正确性视角。

每轮所有评审都要收到完整报告。输出格式：
```
Verdict: PASS | REVISE
Score: 0-120
Pass rationale: ...
Evidence checked: ...
Must-fix issues: ...
Optional improvements: ...
120-level approval statement: ...
```
- **PASS 要求 Score: 120 且 Must-fix issues: none**
- 任一 REVISE → 回 Phase 1（可能连带修订目标）
- 最多 5 轮；同一 must-fix 连续两轮无实质修改 → 报 blocker
- **目标僵化也属 must-fix**：若评审发现"目标该改但没改"，按 REVISE 处理

### Phase 4 — 收敛与交付
- 所有评审 PASS + Score 120
- 交付：`report.md` + `sources-*.jsonl`（合计 500+）+ `rq-versions.md` + `reviewer-*.txt`
- 报告必须标注：数据快照时间、来源条数、置信度、未解缺口

## 强制产物

每次运行必须产出：
- `rq-versions.md` — 目标修订史（含触发证据）
- `sources/*.jsonl` — 来源 ledger，合计 ≥500 条
- `report.md` — 最终报告
- `reviewer-<facet>.txt` — 最终轮评审输出
- `run-manifest.json` — 运行元数据（轮次、来源数、评审结论、隔离状态）

## 来源 Ledger Schema

每行一条：
```json
{"id":"gpu-001","title":"...","url":"...","date":"YYYY-MM-DD|unknown","type":"评测|商品页|新闻|论坛|文档","domain":"...","key_facts":["可引用的具体数据"],"confidence":"high|medium|low","raw_snippet":"..."}
```
规则：
- `key_facts` 必须是具体数据（价格/速度/容量/条款），禁止空泛描述
- 日期精确，查不到写 `unknown`
- 单来源支撑的事实，报告中必须显式标注"未交叉验证"
- **禁止编造 URL 或数据；查不到写"未找到"**

## 评审包（发送给评审 subagent 的内容）

```
You are an isolated reviewer.

Review facet: <视角名>
Your mission: <该视角要保护什么>
Artifact's true job: <报告必须达成什么>
Why distinct: <与其他视角的隔离张力>
User requirements: <预算/用途/约束/验收>
Current artifact: <报告全文>
Evidence: <来源 ledger 统计 + 关键来源>

不假设包外上下文。若发现目标本身该修订，按 REVISE 给出。
```

## 反模式（必须避免）

- ❌ 目标一开始定死，中途不修订
- ❌ 三个评审都在评"质量"（同质化）
- ❌ 凑来源数量但不读内容
- ❌ 用单一来源下结论
- ❌ 没有"使用者视角"评审
- ❌ 数据不标日期/时效，混用不同时间点的报价

## 阻塞条件

- 两个评审给出无法同时满足的矛盾指令 → 升级给用户裁决
- 同一 must-fix 连续两轮无实质修改 → 报阻塞
- 工具反复失败 / 预算耗尽 → 报阻塞

阻塞格式：`Blocked at / Reason / Evidence / What was tried / User decision needed`

## 收敛判定

**仅当**所有选定评审在最终轮返回 `PASS` + `Score: 120` + `Must-fix issues: none`，且来源数 ≥500，且存在目标修订记录（除非首轮即收敛并说明理由），才可宣称完成。
