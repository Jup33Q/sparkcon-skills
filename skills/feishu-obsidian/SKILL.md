---
name: feishu-obsidian
description: >
  飞书文档通过 Obsidian 管理的工作流 Skill。
  当用户要求"写到飞书"、"创建飞书文档"、"飞书记录"、"同步到飞书"、
  "feishu doc"、"lark document" 或任何需要在飞书创建/更新文档时触发。
  本 Skill 定义 Obsidian → 飞书 的标准化 frontmatter 规范与输出格式，
  确保所有飞书文档在 Obsidian 中先创建、格式化，再手动复制到飞书。
  不直接调用飞书 API，只负责在 Obsidian 中生成符合飞书规范的 Markdown 文档。
---

# 飞书-Obsidian 文档协作 Skill

> 所有飞书文档操作，先在 Obsidian 中创建 Markdown 文件，按规范填写 frontmatter，再手动复制到飞书。

---

## 一、工作流

```
用户请求创建飞书文档
    ↓
在 Obsidian 创建 Markdown 文件（按本 Skill 规范）
    ↓
填写 frontmatter 标签（feishu-publish 等）
    ↓
生成飞书格式化的内容
    ↓
用户手动复制到飞书（或后续配置自动同步）
```

---

## 二、Frontmatter 规范（必填）

每个飞书文档必须在文件顶部包含以下 YAML frontmatter：

```yaml
---
title: 文档标题
date: YYYY-MM-DD
tags:
  - feishu-doc          # ⭐ 主标签，标识这是飞书文档
  - feishu-publish      # ⭐ 待发布状态（发布后改为 feishu-published）
  - sparkcon            # 关联项目标签（如适用）
  - <其他主题标签>
feishu-title: 飞书文档标题（可与 title 不同）
feishu-folder: 飞书目录路径（如 "SparkCon/番外"）
feishu-status: 草稿 / 待发布 / 已发布
source: AI Agent 自动同步
author: Jup33Q
---
```

### 标签说明

| 标签 | 含义 | 状态流转 |
|-----|------|---------|
| `feishu-doc` | 标识为飞书文档 | 始终保留 |
| `feishu-publish` | 待发布到飞书 | 发布后改为 `feishu-published` |
| `feishu-published` | 已发布到飞书 | 更新后改为 `feishu-publish` |
| `sparkcon` | 关联 SparkCon 项目 | 项目相关文档使用 |

---

## 三、文档目录结构

```
Obsidian Vault/
└── 02-Projects/
    └── sparkcon/
        └── 📝-飞书文档/
            ├── YYYY-MM-DD_文档标题.md
            └── README.md
```

---

## 四、飞书内容格式规范

### 4.0 ⚠️ 跨平台插件 YAML 兼容性（重要）

Obsidian 知乎插件（v0.4.2）在解析 frontmatter 时，**不支持多行 YAML 列表格式**。以下格式会触发 `No topics found in frontmatter` 错误：

```yaml
# ❌ 错误格式（知乎插件不支持）
zhihu-topics:
  - 黑客松
  - AI编程
```

**正确格式**：使用内联数组（inline array）语法：

```yaml
# ✅ 正确格式
zhihu-topics: [黑客松, AI编程, 人工智能工具, 效率工具, 开发者工具]
```

> 此兼容性问题同样适用于所有使用 Obsidian `metadataCache.getFileCache()` 读取 frontmatter 的第三方插件。在同时生成飞书文档和知乎文章时，统一使用内联数组格式。

### 4.1 标题层级

- 一级标题 `#` → 飞书文档标题（通常不用在正文中）
- 二级标题 `##` → 飞书一级标题
- 三级标题 `###` → 飞书二级标题
- 四级标题 `####` → 飞书三级标题

### 4.2 表格

Obsidian Markdown 表格可直接复制到飞书：

```markdown
| 列1 | 列2 | 列3 |
|-----|-----|-----|
| A   | B   | C   |
```

### 4.3 代码块

使用围栏代码块，标注语言：

```markdown
```python
# 代码内容
```
```

### 4.4 链接

- 普通链接：`[标题](https://url)`
- 飞书不支持 `card` 链接语法，统一使用普通链接

### 4.5 图片

- 使用 `![描述](图片URL)` 或 `![[图片文件名]]`
- 飞书粘贴时自动上传图片

---

## 五、创建流程

### Step 1: 确认文档信息

- 文档标题
- 所属项目/文件夹
- 主题标签

### Step 2: 创建文件

路径模板：
```
02-Projects/<Project>/📝-飞书文档/YYYY-MM-DD_<标题>.md
```

### Step 3: 填写 frontmatter

确保包含 `feishu-doc` 和 `feishu-publish` 标签。

### Step 4: 生成内容

按飞书格式规范编写 Markdown 内容。

### Step 5: 告知用户

提供文件路径和复制到飞书的操作步骤。

---

## 六、示例 frontmatter

```yaml
---
title: ComfyUI-NDI 插件 PR 进展番外
date: 2026-07-18
tags:
  - feishu-doc
  - feishu-publish
  - sparkcon
  - comfyui
  - ndi
  - github-pr
feishu-title: SparkCon 番外｜ComfyUI-NDI 插件 PR 提交进展
feishu-folder: SparkCon/番外
feishu-status: 待发布
source: AI Agent 自动同步
author: Jup33Q
---
```

---

## 七、与 obsidian-sync Skill 的协同

| 场景 | 前置 Skill | 本 Skill 动作 | 后续建议 |
|------|-----------|-------------|---------|
| 创建飞书文档 | `feishu-obsidian` | 在 Obsidian 生成带 frontmatter 的文档 | 用户手动复制到飞书 |
| 项目文档归档 | `obsidian-vault-first` | 读取项目现有笔记 | 生成飞书文档 |
| 更新飞书文档 | 读取 `feishu-published` 文件 | 修改内容，标签改为 `feishu-publish` | 用户手动更新飞书 |

---

*Last updated: 2026-07-18*
*Maintainer: BARF 队员 Jup33Q*
