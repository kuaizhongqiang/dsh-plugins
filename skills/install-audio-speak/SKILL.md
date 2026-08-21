---
name: install-audio-speak
description: 把 speak_text 语音合成（TTS）工具安装到 dsh web（主模型保持 text-only，通过小米 MiMo mimo-v2.5-tts 合成语音并写入本地文件，支持 9 种内置音色、wav/mp3 输出）。当用户要求安装、卸载或排查 audio-speak 插件，或想让 dsh 具备朗读/语音合成能力、配置 MIMO_API_KEY 时使用。
whenToUse: 用户想给 dsh 加语音合成/朗读能力、要求安装或卸载 audio-speak 插件、需要配置 MIMO_API_KEY 或自定义 TTS 端点时。
---

# 安装 audio-speak 插件

把 `speak_text` 语音合成（TTS）工具装进目标电脑的 dsh web。主对话模型保持
text-only，模型调用 `speak_text` 合成语音（小米 MiMo `mimo-v2.5-tts`）并
写入本地文件，返回文件路径（默认写入下载文件夹）。

## 0. 定位插件包

插件包在本仓库 `plugins/audio-speak-dsh-plugin/`，二选一获取：

- 本地已有仓库克隆：直接用 `<仓库根>/plugins/audio-speak-dsh-plugin`
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
1. 复制 `plugins/audio-speak` 到 `%DSH_HOME%\profiles\web\plugins\audio-speak`
2. 幂等地在 `cordis.patch.yml` 追加 `- insert:` 的 `tool-audio-speak` 挂载条目
   （已存在则跳过）

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

## 4. 自定义端点 / 输出目录（可选）

`index.js` 顶部有默认值，也可在 `cordis.patch.yml` 的 `tool-audio-speak`
条目里覆盖：

```yaml
- insert:
    - id: tool-audio-speak
      name: './plugins/audio-speak/index.js'
      config:
        baseURL: https://你的端点/v1
        model: mimo-v2.5-tts
        apiKeyEnv: YOUR_KEY_ENV
        timeoutMs: 60000
        defaultOutputDir: C:/path/to/output
```

> YAML 字符串里的反斜杠需转义或使用正斜杠（如上例）。

## 5. 重启并验证

1. 重启 web 实例：停掉旧进程后运行 `dsh web`（或 `npx @deepseek-ai/dsh web`）
2. 在对话里让模型朗读一段文字（如“用冰糖音色读出：欢迎使用 dsh”），
   模型应调用 `speak_text` 并给出音频文件路径；检查默认输出目录
   （下载文件夹）出现 `mimo-speech-*.wav/mp3`

## 6. 排查

- 报 `unknown voice`：音色必须是内置清单之一（mimo_default / 冰糖 / 茉莉 /
  苏打 / 白桦 / Mia / Chloe / Milo / Dean）
- 报 `format must be one of wav/mp3`：本工具只落盘 wav/mp3（带文件头可
  直接播放）；pcm16 裸流未开放
- 报 `extension does not match format`：`output_path` 的扩展名要与
  `format` 一致；或只给目录让工具自动生成文件名

## 7. 卸载

1. 编辑 `%DSH_HOME%\profiles\web\cordis.patch.yml`，删除
   `tool-audio-speak` 条目
2. 删除 `%DSH_HOME%\profiles\web\plugins\audio-speak\`
3. （可选）移除 credentials / 环境变量中的 `MIMO_API_KEY`
4. 重启 `dsh web`
