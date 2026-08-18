# dsh-plugins — DSH 插件合集

面向原生 dsh（npm 版 `@deepseek-ai/dsh`）的插件合集。每个插件是一个
**自包含安装包**（`install.ps1` + `plugins/` + 说明文档），并配套一个
**安装技能（skill）**：技能描述该插件的安装方法，让你在 dsh 会话里
按需**选择安装哪些插件**。

## 目录结构

```
dsh-plugins/
├── plugins/                    插件安装包（自包含，可直接独立安装）
│   ├── describe-image-dsh-plugin/
│   ├── unity-mcp-dsh-plugin/
│   ├── video-read-dsh-plugin/
│   ├── audio-read-dsh-plugin/
│   ├── audio-speak-dsh-plugin/
│   ├── credentials-dsh-plugin/
│   ├── stock-dsh-plugin/
│   ├── deepseek-balance-dsh-plugin/
│   └── deepseek-recharge-dsh-plugin/
├── skills/                     安装技能：描述每个插件的安装方法，可选择安装
│   ├── README.md               技能机制与安装说明
│   ├── install-skills.ps1      一键把技能装进 %DSH_HOME%\skills（可选子集）
│   ├── install-describe-image/SKILL.md
│   ├── install-unity-mcp/SKILL.md
│   ├── install-video-read/SKILL.md
│   ├── install-audio-read/SKILL.md
│   ├── install-audio-speak/SKILL.md
│   ├── install-credentials/SKILL.md
│   ├── install-stock/SKILL.md
│   ├── install-deepseek-balance/SKILL.md
│   └── install-deepseek-recharge/SKILL.md
├── README.md
└── LICENSE                     MIT
```

## 快速开始

1. **装技能**（把“怎么装插件”教给 dsh，只需一次）：

   ```powershell
   # 安装全部技能（幂等，可重复执行）
   powershell -ExecutionPolicy Bypass -File .\skills\install-skills.ps1

   # 或只装选中的插件技能（“选择安装哪些插件”）
   powershell -ExecutionPolicy Bypass -File .\skills\install-skills.ps1 -Skills install-unity-mcp
   ```

2. **选插件**：在 dsh 会话里输入 `/install-describe-image`，或直接说
   “安装 describe-image 插件”，Agent 会加载对应技能并按正文一步步执行安装。

3. **验证**：重启 `dsh web` 后按各插件 README 的“重启并验证”步骤确认。

> 每个插件包也可脱离技能独立使用：进入 `plugins/<name>-dsh-plugin/` 目录
> 直接运行 `install.ps1`，详见各包 README。

## 插件清单

| 插件 | 功能 | 额外依赖 | 安装技能 |
|------|------|----------|----------|
| [describe-image](plugins/describe-image-dsh-plugin/README.md) | 图片理解工具：主模型保持 text-only，调用 `describe_image` 经 OpenAI 兼容 vision 端点（默认 Xiaomi MiMo `mimo-v2.5`）读图 | `MIMO_API_KEY`（或自建端点） | `install-describe-image` |
| [unity-mcp](plugins/unity-mcp-dsh-plugin/README.md) | MCP for Unity 桥：模型获得 `mcp__unity__*`（48 个 Unity Editor 工具），自带监督器自动拉起服务器 | Unity 项目 + MCP for Unity 客户端包 + uv/uvx | `install-unity-mcp` |
| [video-read](plugins/video-read-dsh-plugin/README.md) | 视频理解工具：调用 `read_video`（MiMo `mimo-v2.5` 全模态）理解视频，支持本地路径（mp4/mov/avi/wmv）或公网 URL | `MIMO_API_KEY` | `install-video-read` |
| [audio-read](plugins/audio-read-dsh-plugin/README.md) | 音频读取工具：`transcribe_audio` 语音转写（`mimo-v2.5-asr`，mp3/wav）+ `understand_audio` 音频理解（`mimo-v2.5`，mp3/wav/flac/m4a/ogg） | `MIMO_API_KEY` | `install-audio-read` |
| [audio-speak](plugins/audio-speak-dsh-plugin/README.md) | 语音合成工具：调用 `speak_text`（MiMo `mimo-v2.5-tts`）合成语音写入本地文件，9 种内置音色、wav/mp3 输出 | `MIMO_API_KEY` | `install-audio-speak` |
| [credentials](plugins/credentials-dsh-plugin/README.md) | 凭证管理工具：`credentials_list` / `credentials_set` / `credentials_unset` / `credentials_verify` 在对话里管理 `%DSH_HOME%\.credentials.yaml`，走官方 seam、永不暴露 key 值，set 可挂 approval 闸门 | 无 | `install-credentials` |
| [stock](plugins/stock-dsh-plugin/README.md) | A股行情与自选股：`stock_quote`/`stock_kline`/`stock_indicators`（MA/MACD/RSI/KDJ/ATR）/`stock_market_overview`/`watchlist_*`/`stock_daily_collect`/`stock_report`，腾讯公开接口、零密钥 | 无 | `install-stock` |
| [deepseek-balance](plugins/deepseek-balance-dsh-plugin/README.md) | DeepSeek 余额查询：`deepseek_balance` 走官方 `GET /user/balance` 接口返回总/充值/赠送余额 | `DEEPSEEK_API_KEY` | `install-deepseek-balance` |
| [deepseek-recharge](plugins/deepseek-recharge-dsh-plugin/README.md) | DeepSeek 充值辅助：`deepseek_recharge` 查余额作上下文并打开平台充值页（官方无充值 API，网页端支付） | `DEEPSEEK_API_KEY` | `install-deepseek-recharge` |

## 环境要求（目标电脑）

- Node.js `^22.19 || >=24`
- dsh 已安装：`npm install -g @deepseek-ai/dsh`（或用 `npx @deepseek-ai/dsh`）
- 至少启动过一次 `dsh web`（初始化 `%DSH_HOME%\profiles\web`）
- 各插件额外依赖见上表与各包 README

## 贡献约定

- 新插件放 `plugins/<name>-dsh-plugin/`，结构：`install.ps1`（幂等，可重复执行）
  + `plugins/<name>/`（插件代码）+ `README.md` +（如需要）`.env.example`
- **必须**配套安装技能：`skills/install-<name>/SKILL.md`，约定见
  [`skills/README.md`](skills/README.md)
- 包内不含任何 API key；密钥配置走 `%DSH_HOME%\.credentials.yaml` 或环境变量，
  不要把真实 key 写进仓库或分享的文件

## License

MIT © 2026 kuaizhongqiang
