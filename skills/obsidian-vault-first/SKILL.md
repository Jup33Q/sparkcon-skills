---
name: obsidian-vault-first
description: >
  强制要求 Agent 在处理任何与项目、系统、技术栈、知识管理相关的任务之前，
  必须先读取 Obsidian Vault 的 MOC（地图 of content）文件，获取当前上下文和项目状态。
  本 Skill 为强制性前置条件，不可跳过。
version: 1.0.0
category: system-ops
trigger:
  - 任何涉及代码项目、技术栈、磁盘/系统管理、开发工作流、知识查询的任务
  - 用户提及 项目、代码、开发、系统、整理、分析、优化 等关键词
  - 会话开始时（如果判断任务可能涉及用户工作区或项目）
  - 用户没有明确说明 不需要看 Obsidian 的情况下
---

# Obsidian Vault 优先访问（强制性）

> ⚠️ **本 Skill 为强制前置条件。** 在回答用户问题、执行操作、或制定计划之前，必须先完成以下步骤。

---

## 一、触发条件

以下场景**必须**先执行 Vault 读取：

1. 涉及任何代码项目的任务（修改、分析、调试、整理、新增功能）
2. 涉及系统/磁盘/文件管理的任务（清理、整理、空间管理、备份）
3. 涉及技术栈、工具链、环境配置的任务
4. 涉及知识查询、历史上下文回顾的任务
5. 会话开始时，如果用户请求与日常开发/工作相关
6. 用户请求 整理、分析、调查、报告 等系统性任务

**例外**：纯闲聊、与当前工作区无关的通用知识问答、明确声明 不需要看 Obsidian 的任务。

---

## 二、强制读取顺序（不可跳过、不可颠倒）

### Step 1: 读取总览导航

```
路径: ~/Documents/Obsidian Vault/00-MOC/README.md
```

**目的**：了解仓库全貌、结构规范、Agent 查阅指南。

**提取信息**：
- 仓库结构（00-MOC / 01-Agent-Chat-History / 02-Projects / 03-Knowledge-Base / 04-Tools-Scripts / 99-Archive）
- 命名规范（目录前缀、文件命名规则）
- 查阅优先级（MOC → 项目索引 → 今日待办 → 项目目录 → 知识域）

### Step 2: 读取今日待办（当前会话上下文）

```
路径: ~/Documents/Obsidian Vault/00-MOC/今日待办.md
```

**目的**：获取当前会话的进展、已完成事项、待办事项、最近会话历史。

**提取信息**：
- 本次完成 ✅ 列表（避免重复劳动）
- 当前待办 ⏳ 列表（延续性工作）
- 最近会话历史（关联上下文）
- 更新规则（如何维护本文件）

### Step 3: 读取项目索引

```
路径: ~/Documents/Obsidian Vault/00-MOC/项目索引.md
```

**目的**：获取所有活跃项目的速查信息，快速定位相关项目。

**提取信息**：
- 活跃项目列表（名称、路径、类型、状态、技术栈）
- 核心交付物与下一步行动
- 工具脚本集
- 更新日志

---

## 三、读取后动作（必须执行）

完成 Step 1-3 后，根据任务类型进行**定向读取**：

| 任务类型 | 定向读取路径 |
|----------|-------------|
| 涉及特定项目 | `02-Projects/<ProjectName>/` 下的项目概述.md |
| 涉及技术栈查询 | `03-Knowledge-Base/<Domain>/` 下的对应文档 |
| 涉及历史上下文 | `01-Agent-Chat-History/2026-07/` 下的对应日期文件 |
| 涉及系统/磁盘/清理 | `03-Knowledge-Base/系统运维/` 下的报告 |
| 涉及脚本工具 | `04-Tools-Scripts/` 下的对应目录 |

---

## 四、写入规范（如果修改了 Vault 内容）

如果本次任务导致 Vault 内容需要更新（新增项目、更新状态、记录清理成果等），必须：

1. **更新今日待办**：将本次完成的事项加入 `✅ 本次完成`，将新发现的事项加入 `⏳ 当前待办`
2. **更新项目索引**：如果新增/修改了项目，更新 `00-MOC/项目索引.md`
3. **更新知识库**：如果产生了可复用的知识，写入 `03-Knowledge-Base/` 对应域
4. **归档聊天历史**：如果本次会话有重要上下文，按规范写入 `01-Agent-Chat-History/YYYY-MM/`

---

## 五、路径常量（速查）

```
VAULT_ROOT = ~/Documents/Obsidian Vault/
MOC_DIR    = ~/Documents/Obsidian Vault/00-MOC/
CHAT_DIR   = ~/Documents/Obsidian Vault/01-Agent-Chat-History/
PROJ_DIR   = ~/Documents/Obsidian Vault/02-Projects/
KB_DIR     = ~/Documents/Obsidian Vault/03-Knowledge-Base/
TOOLS_DIR  = ~/Documents/Obsidian Vault/04-Tools-Scripts/
ARCHIVE_DIR= ~/Documents/Obsidian Vault/99-Archive/

MOC_README = ~/Documents/Obsidian Vault/00-MOC/README.md
MOC_TODO   = ~/Documents/Obsidian Vault/00-MOC/今日待办.md
MOC_INDEX  = ~/Documents/Obsidian Vault/00-MOC/项目索引.md
```

---

## 六、错误处理

- 如果 Vault 文件不存在或无法读取：记录错误，向用户报告，但仍尝试基于已有上下文完成任务
- 如果 Vault 内容为空或格式异常：记录异常，使用用户消息中的上下文作为 fallback
- 如果用户明确说 不要看 Obsidian ：跳过本 Skill，但记录本次跳过原因

---

## 七、更新与维护

本 Skill 的维护责任：
- 当 Vault 结构发生重大变更时（新增目录、重命名等），同步更新本 Skill 中的路径常量
- 当 Obsidian 升级导致 YAML frontmatter 格式变化时，同步更新
- 定期检查 Vault 路径有效性
