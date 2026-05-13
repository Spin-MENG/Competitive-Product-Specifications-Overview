---
name: spec-inventory
description: 竞品规格盘点流水线 —— 给定 CSV 种子或 ASIN 列表，扩展到 Top N 主流竞品，并行 web research agent 拉硬件规格 / 协议代际 / 价格，输出 spec_matrix CSV + 黄昏绮景配色 HTML 报告。**硬性区分 ticket / PPT 明示项 vs 待探讨项**，禁止 agent 给"待探讨项"自动填值（避免越权预设）。触发场景：用户给一份 Amazon scraper CSV 要"做竞品规格盘点"/"竞品硬件对比"/"Top N 主流产品的协议代际盘点"；或用户显式调用 `/spec-inventory`。本 skill **仅在显式调用 / 明确需求时触发**，不要主动加载。
---

# spec-inventory · 竞品规格盘点流水线

## 这个 skill 解决什么问题

产品立项 / 竞争分析时，需要对**主流竞品做硬件规格、协议代际、价格的盘点**。手动做这件事要：

1. 从 Amazon scraper / 厂商页 / 评测站交叉查 chipset / RAM / Flash / 天线 / 端口（CSV 文案里没有）
2. 反复掉进"未上市新品锚点越权预设"陷阱（PPT 没说的字段被分析师自己填了）
3. 反复做 ECharts 配色 / 表格组件等模板化工作

本 skill 沉淀以上三件事到一个 6 步可执行 pipeline。

## 核心契约（不要违反）

1. **明示项 vs 待探讨项硬隔离**：从 ticket / PPT 提取的"立项明示"信息（如目标售价、不做独立路由）才能在 HTML 锚点 / 推荐章节中预设；"待探讨"字段（如 chipset / WiFi 代际 / 形态）禁止 agent 自动填值。详见 [`references/anchor-isolation.md`](references/anchor-isolation.md)。

2. **子 agent 输出必须有真实 URL**：并行 research agent 的产出**必须包含 ≥1 个真实可访问的 source URL**（厂商官网 / FCC database / 评测站等）；agent 若声明"无法联网"或全部字段都是推测，**降级标"未公开"**，不接受瞎填。详见 [`references/agent-prompt.md`](references/agent-prompt.md)。

3. **HTML 输出统一调用 `data-team-skills:html-report` skill**：本 plugin **不自维护** HTML 模板 fork，所有报告生成必须委托给 html-report skill（黄昏绮景 7 色 + ECharts v5 + Plus Jakarta Sans/Noto Sans SC + 4 section 结构）。这样 html-report 升级时本 plugin 自动受益，避免模板分叉。

4. **按品类 schema 驱动**：字段定义 + **雷达 6 维评分规则** 全部来自 `configs/<category>.yaml`，不要在 skill 主流程或 HTML 模板里硬编码"路由器有 chipset 字段"或"雷达画 WiFi 代际"。雷达评分规则支持 3 种类型：`enum`（枚举映射）/ `numeric_extract`（regex 抽数字 + scale）/ `percentile`（盘点内百分位 1-5），详见 [`references/workflow.md`](references/workflow.md) Step 5.5。

## 触发与不触发

**触发**（满足任一）：
- 用户显式调用 `/spec-inventory` 或 `/competitive-product-specs:spec-inventory`
- 用户给一份 Amazon / 电商 / 评测站 scraper CSV，明确要"做竞品规格盘点"/"硬件规格对比"/"协议代际盘点"
- 用户给一份产品 ticket / PPT，要为新品立项做"Top N 主流竞品 spec matrix"

**不触发**：
- 用户只说"分析这堆评论"→ 走评论分析 skill（如 `social-reviews-analyzer`）
- 用户要"用户画像" / "聚类" → 走 `customer-persona-clustering`
- 用户要"写报告 HTML" 但没指明竞品盘点 → 走 `html-report`
- 用户在问"这个能不能做"等闲聊性问题 → 不要加载

## 6 步主流程

详细每步操作见 [`references/workflow.md`](references/workflow.md)。

### Step 1 · 收集输入并提取锚点

- **CSV 路径**（可选）：用户给的 scraper 输出，从中提取 brand / model / ASIN
- **品类**：路由 / 厨具 / SaaS / 自定义。对应加载 `configs/<category>.yaml`
- **ticket / PPT 路径**（可选）：用于提取明示项 + 待探讨项

提取出三类信息：
1. `seed_products`：CSV 里的产品（去重后）
2. `anchor_explicit`：ticket / PPT **明示**的产品参数（如目标售价 ~$80）
3. `anchor_open`：ticket / PPT **明示待探讨**的字段（如形态 / 协议代际未定）

### Step 2 · 候选扩展到 Top N

CSV 通常只覆盖少数 SKU。结合品类常识 + web search 补全到 Top N（默认 10-12）。**让用户确认候选名单**再继续。

### Step 3 · 加载品类 schema

```yaml
# configs/router.yaml 示例
category: router
required_fields:
  - brand, model, category, wifi_protocol, ports, chipset, ram_mb, ...
optional_fields:
  - mlo_support, mesh_protocol, max_nodes, ...
standardize:
  chipset: "Vendor Model @ClockGHz"   # 规范化模板
  price: "EUR (3-pack 等效)"
viz:
  radar_dims: [generation, bands, port_speed, coverage, value, openness]
  scatter: [x=generation, y=price]
```

### Step 4 · 并行 research agent 拉规格

分批 3-4 产品/agent，**N 个并行**。每个 agent 拿到品类 schema + 产品清单 + agent-prompt 模板。

**输出校验**：每 agent 返回后，检查
- ≥1 真实 URL（不接受"无法联网"）
- 未填字段标"未公开"
- 推测字段必须标注"(推测)"
- 跨 agent 重复产品（误派）排除

不合格 agent **重新派发**或降级标记。

### Step 5 · 合并 → CSV

按 schema 字段顺序合并所有 agent 输出到 `spec_matrix_<category>_top<N>_<date>.csv`。CSV 是后续 U21 价格分布 / U22 主报告整合的输入。

### Step 5.5 · 雷达评分计算（schema-driven）

读 `configs/<category>.yaml` 的 `viz.radar`，按 6 个 dim 配置（enum / numeric_extract / percentile）把 CSV 原始字段值映射到 1-5 整数分。按 `grouping_field`（如 router 的 `category` / kitchen 的 `subcategory`）分组，每组 ≥ `min_per_group` 才画。详见 [`references/workflow.md`](references/workflow.md) Step 5.5。

**anchor isolation 在雷达图强制规则**：新品 6 维中至少 5 维必须在 `anchor_explicit` 才允许画占位锚点；否则**不画新品雷达系列**（MeshNode 这次 6 维中只有"性价比"有 PPT 依据，故不画）。

### Step 6 · 委托 html-report skill 生成 HTML 报告

**不要在本 skill 内自己写 HTML 骨架**。把 Step 1-5 准备好的数据打包成 section brief，通过 `Skill` 工具调用 `data-team-skills:html-report`，让它用自己的 `assets/template.html` + 工厂函数 `C` / `tooltipStyle()` / `axisStyle()` / `lerpColor()` 渲染。

**handoff 时给 html-report skill 的输入清单**（一句话 brief + 数据文件路径）：

| 输入 | 内容 |
|---|---|
| 报告标题 | `<品类> 主流竞品 Top <N> 规格盘点` |
| 数据来源 | CSV / 厂商官网 / FCC database / 评测站 |
| 章节结构 | 4 section：概览 → 代际+形态分布 → 价格×代际散点+规格矩阵 → 结论与建议 |
| 数据文件 | `spec_matrix_<category>_top<N>_<date>.csv`（Step 5 产物）|
| anchor 信息 | `anchor_explicit` 字典 + `anchor_open` 列表（Step 1 提取） |
| 图表清单 | 4 张分布图（donut/donut/bar/donut）+ 1 张散点图（含新品锚点）|

**4 个章节的内容由本 skill 准备**（html-report 负责套模板）：

1. **概览** · 4 个 ad-kpi-card（SKU 数 / 类别拆分 / 价格区间 / 代际覆盖）+ orange warning insight 交代 anchor_explicit / anchor_open
2. **代际与形态分布** · 4 张 ECharts 图（协议 donut / 形态 donut / 频段 bar / chipset donut）+ teal insight 观察
3. **价格 × 代际散点 + 完整规格矩阵** · scatter + `.asin-tbl` 表 + orange warning 解读
4. **结论与建议** · `.summary-grid` 双列（左 = 客观结论 / 右 = 分析师建议 + disclaimer）+ glossary 字段空白说明

**HTML 内 anchor 规则**（强制 · 在传给 html-report 的 brief 中明示）：
- 新品的散点 / 雷达只在 `anchor_explicit` 维度放点
- `anchor_open` 维度的图禁止画新品占位（避免预设）
- 章节底部 warning insight 明确列出"PPT 明示项"和"PPT 待探讨项"两份清单

## 路径约定

输入 ticket 目录（用户已有的项目根，如 `65_MeshNode-Performance-Price-Conversion-Forecast/`）：

```
<编号>_<slug>/
├── docs/data/analyzed/
│   └── spec_matrix_<category>_top<N>_<date>.csv     ← Step 5 产物
└── <编号>_<slug>/                                    ← 交付包子目录
    └── U12_spec_inventory_top<N>.html               ← Step 6 产物（最终报告）
```

如果用户没有 ticket 目录（独立竞品调研），落到 cwd 下：
```
./spec_inventory_<category>_<date>/
├── spec_matrix.csv
└── report.html
```

## 反模式

- ❌ Agent 没用 WebSearch / WebFetch 就交活 → 必须校验 source_urls，不接受
- ❌ 给 anchor_open 字段预设值（如"MeshNode 走 WiFi 7 dual-band"，PPT 没说就不能写）
- ❌ 改黄昏绮景配色 / 字体
- ❌ 引入 Chart.js / D3 等其他图表库
- ❌ 当用户只是问"能不能做"闲聊时主动加载本 skill
- ❌ 雷达图给只有 4 个产品的子集硬画 → 信息密度不够时换成 scatter / table（这次 Repeater 雷达就是这样砍掉的）

## 文件结构

```
skills/spec-inventory/
├── SKILL.md                          # 你正在读的这个文件
├── assets/
│   └── csv-schema.md                 # 公共 CSV 字段定义
│                                     # （HTML 模板委托 data-team-skills:html-report，本目录不存 HTML fork）
├── configs/
│   ├── router.yaml                   # 路由器品类（含 MeshNode 项目沉淀的字段）
│   ├── kitchen.yaml                  # 厨具品类（最小骨架，按需扩展）
│   └── saas.yaml                     # SaaS 品类（最小骨架，按需扩展）
└── references/
    ├── workflow.md                   # 详细每步操作
    ├── anchor-isolation.md           # 明示项 vs 待探讨项 隔离规则
    └── agent-prompt.md               # 子 agent prompt 模板 + 校验
```

## 依赖

本 plugin **强依赖** `data-team-skills:html-report` skill（GL.iNet 数据组内部）。
请先安装 data-team-skills plugin，否则 Step 6 无法完成。详见 [README.md](../../README.md) 安装章节。

## 参考来源

- **黄昏绮景配色**：源于 GL.iNet 数据组 `html-report` skill（外部配色出处 `References/IMG_8521.JPG` 科研Sci配色）
- **流水线方法论**：源于 ticket #65 MeshNode 项目 U12 工作单元
- **anchor isolation 教训**：MeshNode 项目第一版雷达图错误预设 6 维分数（仅价格档来自 PPT 明示），被需求方质疑后回查 PPT slide 7 明示 ID 形态 / 性能档为"待探讨"，遂校正
- **agent 校验教训**：MeshNode 项目并行 B agent 声明"无法实际联网"但仍输出大量"推测"字段，主控用 WebSearch 实地核验后发现 ASUS BT8 实际是 tri-band BE14000 而非 agent 写的 dual-band BE7200
