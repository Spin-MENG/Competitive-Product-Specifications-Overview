# 详细工作流 · workflow.md

本文档是 SKILL.md 6 步主流程的展开。**只在执行流程时读这个文件**，不要提前全读。

---

## Step 1 · 收集输入并提取锚点

### 1.1 收集用户输入

向用户问 4 项（已提供的跳过）：

| 字段 | 必需 | 类型 | 示例 |
|---|---|---|---|
| `csv_path` | 可选 | 文件路径 | `dataset_amazon-product-scraper.csv` |
| `category` | 必需 | enum | `router` / `kitchen` / `saas` / `custom` |
| `ticket_path` | 可选 | 文件路径 | `docs/ticket.md` |
| `ppt_path` | 可选 | 文件路径 | `MeshNode方案.pptx` |
| `target_n` | 可选 | int (默认 10-12) | `12` |
| `market` | 可选 | str | `EU 5 站点` / `US` / `中国` |

### 1.2 解析 CSV → seed_products

如有 CSV：

```python
# 读 CSV，提取 brand / model / ASIN 三元组
# 去重（同一 model 跨多市场只算 1）
# 输出：seed_products = [{brand, model, asin, ...}, ...]
```

CSV 字段映射要灵活——Amazon scraper 格式、Helium10 导出、自建 CSV 字段名各不同。先 head -3 看 header，再决定映射。

### 1.3 解析 ticket + PPT → 锚点

**ticket.md** 通常含明确目标售价、产品定位、不做什么。读全文，**人工抽取**：

```yaml
anchor_explicit:
  target_price_usd: 80
  positioning: "接 Flint 主路由做节点扩展"
  not_doing: ["独立路由能力"]

anchor_open:
  - wifi_generation       # ticket / PPT 未定
  - form_factor           # 列为待探讨
  - mesh_protocol         # 未定
  - chipset               # 未定
```

**PPT 解析**用 `python-pptx`：
```bash
pip install python-pptx --quiet
python3 -c "from pptx import Presentation; p=Presentation('xxx.pptx'); [print(s.text_frame.text) for slide in p.slides for s in slide.shapes if s.has_text_frame]"
```

特别留意 "后续希望探讨的问题" / "待定" / "TBD" 等关键词——这些章节的内容**必须**进 anchor_open 黑名单。

### 1.4 输出 Step 1 总结给用户确认

格式：

```
**输入摘要**
- CSV：8 行 → 4 个独立 SKU（[列名]）
- 品类：router
- ticket / PPT 明示项：目标售价 ~$80 / 接 Flint 主路由 / 不做独立路由
- ticket / PPT 待探讨项：WiFi 代际 / 形态 / Mesh 协议 / Chipset

请确认是否继续。
```

---

## Step 2 · 候选扩展到 Top N

### 2.1 缺口计算

```
need_more = target_n - len(seed_products)
```

CSV 4 个 + 目标 12 个 → 需补 8 个。

### 2.2 用 WebSearch 找候选

按品类 + 市场 + 时间 范围搜索 "Top mesh router 2026 Europe" 等关键词。**关键是覆盖品牌阵营**（TP-Link / NETGEAR / ASUS / Linksys / Google / eero / AVM / Devolo）和**形态多样性**（mesh kit / single repeater / powerline-hybrid / plug-in）。

### 2.3 向用户列出候选名单确认

```
建议补 8 款到 Top 12：
  Mesh 套装：ASUS ZenWiFi BT8 / Linksys Velop Pro 7 / Google Nest Wifi Pro / TP-Link Deco X50
  Repeater：AVM FRITZ!Repeater 6000 / TP-Link RE815XE / NETGEAR EAX80 / Devolo Mesh WiFi 2

候选名单是否调整？
```

用户确认后进 Step 3。**不确认就停**——别擅自扩。

---

## Step 3 · 加载品类 schema

```bash
# 读 configs/<category>.yaml
# 解析 fields[], standardize{}, viz{}, price_unit
```

如果 category=`custom`，让用户提供 YAML 路径（或在线编辑）。

---

## Step 4 · 并行 research agent 拉规格

### 4.1 分批

每个 agent 负责 3-4 产品。`N_agents = ceil(target_n / 4)`，**最多 4 个 agent 并行**（再多 token 浪费 + 收尾管理成本高）。

### 4.2 派发 prompt

模板见 [`agent-prompt.md`](agent-prompt.md)。每个 agent 接收：
- 品类 schema 字段清单
- 该 agent 负责的产品名单
- 输出格式要求（markdown 表格 + source URL 列表）
- **校验规则**：必须用 WebSearch / WebFetch；必须 ≥1 真实 URL；推测字段标 "(推测)"；不知道就标 "未公开" 不许编

**关键**：用 `run_in_background: true` 并行起多个 agent，**单条消息多 Agent 调用**最大化并行。

### 4.3 接收 + 校验

每个 agent 完成回来后立刻校验：

```
检查项：
[ ] 输出 markdown 表格行数 = 派发产品数
[ ] 每行至少有 1 个非"未公开"字段
[ ] 表格下方有 "## 资料来源" 或类似 URL 清单
[ ] URL 不全是 amazon.com 商详（要有厂商页 / 评测站 / FCC）
[ ] agent 是否在文字中声明"无法联网"——若是，重派或主控自己用 WebSearch 校验关键字段
```

不合格 → SendMessage 让 agent 修正，或主控 WebSearch 自补 1-2 关键字段后通过。

### 4.4 主控二次校验

特别是 **CSV 给的 4 个 SKU 之外**的"补充候选" + **agent 自称已查到的关键字段**，主控用 WebSearch 抽查 2-3 个：

- 关键字段：chipset 型号、RAM 容量、价格
- 抽查 hit ratio < 70% → 整批结果不可信，重做

这一步就是 MeshNode 项目里发现 B agent 写错 ASUS BT8 频段配置的环节。

---

## Step 5 · 合并 → CSV

```python
# 按 schema 字段顺序合并所有 agent 输出
# 缺失字段填 "未公开"
# 价格统一货币（按 schema 的 price_unit）
# CSV 表头 = required_fields + optional_fields + ["source_note"]
```

落盘到：
- 有 ticket 目录：`<root>/docs/data/analyzed/spec_matrix_<category>_top<N>_<YYYYMMDD>.csv`
- 无 ticket 目录：`./spec_inventory_<category>_<YYYYMMDD>/spec_matrix.csv`

---

## Step 6 · 委托 html-report skill 生成 HTML 报告

**核心原则**：本 plugin **不自维护** HTML 模板，统一委托给 `data-team-skills:html-report` skill。这样模板升级、配色调整、组件扩展由 html-report skill owner 集中维护，本 plugin 自动受益。

### 6.1 准备 section brief

把 Step 1-5 沉淀的数据打包成一份 brief，供 html-report skill 渲染。Brief 是一个内存数据结构（也可落到 markdown 文件方便 html-report 读取）：

```yaml
report_title: "<品类> 主流竞品 Top <N> 规格盘点 — <项目名> (Ticket <编号>)"
project_badge: "<组织> · <工作单元>"
meta:
  ticket: "<编号> · <slug>"
  data_date: "<YYYY-MM-DD>"
  market: "<EU 5 站点 / US / CN / ...>"
  work_unit: "U12 · 规格盘点"

sections:
  overview:                 # Section 1
    kpi_cards: [...]        # 4 个 ad-kpi-card 数据
    anchor_explicit: {...}  # PPT 明示项
    anchor_open: [...]      # PPT 待探讨项
    insight_type: warning   # 用 .insight.warning 框

  distribution:             # Section 2
    charts:
      - id: proto, type: donut, data: [...], title: "WiFi 协议代际分布"
      - id: form,  type: donut, data: [...], title: "产品形态分布"
      - id: band,  type: bar,   data: [...], title: "频段配置分布"
      - id: chipset, type: donut, data: [...], title: "Chipset 阵营"
    insight: "..."

  pricing_matrix:           # Section 3
    scatter:
      mesh_data: [...]
      repeater_data: [...]
      anchor_data: [...]    # **遵守 anchor isolation 规则**
    table:
      headers: [...]
      rows: [...]
    insight: "..."

  summary:                  # Section 4
    findings: [...]         # 左列：客观结论
    recommendations: [...]  # 右列：建议（带 disclaimer banner）
    field_gaps: [...]       # glossary 列表

footer_sources: "..."
output_path: "<编号>_<slug>/<编号>_<slug>/U12_spec_inventory_top<N>.html"
```

### 6.2 调用 html-report skill

通过 Skill 工具切换到 html-report 上下文，让它执行自己的 Step 1-6（读 template.html → 套 brief 数据 → 落盘）：

```
Skill(skill: "data-team-skills:html-report", args: "<brief 摘要 + brief 文件路径>")
```

或者，如果 Skill 工具不可用 / 同事是手动 clone 方式安装，**fallback** 为：

```
Read /Users/<user>/.claude/plugins/cache/data-team-skills/.../skills/html-report/assets/template.html
→ 在主 conversation 内按 html-report 工作流套模板
→ 输出到 output_path
```

两种路径都要把 brief 中的 anchor 规则透传过去：
- 散点图新品锚点**只用 anchor_explicit** 字段
- 雷达图 ≥ 2 维落在 anchor_open → 不画新品锚点
- 推荐章节顶部加 orange disclaimer banner

### 6.3 强制 anchor 规则（无论谁渲染都要遵守）

详见 [anchor-isolation.md](anchor-isolation.md)：

- 散点 x 轴若为 anchor_open 字段 → 用占位值 0.5 + label "（代际待定）"
- 雷达若 ≥ 2 维落在 anchor_open → 不画新品锚点 + 章节下方 insight 注明
- 推荐章节顶部 disclaimer：
  ```
  ⚠️ 本节是 U12 分析师推论建议，非 PPT 明示。
  ```

### 6.4 浏览器验证

```bash
open <output_path>
```

让用户确认：
- 图表是否正确渲染（黄昏绮景配色 + ECharts）
- 字体加载（Plus Jakarta Sans + Noto Sans SC）
- 响应式（窄屏正常换行）
- 数据是否对齐表格行数
- anchor isolation 规则是否生效（新品在 anchor_open 字段是否未被预设）

---

## 完整 timing 估算

| Step | 单人手动 | 用本 skill |
|---|---|---|
| 1. 收集 + 锚点解析 | 30 min | 5 min |
| 2. 候选扩展 | 60 min | 10 min |
| 3. schema 加载 | — | < 1 min |
| 4. 并行规格抓取 | 4-6 h | 30 min（并行 + 校验） |
| 5. CSV 合并 | 30 min | < 5 min |
| 6. HTML 报告 | 4 h | 30 min |
| **合计** | **9-11 h** | **1.5 h** |

节省 ~85%，主要省在并行抓取 + HTML 模板化两块。
