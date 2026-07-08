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

用户未指定时间：

- 分析月份不是必填项，不得因为用户未输入分析月份而阻塞。
- 候选当前期：当前月 1 日至当前日期。
- 候选上期：上月 1 日起，同等天数窗口。
- 例：7 月 8 日运行时，候选窗口为 7 月 1 日至 7 月 8 日 vs 6 月 1 日至 6 月 8 日。
- 执行每个数据项时，必须检查该 MCP 或网页证据实际能取到的最新日期。若最新日期早于当前日期，以实际最新日期截断当前期，并用上月同等日数重新生成上期窗口。
- 例：7 月 8 日运行但领星广告数据只返回到 7 月 6 日，则广告花费窗口为 7 月 1 日至 7 月 6 日 vs 6 月 1 日至 6 月 6 日。
- 不同数据源存在数据延迟时，不强行使用同一个结束日期；最终证据必须写明该因素实际使用的时间窗口。

用户只指定一个时间段：

- 当前期：用户指定的月份、连续月份段或日期段。
- 对比期：当前期之前的等长时间段。
- 例：用户指定 2026 年 6 月，当前期为 2026-06-01 至 2026-06-30，对比期为 2026-05-01 至 2026-05-31。
- 例：用户指定 2026 年 4-6 月，当前期为 2026-04-01 至 2026-06-30，对比期为 2026-01-01 至 2026-03-31。
- 如果用户指定的时间段尚未结束，当前期结束日期按当前日期或实际可调取最新日期截断，对比期按同等天数生成。

用户明确指定两个时间段：

- 严格按用户指定的两个时间段运行，不强制改成上月、同期或等长窗口。
- 例：2026 年 1-3 月 vs 2026 年 4-6 月。
- 例：2026 年 2 月 vs 2026 年 6 月。
- 如果用户没有说明哪段是分析期，默认较晚时间段为当前期/分析期，较早时间段为对比期。
- 如果用户明确说“用 A 对比 B”“分析 A 相比 B”，按用户语义把 A 作为当前期/分析期，把 B 作为对比期。
- 两个时间段天数不一致时，所有可累加指标必须同时计算总量和日均值。除非用户明确要求比较总量，否则方向判断优先使用日均值。
- 时间段已完整结束但 MCP 只返回部分数据时，不得静默忽略缺失日期；最终证据必须说明实际可用日期范围。若可用范围不足以覆盖任一指定时间段，标记该验证项 blocked。

时间型工具与月份型工具：

- 支持日期范围的工具：使用当前期和上期的实际日期窗口。
- 只支持月份参数的工具：单月对比时按对应月份调用；连续月份段对比时逐月调用并聚合。
- 可累加指标，如流量、广告花费、搜索量：同等长度时间段优先比较期间总量；不同长度时间段优先比较日均值或月均值，同时输出总量。
- 状态型指标，如变体数量、Listing 状态：优先比较每个时间段最后一个可验证日期或月份的状态。
- 覆盖型指标，如自然关键词数量：单月用该月数量；多月用期间月均自然关键词数量，并在证据中保留期末数量。
- 排名型指标，如自然搜索关键词排名：单月用该月排名；多月用期间平均自然排名，并列举影响最大的关键词。

SIF 自然搜索：

- 完整月：`time_type=month` / `timePieceType=month`，`time_value` / `timePieceValue` 填月份首日，例如 `2026-06-01`。
- 连续月份段：逐月调用 `time_type=month` / `timePieceType=month`，再按本节的可累加、覆盖型、排名型规则聚合。
- 当前月未结束且 SIF 不支持自定义日期：优先使用该 SIF 工具支持的 `week` 最新完整周，并与向前 28 天的上月同口径周对比；工具不支持 `week` 或周数据无法覆盖当前月已发生区间时，再使用 `latelyDay=7/30` 或 `lately=7/30`。当前月已发生区间不超过 10 天时优先 7 天口径，超过 10 天时优先 30 天口径。最终证据必须说明实际使用的 SIF 时间口径，不允许改用其他数据源。

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
  "search_field": "parent_asin",
  "search_value": ["<4.3 领星确认的父 ASIN>"],
  "summary_field": "parent_asin",
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
total_sum
```

固定用法：

- 上游周数据分析 Skill 已提供流量方向时，流量方向可直接使用上游结果。
- 上游未提供流量方向时，必须用本工具当前期 vs 上期的 `sessions_total` 判断。
- 历史父商品整体到货/断货必须用本工具当前期 vs 上期的 `afn_fulfillable_quantity`。
- 父商品口径必须使用 `search_field=parent_asin` 与 `search_value=[父 ASIN]` 限定，不允许查询整店后用未过滤汇总值代替父商品。
- 如果返回结果包含父商品汇总行，优先读取该汇总行；如果返回结构只提供 `total_sum`，必须确认请求已经用父 ASIN 限定后，才能读取 `total_sum` 中的 `sessions_total`、`page_views_total`、`volume`、`amount`、`afn_fulfillable_quantity`。
- 当前本机实测状态：2026-07-08 对 `B0FNVXJCN8` / 美国站可正常返回父商品限定后的 `total_sum`，可读取 `sessions_total`、`page_views_total`、`volume`、`amount`、`afn_fulfillable_quantity`。

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
  "search_text": "<父商品子体并集中的单个子 ASIN>",
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
- 必须对父商品子体并集逐个调用本工具，`search_text` 填单个子 ASIN；每次调用后只保留返回行中 `asin` 与该子 ASIN 完全一致的记录。
- 父商品广告花费 = 子体 `spends` 汇总。
- 当前期 `spends` > 上期 `spends`，命中广告花费增加。
- 当前期 `spends` < 上期 `spends`，命中广告花费减少。
- 本地 schema 实测：入参必填 `report_date`、`profile_id`，支持 `search_text` 按 ASIN/SKU 模糊搜索，花费字段名使用 `spends`。
- 当前本机运行实测：2026-07-08 对 `B0FNVXJCN8` / 美国站按 8 个子 ASIN 逐个调用成功，返回行在 `data.data[]` 中；必须以实际成功返回 `asin/spends` 为 verified 条件。

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

单月或时间段末月固定入参：

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

对比期同样调用一次，`month` 改为对比期 `yyyyMM`。如果用户指定的是连续月份段，当前期使用分析期最后一个可验证月份，对比期使用对比期最后一个可验证月份。

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
- 分析期末月数量 > 对比期末月数量，命中变体增加。
- 分析期末月数量 < 对比期末月数量，命中变体拆分/减少。
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

单月对比时，当前期和对比期分别调用。连续月份段对比时，对两个时间段内每一个完整月份分别调用，再聚合。当前月未结束时优先改用 `week` 同口径：

```json
{
  "timePieceType": "week",
  "timePieceValue": "<SIF 最新完整周周日 yyyy-MM-dd>"
}
```

对比期同样使用 `week`，`timePieceValue` 填分析周向前 28 天的周日日期，用作上月同口径周。

如果 `week` 不可用、最新完整周与当前月已发生分析区间没有交集，或对比周无数据，再退到：

```json
{
  "timePieceType": "latelyDay",
  "timePieceValue": "7"
}
```

当前月已发生区间超过 10 天时，`latelyDay` 优先使用 `30`；不超过 10 天时优先使用 `7`。但 `latelyDay` 只能作为周口径不可用时的兜底，最终证据必须说明实际使用的 SIF 时间口径。

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

- 父商品自然关键词数量 = 父商品分析期子体集合和对比期子体集合并集中，返回 `list[].asin` 命中的行的 `natural` 汇总。
- 单月对比时，分析期汇总 > 对比期汇总，命中自然搜索关键词数量增加；分析期汇总 < 对比期汇总，命中自然搜索关键词数量减少。
- 连续月份段对比时，分别计算分析期月均自然关键词数量和对比期月均自然关键词数量；分析期月均值 > 对比期月均值，命中增加；分析期月均值 < 对比期月均值，命中减少。最终证据同时输出月均值和期末值。
- 当前月未结束且使用周口径时，分析周自然关键词数量 > 上月同口径周，命中自然搜索关键词数量增加；分析周自然关键词数量 < 上月同口径周，命中自然搜索关键词数量减少。
- 本机实测状态：2026-07-08 对 `B0FNVXJCN8` / 美国站，`week=2026-06-28` 与 `week=2026-05-31` 均可正常返回 `list[].asin` 与 `list[].natural`。

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

单月对比时，当前期和对比期分别调用。连续月份段对比时，对两个时间段内每一个完整月份分别调用，再聚合。当前月未结束时改用：

```json
{
  "time_type": "lately",
  "time_value": "7"
}
```

当前月已发生区间不超过 10 天时优先使用 `time_value=7`；超过 10 天时优先使用 `time_value=30`。如果优先口径返回数据不足，再切换到另一个 `lately` 口径，但仍必须使用同一个 SIF 工具。最终证据必须说明实际使用的 SIF 时间口径。

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
- 单月对比时，按分析期与对比期的自然排名变化判断方向。
- 连续月份段对比时，对同一关键词分别计算分析期平均自然排名和对比期平均自然排名；分析期平均排名数字更小代表排名上升，数字更大代表排名下降。
- `improving` 数量和贡献高于 `declining/volatile/gap_detected` 时，支持自然搜索关键词排名上升。
- `declining/volatile/gap_detected/no_organic` 数量和贡献高于 `improving` 时，支持自然搜索关键词排名下降。
- 最终输出只列举影响最大的关键词，不罗列全部关键词。
- 影响最大关键词排序规则：先按 `abs(top_keywords[].contri_change)` 从高到低排序；`contri_change` 缺失时按 `top_keywords[].click_share` 从高到低排序；仍无法排序时按 `rank_evolution` 中自然排名变化幅度从高到低排序。
- 默认输出 Top 3 影响关键词；如果可验证关键词不足 3 个，输出全部可验证关键词；如果一个关键词都无法取得对比期自然排名、分析期自然排名或排名变化，不得输出自然搜索关键词排名因素。
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

- 对每个主要关键词，取分析期月份和对比期月份对应的 `volumes[]`。
- 单月对比时，分析期搜索量 > 对比期，命中主要关键词搜索趋势上升；分析期搜索量 < 对比期，命中主要关键词搜索趋势下降。
- 连续月份段对比时，分别计算分析期总搜索量、对比期总搜索量、分析期月均搜索量、对比期月均搜索量。两个时间段月份数相同则优先用总搜索量判断；月份数不同则优先用月均搜索量判断。
- 多数主要关键词分析期搜索量或月均搜索量 > 对比期，命中主要关键词搜索趋势上升。
- 多数主要关键词分析期搜索量或月均搜索量 < 对比期，命中主要关键词搜索趋势下降。
- 只用 `market_get_keyword_history` 的原始 ABA 搜索量 `volumes[]` 判断搜索趋势。
- 不使用 `market_get_keyword_demand` 的 `action_hint`、`interpretation`、`diagnosis` 输出结论或建议。

当前月未结束时的固定替代口径：

- 先按 `granularity=month` 调用一次。如果分析期月份没有出现在 `dates[]`，不得直接 blocked，继续使用同一工具改为 `granularity=week`。
- 周度调用仍使用 6.3 选出的同一批主要关键词，不重新换词。
- SIF 周度 `dates[]` 的日期按该周周日作为周起始日，周区间为该日期至该日期后第 6 天。
- 分析周：取 SIF 返回的最新周度日期，且该周区间必须与当前月已发生分析区间有交集。
- 对比周：取分析周日期向前 28 天的周度日期，用作“上月同口径周”。
- 对每个主要关键词，只有同时存在分析周和对比周 `volumes[]` 时，才纳入本项判断；缺少任一周数据的关键词不参与趋势方向判断，但要在内部验证状态中记录为该关键词无可比周度数据。
- 如果所有主要关键词都没有可比周度数据，才标记本验证项 blocked。
- 如果至少 1 个主要关键词有可比周度数据，本验证项记为 `verified`；最终证据必须写明“使用 SIF 最新完整周替代口径”，并输出有可比数据的关键词数量、上升/下降数量及代表关键词的前后搜索量。
- 周度替代口径只用于主要关键词搜索趋势，不改变领星流量、领星广告、卖家精灵 Deals 或其他数据项的时间窗口。

阻塞条件：

- `market_get_keyword_history` 不可用。
- 主要关键词列表为空。
- `volumes[]` 或 `dates[]` 缺失。
- 完整月分析时，分析期或对比期月份无法对应到数据点。
- 当前月未结束且使用周度替代口径时，SIF 最新周与当前月已发生分析区间没有交集。
- 当前月未结束且使用周度替代口径时，全部主要关键词都缺少分析周或对比周搜索量。

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
- 至少 1 条 `accessible=true` 且时间、内容、类型都满足，才允许将对应因素判定为命中。
- 如果已完整执行 7.1 的全部搜索组合，且没有找到满足时间、内容、类型要求的可访问页面，则对应验证项仍为 `verified`，证据结论为“未发现合格站外/联盟客推广证据”，最终不输出该因素。
- 搜索无结果、搜索结果与 ASIN/品牌/产品词不相关、或结果页面时间不在分析区间内，都属于“已验证但未命中”，不是 blocked。
- 只有无法完成搜索、无法打开候选页面、候选页面存在相关推广但时间或类型无法判断时，才标记 blocked。

### 7.3 分类

站外推广：

- `promotion_type=operator_offsite`
- 证据显示品牌官方、卖家主动活动、主动投放、官方折扣码、官方社群推广等。

联盟客推广：

- `promotion_type=affiliate_self_initiated`
- 证据显示 Amazon Associates、网红、测评站、导购站、第三方 Deal 分享等。

阻塞条件：

- 无法搜索互联网。
- 搜索结果中存在疑似相关候选网页，但无法打开候选网页验证内容。
- 搜索结果中存在疑似相关候选网页，但只能看到搜索结果摘要，没有可访问页面。
- 候选网页内容命中 ASIN、品牌或产品词，但无法判断时间是否落在分析区间内。
- 候选网页内容命中 ASIN、品牌或产品词，且时间可能落在分析区间内，但无法区分运营主动推广或第三方自发推广。
- 已完整搜索且没有找到合格网页证据时，不按本节阻塞；该验证项记为 verified 且未命中。

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

## 9. 最终输出证据转换

本文件中的字段名只用于内部取数和校验。最终输出给用户时，必须把内部字段和计算过程转换成业务语言：

- 不输出 MCP 字段名、接口名、数据库名、字段表达式、等号判断或原始 JSON 路径。
- 不输出“分析期高于对比期”“分析期低于对比期”“有变化”等泛泛描述。
- 必须输出可核验的业务指标：对比期值、分析期值、变化值、变化率、子体数量、异常数量、影响日期、影响天数、链接或页面日期。
- 库存类证据只写“无可售库存”“恢复可售库存”“部分子体断货”“父商品整体断货”等自然语言。
- 广告类证据必须写明对比期花费、分析期花费、变化金额、变化率和币种。
- 自然搜索类证据必须写明关键词数量、排名数值或搜索趋势数值的前后对比。

如果内部验证已经命中，但无法把证据转换成上述业务指标，视为该验证项未完成，不得输出该因素。
