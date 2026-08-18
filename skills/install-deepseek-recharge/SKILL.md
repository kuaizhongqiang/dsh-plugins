---
name: install-deepseek-recharge
description: 把 deepseek_recharge 充值辅助工具安装到 dsh web（DeepSeek 无公开充值 API，工具查询当前余额作上下文并打开平台充值页 platform.deepseek.com/usage，付完可再查余额验证；key 走 credentials seam 的 DEEPSEEK_API_KEY）。当用户要求安装、卸载或排查 deepseek-recharge 插件，或想快速跳转 DeepSeek 充值页时使用。
whenToUse: 用户想给 DeepSeek 充值/打开充值页、要求安装或卸载 deepseek-recharge 插件、或余额不足需要引导充值时。
---

# 安装 deepseek-recharge 插件

把 `deepseek_recharge` 充值辅助工具装进目标电脑的 dsh web。

**背景**：DeepSeek **没有公开充值 API**，充值只能在平台网页端完成
（实名认证后支付宝/微信）。本工具把充值路径缩到最短：查当前余额作上下文
→ 打开平台充值页（默认 `https://platform.deepseek.com/usage`）→ 返回
充值页 URL 与余额。

## 0. 定位插件包

插件包在本仓库 `plugins/deepseek-recharge-dsh-plugin/`，二选一获取：

- 本地已有仓库克隆：直接用 `<仓库根>/plugins/deepseek-recharge-dsh-plugin`
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
1. 复制 `plugins/deepseek-recharge` 到 `%DSH_HOME%\profiles\web\plugins\deepseek-recharge`
2. 幂等地在 `cordis.patch.yml` 追加 `- insert:` 的 `tool-deepseek-recharge` 挂载条目
   （已存在则跳过）

确认输出出现 `OK  plugin copied`、`OK  profile patch entry added`（或
`SKIP ... already present`）。

## 3. 配置 DEEPSEEK_API_KEY

工具按 credentials seam → 环境变量顺序解析，任选其一：

- credentials 服务：在 `%DSH_HOME%\.credentials.yaml` 加入
  ```yaml
  DEEPSEEK_API_KEY: <你的 key>
  ```
- 或设置环境变量 `DEEPSEEK_API_KEY`

> 绝不要把真实 key 写进仓库或分享的文件；`.env.example` 只是模板。

## 4. 可选配置

```yaml
- insert:
    - id: tool-deepseek-recharge
      name: './plugins/deepseek-recharge/index.js'
      config:
        rechargeUrl: https://platform.deepseek.com/usage   # 充值页
        autoOpen: true       # false = 只返回 URL 不自动开浏览器
```

## 5. 重启并验证

1. 重启 web 实例：停掉旧进程后运行 `dsh web`（或 `npx @deepseek-ai/dsh web`）
2. 在对话里让模型打开充值页，确认默认浏览器弹出平台页面
3. 充值完成后用 `deepseek_balance` 验证余额变化

## 6. 排查

- 浏览器没弹出来：`autoOpen` 被配置为 false，或系统默认浏览器设置异常；
  工具返回的 URL 可手动访问
- 报 `no credential for DEEPSEEK_API_KEY`：按步骤 3 配置 key
- 报 `endpoint answered 401`：key 无效或过期
- 充值页打不开/改版：更新配置里的 `rechargeUrl`

## 7. 卸载

1. 编辑 `%DSH_HOME%\profiles\web\cordis.patch.yml`，删除
   `tool-deepseek-recharge` 条目
2. 删除 `%DSH_HOME%\profiles\web\plugins\deepseek-recharge\`
3. （可选）移除 credentials / 环境变量中的 `DEEPSEEK_API_KEY`
4. 重启 `dsh web`
