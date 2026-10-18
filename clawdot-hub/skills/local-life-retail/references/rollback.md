# 改主意会清掉什么

下单前改前面某一项时，下面这些动作会清掉后面已经选好的东西，清掉的要重新选、重新加；表里没列的动作不清东西。
确认页清掉以后，重新预览，就会出新的。
重选**同一个**地址什么都不清。

## 零售

| 想改什么 | 调哪个 | 会清掉 |
|---|---|---|
| 换规格、改数量 | `retail_browse` · `select` / `remove` | 确认页 |
| 换收货地址、新增或改地址 | `retail_address` · `select` / `save` | 确认页 |
| 重新预览 | `retail_checkout` · `preview` | 确认页 |
| 搜商品、看规格 | `retail_browse` · `search` / `item` | 不清 |
| 列已存地址、查区划 | `retail_address` · `search` / `divisions` | 不清 |
