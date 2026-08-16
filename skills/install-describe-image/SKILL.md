---
name: install-describe-image
description: 把 describe_image 图片理解工具安装到 dsh web（主模型保持 text-only，通过可配置的 OpenAI 兼容 vision 端点读图）。当用户要求安装、卸载或排查 describe-image 插件，或想让 dsh 具备图片理解能力、配置 MIMO_API_KEY / vision 端点时使用。
whenToUse: 用户想给 dsh 加图片理解能力、要求安装或卸载 describe-image 插件、需要配置 MIMO_API_KEY 或自定义 vision 端点时。
---

# 安装 describe-image 插件

把 `describe_image` 图片理解工具装进目标电脑的 dsh web。主对话模型保持
text-only，模型调用 `describe_image` 读图（attachmentId 或本地 path），由
OpenAI 兼容 vision 端点（默认 Xiaomi MiMo `mimo-v2.5`）返回文字描述。

## 0. 定位插件包

插件包在本仓库 `plugins/describe-image-dsh-plugin/`，二选一获取：

- 本地已有仓库克隆：直接用 `<仓库根>/plugins/describe-image-dsh-plugin`
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
1. 复制 `plugins/describe-image` 到 `%DSH_HOME%\profiles\web\plugins\describe-image`
2. 幂等地在 `cordis.patch.yml` 追加 `- insert:` 的 `tool-describe-image` 挂载条目
   （已存在则跳过）
3. 复制并运行 `patch-apiproxy.mjs`（text-only 主模型收到图片时转为 hint 文本
   而非拒绝，幂等）

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

## 4. 自定义 vision 端点（可选）

`index.js` 顶部有默认值，也可在 `cordis.patch.yml` 的 `tool-describe-image`
条目里覆盖：

```yaml
- insert:
    - id: tool-describe-image
      name: './plugins/describe-image/index.js'
      config:
        baseURL: https://你的端点/v1
        model: 你的模型
        apiKeyEnv: YOUR_KEY_ENV
        maxTokens: 4096
```

`mimo-v2.5` 是 reasoning 模型，`maxTokens` 建议 ≥ 2048，否则推理过程可能
耗尽预算导致 `content` 为空。

## 5. 重启并验证

1. 重启 web 实例：停掉旧进程后运行 `dsh web`（或 `npx @deepseek-ai/dsh web`）
2. 在对话里附一张图（或给本地路径），模型应调用 `describe_image` 并返回描述
3. 观察到 hint 文本 `[图片附件 <id> ...]` = apiproxy 补丁生效

## 6. 排查

- `patch-apiproxy.mjs` 报 `admission block not found`：补丁按 npm 版
  `dsh-host-apiproxy` 的文本锚点定位，dsh 大版本升级后锚点失效，需按新版
  `lib/index.js` 结构更新 `patch-apiproxy.mjs`
- 返回 `content` 为空：`maxTokens` 太小，调大（≥ 2048）
- `attachmentId` 需要图片以会话附件形式存在（GUI 粘贴/上传）；磁盘路径
  `path` 参数不受此限制

## 7. 卸载

1. 编辑 `%DSH_HOME%\profiles\web\cordis.patch.yml`，删除
   `tool-describe-image` 条目
2. 删除 `%DSH_HOME%\profiles\web\plugins\describe-image\` 与
   `%DSH_HOME%\profiles\web\patch-apiproxy.mjs`
3. （可选）移除 credentials / 环境变量中的 `MIMO_API_KEY`
4. 重启 `dsh web`
