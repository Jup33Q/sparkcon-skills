---
name: obsidian-sync
description: >
  将项目文档、系统状态、技能清单、聊天历史等内容同步写入用户的 Obsidian Vault。
  当用户说 "同步到 Obsidian"、"写入 Obsidian"、"记录到 Obsidian"、"更新 Obsidian"、
  "同步文档到 obsidian"、"归档到 obsidian"、"obsidian 备份" 等时使用本 Skill。
  覆盖项目资料、系统工具与模型配置、Agent 聊天历史、技能列表等多类内容的归档写入。
version: 1.0.0
category: system-ops
trigger:
  - 同步到 obsidian
  - 写入 obsidian
  - 记录到 obsidian
  - 更新 obsidian
  - 归档到 obsidian
  - obsidian 备份
  - 同步文档到 obsidian
  - 同步项目到 obsidian
  - 把聊天历史写入 obsidian
  - 把系统信息同步到 obsidian
---

# Obsidian Vault 同步写入（归档与知识沉淀）

> 本 Skill 负责将外部内容**写入** Obsidian Vault，与 `obsidian-vault-first`（读取优先）互补。
> 执行写入前，如任务涉及当前项目或系统状态，建议先读取 `obsidian-vault-first` 获取 Vault 上下文。

---

## 一、Vault 路径常量（必须与 obsidian-vault-first 保持一致）

```
VAULT_ROOT  = ~/Documents/Obsidian Vault/
MOC_DIR     = ~/Documents/Obsidian Vault/00-MOC/
CHAT_DIR    = ~/Documents/Obsidian Vault/01-Agent-Chat-History/
PROJ_DIR    = ~/Documents/Obsidian Vault/02-Projects/
KB_DIR      = ~/Documents/Obsidian Vault/03-Knowledge-Base/
TOOLS_DIR   = ~/Documents/Obsidian Vault/04-Tools-Scripts/
ARCHIVE_DIR = ~/Documents/Obsidian Vault/99-Archive/
```

---

## 二、内容分类与写入路径

| 内容类型 | 目标目录 | 文件名规范 | 示例 |
|---------|---------|-----------|------|
| 项目文档 / README | `02-Projects/<ProjectName>/` | `README.md` 或 `项目概述.md` | `02-Projects/StreamDiffusion-mac/项目概述.md` |
| 项目技术笔记 / 修复记录 | `02-Projects/<ProjectName>/` | `<主题>.md` | `02-Projects/ComfyUI-NDI/修复笔记.md` |
| 系统工具与模型配置 | `03-Knowledge-Base/` | `系统工具与模型配置总览.md` | `03-Knowledge-Base/系统工具与模型配置总览.md` |
| 技术栈知识 / 命令速查 | `03-Knowledge-Base/<Domain>/` | `<主题>.md` | `03-Knowledge-Base/Apple-Container/命令速查与网络配置.md` |
| Agent 聊天历史 | `01-Agent-Chat-History/YYYY-MM/` | `YYYY-MM-DD_NNN_主题摘要.md` | `01-Agent-Chat-History/2026-07/2026-07-13_001_Obsidian结构整理.md` |
| 脚本与工具 | `04-Tools-Scripts/<ProjectName>/` | 保留原始文件名 | `04-Tools-Scripts/StreamDiffusion-mac/camera.py` |
| 过时或合并后的旧文档 | `99-Archive/` | 保留原始文件名或加前缀 | `99-Archive/旧项目笔记.md` |

---

## 三、写入前检查清单

1. **确认 Vault 根目录存在**：`~/Documents/Obsidian Vault/` 是否存在？
2. **确认目标目录存在**：若不存在，按 Vault 结构创建目录。
3. **检查文件冲突**：若目标文件已存在，询问用户是否覆盖、追加或重命名。
4. **Frontmatter 规范**：Markdown 文件顶部必须包含 YAML frontmatter：
   ```yaml
   ---
   title: 文档标题
   date: YYYY-MM-DD
   tags: [tag1, tag2]
   source: 来源描述（如 "Kimi Work 自动同步" / "Agent 会话导出"）
   ---
   ```
5. **WikiLink 兼容性**：若文档间需要互相链接，使用 Obsidian WikiLink 格式 `[[目标文件名]]`。

---

## 四、典型同步工作流

### 4.1 同步项目文档到 Vault

```
1. 读取项目源文件（如 GitHub README、本地 README.md）
2. 在 02-Projects/ 下创建或确认项目目录
3. 转换或复制内容，补充 YAML frontmatter
4. 写入目标文件（如 02-Projects/<Name>/README.md）
5. 更新 00-MOC/项目索引.md（如项目索引有变化）
6. 向用户确认写入完成，并提供 Vault 内路径
```

### 4.2 同步系统工具与模型配置

```
1. 收集系统状态（Skills 列表、Ollama 模型、ComfyUI 模型、应用版本等）
2. 整理为 Markdown 表格和分类列表
3. 写入 03-Knowledge-Base/系统工具与模型配置总览.md
4. 补充 YAML frontmatter（date、tags、source）
5. 询问用户是否需要设置定时刷新（cron job）
```

### 4.3 记录 Agent 聊天历史

```
1. 提取本次会话的主题摘要（1-3 个关键词）
2. 按日期命名：YYYY-MM-DD_NNN_主题摘要.md
   - NNN 为当天序号，从 001 开始
3. 写入 01-Agent-Chat-History/YYYY-MM/
4. 内容结构：
   - 会话主题与目标
   - 关键决策与结果
   - 待办事项（如有）
   - 相关文件/链接
5. 更新 01-Agent-Chat-History/README.md（如有目录索引）
```

---

## 五、跨 Skill 协同

| 场景 | 前置 Skill | 本 Skill 动作 | 后续建议 |
|------|-----------|-------------|---------|
| 项目开发中整理知识 | `obsidian-vault-first` 读取项目现有笔记 | 将新调研/结论写入 03-Knowledge-Base/ | 更新 00-MOC/今日待办.md |
| 安装新工具后记录 | 无 | 将工具信息追加到 03-Knowledge-Base/系统工具与模型配置总览.md | 询问是否设置 cron 自动同步 |
| 会话结束归档 | 无 | 将聊天摘要写入 01-Agent-Chat-History/ | 无 |
| 项目仓库文档同步 | 无 | 将 README/使用说明写入 02-Projects/<Name>/ | 更新 00-MOC/项目索引.md |

---

## 六、错误处理

- **Vault 根目录不存在**：提示用户确认 Obsidian Vault 路径，记录异常并中止写入。
- **目录无写入权限**：报告权限问题，尝试修复（如 `chmod` / `chown`），或建议用户手动处理。
- **文件已存在**：默认行为为**追加到文件末尾**（带分隔线），除非用户明确说"覆盖"。
- **内容格式不支持**：纯文本/HTML 等非 Markdown 内容，先转换为 Markdown 再写入。

---

## 七、维护与更新

- 当 Vault 结构变更（如新增目录层级）时，同步更新本 Skill 的"内容分类与写入路径"表。
- 当 Obsidian YAML frontmatter 规范变化时，同步更新"写入前检查清单"。