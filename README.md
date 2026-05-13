# Competitive Product Specifications Overview

> 竞品规格盘点流水线 Claude Code Plugin —— 给定 CSV 种子，扩展到 Top N 主流竞品，并行 web research 拉硬件规格 / 协议代际 / 价格，输出 spec_matrix CSV + 黄昏绮景配色 HTML 报告。

**核心差异化**：硬性区分 ticket / PPT 明示项 vs 待探讨项，禁止子 agent 给"待探讨项"自动填值（避免越权预设）。

---

## 这个 plugin 解决的痛点

产品立项 / 竞争分析时反复要做的事：

1. 从 Amazon scraper / 厂商页 / 评测站交叉查 chipset / RAM / Flash / 天线 / 端口（Amazon 商详文案里没有）
2. **不小心给"未上市新品"的待定字段预设了值**（被需求方质疑后才发现 PPT 明确写了"待探讨"）
3. 反复做 ECharts 配色 / 表格组件 / 章节骨架等模板化工作
4. 子 agent 可能声明"无法联网"但仍输出大量"推测"字段（B agent 翻车）

本 plugin 沉淀以上 4 件事到一个 6 步可执行 pipeline。**预计单次盘点从 9-11 h 压到 1.5 h（节省 ~85%）**。

---

## 前置依赖

本 plugin 的 Step 6（HTML 报告生成）**强依赖** `data-team-skills:html-report` skill（GL.iNet 数据组内部）。

**为什么**：我们不在这里维护一份 HTML 模板 fork —— 配色 / 字体 / 组件类名 / ECharts 工厂函数全部由 html-report skill owner 集中维护，避免分叉。本 plugin 把 Step 1-5 的数据打包成 section brief，委托 html-report 完成最后一公里渲染。

**如何获取 html-report**：

- GL.iNet 数据组同事：装 `data-team-skills` plugin（含 html-report 等 5 个 skill）
- 外部用户：fork 本 repo，把 SKILL.md / workflow.md 里 "委托 html-report" 部分替换为你自己的 HTML 模板路径，或参考 [GL.iNet 数据组 html-report skill 设计系统](#致谢)（黄昏绮景 7 色 + ECharts v5）自建

## 安装方式

### 选项 1 · marketplace（推荐）

在 Claude Code 里加 marketplace 来源：

```bash
claude marketplace add Spin-MENG/Competitive-Product-Specifications-Overview
```

然后启用本 plugin：

```bash
claude plugin install competitive-product-specs
```

### 选项 2 · 手动 clone

```bash
git clone https://github.com/Spin-MENG/Competitive-Product-Specifications-Overview.git ~/.claude/plugins/competitive-product-specs
```

重启 Claude Code 即可生效。**确保已先装 data-team-skills 或等效 html-report skill。**

---

## 触发方式

### 方式 A · slash command

```
/spec-inventory
```

或带 plugin 命名空间：

```
/competitive-product-specs:spec-inventory
```

### 方式 B · 自然语言

下列任一描述会触发本 skill：

- "这份 CSV 是 Amazon 抓的竞品，做一份 Top 10 主流路由器规格盘点"
- "为我们 [新品名] 立项做竞品硬件 / 协议代际盘点"
- "这份 ticket 的 U12 帮我做规格盘点"

---

## 输入

| 输入 | 必填 | 来源 |
|---|---|---|
| 品类 | ✓ | `router` / `kitchen` / `saas` / 自定义 |
| CSV 路径 | – | Amazon / Helium10 / 自建 scraper 输出 |
| ticket / PPT 路径 | – | 用于提取 anchor_explicit + anchor_open |
| Top N | – | 默认 10-12 |
| 目标市场 | – | 默认按品类 YAML 的 `market_default` |

---

## 输出

### 1. spec_matrix CSV

```
spec_matrix_<category>_top<N>_<YYYYMMDD>.csv
```

字段由 `configs/<category>.yaml` 定义。CSV 是后续报告整合 / 价格分布 / 销量建模的输入。

### 2. 黄昏绮景 HTML 报告

```
U12_spec_inventory_top<N>.html
```

4 sections：
1. 概览 + KPI 卡 + 锚点说明
2. 协议代际 + 形态 + 频段 + Chipset 阵营 4 张图
3. 6 维雷达对比 + 价格×代际散点 + **配置综合分×价格 性价比四象限 matrix** + 完整规格矩阵表
4. 结论与建议（左列客观结论 / 右列分析师推论 + disclaimer）

**性价比四象限 matrix**（YAML `viz.value_matrix.enabled` 启用）：

```
       价格高
         │
  溢价警示 │ 高端旗舰
   (⚠)   │   (★)
─────────┼─────────  ← Y 中位数
   入门  │ 性价比甜点
   (○)   │   (✓)
         │
       价格低
低配置 ◄────► 高配置
       ↑ X 中位数
```

X 轴 = 雷达 N 维（排除 value 维）总和 ÷ 维数 → 1-5；Y 轴 = 价格中位数；自动按盘点 12 款的 x/y 中位数切 4 象限。**anchor isolation**：新品 x 任一维度待定 → 只画 y 价格水平线，不画 x 散点。

---

## 3 个品类示例

### `configs/router.yaml`（最完整，沉淀自 GL.iNet MeshNode 项目）

- 14 个 required 字段：brand / model / wifi_protocol / phy_rate / band_config / mlo_support / mesh_protocol / wan_lan_ports / antenna / chipset_soc / ram_mb / flash_mb / coverage / form_factor / price
- 视化：协议代际 donut / 形态 donut / 频段 bar / Chipset donut / 价格×代际 scatter / 6 维 radar
- 雷达 6 维（schema-driven，YAML 可改）：WiFi 代际 / 频段数 / 端口速率 / 覆盖面积 / 性价比 / 生态开放度

### `configs/kitchen.yaml`（骨架示例）

- 8 个 required 字段：brand / model / subcategory / capacity / power / control_type / temp_range / safety_cert / price
- 适用：空气炸锅 / 料理机 / 电饭煲 / 净水器 / 咖啡机
- 雷达 6 维：容量 / 功率 / 智能化 / 安全认证 / 性价比 / 静音度

### `configs/saas.yaml`（骨架示例）

- 9 个 required 字段：brand / product_name / subcategory / deployment_model / pricing_tiers / api_availability / sso_support / data_residency / compliance
- 适用：CRM / 数据分析 / 团队协作 / 客服系统
- 雷达 6 维：集成生态 / 合规深度 / API 开放度 / 认证安全 / 性价比 / 部署灵活度

### 自定义品类

复制 `configs/router.yaml` 改字段即可。结构对 Claude 透明，新加 YAML 自动可用。

**雷达维度高度可配**——3 种评分规则类型即可覆盖绝大多数场景：

| 规则类型 | 适用 | 例 |
|---|---|---|
| `enum` | 字段是离散枚举（含 `match_mode: contains` 处理含子串场景） | WiFi 协议 / 控制方式 / Mesh 协议 |
| `numeric_extract` | 字段是带单位的文本，需 regex 抽数字 | 端口速率（`2× 2.5GbE` 抽 2.5）/ 安全认证（`CE/GS` 抽计数） |
| `percentile` | 字段是连续值，按盘点内百分位映射 1-5 | 价格 / 覆盖面积 / 容量 / 功率 |

例（router.yaml 节选）：

```yaml
viz:
  radar:
    enabled: true
    grouping_field: category
    min_per_group: 4
    dims:
      - key: generation
        label: WiFi 代际
        source: wifi_protocol
        type: enum
        scale: {"WiFi 5": 1, "WiFi 6": 2, "WiFi 6E": 3, "WiFi 7 dual": 4, "WiFi 7 tri": 5}
      - key: port_speed
        label: 端口速率
        source: wan_lan_ports
        type: numeric_extract
        pattern: '(\d+(?:\.\d+)?)\s*GbE'
        aggregate: max
        scale: {"1": 2, "2.5": 4, "5": 5, "10": 5}
      - key: value
        label: 性价比
        source: typical_price_eur
        type: percentile
        direction: descending     # 价格低高分
        bins: 5
      # ... 其余 3 维
```

---

## 工作流概览

详见 [`skills/spec-inventory/references/workflow.md`](skills/spec-inventory/references/workflow.md)。

| Step | 操作 | 输出 |
|---|---|---|
| 1 | 收集输入 + 解析 ticket / PPT | seed_products + anchor_explicit + anchor_open |
| 2 | 候选扩展到 Top N（用户确认） | 完整候选名单 |
| 3 | 加载品类 schema | YAML 字段定义 |
| 4 | 并行 N 个 research agent 拉规格 + 校验 | 每 agent 4 产品 × N 字段 |
| 5 | 合并到 CSV | spec_matrix_*.csv |
| 6 | 生成黄昏绮景 HTML | U12_*.html |

---

## 核心契约（不要违反）

1. **明示项 vs 待探讨项硬隔离**：详见 [`references/anchor-isolation.md`](skills/spec-inventory/references/anchor-isolation.md)
2. **子 agent 输出校验**：详见 [`references/agent-prompt.md`](skills/spec-inventory/references/agent-prompt.md)
3. **黄昏绮景配色 + ECharts v5 单文件**：复用现有 html-template.html 不另起炉灶
4. **按品类 schema 驱动**：字段定义在 YAML，主流程不硬编码

---

## 真实案例

本 plugin 的所有规则都来自 GL.iNet 数据组 MeshNode 项目（ticket #65）的真实迭代：

### Case 1 · MeshNode 锚点教训

第一版 HTML 雷达图给 MeshNode 锚点填了 6 维分数 `[4,3,4,3,5,4]`，被需求方质疑后回查 PPT slide 7 明示 ID 形态 / 性能档为"后续待探讨问题"。校正：移除雷达图新品锚点，散点图只保留价格档（PPT 唯一明示），章节顶部加 disclaimer。

→ 沉淀为 [`anchor-isolation.md`](skills/spec-inventory/references/anchor-isolation.md) 硬规则。

### Case 2 · B agent 翻车教训

并行 3 个 research agent，其中 B agent 自己声明"由于环境限制，我无法实际访问网络"但仍输出大量"推测"字段，导致 ASUS BT8 频段配置写错（实际 tri-band BE14000 vs agent 写的 dual-band BE7200）。主控用 WebSearch 二次校验才发现。

→ 沉淀为 [`agent-prompt.md`](skills/spec-inventory/references/agent-prompt.md) 输出校验 checklist + 主控二次抽检规则。

---

## 文件结构

```
.
├── README.md                                          # 你正在读的这个文件
├── LICENSE                                            # MIT
├── .claude-plugin/
│   └── marketplace.json                               # Plugin 元数据
└── skills/
    └── spec-inventory/
        ├── SKILL.md                                   # Skill 主入口
        ├── assets/
        │   └── csv-schema.md                          # 公共 CSV 字段定义
        │                                              # （HTML 模板委托 data-team-skills:html-report,
        │                                              #  不在本 plugin 维护 fork）
        ├── configs/
        │   ├── router.yaml                            # 路由器品类（最完整）
        │   ├── kitchen.yaml                           # 厨具品类（骨架）
        │   └── saas.yaml                              # SaaS 品类（骨架）
        └── references/
            ├── workflow.md                            # 6 步主流程详细操作
            ├── anchor-isolation.md                    # 明示项 vs 待探讨项 硬隔离规则
            └── agent-prompt.md                        # 子 agent prompt + 输出校验
```

---

## 已知限制

1. **第一次盘点时 candidate_hints 可能不全**：YAML 里只列了主流品牌，长尾品牌（如新兴 D2C）需要用户在 Step 2 手工补
2. **CSV 字段映射依赖 head 抽样**：Amazon scraper 字段名变化时需手工调整
3. **HTML 报告字数**：12 款 × 14 字段表格在 1160px 容器内需横向滚动；不适合 4K 高密屏一屏看全
4. **WebSearch 在中国大陆**：部分国际站点访问受限，建议主控环境有代理；本 plugin 不内置代理逻辑
5. **价格货币**：单次盘点只用一种货币（YAML `price_unit`）；跨市场对比需多次跑后手工合并

---

## 贡献

PR 欢迎，特别是：

- 新品类 YAML（automotive / outdoor / 服装 / 美妆 ...）
- HTML 模板的可访问性（a11y）改进
- agent prompt 在小语种市场（DE / FR / JP）的本地化模板
- CSV 字段映射对其他 scraper（Helium10 / SimilarWeb）的预设

不接受改变 `:root` 配色或换图表库的 PR——这是组内契约。

---

## 协议

MIT License（见 LICENSE 文件）

---

## 致谢

- 黄昏绮景配色 + ECharts 工厂函数：源于 GL.iNet 数据组 [`html-report` skill](https://github.com/) 的设计系统
- 流水线方法论：源于 MeshNode 项目（ticket #65）U12 工作单元的实际迭代
- 教训沉淀：感谢需求方对"MeshNode 锚点是否合法"的及时质疑
