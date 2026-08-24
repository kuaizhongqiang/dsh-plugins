# stock analysis tools for native dsh

A 股（沪深）行情分析、自选股管理、**舆情分析与交易建议**插件，17 个工具全部**仅依赖
腾讯公开行情接口**（无需任何 API key），技术指标在插件内零依赖计算，
主模型负责解读、舆情矛盾分析与报告撰写。

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
| `sentiment_sources()` | 舆情**权威信息源白名单**：官方（政府网/部委/新华社/人民日报/央视/交易所公告）+ 主流财经媒体（财联社/证券时报/上证报/中证报等），舆情搜索只采信清单内来源 |
| `sentiment_pick(limit?)` | 从客观数据挑选当日**需舆情调查的股票**（默认最多 5 只）：异动（涨跌停/大幅波动/量比/振幅）+ 技术信号（RSI 超买超卖/KDJ 极端/均线破位突破/MACD）打分排序 |
| `sentiment_record(...)` | 持久化一只股票的舆情分析结论：消息摘要、来源、受众定位、解读方向、矛盾检验命中、反身性、倾向、结论 |
| `sentiment_list(date?)` | 列出历史舆情结论（可按日过滤），每日分析前回顾防重复调查 |
| `advice_calc(code, action?, risk_profile?)` | 按技术位计算建议价格与仓位：**触发价/目标价/止损价/仓位比例**（ATR 波动率 + 风险偏好 0~10，默认 6.5 定档） |
| `position_record(...)` | 记录一条交易建议到本地（状态默认**未执行 pending**），同股同类未执行建议会提示防仓位重复叠加 |
| `position_list(status?)` | 列出建议（默认未执行），返回未执行建议仓位合计 |
| `position_update(id, status)` | 用户反馈执行状态：`executed`=已执行 / `cancelled`=已取消；未反馈保持 pending |

## 数据与存储

- 行情：`https://qt.gtimg.cn/q=<symbol>`（GBK 编码，插件内解码；批量查询一次请求）
- 日K：`https://web.ifzq.gtimg.cn/appstock/app/fqkline/get`（前复权）
- 限速：请求间隔 ≥ 250ms；日K 按代码按日缓存，避免重复全量拉取
- 用户数据：`%DSH_HOME%\stock\`（`watchlist.json` · `kline-cache.json` ·
  `daily/YYYY-MM-DD.json` · `reports/*.md` · `sentiment.json` · `positions.json`）

## 每日分析 SOP（模型按此流程执行）

1. **客观数据**：`stock_market_overview` + 全自选股行情/指标（`stock_daily_collect` 或批量行情）
2. **挑选舆情对象**：`sentiment_pick`（默认 ≤5 只，原则每日不超过 5 只）
3. **舆情搜索**：`sentiment_sources` 拿白名单 → 对每只候选仅从白名单网站搜索（官方+财经）
4. **矛盾分析**（对每条消息，核心 C+D）：
   - **C 受众检验**：说给谁听的？老百姓→反着看；机构→警告或通知；外资→兑现验证；产业界→看订单/成本
   - **D 利益检验**：谁受益？高位利好=诱多出货嫌疑，低位利空=洗盘吸筹嫌疑
   - 辅助：A 动机（为何现在说）、B 反说（越缺越说）、E 措辞（适时/研究=没承诺）、F 兑现（说做时间差）
   - **反身性定性**：政府行为→传导路径→对实际价值的影响（暂不量化）
5. **记录**：每只调用 `sentiment_record` 持久化结论
6. **建议**：`advice_calc` 算价格/仓位 → 综合技术面+消息面出最终建议 →
   `position_record` 入库（状态 pending，待用户反馈执行）
7. **输出**：精选**最有把握且收益最大**的建议（最多 3 条），格式含
   动作/现价/触发价/目标价/止损价/仓位%/一句话逻辑；用户反馈后 `position_update` 更新状态

> 风险偏好默认 6.5（0 最保守 ~ 10 最激进）。用户未反馈执行时，建议保持 pending，
> `position_list` 会累计未执行仓位，防止遗忘与重复叠加。

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
   - “今天哪些股票需要做舆情？” → `sentiment_pick`
   - “给 600028 算一条买入建议” → `advice_calc` + `position_record`
   - “看看我有哪些没执行的建议” → `position_list`

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
