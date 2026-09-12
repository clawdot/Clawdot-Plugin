# ClawDot Hub · Claude Code 插件

点外卖、叫跑腿。**登录授权即用，不用手填 API Key。**

## 装

```bash
/plugin marketplace add <本仓地址>
/plugin install clawdot-hub@clawdot
```

## 授权（装完还要这一步）

装上**不等于连上**。首次用到它时服务端会回 401，Claude Code 把这个 server 标成「需要授权」，
你要自己打开 `/mcp` 面板，跟着浏览器走一遍手机号 + 验证码。

```
/mcp        # 在里面对 clawdot-hub 完成授权
```

授权一次，票 1 小时有效、自动续期，不用反复登录。

## ⚠️ 装之前先看这条

**如果你以前手动 `claude mcp add` 过同名的 server，插件里的配置会被完全遮蔽，而且不会报错。**

Claude Code 的优先级是 Local > Project > User > **Plugin**；同名时高优先级那份**整条**生效，
字段不合并。所以你会连到旧地址上，却看不到任何提示。

装/升级前先清掉手动配置：

```bash
claude mcp list                 # 看有没有同名的
claude mcp remove clawdot-hub   # 有就先删掉
```

## 装了什么

| | |
|---|---|
| `.mcp.json` | 远程 MCP server（`https://hub.clawdot.net/mcp`，OAuth 2.1） |
| `skills/takeout-order/` | 怎么用这套能力的说明书：下单前的硬规则、7 步流程、工具表、错误码表 |

工具本身由 server 提供，装插件不会把工具塞进你本地。

## 不带 skill 只要连接

不想要说明书、只想连上：

```bash
claude mcp add --transport http clawdot-hub https://hub.clawdot.net/mcp
```

然后 `/mcp` 授权。能力一样，只是模型少了那份怎么用的指引。
