---
name: local-life-retail
description: 帮用户买日用百货（零售），连的是带 create_task、retail_browse 的那组工具（先开 task，再一组一组往下走）时用。讲每一单怎么走、什么算同意下单、一笔只买一个规格、下单前改主意怎么回退、两单怎么确认、用户说付好了怎么办、金额变了怎么办、地址和手机号怎么说、退货退款怎么办。触发：买纸巾 / 买洗衣液 / 帮我买个 X（日用品、百货）/ 我买的东西到哪了 / 要退货、退款。
description_zh: 登录就能用：搜商品、挑规格，按真实价格出确认页，你点头才建收款单、由你自己付款；能查付款和物流，也能退货退款。
description_en: Sign in and buy daily essentials: search products and pick a spec, see the real price on a confirmation page; a payment link is created only after you confirm, and you pay it yourself; track payment and shipping, or request a refund.
version: 0.1.0
author: ClawDot
---

# 本地生活任务（零售）

这组工具按「一单一个 task」走：先开 task，之后每一步都照返回里的下一步去调。
每个参数什么意思、哪一步会被拒，工具说明和返回里都有，这里不重复；这里只讲工具判断不了、要你来判断的事：用户到底想要什么、几个 task 怎么记、对用户怎么说。

## 1. 怎么走

- 每一单先调 `create_task`，只带 `idempotency_key`，不用带 `capability`。
- `idempotency_key`（`create_task`、`fork_task` 都要带）：每开一单生成一个随机 UUID，只在同一次调用超时重试时带同一个；不要按内容、日期、序号起名，也不要沿用别的单用过的键。
- 刚开的 task 还没定办什么，返回里的 `next_action.summary` 列了各业务的第一步：买日用百货从 `retail_browse` · `search` 开始，退货退款从 `retail_aftersale` · `locate` 开始。第一次调了哪个业务，这个 task 就归哪个业务，之后不能再调别的业务；用户中途又要办另一件事，用新的 `idempotency_key` 另开一个 task。
- 之后照返回里的 `next_action.tool`、`next_action.action` 调，每次都带 `task_id`。`next_action.options` 原样摆给用户挑，用户挑中的那一项的 `ref` 原样回传，不改、不自己拼。
- 要下单，用确认页那一项的 `ref` 当 `confirm_ref`，而且要等用户同意之后。什么算同意下单：
  - 只有用户**看过这张确认页的实付之后**、针对它说「下单 / 确认 / 就这样」才算同意。
  - 在确认页出来之前说的「结算 / 买单 / 下单吧」，只表示「出确认页给我看」：调 `retail_checkout` · `preview` 之后，先把商品、规格、数量、收货地址、实付摆给用户，问一句「确认下单吗？」，等用户回复再调 `retail_place_order`。
  - 「继续」「往下走」「你直接帮我选」都不算同意下单。
  - 用户回的是问题或别的事，先答，再问一次要不要下单。

- 收款单建好以后没有取消：没付的不用管，到期自动失效；付了想退，另开一个 task 走退货退款。

- 用户问订单（「看看我之前的单」「上一单到哪了」），任何时候都能查，不用管 task 走到哪一步：在手头的 task 里（刚开的也行）调 `retail_fulfillment` · `list` 列最近的单，要看某一单就带它的 `order_id` 调 `status`（看物流调 `logistics`）。查单不会清掉已选的东西，查完接着原来那一步走。

## 2. 一个 task = 一样商品 = 一单

- 一个 task 只买一个规格（同一个规格可以买几件），最后只成一单。两样商品就是两个 task、两单、两次付款。
- 每开一个 task 都算一次费。不要为了「先看看」去开 task。

## 3. 下单前改主意

- 有两种回退，按用户的意思选：
  - 只想改一项：在原 task 里重调定这一项的那一步，比如换规格先用 `retail_browse` · `remove` 清掉再选，改数量重调 `retail_browse` · `select`，换地址重调 `retail_address` · `select`。不开新 task、不多计费。
  - 想留着现在这一版、另试一版（「先别动这个，再配一份看看」）：对原 task 调 `fork_task`，带新的 `idempotency_key`。新 task 带着商品、规格、数量和收货地址，原 task 原样留着，之后哪一版都能接着走到下单。fork 会多计一次费；确认页不带过去，新 task 里要重新预览。
- 有的改动会清掉后面已经选好的东西，比如换规格、换地址都会让确认页作废。哪一步清什么见 @references/rollback.md；会清掉用户已经挑好的东西时，先跟用户说清楚再调。
- 下了单就回不去了：再要一单调 `create_task` 新开；「照上一单再来一份」用 `fork_task`。两个都要带这一单新的 `idempotency_key`，超时重试带同一个。还没开始办事的 task、下单结果不明的 task 不能 fork。

## 4. 付款

- 用户说「付好了」，先调 `retail_fulfillment` · `status` 查一次再回话，不要直接当成已付。
- 建收款单时金额变了，要等用户对新金额点了头，才再确认一次。

## 5. 两单怎么确认

- 两样商品的规格、地址都选好以后，挨着各自预览（`retail_checkout` · `preview`），把两张确认页摆在同一屏：各写商品、规格、实付多少，讲明是两份收款单、两次付款。
- 用户明确说两单都下，才两单一起下，各用各的确认页那一项的 `ref`；否则一单一单问、一单一单下。
- 两单来回切，找不准某一单的确认页了，就对那个 `task_id` 调 `get_task` 取回；确认页已经过期，就重新预览那一单。

## 6. 买日用百货时怎么问、怎么摆

- **地址**：已存地址只有一个，直接选，到确认页时说一句送到哪；有好几个，摆出来让用户挑，不替用户选。
- **手机号**：只用来填收货地址；不复述完整号码，要提就说返回里打码后的那串。

## 7. 退货退款

- 退款说明、退货运单号只用用户自己说的：要求写说明的原因，请用户说一句，原样带上，不替用户写、不改写；运单号要用户给的真实单号。
- 申请退款、回传运单、撤销售后都和下单一样，先出确认页、等用户同意再提交；撤销前先讲明撤了钱不退。
- 进度以 `retail_aftersale` · `progress` 查到的为准：审核通过不等于钱到账，到账了才说退了多少。

## 8. 对用户说话

- 不说工具名、`task_id`、`ref` 这些；说商品名、规格、价钱。
- 两单就说两单（「纸巾一单、洗衣液一单，分开付」），不要说成一单，也不要只报一个合计让用户以为只付一次。
- 金额只用确认页上的实付，不自己估。
