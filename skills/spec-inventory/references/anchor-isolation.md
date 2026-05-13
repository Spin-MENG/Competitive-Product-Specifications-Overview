# Anchor Isolation · 明示项 vs 待探讨项 硬隔离规则

> 这是本 skill 最核心的差异化逻辑。源于 GL.iNet 数据组 MeshNode 项目（ticket #65）的一次教训：**新品锚点的非价格维度被分析师主观预设了，被需求方质疑后才发现 PPT 明确列为"待探讨"。**

---

## 问题模式

新品立项时，需求方给的 PPT / ticket 通常**只锁定少数关键参数**（如目标售价、市场定位、不做什么），而**多数硬件 / 形态 / 协议字段都是 slide 7 那种"后续希望探讨的问题"**。

分析师做竞品对标时，本能地想"把新品也画在图上"——于是给新品的 chipset / 频段 / 端口速率 / 形态都倒推填值。但这不是"分析"，而是**越权预设了需求方还没决策的内容**。

后果：
1. 报告把"未决"装扮成"已决"，需求方据此做的下游决定就基于错位前提
2. 一旦真实立项与盘点预设不一致，前期所有"建议 / 对标"全部失效
3. 数据组信誉打折

---

## 解决方案：两份清单 + 硬规则

### 清单 1 · anchor_explicit（明示项）

ticket / PPT 中**白纸黑字**写出来的产品参数。例：

```yaml
anchor_explicit:
  target_price_usd: 80               # PPT slide 3 明示
  positioning: "Flint 主路由的节点扩展"   # PPT slide 3 明示
  not_doing: ["独立路由能力"]           # ticket.md 第 8 行明示
```

**判定标准**：能引用到具体 slide / 段落 / 行号。

### 清单 2 · anchor_open（待探讨项）

ticket / PPT 中**明确列为"待定" / "待探讨" / "TBD" / "后续讨论"**的字段。例：

```yaml
anchor_open:
  - wifi_generation     # PPT slide 7 "后续希望探讨"
  - band_config         # PPT slide 7
  - port_speed          # PPT slide 7
  - mesh_protocol       # PPT slide 7
  - form_factor         # PPT slide 7 "ID 设计形态"明示待探讨
  - chipset             # 未明示但通常需待选型
  - antenna_count       # 未明示但需立项确认
```

**判定标准**：要么 ticket / PPT 显式说待定，要么属于硬件 / 工艺细节通常需立项后才确定的字段。**保守归类为 anchor_open**——宁可缺画也不要错画。

---

## 硬规则（生成 HTML 报告时强制）

### 规则 1 · 散点图

- ✅ `anchor_explicit` 包含 price → 新品散点的 y 轴（价格）可放点
- ❌ `anchor_open` 含 wifi_generation → 新品散点的 x 轴（代际）**不能取具体值**

操作：x 轴用占位值（如 `0.5` 表示"待定"），点 label 标 "（代际待定）" 或 "（PPT 未定）"。

### 规则 2 · 雷达图

- 雷达需要 6 个维度同时取值。如果 ≥ 2 个维度落在 `anchor_open` 黑名单 → **不画新品锚点**
- 在雷达图下方 insight 注明："本图不画 [新品名] 6 维占位，因 PPT 仅明示 [x 维]，其他 [y 维] 列为待探讨"

### 规则 3 · 推荐章节

`{{SUMMARY_RECOMMENDATIONS}}` 右列的"立项建议"可以包含分析师对 `anchor_open` 字段的推论，但**必须**：

1. 章节顶部加 orange disclaimer：
   ```
   ⚠️ 本节是 U12 竞品分析师的推论建议，非 PPT 明示。
   ```

2. 每条建议明确标注是基于哪个竞品基线推论（如"对标 AVM 6000 desktop 形态"）

3. **不在 HTML 任何图表 / KPI 卡 / 表格的"新品列"里固化推论值**——推论只活在"建议"文本块里。

### 规则 4 · TL;DR 顶部

TL;DR 必须显示 `anchor_explicit` 和 `anchor_open` 两份清单，让读者一眼看到"什么已定 / 什么未定"。

---

## 反例（这次踩到的坑）

```diff
# 雷达图的 6 维分数
- {name:'MeshNode (目标 ~$80, 立项假设)', value:[4,3,4,3,5,4], color:P[2]}
+ // MeshNode 6 维分数 PPT 未定，故不画占位锚点
```

```diff
- [4,225,'WiFi 7 dual (假设)','MeshNode 3 节点 (~$80×3)','Anchor']
+ // x 值为 0.5 = 横轴外的"代际未定"占位，仅锚 y 轴价格
+ [0.5,225,'PPT 未定','MeshNode 3 节点 (~$80×3≈225EUR; PPT 仅明示价格)','Anchor']
```

```diff
- <h2>对 MeshNode 立项的 4 个具体建议</h2>
+ <h2>对 MeshNode 立项的 4 个具体建议</h2>
+ <div class="insight" style="background:#FDEBD3;">
+   ⚠️ 本节是 U12 竞品分析师的推论建议，非 PPT 明示。
+   PPT 仅锁定 ~$80 / 接 Flint 主路由 / 不做独立路由 3 项；
+   形态 / 协议 / chipset 全部待立项。
+ </div>
```

---

## 实施 checklist

执行 Step 6 生成 HTML 前自检：

- [ ] `anchor_explicit` 字典已落地（≥1 字段）
- [ ] `anchor_open` 列表已落地（可以为空，但要主动 declare 已检查 PPT 全文）
- [ ] 散点图新品锚点的 x/y 仅用 `anchor_explicit` 字段
- [ ] 雷达图若 ≥2 维落在 `anchor_open` → 不画新品
- [ ] 推荐章节顶部有 disclaimer banner
- [ ] TL;DR 显式列出两份清单
- [ ] 不在表格 / KPI 卡 "新品列" 固化任何推论

任一项未通过 → 不要交付 HTML，先回到 Step 1 重新核对 anchor。
