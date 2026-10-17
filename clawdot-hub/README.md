# ClawDot Hub · Claude Code 插件

点外卖、叫跑腿：连的是任务接口（`/mcp/task`），外卖、跑腿两份说明书一起装。**登录授权即用，不用手填 API Key。**

## 装

```bash
/plugin marketplace add <本仓地址>
/plugin install clawdot-hub@clawdot
```

## 连接和授权（装完还要这一步）

装上**不等于连上**。首次用到它时服务端会回 401，Claude Code 把这个 server 标成「需要授权」，
你要自己打开 `/mcp` 面板，选 `clawdot-task`，跟着浏览器走一遍连接：登录（或注册）开发者账号 → 选一个专用 agent（或新建一个）。这一步不填手机号。

```
/mcp        # 在里面对 clawdot-task 完成授权
```

连上之后，第一次用外卖、跑腿时，AI 会先在对话里问你的手机号，再给你一条授权链接：打开，用同一个手机号完成授权 → 回到对话说一声，它下一次调用时自动核验、接着往下办，不用回别的页面点确认。
连接一次，票 1 小时有效、自动续期，不用反复登录；同一个专用 agent 上连的其他客户端也不用再授权。
一个专用 agent 对应一个手机号，要换手机号就重新连接、新建一个专用 agent（免费档最多 3 个 agent，注册时送 1 个，还能建 2 个专用的）。

## ⚠️ 装之前先看这条

**同一个 Claude Code 里只留一个连接。** 手动加过别的名字或地址的（比如连 `/mcp` 的 `clawdot-hub`），先删掉，不然两个连接并存：

```bash
claude mcp list                 # 看有哪些
claude mcp remove clawdot-hub   # 手动加过的才要删；插件自带的那条随插件升级自动换掉
```

## 装了什么

| | |
|---|---|
| `.mcp.json` | 远程 MCP server `clawdot-task`（任务接口 `https://hub.clawdot.net/mcp/task`，OAuth 2.1） |
| `skills/local-life-food/` | 点外卖怎么走：一单一个 task、什么算同意下单、改主意怎么回退、两单怎么确认、怎么问怎么摆 |
| `skills/local-life-errand/` | 叫跑腿怎么走：同上的共用规则，加上挑运力、退单、查单看骑手 |

工具本身由 server 提供，装插件不会把工具塞进你本地。

## 不带 skill 只要连接

不想要说明书、只想连上：

```bash
claude mcp add --transport http clawdot-task https://hub.clawdot.net/mcp/task
```

然后 `/mcp` 授权。能力一样，只是模型少了那份怎么用的指引。
