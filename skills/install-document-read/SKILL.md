---
name: install-document-read
description: 把 read_document 文档理解工具安装到 dsh web（综合读取 Word docx / Excel xlsx / PDF：提取文字 + 内嵌图片逐张走 MiMo vision 描述，解析由 Python python-docx/openpyxl/PyMuPDF 完成，需配置 MIMO_API_KEY）。当用户要求安装、卸载或排查 document-read 插件，或想让 dsh 具备读取 Word/Excel/PDF 文档文字与图片的能力时使用。
whenToUse: 用户想读取/总结 Word、Excel、PDF 文档内容或提取文档内图片、要求安装或卸载 document-read 插件、或文档读取功能不可用需要排查时。
---

# 安装 document-read 插件

把 `read_document` 工具装进目标电脑的 dsh web：读取 **Word (.docx/.docm) /
Excel (.xlsx) / PDF (.pdf)** 文档，返回提取文字 + 内嵌图片的 vision 描述。
解析由随包分发的 `parse_document.py` 完成（依赖 Python 与
python-docx / openpyxl / PyMuPDF），图片理解走 OpenAI 兼容 vision 端点
（默认 Xiaomi MiMo `mimo-v2.5`，key 走 credentials seam `MIMO_API_KEY`）。

## 0. 定位插件包

插件包在本仓库 `plugins/document-read-dsh-plugin/`，二选一获取：

- 本地已有仓库克隆：直接用 `<仓库根>/plugins/document-read-dsh-plugin`
- 没有克隆：`git clone https://github.com/kuaizhongqiang/dsh-plugins`，
  或让用户下载 Release 压缩包并解压

以下步骤均在插件包目录内执行。

## 1. 检查前置（目标电脑）

1. Node.js 满足 dsh 要求：`node -v` 应输出 `^22.19 || >=24`
2. dsh 已安装：`npm install -g @deepseek-ai/dsh`（或确认 `npx @deepseek-ai/dsh` 可用）
3. `%DSH_HOME%\profiles\web` 已初始化（`$env:DSH_HOME` 缺省 `~/.dsh`）。
   至少启动过一次 `dsh web`；没有就先用 `dsh web` 启动一次再继续
4. Python 与解析库（解析后端）：

   ```powershell
   python --version            # 需 3.9+
   python -m pip install python-docx openpyxl PyMuPDF
   ```

   国内网络装不上时先设代理再装：
   `$env:HTTPS_PROXY='http://127.0.0.1:10808'; python -m pip install python-docx openpyxl PyMuPDF`

## 2. 执行安装脚本

```powershell
powershell -ExecutionPolicy Bypass -File .\install.ps1
```

脚本会（幂等，可重复执行）：
1. 复制 `plugins/document-read` 到 `%DSH_HOME%\profiles\web\plugins\document-read`
2. 幂等地在 `cordis.patch.yml` 追加 `- insert:` 的 `tool-document-read` 挂载条目
   （已存在则跳过）
3. 探测 Python 解析库；缺失时给出安装命令（不阻塞安装）

确认输出出现 `OK  plugin copied`、`OK  profile patch entry added`（或
`SKIP ... already present`）。

## 3. 配置 MIMO_API_KEY

工具按 credentials seam → 环境变量顺序解析，任选其一：

- credentials 服务：在 `%DSH_HOME%\.credentials.yaml` 加入
  ```yaml
  MIMO_API_KEY: <你的 key>
  ```
- 或设置环境变量 `MIMO_API_KEY`

> 绝不要把真实 key 写进仓库或分享的文件；`.env.example` 只是模板。
> 只提取文字不描述图片时可以不配 key（工具会跳过图片描述）。

## 4. 重启并验证

1. 重启 web 实例：停掉旧进程后运行 `dsh web`（或 `npx @deepseek-ai/dsh web`）
2. 在对话里给一个文档路径或 URL，如“读一下 `C:\temp\表.xlsx` 的内容”，
   模型应调用 `read_document` 并返回提取文字 + 内嵌图片描述

## 5. 排查

- 报 `python parser error: missing python library ...`：按步骤 1.4 安装对应库
- 报 `python parser produced invalid JSON`：Python 版本过旧或解析库版本冲突，
  升级 Python ≥3.9 后重装库；也可在插件配置里改 `pythonPath` 指向其它解释器
- 报 `no credential for MIMO_API_KEY`：按步骤 3 配置 key；若不需图片描述，
  可在调用时传 `maxImages: 0`
- 报 `vision endpoint answered 401`：key 无效或过期，换新 key
- 报 `timed out`：网络问题或文档图片较多，稍后重试或调大配置 `timeoutMs`
- 不支持 `.doc` / `.xls`（老二进制格式），只支持 `.docx/.docm/.xlsx/.pdf`

## 6. 卸载

1. 编辑 `%DSH_HOME%\profiles\web\cordis.patch.yml`，删除
   `tool-document-read` 条目
2. 删除 `%DSH_HOME%\profiles\web\plugins\document-read\`
3. （可选）移除 credentials / 环境变量中的 `MIMO_API_KEY`
4. 重启 `dsh web`
