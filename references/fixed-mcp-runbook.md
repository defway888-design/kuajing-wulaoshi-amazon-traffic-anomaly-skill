# 固定 MCP 数据运行手册

本文件是执行标准，不是参考建议。运行本 Skill 时必须逐项执行本文件列出的 MCP、工具/数据库、参数和字段。任何一步未完成，都不能输出流量异动结果。

## 1. 不可替代规则

- 父商品 ASIN、市场、子体集合、库存和广告花费：只用领星 MCP。
- Deals、变体数量、Listing 状态：只用卖家精灵 MCP。
- 自然搜索关键词排名、自然搜索关键词数量、主要关键词搜索趋势：只用 SIF MCP。
- 站外推广和联盟客推广：只用互联网搜索证据。
- 不允许因为其他 MCP 也返回相似字段，就替代指定 MCP。
- 不允许用用户手动上传的 Keepa 文件替代卖家精灵 MCP。
- 不允许在部分验证失败时输出可能因素。必须全量验证完成后再输出。

## 2. 运行状态

每个验证项必须生成内部状态：

```json
{
  "item": "验证项名称",
  "status": "verified | blocked",
  "required_source": "指定 MCP 或互联网搜索",
  "required_tool": "指定工具/数据库",
  "required_fields": [],
  "evidence": {},
  "blocked_reason": ""
}
```

全部 `status=verified` 后，才允许输出最终结果。最终结果仍只展示命中的可能因素，不展示未命中因素。

## 3. 固定时间参数

完整月份：

- 当前期：分析月 1 日至分析月最后一天。
- 上期：上一个自然月 1 日至上一个自然月最后一天。

当前分析月未结束：

- 当前期：分析月 1 日至当前已发生日。
- 上期：上月 1 日起，同等天数窗口。

SIF 自然搜索：

- 完整月：`time_type=month` / `timePieceType=month`，`time_value` / `timePieceValue` 填月份首日，例如 `2026-06-01`。
- 当前月未结束：优先 `latelyDay=30` 或 `lately=30`；数据不足时用 `latelyDay=7` 或 `lately=7`；不允许改用其他数据源。

## 4. 领星 MCP 固定步骤

领星 MCP 必须使用 HTTPS：

```text
https://openmcp.lingxing.com/mcp-servers/lingxing-mcp
```

如果配置为 `http://openmcp.lingxing.com/...` 并返回 `405 Method Not Allowed`，先切换到 HTTPS 后再运行。

### 4.1 获取店铺和市场

工具/数据库：

```text
LingXing-MCP.get_my_sids
```

必取字段：

```text
sid
country
name / 店铺名
```

用途：

- 确认用户指定市场对应的 `sid`。
- 如果同一 ASIN 存在多个市场，必须让用户选择市场或明确选择全部市场。
- 本地实测状态：可正常返回 `sid`、店铺名和国家。

阻塞条件：

- 无法调用 `get_my_sids`。
- 返回结果没有 `sid`。
- 国家/站点信息无法与用户指定市场匹配。

### 4.2 获取广告授权店铺

工具/数据库：

```text
LingXing-MCP.ad_auth_shops
```

固定入参：

```json
{}
```

必取字段：

```text
data[].sid
data[].profile_id
data[].country
data[].alias
data[].store_id
```

固定用法：

- 用 4.1 得到的 `sid` 匹配广告授权店铺的 `data[].sid`。
- 广告商品报表必须使用匹配到的 `data[].profile_id`。
- 本地实测状态：可正常返回 `sid`、`profile_id`、`country`、`alias`。

阻塞条件：

- 无法调用 `ad_auth_shops`。
- 目标 `sid` 没有对应 `profile_id`。

### 4.3 父商品 ASIN、当前子体集合、当前 Listing 状态

工具/数据库：

```text
LingXing-MCP.erp_listing
```

固定入参：

```json
{
  "offset": 0,
  "length": 200,
  "pvi_ids": "",
  "exact_search": "1"
}
```

分页：

```text
offset 从 0 开始递增，直到 offset >= total 或返回 list 为空。
```

必取字段：

```text
parent_asin
asin1
seller_sku
local_sku
status
status_text
afn_fulfillable_quantity
fulfillment_channel_type
sid
marketplace
item_name
```

固定用法：

- 父商品 ASIN：只认 `parent_asin`。
- 当前子体 ASIN：使用 `asin1`。
- 当前父商品子体集合：筛选 `sid=<目标 sid>` 且 `parent_asin=<父 ASIN>` 的全部行。
- 当前 Listing 状态：使用 `status`、`status_text`。
- `afn_fulfillable_quantity` 只作为 Listing 侧辅助字段；当前 FBA 库存验证必须以 4.4 的 `get_fba_stock_list` 为主。
- 部分子体断货：先用本步骤确定当前在售 FBA 子体，再用 4.4 的 FBA 可售库存字段判断。
- 本地实测状态：可正常返回 `parent_asin`、`asin1`、`seller_sku`、`status_text`、`status`、`afn_fulfillable_quantity`、`sid`。注意：`sids/search_value` 入参在当前实测中没有稳定过滤效果，因此必须分页后按返回字段二次筛选。

阻塞条件：

- 无法通过 `parent_asin` 确认父商品。
- 用户输入子 ASIN 时，领星无法把它归属到父商品。
- 缺少 `asin1`、`parent_asin`、`sid`、`status`、`status_text` 或 `afn_fulfillable_quantity`。
- 不能完成全量分页。

### 4.4 当前 FBA 仓库库存

工具/数据库：

```text
LingXing-MCP.get_fba_stock_list
```

固定入参：

```json
{
  "sid": "<4.1 得到的 sid>",
  "fulfillment_channel_type": "FBA",
  "is_hide_zero_stock": 0,
  "offset": 0,
  "length": 200,
  "sort_field": "available_total",
  "sort_type": "desc",
  "is_cost_page": 0
}
```

分页：

```text
offset 从 0 开始递增，直到 offset >= total 或返回 list 为空。
```

必取字段：

```text
sid
parent_asin_real
asin
seller_sku
fnsku
afn_fulfillable_quantity
available_total
total_onhand_quantity
total_fulfillable_quantity
fulfillment_channel_type
```

固定用法：

- 当前 FBA 可售库存使用 `afn_fulfillable_quantity`。
- 当前 FBA 可用库存可参考 `available_total`，但断货判断以 `afn_fulfillable_quantity` 为准。
- 父商品当前 FBA 库存 = 父商品当前在售 FBA 子体对应 `afn_fulfillable_quantity` 汇总。
- 部分子体断货 = 4.3 识别出的当前在售 FBA 子体中，任一子体在本工具中 `afn_fulfillable_quantity=0`。
- 本地实测状态：可正常返回 `parent_asin_real`、`asin`、`seller_sku`、`sid`、`afn_fulfillable_quantity`、`available_total`、`total_onhand_quantity`、`total_fulfillable_quantity`。注意：搜索入参在当前实测中没有稳定过滤效果，因此必须分页后按返回字段二次筛选。
- `query_fba_valid_list` 不作为本 Skill 的固定库存验证工具，不能用于通过本项 verified。

阻塞条件：

- 无法调用 `get_fba_stock_list`。
- 缺少 `parent_asin_real` 或无法用 4.3 的父子体集合完成二次筛选。
- 缺少 `afn_fulfillable_quantity`。
- 不能完成全量分页。

### 4.5 历史流量与历史库存口径

工具/数据库：

```text
LingXing-MCP.query_product_performance_asin_lists
```

固定入参：

```json
{
  "sids": "<4.1 得到的 sid>",
  "start_date": "<时间窗口开始 yyyy-MM-dd>",
  "end_date": "<时间窗口结束 yyyy-MM-dd>",
  "date_range_type": 0,
  "date_type": "purchase",
  "currency_code": "CNY",
  "summary_field": "asin",
  "turn_on_summary": 1,
  "sort_field": "volume",
  "sort_type": "desc",
  "offset": 0,
  "length": 500
}
```

分页：

```text
offset 从 0 开始递增，直到 offset >= total 或返回列表为空。
```

必取字段：

```text
asin
parent_asin
sessions_total
page_views_total
volume
amount
afn_fulfillable_quantity
```

固定用法：

- 上游周数据分析 Skill 已提供流量方向时，流量方向可直接使用上游结果。
- 上游未提供流量方向时，必须用本工具当前期 vs 上期的 `sessions_total` 判断。
- 历史父商品整体到货/断货必须用本工具当前期 vs 上期的 `afn_fulfillable_quantity`。
- 当前本机实测状态：该工具 `tools/list` 可见，但多组参数在 2026-07-08 实测均返回 `data.msg=查询异常`，没有拿到可验证的历史字段。

阻塞条件：

- 工具返回 `查询异常`。
- 缺少 `sessions_total` 且上游周数据分析 Skill 未提供流量方向。
- 缺少 `afn_fulfillable_quantity` 时，不能验证历史到货/断货。
- 当前期或上期任一时间窗口无法取得。

### 4.6 广告花费

工具/数据库：

```text
LingXing-MCP.ad_campaign_product_report
```

固定入参：

```json
{
  "profile_id": "<4.2 得到的 profile_id>",
  "report_date": "<时间窗口开始 yyyy-MM-dd> - <时间窗口结束 yyyy-MM-dd>",
  "page": 1,
  "length": 500,
  "sort_field": "spends",
  "sort_type": "desc",
  "with_ring": false
}
```

分页：

```text
page 从 1 开始递增，直到 page 覆盖 total 或返回列表为空。
```

必取字段：

```text
asin
sku
spends
```

固定用法：

- 只统计父商品当前子体集合和上期子体集合并集中的 `asin`。
- 父商品广告花费 = 子体 `spends` 汇总。
- 当前期 `spends` > 上期 `spends`，命中广告花费增加。
- 当前期 `spends` < 上期 `spends`，命中广告花费减少。
- 本地 schema 实测：入参必填 `report_date`、`profile_id`，花费字段名使用 `spends`。
- 当前本机运行实测：2026-07-08 调用该工具多次返回 `服务器繁忙`，未拿到可验证结果行。因此执行时必须以实际成功返回 `asin/spends` 为 verified 条件。

阻塞条件：

- 无法调用 `ad_campaign_product_report`。
- 返回 `服务器繁忙`。
- 缺少 `asin` 或 `spends`。
- 广告报表无法按当前期和上期分别取得。
- 不允许用 SIF 广告曝光、卖家精灵销量或任何其他工具替代领星广告花费。

## 5. 卖家精灵 MCP 固定步骤

卖家精灵 MCP 使用 `mcp__sellersprite_mcp`。如果当前环境只暴露 `mcp__sellersprite_mcp_2`，必须使用同名工具和同样参数；不得改用其他 MCP。

执行本章前，如本机已安装 `kuajing-wulaoshi-sellersprite-mcp-database`，先读取其 `references/sellersprite-mcp-database.md` 确认卖家精灵 MCP 的通用工具边界。通用数据库说明不能覆盖本章为流量异动写死的工具、参数、字段和阻塞条件。

### 5.1 变体数量

工具/数据库：

```text
mcp__sellersprite_mcp.competitor_lookup
```

当前期固定入参：

```json
{
  "request": {
    "marketplace": "<市场>",
    "month": "<yyyyMM>",
    "asins": ["<父 ASIN>"],
    "variation": "N",
    "returnFields": "asin,parent,variations,variationCount,variationList,title,brand",
    "page": 1,
    "size": 40
  }
}
```

上期同样调用一次，`month` 改为上期 `yyyyMM`。

必取字段：

```text
asin
parent
variations
variationCount
variationList
```

固定用法：

- 优先使用 `variations`。
- 如果 `variations` 缺失，再用 `variationCount`。
- 如果 `variations` 和 `variationCount` 都缺失，再用 `variationList` 的数量；不得用 Keepa、SIF 或领星替代卖家精灵变体数量。
- 当前期数量 > 上期，命中变体增加。
- 当前期数量 < 上期，命中变体拆分/减少。
- 本机实测状态：`competitor_lookup` 对 ASIN `B0DWXBCQVP` 可正常返回 `parent`、`asin`、`title`、`brand`、`variations`；其中 `variationCount=null`、`variationList=null`，所以固定优先字段必须是 `variations`。

阻塞条件：

- `competitor_lookup` 不可用。
- `variations`、`variationCount` 和 `variationList` 都缺失。
- 当前期或上期任一月份无法取得。

### 5.2 Deals 促销

工具/数据库：

```text
mcp__sellersprite_mcp.keepa_info
```

对父商品子体并集逐个调用。固定入参：

```json
{
  "asin": "<子 ASIN>",
  "marketplace": "<市场>",
  "startTimestamp": "<时间窗口开始毫秒时间戳>",
  "endTimestamp": "<时间窗口结束毫秒时间戳>",
  "dailyLatest": false,
  "returnFields": "asin,parentAsin,variationAsins,dealPrice"
}
```

当前期和上期分别调用。

必取字段：

```text
asin
parentAsin
variationAsins
dealPrice
dealPrice[].timePoint
dealPrice[].value
```

固定用法：

- 任一子体任一时间点 `dealPrice[].value > 0`，该父商品在该期间存在 Deals。
- `dealPrice[].value = -1` 或缺失有效正数，不判断为 Deals。
- 上期无 Deals、当前期有 Deals，命中 Deals 促销开始。
- 上期有 Deals、当前期无 Deals，命中 Deals 促销结束。
- Coupon 不在本 Skill 判断。

阻塞条件：

- `keepa_info` 不可用。
- `dealPrice` 字段缺失。
- 不能覆盖父商品子体并集。

### 5.3 Listing 状态异常 / 被抑制

工具/数据库：

```text
mcp__sellersprite_mcp.keepa_info
```

对当前父商品子体逐个调用。固定入参：

```json
{
  "asin": "<子 ASIN>",
  "marketplace": "<市场>",
  "dailyLatest": true,
  "returnFields": "asin,parentAsin,productStatus"
}
```

必取字段：

```text
asin
parentAsin
productStatus
```

固定用法：

- `productStatus=STANDARD` 判断为正常。
- 任一子体 `productStatus` 不是 `STANDARD`，命中父商品 Listing 状态异常。
- 断货导致的不可售不在本项重复归因。

阻塞条件：

- `keepa_info` 不可用。
- `productStatus` 字段缺失。
- 当前父商品子体无法全部查询。

## 6. SIF MCP 固定步骤

SIF MCP 使用 `mcp__sif_mcp`。运行前先调用：

```text
mcp__sif_mcp.ping
```

`ping` 失败则 SIF 三项全部 blocked，禁止输出最终结果。

### 6.1 自然搜索关键词数量

工具/数据库：

```text
mcp__sif_mcp.ops_get_listing_keyword_distribution
```

固定入参：

```json
{
  "asin": "<领星确认的父商品下任一当前在售子 ASIN；如 SIF 接受父 ASIN，则用父 ASIN>",
  "country": "<市场>",
  "timePieceType": "month",
  "timePieceValue": "<月份首日 yyyy-MM-01>",
  "dimension": "asin",
  "showType": 1,
  "pageNum": 1,
  "pageSize": 100
}
```

当前期和上期分别调用。当前月未结束时改用：

```json
{
  "timePieceType": "latelyDay",
  "timePieceValue": "30"
}
```

必要时再改为 `timePieceValue=7`，但仍必须使用同一个 SIF 工具。

分页：

```text
pageNum 从 1 开始递增，直到 pageNum 覆盖 total 或返回 list 为空。
```

必取字段：

```text
total
list[]
list[].asin
list[].total
list[].natural
list[].ad
list[].sp
list[].rec
list[].brand
list[].vedio
```

固定用法：

- 父商品自然关键词数量 = 父商品当前子体集合和上期子体集合并集中，返回 `list[].asin` 命中的行的 `natural` 汇总。
- 当前期汇总 > 上期汇总，命中自然搜索关键词数量增加。
- 当前期汇总 < 上期汇总，命中自然搜索关键词数量减少。

阻塞条件：

- `ops_get_listing_keyword_distribution` 不可用。
- `list[].natural` 缺失。
- 当前期或上期无法取得同口径数据。
- 不能完成全量分页。

### 6.2 自然搜索关键词排名

工具/数据库：

```text
mcp__sif_mcp.market_get_asin_keyword_signals
```

固定入参：

```json
{
  "asin": "<领星确认的父商品下任一当前在售子 ASIN；如 SIF 接受父 ASIN，则用父 ASIN>",
  "country": "<市场>",
  "time_type": "month",
  "time_value": "<月份首日 yyyy-MM-01>",
  "listingSearch": true,
  "topN": 300
}
```

当前期和上期分别调用。当前月未结束时改用：

```json
{
  "time_type": "lately",
  "time_value": "30"
}
```

必要时再改为 `time_value=7`，但仍必须使用同一个 SIF 工具。

必取字段：

```text
primary_signals
primary_signals.declining[]
primary_signals.gaining[]
primary_signals.rank_gaps[]
top_keywords[]
top_keywords[].keyword
top_keywords[].rank_evolution
top_keywords[].contri_change
top_keywords[].click_share
top_keywords[].sp_rank
top_keywords[].sb_rank
top_keywords[].sbv_rank
```

固定用法：

- 只用 `rank_evolution` 判断自然排名方向，判断对象限定为 `top_keywords[].keyword` 对应的自然排名演变。
- `improving` 数量和贡献高于 `declining/volatile/gap_detected` 时，支持自然搜索关键词排名上升。
- `declining/volatile/gap_detected/no_organic` 数量和贡献高于 `improving` 时，支持自然搜索关键词排名下降。
- 不能使用广告排名字段 `sp_rank/sb_rank/sbv_rank` 判断自然排名方向。
- 不调用 `market_get_keyword_competition` 判断本项；竞争格局不属于本 Skill 的流量因素。
- 不输出 SIF 工具中的行动建议、策略建议或竞争建议。

阻塞条件：

- `market_get_asin_keyword_signals` 不可用。
- `top_keywords[].rank_evolution` 缺失。
- 当前期或上期无法取得同口径数据。

### 6.3 主要关键词搜索趋势

工具/数据库 A：

```text
mcp__sif_mcp.market_get_asin_keyword_signals
```

先用 6.2 的 `top_keywords[]` 选出主要关键词：

```text
top_keywords[].keyword
top_keywords[].click_share
top_keywords[].contri_change
```

固定选词：

- 按 `click_share` 从高到低取前 10 个。
- 如果少于 10 个，取全部。

工具/数据库 B：

```text
mcp__sif_mcp.market_get_keyword_history
```

固定入参：

```json
{
  "keywords": ["<主要关键词 1>", "<主要关键词 2>"],
  "country": "<市场>",
  "granularity": "month"
}
```

必取字段：

```text
keywords[]
keywords[].keyword
keywords[].data_points
keywords[].dates[]
keywords[].volumes[]
keywords[].ranks[]
keywords[].top3_click_shares[]
keywords[].top3_conversion_shares[]
keywords[].latest.date
keywords[].latest.volume
keywords[].latest.rank
```

固定用法：

- 对每个主要关键词，取当前期月份和上期月份对应的 `volumes[]`。
- 多数主要关键词当前期搜索量 > 上期，命中主要关键词搜索趋势上升。
- 多数主要关键词当前期搜索量 < 上期，命中主要关键词搜索趋势下降。
- 只用 `market_get_keyword_history` 的原始 ABA 搜索量 `volumes[]` 判断搜索趋势。
- 不使用 `market_get_keyword_demand` 的 `action_hint`、`interpretation`、`diagnosis` 输出结论或建议。

阻塞条件：

- `market_get_keyword_history` 不可用。
- 主要关键词列表为空。
- `volumes[]` 或 `dates[]` 缺失。
- 当前期或上期月份无法对应到数据点。

## 7. 互联网搜索固定步骤

互联网搜索用于站外推广和联盟客推广，不属于 MCP 替代。

### 7.1 搜索词

必须至少搜索以下组合：

```text
"<父 ASIN>"
"<任一子 ASIN>"
"<品牌> <产品词> deal"
"<品牌> <产品词> promo code"
"<品牌> <产品词> discount"
"<品牌> <产品词> Facebook"
"<品牌> <产品词> TikTok"
"<品牌> <产品词> YouTube"
"<品牌> <产品词> Instagram"
"<品牌> <产品词> Amazon Associates"
```

### 7.2 必取网页证据字段

每条候选网页必须记录：

```text
url
title
snippet
page_publish_date
page_content_match
matched_token
matched_type
promotion_type
accessible
```

字段规则：

- `matched_token` 必须命中 ASIN、品牌或产品词。
- `page_publish_date` 或页面可识别时间必须落在分析区间内。
- `promotion_type` 只能是 `operator_offsite`、`affiliate_self_initiated`、`unknown`。
- 至少 1 条 `accessible=true` 且时间、内容、类型都满足，才算对应验证项 verified。

### 7.3 分类

站外推广：

- `promotion_type=operator_offsite`
- 证据显示品牌官方、卖家主动活动、主动投放、官方折扣码、官方社群推广等。

联盟客推广：

- `promotion_type=affiliate_self_initiated`
- 证据显示 Amazon Associates、网红、测评站、导购站、第三方 Deal 分享等。

阻塞条件：

- 无法搜索互联网。
- 没有打开候选网页验证内容。
- 只有搜索结果摘要，没有可访问页面。
- 无法判断时间是否落在分析区间内。
- 无法区分运营主动推广或第三方自发推广。

## 8. 最终输出前检查

输出最终结果前，必须确认：

```text
parent_market_verified=true
traffic_direction_verified=true
current_parent_child_inventory_verified=true
historical_arrival_stockout_verified=true
ad_spend_verified=true
deals_verified=true
offsite_verified=true
variation_verified=true
listing_status_verified=true
natural_rank_verified=true
natural_keyword_count_verified=true
main_keyword_trend_verified=true
affiliate_verified=true
```

任何一个不是 `true`，都必须输出阻塞模板，不得输出“可能的因素”。
