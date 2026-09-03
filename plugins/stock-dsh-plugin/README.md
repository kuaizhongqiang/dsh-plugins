# Stock 插件（tool-stock）

A 股行情分析 / 舆情 / 交易建议 / 模拟盘插件。数据仅来自腾讯公开行情接口
（无需 API key），技术指标（MA/MACD/RSI/KDJ/ATR）插件内零依赖计算。

## 工具

| 工具 | 作用 |
|---|---|
| `stock_quote` | 实时行情（价格/涨跌/换手/PE/PB/市值/五档） |
| `stock_kline` | 日 K（前复权，本地缓存） |
| `stock_indicators` | MA / 量均线 / MACD / RSI / KDJ / ATR |
| `stock_market_overview` | 上证/深成/创业板/沪深300 快照 |
| `watchlist_add` / `watchlist_remove` / `watchlist_list` | 自选股管理 |
| `stock_daily_collect` | 收集当日自选股+指数快照（每日幂等） |
| `stock_report` | 生成个股 Markdown 报告骨架 |
| `sentiment_sources` | 舆情权威信息源白名单 |
| `sentiment_pick` | 按当日异动+技术信号挑需调查的候选（≤5 只） |
| `sentiment_record` / `sentiment_list` | 舆情结论持久化与回顾 |
| `advice_calc` | 按 ATR/均线/支撑压力/风险偏好计算建议价（触发/目标/止损/仓位） |
| `position_record` / `position_list` / `position_update` | 交易建议（挂单/预测）记录 |
| `paper_init` / `paper_account` / `paper_execute_advice` / `paper_trade` | 模拟盘账户与成交 |
| `paper_settle` | T+1 验单结算（建议即挂单） |

## 时间模型（时间周期为日）

每条建议自动标注产生时段 `phase`（`pre_market` 盘前 / `intraday` 盘中 /
`after_hours` 盘后）与数据基准日 `dataDate`：

- **T0 盘后**（交易日 15:00 后，含晚间/周末/节假日）：预测基于 T0 及以前
  全部已收盘数据；挂单最早只能在 T+1（下一交易日）成交。
- **T0 盘中**（9:30–15:00 未收盘）：预测**忽略 T0 当日数据**（当日 K 线未
  走完），按 T-1 及以前已收盘数据分析与挂单。
- `advice_calc` 自动遵守该规则：盘中调用时只用 T-1 收盘数据定档价位。
- `paper_settle` 统一取「建议日期之后第一个交易日」的最高/最低价区间判定
  挂单是否成交（成交按挂单价记账），绝不用建议日当天区间（避免马后炮）。

用户数据在 `%DSH_HOME%\stock\`（watchlist.json / kline-cache.json /
daily/YYYY-MM-DD.json / reports/*.md / sentiment.json / positions.json /
paper.json）。

## 安装

```powershell
powershell -ExecutionPolicy Bypass -File .\install.ps1
```

脚本幂等：复制插件（打印版本）→ `cordis.patch.yml` 追加 `tool-stock` 条目。
重启 `dsh web` 后即可在对话里使用。

## 配置（可选，cordis.patch.yml）

```yaml
- insert:
    - id: tool-stock
      name: './plugins/stock/index.js'
      config:
        klineDays: 150
        dataRoot: C:/path/to/stock
        timeoutMs: 15000
```

## 卸载

1. 编辑 `%DSH_HOME%\profiles\web\cordis.patch.yml`，删除 `tool-stock` 条目
2. 删除 `%DSH_HOME%\profiles\web\plugins\stock\`
3. 重启 `dsh web`（不删除 `%DSH_HOME%\stock\` 用户数据）
