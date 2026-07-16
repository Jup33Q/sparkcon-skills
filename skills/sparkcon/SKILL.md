---
name: sparkcon
description: >
  专为 Spark 黑客松（Spark Hackathon）设计的参赛协作技能。
  覆盖从组队、选题、开发到展示的完整黑客松生命周期。
  提供工作流模板、提示词工程、队友协作节奏、技术选型建议、演示准备清单等。
  触发场景：用户提到 "spark 黑客松"、"hackathon"、"组队开发"、"48小时开发"、
  "演示准备"、"黑客松选题"、"BARF"、"第二届 spark" 或任何与 Spark 大会/黑客松相关任务。
  也用于同步 skills 文件到 GitHub 仓库、生成黑客松技术报告、管理参赛文档。
---

# SparkCon 黑客松协作技能

> 面向第二届 Spark 黑客松及同类黑客松活动的全方位协作指南。
> 本 Skill 由 BARF 队员 Jup33Q 创建并维护。

---

## 一、黑客松生命周期工作流

### 1.1 赛前准备（D-7 ~ D-1）

```
1. 研究往届优秀作品与技术栈
2. 梳理个人 skills 清单（调用本 Skill 生成汇总）
3. 确认组队人选与角色分工
4. 预研潜在技术方案
5. 准备开发环境（IDE、依赖、云资源）
6. 同步 skills 到团队共享仓库
```

### 1.2 选题与立项（D-Day 0~2h）

| 步骤 | 动作 | 产出 |
|-----|------|------|
| 头脑风暴 | 每人提出 3 个想法 | 想法池（>10个） |
| 筛选矩阵 | 技术可行性 × 创新性 × 演示效果 | Top 3 候选 |
| 快速验证 | 核心功能 PoC（30分钟） | 可行性结论 |
| 最终确定 | 全队投票，锁定选题 | 项目 README + 路线图 |

### 1.3 开发冲刺（D-Day 2h ~ D+1 24h）

```
Phase 1: 基础架构（2h）
  - 项目脚手架搭建
  - CI/CD 配置
  - 团队 Git 工作流确立

Phase 2: 核心功能（8h）
  - MVP 功能实现
  - 每日 standup（每 4h 同步一次）
  - 持续集成，保持主分支可运行

Phase 3: 集成与打磨（6h）
  - 功能集成测试
  - UI/UX 优化
  - 性能调优

Phase 4: 演示准备（4h）
  - Demo 脚本编写
  - 演示数据准备
  - 备用方案（offline fallback）
```

### 1.4 演示与收尾（D+2 上午）

- 演示前 30 分钟完整彩排
- 准备 3 分钟电梯演讲版本
- 准备 Q&A 常见问题答案
- 提交作品与文档

---

## 二、技术选型速查表

| 场景 | 推荐技术栈 | 备注 |
|-----|-----------|------|
| 实时协作 | Elixir + Phoenix LiveView | 低延迟、高并发 |
| AI 应用 | Python + FastAPI + 模型 API | 快速原型 |
| 数据可视化 | Python + Seaborn/Dash | 黑客松可视化标配 |
| 前端原型 | React + Tailwind CSS | 快速 UI 迭代 |
| 音频/音乐 | Strudel Live Coding | 创意音乐项目 |
| 文档/报告 | Markdown + PPTX Skill | 快速生成演示稿 |
| 金融数据 | stock-assistant + yahoo_finance | 量化/FinTech 项目 |

---

## 三、Skills 资产同步

### 3.1 生成本地 Skills 清单

使用以下命令汇总所有可用 skills：

```bash
# 列出所有 managed skills
ls -1 ~/Library/Application\ Support/kimi-desktop/daimon-share/daimon/skills/ | grep -v stock-assistant

# 列出所有 plugin skills
ls -1 ~/Library/Application\ Support/kimi-desktop/daimon-share/daimon/runtime/kimi-code/home/plugins/managed/
```

### 3.2 同步到 GitHub 仓库

```bash
# 1. 创建/克隆仓库
git clone https://github.com/Jup33Q/sparkcon-skills.git
cd sparkcon-skills

# 2. 复制 skills 内容
cp -r ~/Library/Application\ Support/kimi-desktop/daimon-share/daimon/skills/* ./skills/

# 3. 提交并推送
git add .
git commit -m "sync: 更新 skills 清单 $(date +%Y-%m-%d)"
git push origin main
```

---

## 四、团队协作文档模板

### 4.1 项目 README 模板

```markdown
# [项目名称] — Spark 黑客松作品

## 团队
- 队名：BARF
- 队员：Jup33Q, ...

## 项目概述
（一句话描述 + 三句话展开）

## 技术栈
- 前端：...
- 后端：...
- AI/数据：...

## 快速开始
```bash
git clone ...
cd ...
# 安装与运行命令
```

## 演示要点
1. ...
2. ...
3. ...

## 未来展望
- ...
```

### 4.2 每日 Standup 模板

```markdown
## Standup YYYY-MM-DD HH:MM

### 已完成
- [ ] 任务 A
- [ ] 任务 B

### 进行中
- [ ] 任务 C（预计完成时间）

### 阻塞/需要帮助
- ...

### 下阶段重点
- ...
```

---

## 五、提示词工程（Prompt Engineering）

### 5.1 快速原型提示词模板

```
我需要为一个黑客松项目快速实现 [功能描述]。
技术栈：[技术栈]
约束条件：
- 必须在 [时间] 内完成
- 代码必须可直接运行
- 优先使用简单方案，不追求完美

请提供：
1. 最小可行代码实现
2. 运行步骤
3. 已知限制
```

### 5.2 调试辅助提示词

```
我在黑客松中遇到以下错误：
[错误信息/截图]

上下文：
- 技术栈：...
- 最近修改：...

请帮我：
1. 分析可能原因
2. 提供 3 种解决方案（从快到慢）
3. 推荐最优解及理由
```

---

## 六、黑客松 checklist

### 赛前
- [ ] 开发环境配置完成
- [ ] GitHub 账号/仓库就绪
- [ ] 云服务/API Key 申请
- [ ] Skills 清单已同步到团队

### 赛中（每 8 小时检查）
- [ ] 主分支可运行
- [ ] 所有队员有最新代码
- [ ] 文档与代码同步更新
- [ ] 演示环境正常

### 赛前 2 小时
- [ ] Demo 脚本背诵完成
- [ ] 演示数据准备就绪
- [ ] 备用设备/网络方案
- [ ] 截图/录屏备份

---

## 七、资源索引

| 资源 | 路径/链接 | 说明 |
|-----|----------|------|
| Skills GitHub 仓库 | https://github.com/Jup33Q/sparkcon-skills | 团队共享 skills 同步 |
| Obsidian 知识库 | `02-Projects/zhihu-obsidian-manager/` | 文档与文章管理 |
| 技术栈技能 | `elixir-phx-liveview-ecosystem`, `swarm-coding` | 核心技术支持 |
| 数据与可视化 | `seaborn-visualization`, `stock-assistant` | 数据类项目 |
| 演示与文档 | `pptx`, `md-to-pdf` | 演示材料生成 |

---

*本 Skill 最后更新：2026-07-18*
*维护者：BARF 队员 Jup33Q*
