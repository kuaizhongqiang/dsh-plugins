# stock analysis tools for native dsh

A 股（沪深）行情分析与自选股管理插件，15 个工具全部**仅依赖腾讯公开行情接口**
（无需任何 API key），技术指标在插件内零依赖计算，主模型负责解读与报告撰写。

## 包里有什么

```
stock-dsh-plugin/
├── install.ps1                    一键安装脚本（幂等，可重复执行）
├── README.md                      本说明
└── plugins/
    └── stock/
        ├── index.js               股票工具插件（自包含，仅依赖 npm 版已有的
        │                          dsh-tools / schemastery）
        └── package.json           ESM 声明（消除 MODULE_TYPELESS 警告）
```

> 数据源为腾讯公开接口，不需要 `.env.example`（无密钥）。

## 前置要求（目标电脑）

- Node.js（dsh 要求 `^22.19 || >=24`）
- dsh npm 版已安装：`npm install -g @deepseek-ai/dsh`
- 至少启动过一次 `dsh web`，让 `profiles/web` 目录初始化

## 安装（目标电脑上）

```powershell
cd stock-dsh-plugin
powershell -ExecutionPolicy Bypass -File .\install.ps1
```

脚本会：
1. 把 `plugins/stock` 复制到 `%DSH_HOME%\profiles\web\plugins\stock`
2. 幂等地在 `cordis.patch.yml` 追加 `- insert:` 挂载条目（已存在则跳过）

## 工具清单

| 工具 | 作用 |
|------|------|
| `stock_quote(code)` | 实时报价（现价/涨跌/量额/换手/PE/PB/市值/五档），支持 `600519` / `sh600519` / `600519.SH` |
| `stock_kline(code, days?)` | 日K（前复权），默认最近 60 根，缓存 150 交易日按日刷新 |
| `stock_indicators(code)` | 技术指标最新值：MA(5/10/20/60)、均量(5/10)、MACD(12,26,9)、RSI(6/12/24)、KDJ(9,3,3)、ATR(14) |
| `stock_market_overview()` | 四大指数快照（上证/深成/创业板/沪深300） |
| `watchlist_add(code)` / `watchlist_remove(code)` / `watchlist_list()` | 自选股管理（JSON 持久化） |
| `stock_daily_collect(force?)` | 手动触发每日收集：自选股收盘快照+指标+指数 → `daily/YYYY-MM-DD.json`（当日幂等，force 可覆盖） |
| `stock_report(codes, output_path?)` | 生成 Markdown 报告骨架（行情表+指标表）→ `reports/*.md`，模型补充分析 |
| `sentiment_sources()` / `sentiment_pick(limit?)` / `sentiment_record(...)` / `sentiment_list(date?)` | 舆情分析白名单/候选挑选/结论记录/历史回顾（每日≤5只） |
| `advice_calc(code, action?, risk_profile?)` | 按 ATR/均线/支撑压力计算建议触发价/目标价/止损价/仓位 |
| `position_record(...)` / `position_list(status?)` / `position_update(id, status)` | 交易建议台账（pending→executed/cancelled） |
| `paper_init(initial_cash?, force?)` / `paper_account()` / `paper_execute_advice(id, ...)` / `paper_trade(code, action, shares, ...)` | 模拟盘（10万本金"建议即操作"）：开户/账户/建议执行成交/手工调仓 |

## 数据与存储

- 行情：`https://qt.gtimg.cn/q=<symbol>`（GBK 编码，插件内解码）
- 日K：`https://web.ifzq.gtimg.cn/appstock/app/fqkline/get`（前复权）
- 限速：请求间隔 ≥ 250ms；日K 按代码按日缓存，避免重复全量拉取
- 用户数据：`%DSH_HOME%\stock\`（`watchlist.json` · `kline-cache.json` ·
  `daily/YYYY-MM-DD.json` · `reports/*.md` · `sentiment.json` ·
  `positions.json` · `paper.json`）

## 可选配置

`cordis.patch.yml` 条目里可覆盖：

```yaml
- insert:
    - id: tool-stock
      name: './plugins/stock/index.js'
      config:
        klineDays: 150      # 日K缓存深度（默认 150）
        dataRoot: C:/path/to/stock   # 数据目录（默认 %DSH_HOME%\stock）
        timeoutMs: 15000    # 请求超时（毫秒）
```

## 重启并验证

1. 重启 web 实例：停掉旧进程后运行 `dsh web`（或 `npx @deepseek-ai/dsh web`）
2. 对话里试：
   - “茅台现在什么价？” → `stock_quote`
   - “把 600519、000001 加进自选股” → `watchlist_add` ×2
   - “收集今日行情” → `stock_daily_collect`（收盘后调用最有意义）
   - “出一份 600519 的分析报告” → `stock_report` + 模型补充分析

## 已知边界

- 免费公开接口有限速与稳定性风险；行情仅供学习参考，**不构成投资建议**
- 代码推断规则：`60/68/9x → 沪`，`00/30/20 → 深`；指数用显式代码
  （如 `sh000001` 上证指数，与深市 `000001` 平安银行区分）
- 盘中调用 `stock_daily_collect` 收集的是盘中数据；建议收盘后触发
- 指标需要足够 K 线：MA60 至少 60 根（缓存 150 根足够）；新股数据不足时
  对应指标返回 null

## 卸载

1. 编辑 `%DSH_HOME%\profiles\web\cordis.patch.yml`，删除 `tool-stock` 条目
2. 删除 `%DSH_HOME%\profiles\web\plugins\stock\`
3. 重启 `dsh web`

（卸载不会删除 `%DSH_HOME%\stock\` 下的自选股/快照/报告数据。）
