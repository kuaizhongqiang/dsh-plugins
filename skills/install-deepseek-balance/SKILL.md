---
name: install-deepseek-balance
description: 把 deepseek_balance 余额查询工具安装到 dsh web（通过 DeepSeek 官方公开接口 GET /user/balance 查询账户余额，key 走 credentials seam 的 DEEPSEEK_API_KEY，输出不暴露 key）。当用户要求安装、卸载或排查 deepseek-balance 插件，或想查 DeepSeek API 余额/剩余额度/是否需要充值时使用。
whenToUse: 用户想查 DeepSeek API 账户余额/剩余额度、要求安装或卸载 deepseek-balance 插件、或余额查询功能不可用需要排查时。
---

# 安装 deepseek-balance 插件

把 `deepseek_balance` 余额查询工具装进目标电脑的 dsh web。通过
**DeepSeek 官方公开接口** `GET https://api.deepseek.com/user/balance`
查询账户余额，key 走 credentials seam（`DEEPSEEK_API_KEY`），输出中
**永不暴露 key**。

## 0. 定位插件包

插件包在本仓库 `plugins/deepseek-balance-dsh-plugin/`，二选一获取：

- 本地已有仓库克隆：直接用 `<仓库根>/plugins/deepseek-balance-dsh-plugin`
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
1. 复制 `plugins/deepseek-balance` 到 `%DSH_HOME%\profiles\web\plugins\deepseek-balance`
2. 幂等地在 `cordis.patch.yml` 追加 `- insert:` 的 `tool-deepseek-balance` 挂载条目
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

## 4. 重启并验证

1. 重启 web 实例：停掉旧进程后运行 `dsh web`（或 `npx @deepseek-ai/dsh web`）
2. 在对话里问“DeepSeek 还剩多少余额？”，模型应调用 `deepseek_balance`
   并返回总余额/充值余额/赠送余额

## 5. 排查

- 报 `no credential for DEEPSEEK_API_KEY`：按步骤 3 配置 key
- 报 `endpoint answered 401`：key 无效或过期，换新 key
- 报 `timed out`：网络问题或接口波动，稍后重试
- 余额不足时 `is_available` 为 false；充值请用配套的
  `install-deepseek-recharge` 技能安装充值插件

## 6. 卸载

1. 编辑 `%DSH_HOME%\profiles\web\cordis.patch.yml`，删除
   `tool-deepseek-balance` 条目
2. 删除 `%DSH_HOME%\profiles\web\plugins\deepseek-balance\`
3. （可选）移除 credentials / 环境变量中的 `DEEPSEEK_API_KEY`
4. 重启 `dsh web`
