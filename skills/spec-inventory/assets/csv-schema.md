# CSV Schema · spec_matrix 字段定义

> Step 5 输出的 CSV 字段定义。所有品类共通的元字段 + 按 `configs/<category>.yaml` 注入的品类字段。

## 公共字段（所有品类）

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `brand` | str | ✓ | 品牌名（按厂商官网命名，不要简写） |
| `model` | str | ✓ | 完整型号（如 "Deco BE25"，不只是 "BE25"） |
| `category` | str | ✓ | 大类（如 `router/mesh_bundle` / `router/repeater`） |
| `subcategory` | str | – | 子分类（厨具 / SaaS 用，路由可空） |
| `typical_price_eur` 或 `typical_price_usd` | str | ✓ | 当前市场价区间，按 YAML 的 `price_unit` |
| `source_urls` | str | ✓ | 主要资料 URL（分号分隔） |
| `field_gaps` | str | – | 未公开字段说明 |
| `data_date` | YYYY-MM-DD | ✓ | 数据采集日期 |
| `market` | str | ✓ | 市场范围（如 EU 5 站点 / US / 中国） |

## 品类字段

由 `configs/<category>.yaml` 的 `required_fields` + `optional_fields` 注入。
顺序遵循 YAML 中的列表顺序，便于 CSV 表头可读性。

## 字段填值规范

### "未公开" 处理

凡是公开资料查不到的字段，统一填字符串 **`未公开`**（不要 `N/A` / `unknown` / 空字符串混用）。

### "推测" 标注

字段值是基于评测站拆机 / 同代际产品类比 / FCC 内部照片推断的，必须在值后加 **`(推测)`** 后缀。例：

```
chipset_soc = "Qualcomm IPQ5312 @1.1GHz (推测)"
```

### 价格区间

按 YAML `price_basis` 标定。同一行内**只用一种货币**——若需多市场对比，跨市场分多行（同 model 不同 market）。

### 来源 URL

格式：`<vendor-spec-url>; <reviewsite-url>; <fcc-url>`。最少 1 个，目标 2-3 个。
**不接受全是 amazon.com 商详 URL**（这等同于没查）。

## CSV 命名约定

- 文件名：`spec_matrix_<category>_top<N>_<YYYYMMDD>.csv`
- 编码：UTF-8 with BOM（兼容 Windows Excel）
- 分隔符：英文逗号 `,`
- 引号：字段含逗号 / 换行 / 引号时用双引号包裹

## 示例

```csv
brand,model,category,wifi_protocol,phy_rate,band_config,mlo_support,mesh_protocol,wan_lan_ports,chipset_soc,ram_mb,flash_mb,coverage_sqm_3pack,form_factor,typical_price_eur,source_urls,field_gaps,data_date,market
TP-Link,Deco BE25,mesh_bundle,WiFi 7,BE3600 (5G 2882 + 2.4G 688 Mbps),dual-band (2.4 + 5 GHz; no 6 GHz),Yes,AI-Driven Mesh (proprietary),2× 2.5GbE WAN/LAN auto,Qualcomm IPQ5312 @1.1GHz + QCN8402,512,128,~604,desktop puck,210-230,https://tp-link.com/deco-be25;https://mbreviews.com/tp-link-deco-be25-be5000-wifi-7-mesh-system-test-and-review/,Power W not disclosed,2026-05-13,EU 5 站点
```
