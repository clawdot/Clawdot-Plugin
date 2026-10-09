# 工具参考（外卖点单 · 登录版 · MCP 工具）

> 来源：`ClawDot-Hub/src/clawdot_hub/upstream_tools.delivery.json` + `upstream_tools.errand.json`（入参 schema，一 cap 一份）
> + `annotations.delivery.json` + `annotations.errand.json`（暴露策略，同上）+ `contract.py`（hub 加的字段，仅外卖的金额闸相关）。
> **这一版不需要、也看不到"这是哪个用户"这个参数**——旧的直连网关模式里每个工具第一个参数都要传用户授权凭证，
> 这一版由 ClawDot-Hub 在你（AI）看不到的那一层自动补上，34 个暴露工具（外卖 19 + 跑腿 15）的入参里都**没有**这个字段，别自己加。
> 金额单位一律为**分**。坐标一律 GCJ-02。外卖和跑腿共用同一次登录，互不影响。

## 暴露给 AI 的工具（34 个：外卖 19 + 跑腿 15）

### 外卖（19 个）

| 工具 | 用途 | 关键入参 | 关键返回 / 注意 |
|---|---|---|---|
| `search_addresses()` | 列已存地址 / 搜新地址 | `keyword`? · `lat`/`lng`? · `city`? | `saved_addresses[]`（最近用过在前）· `suggestions[]`（带一次性 `token`）· `nearest_address_id` + `distance_meters`（传了坐标时）。POI 搜索必须给 city 或坐标 |
| `select_address()` | 把候选或已存地址落成收货地址 | `contact_name` · `contact_phone` · `suggestion_token`（新）或 `address_id`（已存）· `address_detail`（新地址必填）· `tag`?(≤6 字) | `address_id`(addr_)。缺门牌 → `DETAIL_REQUIRED` |
| `update_address()` | 改已存地址 | `address_id` · `tag`?（只改标签可单独传）· `suggestion_token` + `address_detail`（改位置） | 收件人姓名/电话不能在这改 |
| `delete_address()` | 删已存地址 | `address_id` | 删不存在的会报错 |
| `search_shops()` | 搜店 / 浏览附近 | `keyword`? · `address_id` 或 `lat`+`lng` · `city`? · `offset`(浏览分页) | 每家 `shop_id` + `cart_id`（成对）· `available` + `unavailable_reason` · `matched_items`（仅预览，无 id）· keyword 模式带 `search_match_level`(exact/related) |
| `get_shop_info()` | 店铺名片（营业时间、评分、配送） | `shop_id` · `cart_id` | 不含菜单；比拉菜单轻 |
| `get_shop_menu()` | 拉菜单 | `shop_id` · `cart_id` · `keyword`? · `limit`/`offset`? · `full_menu`(大纲) · `grouped`? | `items[]`(item_id / price / available / min_purchase / sku_options / ingredient_options) · `required_groups[]` · `total_items` / `has_more` / `next_offset` · `search_match_level`(exact/abbrev/fuzzy/partial/none)。大纲项不可直接下单；cart 过期用同一对 id 重调即原地重建 |
| `get_item_options()` | 批量拿商品完整规格（带选中标记） | `cart_id` · `items=[{item_id, sku_id?, ingredient_option_ids?, ingredient_quantities?}]` · `grouped`? | 每个 sku 下嵌自己的 `ingredient_options`；`multi_select` / `max_select` / `max_quantity` / `price_steps`。单个坏 item 不拖垮整批 |
| `batch_get_item_options()` | 多家店一次拿商品完整规格（跨店版 `get_item_options()`） | `carts=[{cart_id, items=[{item_id, sku_id?, ingredient_option_ids?, ingredient_quantities?}]}]` · `grouped`? | `carts[]` 与入参一一对应、同序；每家要么带 `items`（每项与 `get_item_options()` 逐字段相同），要么带 `error`（这家店的车过期等），一家失败不拖垮其余；所有店合计最多 500 个商品，超出报 `BATCH_TOO_LARGE`，只查一家店用 `get_item_options()` |
| `get_item_description()` | 商品详情文案（原料、份量、辣度…） | `cart_id` · `item_id` | `details[]`={label, text}，label 是开放集合；`details=[]` 表示商家没写 |
| `get_shop_share_url()` | 生成店铺 H5 分享短链 | `shop_id` | 短链 |
| `quote_cart()` | 算价（软失败在这暴露） | `shop_id` · `cart_id` · `items` · `address_id`?（不传按购物车所在位置算价；`preview_order()` 仍须传） | `quote_id`(qt_, 10 分钟) · `can_checkout` · `blocking_code`(0/1/2) · `blocking_reason` · `minimum_order_gap`(分) |
| `list_coupons()` | 可用优惠券 | — | `cpn_` 开头的券 id，传给 preview 的 `coupon_ids` |
| `preview_order()` | 下单前预览，签发确认令牌 | `shop_id` · `cart_id` · `address_id` · `items` · `quote_id`? · `coupon_ids`? · `order_remark`? | `preview_id`(prv_) + `confirmation_token`(cf_，**10 分钟、一次性**) · `payable_price` / `delivery_fee` / `packing_fee` · `warnings`(如 SHOP_RESTING)。**失败不重试** |
| `create_order()` | 真实下单 | `preview_id` · `confirmation_token` · `payment_method`? · `callback_url`? · **`hub_confirm_token`?**（仅在上一次调用被金额闸拦下、拿到该 token 后才带，见 @errors.md「金额闸」） | 正常 → `order_id`(ord_) · `status` · `payment_action{action_type, action_url}`（付款链接，原样给用户）· `detail_url`。被闸拦下 → 不返回订单，返回 `need_confirmation=true` 形状（见 @errors.md）。**失败不重试；只在用户显式确认后调** |
| `get_order_status()` | 单笔订单实时状态 | `order_id` | `status` / `status_text` / 配送进度 / 实付 · `detail_url`（未付时也是收银台） |
| `list_orders()` | 历史订单 | `limit`(默认 20，最大 50) · `offset` | 新单在前；`items_snapshot[].line_total` 已含数量；每单带 `address_id` 与 `address` 可读信息；`next_offset`=null 表示没有更多 |
| `get_sign_action()` | 免密支付签约入口（渠道级，被要求时才用） | `return_url` · `format`="h5"/"qrcode" | `action.action_type`=open_h5 / show_qr / none；`action_qr_data_uri` 是图片，不要复述 |
| `get_sign_status()` | 查询是否已签约 | — | `signed` / `sign_status` |

### 跑腿（15 个）

> 跑腿地址簿独立于外卖地址簿（`plat_` 前缀，不是 `addr_`）；`from_address`/`to_address` 是要素对象，键为
> `address_id?`（地址簿已存，可省 `contact_name`/`contact_phone`）或 `(address_text + lat + lng)`（坐标直通，两者仍须带联系人）。
> `goods` 是数组，每项 `{name, qty, price_fen?}`，`name` 写用户原话，平台不解析、只按 `goods_category_code` 定责。

| 工具 | 用途 | 关键入参 | 关键返回 / 注意 |
|---|---|---|---|
| `errand_search_addresses()` | 搜地址 → POI 候选 + 已存地址并列 | `keyword`? · `city`? · `lat`/`lng`?（用户说「从我这儿」、没有地名可搜时传他的坐标反查地名；两样都不给报 `KEYWORD_REQUIRED`；说了地名就别拿坐标顶替 keyword） | `candidates[]`（高德 POI：name/address/lat/lng/adcode）· `saved_matches[]`（同 `errand_list_addresses()` 形状，带 `plat_` id、脱敏电话、门牌，`match_reason`=keyword/same_place/both）。`saved_matches` 非空时优先摆，但**绝不自动选中**，也别把候选和已存地址强行合并 |
| `errand_list_addresses()` | 列跑腿地址簿（与外卖地址簿独立） | — | 每条含 `id`(plat_)/联系人/`contact_phone_masked`/坐标/`detail`(门牌)/`tag`；无地址返回 `{"addresses": []}`，不报错 |
| `errand_save_address()` | 存地址进跑腿地址簿 | `contact_name` · `address` · `lat` · `lng`(必填) · `detail`? · `tag`? · `contact_phone`? | 返回存好的地址（id 带 `plat_`）；电话落库即密文，只回脱敏 |
| `errand_update_address()` | 改地址簿里的某条（只传要改的字段） | `address_id`(必填)；改**位置**须 `address`+`lat`+`lng` 三个一起传 | 返回改完后的完整地址；地址不存在或不属于该用户 → 报错 |
| `errand_delete_address()` | 删地址簿一条（不可撤销） | `address_id`(必填) | `{address_id, deleted:true, deleted_count}`；已下过的订单不受影响（存的是地址快照） |
| `errand_list_goods_categories()` | 列出可选物品品类供用户挑 | — | `categories:[{code, name, valuable}]` 共 15 类（前 7 贵重 `valuable=true`，后 8 普通）；`code` 回填 `errand_quote()` 的 `goods_category_code`；贵重类目应主动问是否保价（`insured`），由用户决定不替他开 |
| `errand_list_schedule_slots()` | 列出可选预约送达时间 | — | `days[].slots[].value` 回填 `scheduled_at`（毫秒时间戳）；一档 15 分钟，最早 45 分钟后、最晚明日 23:45（北京时间）。**只把清单里的 value 原样回填**，不自己算、不复用旧单的值（早过了会被拒）；用户说"尽快"就不传，别传 0 |
| `errand_quote()` | 跑腿询价：多运力报价 + 单次确认令牌 | `from_address` · `to_address` · `goods`(均必填) · `total_weight_g`? · `scheduled_at`? · `person_direct`? · `insured`? · `goods_category_code`? · `remark`?(≤200字) | `{quote_id, quotes:[{company_code, company_name, fee, distance, coupon_fee, estimated_minutes, estimated_arrival_time}], expires_in_seconds}`（预计送达时长 / 时刻，距离缺失时为 null）；命中禁运词（毒品/枪爆/危化品/违法交易物）→ `GOODS_PROHIBITED`（见 @errors.md）；须在有效期内**完成下单**，过期重新询价 |
| `errand_create()` | 核销 `quote_id` 下单（落待支付单，真实扣款） | `quote_id` · `company_code`(必填，须在本次 quote 报价集内) · `callback_url`? | `{order_id(err_), status:pending_payment, pay_status:unpaid, quote_fee, cashier_url, payment_expire_at}` + 整单信息（与 `errand_get_order()` 同一套字段，电话只回脱敏）；`payment_expire_at` 是最晚付款时间、窗很短，**拿到 `cashier_url` 立刻交给用户**；此步只建单**不呼叫骑手**，支付成功后才派单。**不接 hub 金额闸**（不同于外卖 `create_order()`，见 @errors.md）。**失败不重试；只在用户显式确认后调** |
| `errand_list_orders()` | 历史跑腿单（可回填复用） | `limit`?(默认 5，上限 20) · `offset`? · `status`?(逗号分隔多传) · `created_after`?/`created_before`?(北京时间) | `{orders[], next_offset}`(null=没有更多)；`goods_category_code` 可原样带回复用，`scheduled_at` **不可**原样带回（会被拒，需重新问用户约几点）；`status` 传非法值会报错，不会静默返空 |
| `errand_pre_cancel()` | 取消前预检（查违约金，不执行取消） | `order_id`(必填) | `{order_id, cancel_fee, quote_fee, refund_amount}`(分)；骑手已接单后取消可能有违约金，下单后要取消**先调这个**给用户看费用 |
| `errand_cancel()` | 取消跑腿单（真实扣款场景下的退款，不可逆） | `order_id`(必填) · `reason`? | `{order_id, status:cancelled, cancel_fee, refund_amount}`(分)；已支付单按「实付 − 违约金」原路退款，未支付直接关闭，已完成/已取消的单不可再取消 |
| `errand_get_order()` | 跑腿单详情（含进行中订单的实时状态） | `order_id`(必填) | 状态/费用/收发要素/货品/品类/备注 + 进行中的 `status_desc`/`timeline`/`rider`；`scheduled_at` 不要复用（已过期） |
| `errand_get_rider()` | 骑手实时位置 | `order_id`(必填) | `{rider_name, rider_phone, latitude, longitude, time}`(GCJ-02)；仅配送进行中可查 |
| `errand_add_tip()` | 加小费催单（独立支付，真实扣款） | `order_id` · `tip_fee`(必填，分，须>0) | `{order_id, tip_fee, status:pending_payment, cashier_url}`；仅骑手接单前可加、可多次累加；付款期间若骑手已接单会自动退小费 |

## 已禁用（ClawDot-Hub 内部处理，AI 看不到，`tools/list` 里不出现）

### 外卖（5 个）

| 工具 | 为什么禁 |
|---|---|
| `request_user_bind()` | 由 OAuth 授权链替代——登录在接入这一步就做完了（授权页上输手机号 + 验证码），不在对话里 |
| `verify_user_bind()` | 由 OAuth 授权链替代 |
| `get_user_auth_status()` | 授权是否有效由 hub 管；失效时业务调用会直接返回 `consent_required`，不需要单独查 |
| `revoke_user_bind()` | 由 OAuth 的 `/revoke` 替代；解绑不在对话里做 |
| `get_homepage_url()` | 渠道级首页跳转，依赖渠道配置，对点单流程没有价值 |

### 跑腿（4 个）

| 工具 | 为什么禁 |
|---|---|
| `errand_request_user_bind()` | 由 OAuth 授权链替代（只有外卖有授权页，跑腿靠内核互通拿到同一枚授权） |
| `errand_verify_user_bind()` | 由 OAuth 授权链替代 |
| `errand_revoke_user_bind()` | 由 OAuth 的 `/revoke` 替代 |
| `errand_get_auth_status()` | `at_` 有效性由 hub 管；失效回 `consent_required` |

## items 形状（quote / preview 共用，外卖）

```json
[
  {"item_id": "item_x", "quantity": 1,
   "sku_id": "sku_y",
   "ingredient_option_ids": ["opt_a"],
   "ingredient_quantities": [{"option_id": "opt_b", "quantity": 2}],
   "remark": "少冰"}
]
```
- `quantity` 必填且 ≥ `min_purchase`。
- `sku_id` 不传用默认规格；`ingredient_option_ids` 不传用默认加料。
- 所有 id 只在签发它们的 `cart_id` 内有效。

## from_address / to_address / goods 形状（跑腿，`errand_quote()` 用）

```json
{
  "from_address": {"address_id": "plat_xxx"},
  "to_address": {"address_text": "西湖文化广场", "lat": 30.28, "lng": 120.16,
                  "contact_name": "张三", "contact_phone": "138****5678"},
  "goods": [{"name": "文件袋", "qty": 1, "price_fen": 0}]
}
```
- 走 `address_id`（地址簿已存）时 `contact_name`/`contact_phone` 可省，服务端用地址簿里存的；坐标直通形态两者仍必给。
- 纯地址文本（无坐标）暂不支持 POI 解析——坐标必须来自 `errand_search_addresses()` 的候选或已存地址。
- `goods[].name` 写用户原话即可，平台不解析名称，履约与理赔只看 `goods_category_code`。

## 网关签发的引用及有效期（外卖 + 跑腿）

| 前缀 | 是什么 | 有效期 | 失效后 |
|---|---|---|---|
| `cart_` | 购物车（随 shop_id 一起由 search_shops 签发） | 30 分钟空闲 / 1 小时绝对 | 同一对 shop_id + cart_id 重调 `get_shop_menu()` 原地重建 |
| `qt_` | 试算结果 | 10 分钟 | 重调 `quote_cart()` |
| `prv_` | 预览 | 10 分钟 | 重调 `preview_order()` |
| `cf_` | 确认令牌 | 10 分钟，**仅可使用一次** | 重调 `preview_order()` 换新 |
| `cpn_` | 优惠券 | — | 重调 `list_coupons()` |
| `addr_` / `shop_` / `ord_` | 地址 / 店铺 / 订单（外卖） | — | 分别重调 `search_addresses()` / `search_shops()` / 用 `create_order()` 返回的 order_id |
| `hc_` | hub 金额闸确认令牌（**不是**上游发的，只在 `create_order()` 被闸拦下时出现，仅外卖） | ≤10 分钟，绑定这一次 preview，**仅可使用一次** | 过期或对不上 → 重调 `preview_order()` 换新的 preview_id/confirmation_token 再走一遍，见 @errors.md |
| `plat_` | 跑腿地址簿地址 id（独立于外卖的 `addr_`） | — | 重调 `errand_list_addresses()` / `errand_search_addresses()` |
| `err_` | 跑腿订单 id | — | 用 `errand_get_order()` / `errand_list_orders()` 查 |
| `quote_id`（跑腿询价令牌，源文档未给固定前缀字样） | 跑腿报价，单次确认令牌 | 动态值，见 `errand_quote()` 返回的 `expires_in_seconds`（不是固定分钟数） | 过期或已用 → 重调 `errand_quote()` 换新 |

<!-- BEGIN:params -->

## 入参详表（由 MCP schema 生成，勿手改）

这一节是 `tools/list` 回的 `inputSchema` 逐字段展开 —— **它就是服务端此刻真实收什么**。
上面那张表是人工整理的「关键入参」，讲的是怎么用；这张是全量字段，讲的是能传什么。

`必填` 一列为空即选填。**说明一列为空 = 上游没给说明**，不是漏写；
遇到空的别猜字段含义，按上面那张表的用法走，或问我们。

### `batch_get_item_options()` · 外卖

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `carts` | array | ✓ |  |
| `grouped` | boolean |  | true = 按分类分组返回 |
| `lang` | string |  |  |
| `include_chinese` | boolean |  | Optional, default false. true = also return the Chinese original next to every translated field, as a sibling "<key>_zh" key (arrays get a parallel array of the same order and length). Only upstream shop/product data carries "<key>_zh"; notices written by the gateway itself stay in the requested language, and a field whose translation failed has no "<key>_zh". Ignored when the target language is zh. |

### `create_order()` · 外卖

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `preview_id` | string | ✓ | 预订单句柄（`prv_` 开头），`preview_order()` 返回；**原样传，一个字符都不能改** |
| `confirmation_token` | string | ✓ | 下单确认令牌（`cf_` 开头），与 `preview_id` 同时返回；**原样传** |
| `payment_method` | string |  | 支付方式；不传走默认 |
| `callback_url` | string |  | 订单状态变化的回调地址；不传则自行轮询 `get_order_status()` |
| `lang` | string |  |  |
| `include_chinese` | boolean |  | Optional, default false. true = also return the Chinese original next to every translated field, as a sibling "<key>_zh" key (arrays get a parallel array of the same order and length). Only upstream shop/product data carries "<key>_zh"; notices written by the gateway itself stay in the requested language, and a field whose translation failed has no "<key>_zh". Ignored when the target language is zh. |
| `require_phone_verify` | boolean |  |  |
| `hub_confirm_token` | string |  | 二次确认票据。金额超过本平台上限时，本工具不会下单，而是返回 {need_confirmation: true, reason: "amount_over_limit", payable_price, limit_cents, hub_confirm_token}。把金额告诉用户、拿到用户明确同意后，用同样的参数再调一次并带上这里的 hub_confirm_token 才会真正下单。票据 10 分钟内有效，只对这一笔预订单、这一个用户有效。 |

### `delete_address()` · 外卖

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `address_id` | string | ✓ | 已存收货地址的句柄，`search_addresses()` 的 `saved_addresses[]` 或 `select_address()` 返回 |

### `get_item_description()` · 外卖

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `cart_id` | string | ✓ | 购物车句柄，与 `shop_id` 同时返回、**必须成对使用** |
| `item_id` | string | ✓ | 商品句柄，`get_shop_menu()` 返回；**菜单大纲里的 id 不能直接下单**，要带 `keyword` 拉一次 |
| `lang` | string |  |  |
| `include_chinese` | boolean |  | Optional, default false. true = also return the Chinese original next to every translated field, as a sibling "<key>_zh" key (arrays get a parallel array of the same order and length). Only upstream shop/product data carries "<key>_zh"; notices written by the gateway itself stay in the requested language, and a field whose translation failed has no "<key>_zh". Ignored when the target language is zh. |

### `get_item_options()` · 外卖

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `cart_id` | string | ✓ | 购物车句柄，与 `shop_id` 同时返回、**必须成对使用** |
| `items` | array | ✓ | 商品行数组，每行 `item_id` + 数量（+ 已选规格 / 加料） |
| `grouped` | boolean |  | true = 按分类分组返回 |
| `lang` | string |  |  |
| `include_chinese` | boolean |  | Optional, default false. true = also return the Chinese original next to every translated field, as a sibling "<key>_zh" key (arrays get a parallel array of the same order and length). Only upstream shop/product data carries "<key>_zh"; notices written by the gateway itself stay in the requested language, and a field whose translation failed has no "<key>_zh". Ignored when the target language is zh. |

### `get_order_status()` · 外卖

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `order_id` | string | ✓ | 订单句柄，下单时返回 |
| `lang` | string |  |  |
| `include_chinese` | boolean |  | Optional, default false. true = also return the Chinese original next to every translated field, as a sibling "<key>_zh" key (arrays get a parallel array of the same order and length). Only upstream shop/product data carries "<key>_zh"; notices written by the gateway itself stay in the requested language, and a field whose translation failed has no "<key>_zh". Ignored when the target language is zh. |

### `get_shop_info()` · 外卖

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `shop_id` | string | ✓ | 店铺句柄，`search_shops()` 返回；**与 `cart_id` 成对**，后面每一步都要一起带 |
| `cart_id` | string | ✓ | 购物车句柄，与 `shop_id` 同时返回、**必须成对使用** |
| `lang` | string |  |  |
| `include_chinese` | boolean |  | Optional, default false. true = also return the Chinese original next to every translated field, as a sibling "<key>_zh" key (arrays get a parallel array of the same order and length). Only upstream shop/product data carries "<key>_zh"; notices written by the gateway itself stay in the requested language, and a field whose translation failed has no "<key>_zh". Ignored when the target language is zh. |

### `get_shop_menu()` · 外卖

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `shop_id` | string | ✓ | 店铺句柄，`search_shops()` 返回；**与 `cart_id` 成对**，后面每一步都要一起带 |
| `cart_id` | string | ✓ | 购物车句柄，与 `shop_id` 同时返回、**必须成对使用** |
| `address_id` | string |  | 已存收货地址的句柄，`search_addresses()` 的 `saved_addresses[]` 或 `select_address()` 返回 |
| `lat` | number |  | 纬度；与 `lng` 成对，作为 `address_id` 的替代定位方式 |
| `lng` | number |  | 经度，见 `lat` |
| `keyword` | string |  | 菜名关键词；**带上它返回的菜才含规格、才能下单** |
| `limit` | integer |  | 本次最多返回几条 |
| `offset` | integer |  | 分页偏移，用上一次返回的 `next_offset` |
| `full_menu` | boolean |  | true = 要全菜单大纲（快，**不含规格、不能直接下单**） |
| `grouped` | boolean |  | true = 按分类分组返回 |
| `exclude_required_selection` | boolean |  |  |
| `lang` | string |  |  |
| `include_chinese` | boolean |  | Optional, default false. true = also return the Chinese original next to every translated field, as a sibling "<key>_zh" key (arrays get a parallel array of the same order and length). Only upstream shop/product data carries "<key>_zh"; notices written by the gateway itself stay in the requested language, and a field whose translation failed has no "<key>_zh". Ignored when the target language is zh. |

### `get_shop_share_url()` · 外卖

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `shop_id` | string | ✓ | 店铺句柄，`search_shops()` 返回；**与 `cart_id` 成对**，后面每一步都要一起带 |

### `get_sign_action()` · 外卖

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `return_url` | string | ✓ | 完成后跳回的地址 |
| `format` | string |  | 返回格式 |

### `list_coupons()` · 外卖

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `lang` | string |  |  |
| `include_chinese` | boolean |  | Optional, default false. true = also return the Chinese original next to every translated field, as a sibling "<key>_zh" key (arrays get a parallel array of the same order and length). Only upstream shop/product data carries "<key>_zh"; notices written by the gateway itself stay in the requested language, and a field whose translation failed has no "<key>_zh". Ignored when the target language is zh. |

### `list_orders()` · 外卖

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `limit` | integer |  | 本次最多返回几条 |
| `offset` | integer |  | 分页偏移，用上一次返回的 `next_offset` |
| `lang` | string |  |  |
| `include_chinese` | boolean |  | Optional, default false. true = also return the Chinese original next to every translated field, as a sibling "<key>_zh" key (arrays get a parallel array of the same order and length). Only upstream shop/product data carries "<key>_zh"; notices written by the gateway itself stay in the requested language, and a field whose translation failed has no "<key>_zh". Ignored when the target language is zh. |

### `preview_order()` · 外卖

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `shop_id` | string | ✓ | 店铺句柄，`search_shops()` 返回；**与 `cart_id` 成对**，后面每一步都要一起带 |
| `cart_id` | string | ✓ | 购物车句柄，与 `shop_id` 同时返回、**必须成对使用** |
| `address_id` | string | ✓ | 已存收货地址的句柄，`search_addresses()` 的 `saved_addresses[]` 或 `select_address()` 返回 |
| `items` | array | ✓ | 商品行数组，每行 `item_id` + 数量（+ 已选规格 / 加料） |
| `quote_id` | string |  | 试算句柄，`quote_cart()` 返回 |
| `coupon_ids` | array |  | 要抵扣的优惠券，`list_coupons()` 返回 |
| `order_remark` | string |  | 给商家的备注 |
| `lang` | string |  |  |
| `include_chinese` | boolean |  | Optional, default false. true = also return the Chinese original next to every translated field, as a sibling "<key>_zh" key (arrays get a parallel array of the same order and length). Only upstream shop/product data carries "<key>_zh"; notices written by the gateway itself stay in the requested language, and a field whose translation failed has no "<key>_zh". Ignored when the target language is zh. |

### `quote_cart()` · 外卖

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `shop_id` | string | ✓ | 店铺句柄，`search_shops()` 返回；**与 `cart_id` 成对**，后面每一步都要一起带 |
| `cart_id` | string | ✓ | 购物车句柄，与 `shop_id` 同时返回、**必须成对使用** |
| `items` | array | ✓ | 商品行数组，每行 `item_id` + 数量（+ 已选规格 / 加料） |
| `address_id` | string |  | 已存收货地址的句柄，`search_addresses()` 的 `saved_addresses[]` 或 `select_address()` 返回 |
| `lang` | string |  |  |
| `include_chinese` | boolean |  | Optional, default false. true = also return the Chinese original next to every translated field, as a sibling "<key>_zh" key (arrays get a parallel array of the same order and length). Only upstream shop/product data carries "<key>_zh"; notices written by the gateway itself stay in the requested language, and a field whose translation failed has no "<key>_zh". Ignored when the target language is zh. |

### `search_addresses()` · 外卖

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `keyword` | string |  | 地址搜索词；不传则只列已存地址 |
| `lat` | number |  | 纬度；与 `lng` 成对，作为 `address_id` 的替代定位方式 |
| `lng` | number |  | 经度，见 `lat` |
| `city` | string |  | 城市名，缩小搜索范围 |
| `lang` | string |  |  |
| `include_chinese` | boolean |  | Optional, default false. true = also return the Chinese original next to every translated field, as a sibling "<key>_zh" key (arrays get a parallel array of the same order and length). Only upstream shop/product data carries "<key>_zh"; notices written by the gateway itself stay in the requested language, and a field whose translation failed has no "<key>_zh". Ignored when the target language is zh. |

### `search_shops()` · 外卖

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `keyword` | string |  | 搜索词；不传则按位置浏览 |
| `address_id` | string |  | 已存收货地址的句柄，`search_addresses()` 的 `saved_addresses[]` 或 `select_address()` 返回 |
| `lat` | number |  | 纬度；与 `lng` 成对，作为 `address_id` 的替代定位方式 |
| `lng` | number |  | 经度，见 `lat` |
| `city` | string |  | 城市名，缩小搜索范围 |
| `offset` | integer |  | 分页偏移，用上一次返回的 `next_offset` |
| `exclude_required_selection` | boolean |  |  |
| `with_recommendations` | boolean |  |  |
| `chain_brands_only` | boolean |  |  |
| `lang` | string |  |  |
| `include_chinese` | boolean |  | Optional, default false. true = also return the Chinese original next to every translated field, as a sibling "<key>_zh" key (arrays get a parallel array of the same order and length). Only upstream shop/product data carries "<key>_zh"; notices written by the gateway itself stay in the requested language, and a field whose translation failed has no "<key>_zh". Ignored when the target language is zh. |

### `select_address()` · 外卖

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `contact_name` | string | ✓ | 收件人姓名 |
| `contact_phone` | string | ✓ | 收件人手机号 |
| `suggestion_token` | string |  | 地址候选的**一次性**句柄，`search_addresses()` 的 `suggestions[]` 给；用过即失效 |
| `address_id` | string |  | 已存收货地址的句柄，`search_addresses()` 的 `saved_addresses[]` 或 `select_address()` 返回 |
| `address_detail` | string |  | 门牌号（几号楼几室）；**落新地址时必填**，POI 只到楼栋 |
| `tag` | string |  | 地址标签（家 / 公司 这类），可单独改 |

### `update_address()` · 外卖

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `address_id` | string | ✓ | 已存收货地址的句柄，`search_addresses()` 的 `saved_addresses[]` 或 `select_address()` 返回 |
| `suggestion_token` | string |  | 改位置时传；只改 `tag` 不用传 |
| `address_detail` | string |  | 门牌号（几号楼几室）；**落新地址时必填**，POI 只到楼栋 |
| `tag` | string |  | 地址标签（家 / 公司 这类），可单独改 |

### `errand_add_tip()` · 跑腿

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `order_id` | string | ✓ | 订单句柄，下单时返回 |
| `tip_fee` | integer | ✓ | 小费金额（分） |

### `errand_cancel()` · 跑腿

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `order_id` | string | ✓ | 订单句柄，下单时返回 |
| `reason` | string |  | 原因说明 |

### `errand_create()` · 跑腿

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `quote_id` | string | ✓ | 试算句柄，`quote_cart()` 返回 |
| `company_code` | integer | ✓ | 履约商编码 |
| `callback_url` | string |  | 订单状态变化的回调地址；不传则自行轮询 `get_order_status()` |

### `errand_delete_address()` · 跑腿

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `address_id` | string | ✓ | 已存收货地址的句柄，`search_addresses()` 的 `saved_addresses[]` 或 `select_address()` 返回 |

### `errand_get_order()` · 跑腿

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `order_id` | string | ✓ | 订单句柄，下单时返回 |

### `errand_get_rider()` · 跑腿

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `order_id` | string | ✓ | 订单句柄，下单时返回 |

### `errand_list_orders()` · 跑腿

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `limit` | integer |  | 本次最多返回几条 |
| `offset` | integer |  | 分页偏移，用上一次返回的 `next_offset` |
| `status` | string |  | 按订单状态过滤 |
| `created_after` | string |  | 起始时间，过滤用 |
| `created_before` | string |  | 截止时间，过滤用 |

### `errand_pre_cancel()` · 跑腿

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `order_id` | string | ✓ | 订单句柄，下单时返回 |

### `errand_quote()` · 跑腿

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `from_address` | object | ✓ | 取件地址 |
| `to_address` | object | ✓ | 送达地址 |
| `goods` | array | ✓ | 物品描述 |
| `total_weight_g` | integer |  | 总重量（克） |
| `scheduled_at` | integer |  | 预约时间，`errand_list_schedule_slots()` 返回可选时段 |
| `person_direct` | boolean |  | 是否要求专人直送 |
| `insured` | boolean |  | 是否保价 |
| `goods_category_code` | integer |  | 物品分类编码，`errand_list_goods_categories()` 返回 |
| `remark` | string |  | 备注 |

### `errand_save_address()` · 跑腿

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `contact_name` | string | ✓ | 收件人姓名 |
| `address` | string | ✓ | 地址文本 |
| `lat` | number | ✓ | 纬度；与 `lng` 成对，作为 `address_id` 的替代定位方式 |
| `lng` | number | ✓ | 经度，见 `lat` |
| `detail` | string |  | 详细地址（门牌号那一段） |
| `tag` | string |  | 地址标签（家 / 公司 这类），可单独改 |
| `contact_phone` | string |  | 收件人手机号 |

### `errand_search_addresses()` · 跑腿

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `keyword` | string |  | 地名 / 地址关键词（用户说了地名时传，可加 `city` 缩范围）；与 `lat`/`lng` 至少给一样，都不给报 `KEYWORD_REQUIRED`；两样都给时按它搜，坐标只用来把近的排前面 |
| `city` | string |  | 城市名，缩小搜索范围 |
| `lat` | number |  | 纬度，与 `lng` 成对；用户说「从我这儿」、没有地名可搜时只传坐标，回的是这个位置叫什么（所属的几个地名，从具体到宽泛）；同时给了 `keyword` 时只用来排序 |
| `lng` | number |  | 经度，见 `lat` |

### `errand_update_address()` · 跑腿

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `address_id` | string | ✓ | 已存收货地址的句柄，`search_addresses()` 的 `saved_addresses[]` 或 `select_address()` 返回 |
| `contact_name` | string |  | 收件人姓名 |
| `contact_phone` | string |  | 收件人手机号 |
| `address` | string |  | 地址文本 |
| `lat` | number |  | 纬度；与 `lng` 成对，作为 `address_id` 的替代定位方式 |
| `lng` | number |  | 经度，见 `lat` |
| `detail` | string |  | 详细地址（门牌号那一段） |
| `tag` | string |  | 地址标签（家 / 公司 这类），可单独改 |

<!-- END:params -->
