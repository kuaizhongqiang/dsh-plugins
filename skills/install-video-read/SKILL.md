---
name: install-video-read
description: 把 read_video 视频理解工具安装到 dsh web（主模型保持 text-only，通过小米 MiMo 全模态模型 mimo-v2.5 理解视频，支持本地路径 mp4/mov/avi/wmv 或公网 URL）。当用户要求安装、卸载或排查 video-read 插件，或想让 dsh 具备视频理解能力、配置 MIMO_API_KEY 时使用。
whenToUse: 用户想给 dsh 加视频理解能力、要求安装或卸载 video-read 插件、需要配置 MIMO_API_KEY 或自定义视频理解端点时。
---

# 安装 video-read 插件

把 `read_video` 视频理解工具装进目标电脑的 dsh web。主对话模型保持
text-only，模型调用 `read_video` 读视频（本地 path 或公网 url），由小米
MiMo 全模态模型（`mimo-v2.5`）返回文字描述。

## 0. 定位插件包

插件包在本仓库 `plugins/video-read-dsh-plugin/`，二选一获取：

- 本地已有仓库克隆：直接用 `<仓库根>/plugins/video-read-dsh-plugin`
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
1. 复制 `plugins/video-read` 到 `%DSH_HOME%\profiles\web\plugins\video-read`
2. 幂等地在 `cordis.patch.yml` 追加 `- insert:` 的 `tool-video-read` 挂载条目
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

## 4. 自定义端点（可选）

`index.js` 顶部有默认值，也可在 `cordis.patch.yml` 的 `tool-video-read`
条目里覆盖：

```yaml
- insert:
    - id: tool-video-read
      name: './plugins/video-read/index.js'
      config:
        baseURL: https://你的端点/v1
        model: 你的模型
        apiKeyEnv: YOUR_KEY_ENV
        maxTokens: 4096
        timeoutMs: 180000
```

## 5. 重启并验证

1. 重启 web 实例：停掉旧进程后运行 `dsh web`（或 `npx @deepseek-ai/dsh web`）
2. 在对话里给一个视频路径或公网链接（如“分析一下 D:\clips\demo.mp4”），
   模型应调用 `read_video` 并返回视频内容描述

## 6. 排查

- 报 `unsupported video format`：平台仅支持 mp4/mov/avi/wmv，且不保证所有
  变体可识别（官方说明），换标准容器再试
- 报 `over the ...-byte limit`：本地文件走 base64 输入，上限 50 MB 编码串
  （≈37.5 MB 文件）；更大的视频请用公网 URL（≤ 300 MB）
- 返回 `content` 为空：`maxTokens` 太小，调大（≥ 2048）；视频过长可降低
  `fps` 或改问更聚焦的问题

## 7. 卸载

1. 编辑 `%DSH_HOME%\profiles\web\cordis.patch.yml`，删除
   `tool-video-read` 条目
2. 删除 `%DSH_HOME%\profiles\web\plugins\video-read\`
3. （可选）移除 credentials / 环境变量中的 `MIMO_API_KEY`
4. 重启 `dsh web`
