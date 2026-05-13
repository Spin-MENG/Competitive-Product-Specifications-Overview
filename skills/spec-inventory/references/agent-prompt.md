# 子 Agent Prompt 模板与输出校验

> Step 4 派发并行 research agent 用的 prompt 模板 + 校验规则。源于 MeshNode 项目 B agent 翻车教训：agent 自己声明"无法实际联网"但仍输出大量"推测"字段，需主控二次 WebSearch 校验。

---

## Prompt 模板

```text
你是 [产品类别] 竞品规格研究员。请用 **WebSearch + WebFetch 实地查询**，
研究以下 [N] 款产品的硬件规格 / 协议代际 / 价格 / 形态，
**用厂商官网 + 评测站 + FCC database 交叉验证**，不要只用 Amazon 商详。

## 要研究的 [N] 款产品

1. [品牌] [型号] ([简述])
2. ...

## 每个产品必填字段（来自 configs/<category>.yaml）

[把 schema 中的字段一字不落列出来]

## 输出格式（必须遵守）

直接返回 **markdown 表格**，列名 = 上述字段（横向铺开），每行 1 个产品。
表格下方追加 "## 资料来源"，列出主要 URL（每产品 ≥ 1 个权威 URL）。

## 严格规则（违反则结果被退回）

1. **必须用 WebSearch / WebFetch 实地查询**：禁止只凭训练数据答；
   每次回答前调用至少 N×2 次 WebSearch（每产品 ≥ 2 次）。
2. **不许瞎编**：查不到的字段写 "未公开"。可以基于 FCC 拆机
   照片 / 评测站测试推断，但**必须标 "(推测)"** 后缀。
3. **不接受 "无法联网" 借口**：你的工具集明确有 WebSearch / WebFetch；
   如果你声明无法联网，说明你没有调用工具，重新做。
4. **来源 URL 必须真实**：每个 URL 必须是你**实际 fetch 过**的页面，
   不要凭记忆构造 URL。
5. **预算**：响应 ≤ 1500 字（表格 + 来源足够，不要长篇分析）。

## 同类规格对照参考（如有）

- 同价位 / 同代际产品的典型规格（让 agent 知道哪些值是 plausible
  范围、哪些是异常）
- 例如："同价位 WiFi 7 dual-band mesh 套装的 SoC 一般是 Qualcomm
  IPQ5312 / MediaTek MT7976"
```

---

## 输出校验 checklist

主控（dispatcher）每个 agent 返回后立刻跑：

### Hard checks（任一失败 → 退回重做）

- [ ] **输出是 markdown 表格**（不是散文 / 链表 / JSON 等其他格式）
- [ ] **表格行数 = 派发产品数**（少一行就是漏了 / 合并错了）
- [ ] **存在 "## 资料来源" 或 "## Sources" 章节**
- [ ] **来源 URL ≥ 派发产品数**（每产品至少 1 个）
- [ ] **agent 没在文本里说 "我无法联网" / "无法访问网络" / "based on training data"**

### Soft checks（失败 → 主控自补 / 降级）

- [ ] **每行非"未公开"字段 ≥ 50%**（如果一行全是"未公开"，说明这个产品 agent 没查到任何东西，主控用 WebSearch 补 1-2 关键字段）
- [ ] **URL 不全是 amazon.com 商详**（必须有 ≥ 1 个厂商页 / 评测站 / FCC）
- [ ] **"(推测)" 后缀的字段 ≤ 必填字段的 30%**（推测太多 = 实际没查到）

### 主控二次抽检（强制）

无论 agent 报告多漂亮，**抽 2-3 个关键字段亲自 WebSearch 校验**：

```
关键字段 = ['chipset_soc', 'ram_mb', 'price', 'wifi_protocol']

for product in agent_output:
    抽 1 个产品 × 2 个关键字段
    WebSearch "<品牌> <型号> <字段名>"
    对比 agent 给的值 vs WebSearch 结果
    
hit_ratio = 抽中正确 / 抽中总数
if hit_ratio < 70%:
    整批不可信，重做或更换 agent
```

---

## 退回重做的 SendMessage 范式

```
SendMessage(to: <agent_id>, message: """
你之前的输出不合格，原因：

[ ] 你声明 "无法联网" 但你有 WebSearch / WebFetch 工具。
[ ] 你只查了 [X] 个 URL，目标是每产品 ≥ 1 个。
[ ] 字段 [Y] 你填 "推测 IPQ5312"，但没标 "(推测)" 后缀。

请：
1. 实际调用 WebSearch / WebFetch 每产品 ≥ 2 次
2. 表格下方列出每个 URL（要是你 fetch 过的真实页面）
3. 推测字段加 "(推测)" 后缀
4. 重新返回 markdown 表格
""")
```

---

## 主控自救：如果 agent 反复翻车

如果同一批产品 2 个 agent 都翻车，主控直接：

1. **拆得更碎**：1 agent 1 产品
2. **明确给 URL 起点**：在 prompt 里直接给厂商 spec 页 URL 让 agent fetch
3. **降级到主控自己做**：用主上下文 WebSearch + WebFetch 串行查 5-6 个关键字段

不要陷入"agent 翻车 → 再派 agent → 又翻车"的循环。3 次失败必须切换策略。

---

## 实际 MeshNode 项目教训

| 这次发生的 | 对应规则 |
|---|---|
| B agent 自己声明"由于环境限制，我无法实际访问网络" | Hard check #5 |
| B agent 把 ASUS BT8 写成 dual-band BE7200（实际 tri-band BE14000） | 主控二次抽检（用 WebSearch 抓到了） |
| C agent 标注 chipset 都带 "(推测)" 后缀，质量明显高于 B | 这是好的范式，soft check #3 通过 |
| A agent 给的 Deco BE25 chipset = IPQ5312 有 mbreviews 拆机源 URL | Hard check #4 通过 |

→ 沉淀成校验流水线后，类似 B 翻车的 case 应在主控环节被自动捕获，而不是靠主控读完所有 agent 输出后手工质疑。
