# 错误码与重试策略（OAuth 版）

> 外卖上游业务码来源：open-gateway `docs/外卖对外错误码.md`（37 个硬失败码的体系不变）。
> 跑腿业务码/行为来源：跑腿的工具说明原文——跑腿走独立的履约上游，
> **不在**外卖那 37 个码的体系里，目前源文档里只点名了一个正式命名的码，其余是说明文字里点出的失败行为，见下面「跑腿（errand）相关码与行为」。
> hub 自己的错误码来源：`ClawDot-Hub/src/clawdot_hub/contract.py`（`ErrorCode` + `GATE_RESULT_EXAMPLE`），这一层对外卖/跑腿通用。
> code 是稳定契约；message 文案可能变，**不要对 message 做字符串匹配**。

## 三种返回形状，先分清楚是哪一种

**每次调用先看返回体长什么样，再决定按哪张表处理。**

**① hub 网关级错误**（新，这一版特有，外卖/跑腿通用）——`error` 字段是**字符串**，不是对象：
```json
{"error": "consent_required"}
```
只有两个取值会以这个形状出现：
- `consent_required`：这次授权已经失效（比如用户换了账号重新登录，或撤销了这个连接的授权）。**如实告诉用户需要对这个连接重新授权一次**，怎么做见业务流程里的「鉴权模型」一节——没有任何工具可以在对话里把这个问题解决掉，不要尝试猜或编造下一步，也不要反复重试。
- `upstream_error`：hub 连不上上游或协议错误，跟用户的操作无关。按下面「上游瞬时故障」的口径处理：读操作可退避重试一次，`preview_order()` / `create_order()` / `errand_quote()` / `errand_create()` / `errand_add_tip()` / `errand_cancel()` 不重试。

**② hub 金额闸**（新，**只从外卖 `create_order()` 出现，跑腿没有**）——不在 `error` 字段里，是整个返回体：
```json
{
  "need_confirmation": true,
  "reason": "amount_over_limit",
  "payable_price": 12800,
  "limit_cents": 5000,
  "hub_confirm_token": "hc_…",
  "expires_in": 600
}
```
`reason` 的三个取值处理方式不同：

| `reason` | 含义 | 怎么办 |
|---|---|---|
| `amount_over_limit` | 这笔金额超过平台设置的上限，第一次拦截，正常流程 | 把 `payable_price` 和 `limit_cents`（都是分，换算成元）如实告诉用户，明确问一句"金额超出限制，确认要继续下单吗"；拿到明确同意后，把 `hub_confirm_token` **原样**加进参数，`preview_id` / `confirmation_token` 不变，其余参数也不变，再调一次 `create_order()` |
| `amount_unknown` | 引用的 `preview_id` 在 hub 这边找不到记录（多半是隔太久，或者用错了 preview_id） | 这次的 `hub_confirm_token` 没意义，**不要**直接拿去重调；回到 Step 5 重新 `preview_order()` 换一套新的 `preview_id` + `confirmation_token`，再走一次确认流程 |
| `confirm_token_invalid` | 带的 `hub_confirm_token` 对不上、过期或被改过 | 同上：不要重发这个 token，重新 `preview_order()` 换新的一套再来 |

带 `hub_confirm_token` 重调 `create_order()` 时，**`confirmation_token` 这个字段依然要带、不能删**——两个 token 各管各的，一个都不能少。

**跑腿 `errand_create()` / `errand_add_tip()` 不会返回这个形状**（D4：第一版最小闭环不为跑腿建金额闸对子）——金额只信 `errand_quote()` 的 `fee` 或调用方自己传的 `tip_fee`，下单/加小费前的显式确认必须你自己在对话里做完，不能指望服务端再拦一次。

**③ 上游业务错误（透传，形状不变）**：
```json
{"error": {"code": "PUBLIC_REFERENCE_INVALID", "message": "…", "request_id": "req_xxx"}}
```
这个容器形状外卖/跑腿通用，但 `code` 的具体取值来自各自独立的上游——外卖走下面「外卖上游业务硬失败」那张表（37 码体系），跑腿走「跑腿（errand）相关码与行为」，**两套码不能混用**。

**这一版看不到、也不会遇到**跟绑定/登录相关的旧错误码（原来的 `CONSENT_GRANT_INVALID` / `CONSENT_GRANT_EXPIRED` / `SMS_CODE_INVALID` / `ELEME_USER_NOT_FOUND` / `SMS_COOLDOWN` / `SMS_SEND_FAILED`，以及 `get_homepage_url()` 专属的 `HOMEPAGE_URL_CONFIG_MISSING`，跑腿侧同理 `errand_request_user_bind()` 等四个隐藏工具对应的旧码）——那些都需要调用现在已经隐藏的绑定工具才会触发，这一版全部收敛成上面①的 `consent_required`，遇到授权问题只会看到这一个码。

## 软失败（只有 `quote_cart()`，外卖）

正常返回、价格照给，但 `can_checkout=false`：

| `blocking_code` | 含义 | 处理 |
|---|---|---|
| 2 | 店铺必选分类未选（汤底 / 主食） | 按 `required_groups` 让用户补选，重新 quote |
| 1 | 该商品「单点不送」 | 加别的商品凑单，重新 quote |
| 0 | 其余（未达起送 / 售罄 / 规格冲突 / 未达起购…） | `minimum_order_gap`>0 → 差这么多分起送；否则直接展示 `blocking_reason` |

同一问题到 `preview_order()` 会变成硬失败（`MISSING_REQUIRED_SELECTION` / `BELOW_MIN_PURCHASE` / `ORDER_FAILED`）。**能在 quote 前拦的（营业状态、售罄、起购、规格冲突）都有字段，别拖到下单才发现。** 跑腿没有对应的软失败层，`errand_quote()` 失败即失败，按下面的码/行为表处理。

## 外卖上游业务硬失败：按怎么处理分流

| 类型 | code | 怎么办 |
|---|---|---|
| **需要用户参与** | `ADDRESS_CHANGED` `ADDRESS_RESOLUTION_FAILED` | 地址在 App 里被改/删了，让用户重选地址 |
| | `ADDRESS_CREATE_REJECTED` | 上游拒绝这条地址，换一条 |
| **换引用后重试** | `PUBLIC_REFERENCE_INVALID` | 一码多因：message 说是 cart / quote / preview / token / coupon / address 哪个，照它重新获取 |
| | `IDEMPOTENCY_CONFLICT` | 同一 `confirmation_token` 改了内容再用，重新 `preview_order()` 拿新 token |
| | `SUGGESTION_EXPIRED` | 地址候选 token 过期，重新 `search_addresses()` |
| **修参后重试** | `CART_PRICE_INVALID` | 规格入参不合法（单选组选多项、份数超上限、冲突），按 message 修 |
| | `MISSING_REQUIRED_SELECTION` | 店铺必选组没选，回菜单补 |
| | `BELOW_MIN_PURCHASE` | 数量提到 `min_purchase` 以上 |
| | `COORDS_REQUIRED` `CONTACT_REQUIRED` `DETAIL_REQUIRED` `ADDRESS_CONTACT_NAME_TOO_LONG`(>12 字) | 补齐/修正入参 |
| **退避** | `RATE_LIMITED` | 等一会，告诉用户稍等 |
> ℹ️ 下表里像 `ELEME_ERROR` 这种带服务商前缀的码，是**上游原样返回的字面量**，不是我们起的名。
> 它们保留原样是因为 agent 要靠它做分支判断——改写了这份文档就会和真实返回对不上。
> （我们自己写的文字一律不出现服务商名。）

| **上游瞬时故障** | `ELEME_ERROR` `PROVIDER_NOT_AVAILABLE` `SIGN_ADDRESS_NOT_FOUND` | 读操作可退避后重试一次；**preview / create 不重试** |
| **下单失败** | `ORDER_FAILED` | message 有原因（售罄 / 预览过期…）。可能没下成也可能已下成 → 先 `list_orders()` 看，**不自动重下** |
| **hub / 网关侧配置问题，别重试** | `AUTH_REQUIRED` `AUTH_INVALID` `CAP_NOT_BOUND` `BINDING_LIMIT_REACHED` `DEVICE_ID_REQUIRED` | 这是 hub 或网关那一侧的配置/配额问题，不是用户能自己解决的（这一版没有用户可填的 Key）；如实告知暂时无法完成，附 `request_id` 反馈，不要引导用户去改设置 |
| **网关内部** | `INTERNAL_ERROR` `NOT_FOUND` `METHOD_NOT_ALLOWED` | 带 request_id 反馈；不要编原因 |

## 跑腿（errand）相关码与行为

> 下面这些不在外卖那 37 个码的体系里——跑腿走独立的履约上游。目前跑腿工具的说明原文里
> 只正式点名了 `GOODS_PROHIBITED` 这一个 code；其余是说明文字里点出的失败**行为**，没有配专门的 code 名，
> 先按行为描述处理，**不要自己编码名**。

| code | 含义 | 怎么办 |
|---|---|---|
| `GOODS_PROHIBITED` | `errand_quote()` 的 `goods[].name` 或 `remark` 命中禁运校验（毒品/枪爆/危化品/违法交易物），message 会点名是哪个词、属哪一类 | 如实告知用户该物品无法配送，**不要改词重试绕过**；若怀疑是正常商品被误判（如含敏感字的合法商品名），换个更准确的名称重新 `errand_quote()` |

**未命名但要处理的失败行为**（均来自工具说明原文，未见专门 code）：

| 现象 | 触发点 | 处理 |
|---|---|---|
| `quote_id` 超过 `errand_quote()` 返回的 `expires_in_seconds` 还没 `errand_create()` | `errand_create()` | 重新 `errand_quote()` 换一套新的 `quote_id` + 报价集再下单，不要拿旧 `quote_id` 硬试 |
| `scheduled_at` 不是 `errand_list_schedule_slots()` 给出的档位值、传成秒级而非毫秒、早于当前时间或提前超过 1 天 | `errand_quote()` | 400 拒绝；**只把清单里的 value 原样回填**，别自己算、别复用旧单的这个字段（旧单的时间早过了会被拒） |
| `errand_list_orders()` 的 `status` 传了不在允许清单内的值 | `errand_list_orders()` | 报错，不会静默返回空列表；传前核对可选值（pending_payment / pending / dispatching / waiting_rider / rider_reassigning / rider_accepted / rider_arrived / delivering / completed / cancelled / failed / dispatch_failed） |
| 地址不存在或不属于当前用户 | `errand_update_address()` `errand_delete_address()` | 报错；改址前先 `errand_list_addresses()` 核对是哪一条 |
| 骑手已接单后取消，可能有违约金 | `errand_cancel()` | 先 `errand_pre_cancel()` 查 `cancel_fee`/`refund_amount`，如实告诉用户再确认取消 |

## 下单失败的提前判据（在更早的接口上就能看到，外卖）

| 现象 | 最早能发现的工具 | 字段 |
|---|---|---|
| 店打烊 / 超出配送范围 | `search_shops()` | `available=false` + `unavailable_reason` |
| 商品售罄 | `get_shop_menu()` `get_item_options()` | `available=false`；`available_quantity`（0=无货） |
| 未达起送价 | `quote_cart()` | `minimum_order_gap`（分） |
| 单点不送 | `get_item_options()`（promo_labels）/ `quote_cart()` | `blocking_code=1` |
| 必选分类没选 | `get_shop_menu()` | `required_groups` |
| 未达起购份数 | `get_shop_menu()` `get_item_options()` | `min_purchase` |
| 规格互相冲突 | `get_item_options()` | 选项 `available=false` + 冲突说明 |
