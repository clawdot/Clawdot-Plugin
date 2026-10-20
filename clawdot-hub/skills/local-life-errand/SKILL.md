---
name: local-life-errand
description: 用户要叫跑腿（同城取件、送件）时用，还没连上任务接口也用（正文开头讲怎么先帮用户连上）。讲每一单怎么走、什么算同意下单、下单前改主意怎么回退、两单怎么确认、要退一单跑腿怎么办、查单和看骑手。触发：叫跑腿 / 帮我寄个 X / 帮我送个东西过去 / 帮我取个 X / 跑腿到哪了 / 骑手在哪 / 取消跑腿。
description_zh: 登录就能用：帮你叫同城跑腿取件、送件，询价后挑运力，你点头才下单；能查进度、看骑手位置，也能取消。
description_en: "Sign in and book a same-city errand run: set pickup and drop-off, compare carriers by quote, and the order is placed only after you confirm; track progress, see the rider, or cancel."
version: 0.2.2
author: ClawDot
---

# 本地生活任务（跑腿）

### 先连上任务接口

这份说明配的是任务接口上的一组工具，由 `create_task` 起头。先看工具列表里有没有 `create_task`（有的客户端要先搜工具才加载出来，先搜一次）：有，就是已经连上了，直接用，不要因为客户端里别的连接显示要授权就让用户去授权；搜了也调不到 `create_task`，才是这个客户端还没连上任务接口：先停下来帮用户连上；没连上之前什么都查不到、也下不了单，不要自己编结果。各客户端的连法都一样：

- 任务接口的地址：https://hub.clawdot.net/mcp/task
- 在客户端里添加一个远程 MCP 连接，地址填上面这个（各客户端在哪里添加，登录 https://hub.clawdot.net 后在「接入」里按平台查）。连接时会打开连接页：登录（或注册）开发者账号，选一个专用 agent（或新建一个），回到客户端就连上了。
- 客户端打不开浏览器（自建 agent、没有屏幕的设备）：走设备码授权，用户在任意一台能上网的设备上打开链接、输入码，同样登录开发者账号、选专用 agent；这条要先由虾点建档、拿到 client_id。
- 自己的服务端对服务端接入：用 API Key 接任务接口，每个请求带上自己这边的用户编号；API Key 由虾点发放（在控制台申请）。
- 连上以后工具列表里就有 `create_task` 了，照下面「怎么走」办。

这组工具按「一单一个 task」走：先开 task，之后每一步都照返回里的下一步去调。
每个参数什么意思、哪一步会被拒，工具说明和返回里都有，这里不重复；这里只讲工具判断不了、要你来判断的事：用户到底想要什么、几个 task 怎么记、对用户怎么说。

## 1. 怎么走

- 每一单先调 `create_task`，只带 `idempotency_key` 和 `context`，不用带 `capability`。这一次和之后每次调 `create_task`、`update_task` 与各组工具，都在 `context` 的 `utterance` 里放用户这一轮说的原话，逐字照抄，不改写、不总结；这一步用户没说新话（你自己往下走）就带用户最近说的那一句。
- `idempotency_key`（`create_task`、`fork_task` 都要带）：每开一单生成一个随机 UUID，只在同一次调用超时重试时带同一个；不要按内容、日期、序号起名，也不要沿用别的单用过的键。
- 刚开的 task 还没定办什么，返回里的 `next_action.summary` 列了各业务的第一步：叫跑腿从 `errand_address` · `search` 开始。第一次调了哪个业务，这个 task 就归哪个业务，之后不能再调别的业务；用户中途又要办另一件事，用新的 `idempotency_key` 另开一个 task。
- 返回的 `next_action.tool` 是 `errand_authorize`，说明这位用户还没授权：照它调、`action` 用 `start`；回来要手机号就问用户一次，带在 `context.phone` 里再调 `start`，把拿到的授权链接原样发给用户；用户说做完了，同一个工具 `action` 用 `status`（只查、不重发），完成了就照它回的 `next_action` 往下走，没完成就把回来的话转给用户。办事中途任何一步回 consent_required，照返回的话走。
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

## 6. 用户的记忆

- 用户说「记住 / 忘掉 / 你记了我什么 / 给我个链接看看」，说的是下单用的口味、忌口、过敏、家人、常用地点：调 `memory` 工具（不挂 task），不要记进客户端自己的记忆或笔记。
- 记过敏、家人要用户明确同意：用户只是顺口提到（「对了我对虾也过敏」），先问一句要不要记住，同意了才记。
