---
name: local-life-errand
description: 帮用户叫跑腿（同城取件、送件），连的是带 create_task、errand_address 的那组工具（先开 task，再一组一组往下走）时用。讲每一单怎么走、什么算同意下单、下单前改主意怎么回退、两单怎么确认、要退一单跑腿怎么办、查单和看骑手。触发：叫跑腿 / 帮我寄个 X / 帮我送个东西过去 / 帮我取个 X / 跑腿到哪了 / 骑手在哪 / 取消跑腿。
description_zh: 登录就能用：帮你叫同城跑腿取件、送件，询价后挑运力，你点头才下单；能查进度、看骑手位置，也能取消。
description_en: Sign in and book a same-city errand run: set pickup and drop-off, compare carriers by quote, and the order is placed only after you confirm; track progress, see the rider, or cancel.
version: 0.2.0
author: ClawDot
---

# 本地生活任务（跑腿）

这组工具按「一单一个 task」走：先开 task，之后每一步都照返回里的下一步去调。
每个参数什么意思、哪一步会被拒，工具说明和返回里都有，这里不重复；这里只讲工具判断不了、要你来判断的事：用户到底想要什么、几个 task 怎么记、对用户怎么说。

## 1. 怎么走

- 每一单先调 `create_task`，只带 `idempotency_key`，不用带 `capability`。
- `idempotency_key`（`create_task`、`fork_task` 都要带）：每开一单生成一个随机 UUID，只在同一次调用超时重试时带同一个；不要按内容、日期、序号起名，也不要沿用别的单用过的键。
- 刚开的 task 还没定办什么，返回里的 `next_action.summary` 列了各业务的第一步：叫跑腿从 `errand_address` · `search` 开始。第一次调了哪个业务，这个 task 就归哪个业务，之后不能再调别的业务；用户中途又要办另一件事，用新的 `idempotency_key` 另开一个 task。
- 之后照返回里的 `next_action.tool`、`next_action.action` 调，每次都带 `task_id`。`next_action.options` 原样摆给用户挑，用户挑中的那一项的 `ref` 原样回传，不改、不自己拼。
- 已经往后走了（询了价）又回头搜地址，只算「只是看看」：已选的东西一样不动，`next_action` 仍指着原来那一步（比如 `errand_goods` · `choose`），搜到的候选不在 `next_action.options`，在 `result.candidates`。摆给用户看就好；用户要换，再拿其中一项的 `ref` 调 `select`，`summary` 里写了换了会清掉什么。重选同一个地址什么都不清。
- 要下单，用确认页那一项的 `ref` 当 `confirm_ref`，而且要等用户同意之后。什么算同意下单：
  - 只有用户**看过这张确认页的实付之后**、针对它说「下单 / 确认 / 就这样」才算同意。
  - 在确认页出来之前说的「结算 / 买单 / 下单吧」，只表示「出确认页给我看」：让用户挑好运力、调 `errand_goods` · `choose` 之后，先把运力、取件和送达地址、实付摆给用户，问一句「确认下单吗？」，等用户回复再调 `errand_place_order`。
  - 「继续」「往下走」「你直接帮我选」都不算同意下单。
  - 用户回的是问题或别的事，先答，再问一次要不要下单。

- 用户要退一单跑腿：这单是在当前 task 里下的，就在这个 task 里调 `errand_fulfillment` · `pre_cancel`。不是的（新开的对话、别处下的单），新开一个 task，带上订单号 `order_id` 调 `errand_fulfillment` · `pre_cancel`，之后照返回往下走。被拒、说这单在另一个 task 里时，返回带着那个 task 的 `holder_task_id`，到那个 task 里办。

- 用户问订单（「看看我之前的单」「上一单到哪了」），任何时候都能查，不用管 task 走到哪一步：在手头的 task 里（刚开的也行）调 `errand_fulfillment` · `list` 列最近的单，要看某一单就带它的 `order_id` 调 `status`（看骑手调 `rider`）。查单不会清掉已选的东西，查完接着原来那一步走。

## 2. 一个 task = 一趟跑腿 = 一单

- 一个 task 只办一趟跑腿，最后只成一单。两趟跑腿就是两个 task、两单、两次付款。
- 每开一个 task 都算一次费。不要为了「先看看」去开 task。

## 3. 下单前改主意

- 有两种回退，按用户的意思选：
  - 只想改一项：在原 task 里重调定这一项的那一步，比如换取件或送达地址重调 `errand_address` · `select`，改物品重调 `errand_goods` · `quote`。不开新 task、不多计费。
  - 想留着现在这一版、另试一版（「先别动这个，再配一份看看」）：对原 task 调 `fork_task`，带新的 `idempotency_key`。新 task 带着两端地址，原 task 原样留着，之后哪一版都能接着走到下单。fork 会多计一次费；报价不带过去，新 task 里要重新询价。
- 有的改动会清掉后面已经选好的东西，比如换地址会清掉报价。哪一步清什么见 @references/rollback.md；会清掉用户已经挑好的东西时，先跟用户说清楚再调。
- 下了单就回不去了：再要一单调 `create_task` 新开；「照上一单再来一份」用 `fork_task`。两个都要带这一单新的 `idempotency_key`，超时重试带同一个。还没开始办事的 task、下单结果不明的 task 不能 fork。

## 4. 两单怎么确认

- 两趟的地址、物品都定好以后，挨着各自询价、请用户挑运力（`errand_goods` · `choose`），把两张确认页摆在同一屏：各写运力、取送地址、实付多少，讲明是两份跑腿费、两次付款。
- 用户明确说两单都下，才两单一起下，各用各的确认页那一项的 `ref`；否则一单一单问、一单一单下。
- 两单来回切，找不准某一单的确认页了，就对那个 `task_id` 调 `get_task` 取回；确认页已经过期，就重新询价那一单。

## 5. 对用户说话

- 不说工具名、`task_id`、`ref` 这些；说地址、物品、价钱。
- 两单就说两单（「第一趟一单、第二趟一单，分开付」），不要说成一单，也不要只报一个合计让用户以为只付一次。
- 金额只用确认页上的实付，不自己估。
