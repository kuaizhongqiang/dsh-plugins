---
name: install-audio-read
description: 把 transcribe_audio（语音转写，mimo-v2.5-asr，mp3/wav）与 understand_audio（音频理解，mimo-v2.5，mp3/wav/flac/m4a/ogg）两个音频读取工具安装到 dsh web，支持本地路径或公网 URL。当用户要求安装、卸载或排查 audio-read 插件，或想让 dsh 具备语音转写/音频理解能力、配置 MIMO_API_KEY 时使用。
whenToUse: 用户想给 dsh 加语音转写或音频理解能力、要求安装或卸载 audio-read 插件、需要配置 MIMO_API_KEY 或自定义音频端点时。
---

# 安装 audio-read 插件

把 `transcribe_audio` / `understand_audio` 两个音频读取工具装进目标电脑的
dsh web。主对话模型保持 text-only：

- `transcribe_audio`：语音转文字，专用 ASR 模型 `mimo-v2.5-asr`
  （仅 mp3/wav，语言 auto/zh/en）
- `understand_audio`：听音频回答问题，全模态模型 `mimo-v2.5`
  （mp3/wav/flac/m4a/ogg）

## 0. 定位插件包

插件包在本仓库 `plugins/audio-read-dsh-plugin/`，二选一获取：

- 本地已有仓库克隆：直接用 `<仓库根>/plugins/audio-read-dsh-plugin`
- 没有克隆：`git clone https://github.com/kuaizhongqiang/dsh-plugins`，
  或让用户下载 Release 压缩包并解压

以下步骤均在插件包目录内执行。

## 1. 检查前置（目标电脑）

1. Node.js 满足 dsh 要求：`node -v` 应输出 `^22.19 || >=24`
2. dsh 已安装：`npm install -g @deepseek-ai/dsh`（或确认 `npx @deepseek-ai/dsh` 可用）
3. `%DSH_HOME%\profiles\web` 已初始化（`$env:DSH_HOME` 缺省 `~/.dsh`）。
   至少启动过一次 `dsh web`；没有就先用 `dsh web` 启动一次再继续

## 2. 执行安装脚本

```powershell
powershell -ExecutionPolicy Bypass -File .\install.ps1
```

脚本会（幂等，可重复执行）：
1. 复制 `plugins/audio-read` 到 `%DSH_HOME%\profiles\web\plugins\audio-read`
2. 幂等地在 `cordis.patch.yml` 追加 `- insert:` 的 `tool-audio-read` 挂载条目
   （已存在则跳过；一个条目注册 `transcribe_audio` 与 `understand_audio`
   两个工具）

确认输出出现 `OK  plugin copied`、`OK  profile patch entry added`（或
`SKIP ... already present`）。若报 `no dsh profiles found` 或 `web profile
not found`，回到步骤 1 检查 dsh 是否装好、`dsh web` 是否初始化过。

## 3. 配置 MIMO_API_KEY

工具按 credentials seam → 环境变量顺序解析，任选其一：

- credentials 服务：在 `%DSH_HOME%\.credentials.yaml` 加入
  ```yaml
  MIMO_API_KEY: <你的 key>
  ```
- 或设置环境变量 `MIMO_API_KEY`

> 绝不要把真实 key 写进仓库或分享的文件；`.env.example` 只是模板。

## 4. 自定义端点（可选）

`index.js` 顶部有默认值，也可在 `cordis.patch.yml` 的 `tool-audio-read`
条目里覆盖：

```yaml
- insert:
    - id: tool-audio-read
      name: './plugins/audio-read/index.js'
      config:
        baseURL: https://你的端点/v1
        asrModel: mimo-v2.5-asr
        understandModel: mimo-v2.5
        apiKeyEnv: YOUR_KEY_ENV
        maxTokens: 2048
        timeoutMs: 120000
```

## 5. 重启并验证

1. 重启 web 实例：停掉旧进程后运行 `dsh web`（或 `npx @deepseek-ai/dsh web`）
2. 在对话里给一个音频路径或链接：
   - “把 D:\recordings\meeting.wav 转成文字” → `transcribe_audio`
   - “这段音频说了什么/什么情绪？” → `understand_audio`

## 6. 排查

- `transcribe_audio` 报 `unsupported audio format`：ASR 平台限制仅
  mp3/wav；需要 flac/m4a/ogg 用 `understand_audio`
- 报 `over the ...-byte limit`：base64 输入上限 50 MB 编码串（≈37.5 MB 文件）；
  `transcribe_audio` 的 URL 会抓取后本地编码，同样受 37.5 MB 限制；
  `understand_audio` 的 URL 直传（平台 ≤ 100 MB）
- 转写结果为空或语言不对：显式指定 `language`（zh / en）再试

## 7. 卸载

1. 编辑 `%DSH_HOME%\profiles\web\cordis.patch.yml`，删除
   `tool-audio-read` 条目
2. 删除 `%DSH_HOME%\profiles\web\plugins\audio-read\`
3. （可选）移除 credentials / 环境变量中的 `MIMO_API_KEY`
4. 重启 `dsh web`
