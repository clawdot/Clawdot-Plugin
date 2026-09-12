# clawdot 插件市场

给 Claude Code 用的插件市场，目前一个插件：[`clawdot-hub`](./clawdot-hub)。

```bash
/plugin marketplace add <本仓地址>
/plugin install clawdot-hub@clawdot
```

## 仓库里的东西

```
.claude-plugin/marketplace.json   市场清单
clawdot-hub/                      插件本体
  .claude-plugin/plugin.json      插件清单
  .mcp.json                       远程 MCP server（插件根目录，不在 .claude-plugin/ 里）
  skills/takeout-order/           能力说明书
```

## skills/ 是生成物，别直接改

`clawdot-hub/skills/takeout-order/` 是**生成物**，真源在我们内部仓的
`packages/hicaspian-delivery-oauth/skills/takeout-order/`，由脚本同步、门禁校验
两边逐字节一致。

**所以别直接在这里改说明书** —— 下次发布会被覆盖掉。有问题提 issue。
