---
name: interactive-background-alchemist
description: >
  将任意 Canvas 2D/3D 交互特效「熔炼」成可复用 Skill，并注入 Phoenix LiveView 项目的完整工作流。
  覆盖：效果生成 → Skill 封装 → 安装部署 → LiveView Hook 集成 四个阶段。
  与 canvas-effect-*、ide-code-wave-bg、phoenix_liveview_3d_block_wave 等 skill 协同使用。
tags: ["phoenix", "liveview", "canvas", "skill", "background", "hook", "integration"]
---

# 交互化背景炼金术：从 Agent 生成到 LiveView 集成

## 概述

本 Skill 描述如何把一段「随手生成的 Canvas 特效代码」炼成「可安装、可引用、可切换」的标准化 Skill，并最终挂载到 Phoenix LiveView 幻灯片项目中。

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  向 Agent 描述  │ ──▶│  Agent 生成代码  │ ──▶│  熔炼成 Skill   │ ──▶│  LiveView 集成  │
│  想要的背景效果  │    │  (HTML/JS/TS)   │    │  (SKILL.md+assets)│   │  (Hook + HEEx)  │
└─────────────────┘    └─────────────────┘    └─────────────────┘    └─────────────────┘
```

---

## 第一阶段：向 Agent 生成效果（召唤）

### 1.1 Prompt 模板

在任意 Kimi Agent 对话中，使用以下模板发起请求：

```text
请为我生成一个 Canvas 2D 交互背景特效，用于 Phoenix LiveView 幻灯片页面。

## 需求描述
- 主题：{{主题，如「极光呼吸场」「萤火虫星座」「几何矩阵」}}
- 交互：{{交互方式，如「鼠标靠近吸引」「点击爆炸」「滚轮缩放」}}
- 色调：{{主色调 + 背景色，如「cyan/#00E5FF + 深蓝/#06080F」}}
- 性能要求：60fps @ 1080p，粒子数不超过 {{N}} 个
- 输出形式：单文件 vanilla JS，无框架依赖

## 输出要求
1. 一个可直接运行的 `.html` 演示文件（内含 `<canvas>` + `<script>`）
2. 一份剥离出来的纯 `.js` 文件（仅逻辑，无 HTML 包装）
3. 一段 100 字以内的效果描述（用于后续 Skill 的 description 字段）
4. 一份配置参数表（颜色、粒子数、速度等可调整项）
```

### 1.2 推荐 Agent 能力域

| 效果类型 | 推荐描述关键词 | 参考 Skill |
|---------|--------------|-----------|
| 粒子/星云 | "particle nebula, mouse attraction, click explosion" | `canvas-effect-nebula` |
| 流动光带 | "aurora, flowing light bands, mouse reactive" | `canvas-effect-aurora` |
| 发光虫群 | "firefly, constellation, drift and pulse" | `canvas-effect-firefly` |
| 几何网格 | "geometric matrix, grid shapes, proximity rotate" | `canvas-effect-geometric` |
| 光绘轨迹 | "liquid light trail, bezier curve, mouse draw" | `canvas-effect-liquid` |
| 代码字符波 | "IDE code wave, syntax highlight, character dodge" | `ide-code-wave-bg` |
| 3D 方块浪 | "3D block wave, instanced mesh, vertex shader" | `phoenix_liveview_3d_block_wave` |

> 💡 **技巧**：可同时要求 Agent 生成「深色模式」和「浅色模式」两套配色参数，方便后续主题切换。

---

## 第二阶段：熔炼成 Skill（封装）

### 2.1 Skill 文件结构标准

将 Agent 输出的代码整理为以下目录结构：

```
my-canvas-effect/
├── SKILL.md              # 技能元信息 + 使用说明
├── assets/
│   ├── effect.js         # 核心特效逻辑（vanilla JS，无框架）
│   ├── LiveView.tsx      # 可选：React 包装组件（若项目用 LiveView + React）
│   └── demo.html         # 可选：独立演示文件
└── README.md             # 可选：更详细的文档
```

### 2.2 SKILL.md 头信息规范

```markdown
---
name: my-canvas-effect
description: >
  {{100字以内效果描述}}
  Provides a standalone JS file and LiveView integration guide.
  Use when: {{触发条件}}
tags: ["phoenix", "liveview", "canvas", "background"]
---
```

### 2.3 JS 文件接口规范

为了让代码能被 Phoenix LiveView Hook 无缝调用，`.js` 文件需暴露标准接口：

```javascript
// assets/effect.js
class MyCanvasEffect {
  constructor(canvas, options = {}) {
    this.canvas = canvas;
    this.ctx = canvas.getContext('2d');
    this.options = { /* 默认配置 */ ...options };
    this.animationId = null;
    this.resizeHandler = null;
  }

  init() {
    // 初始化画布尺寸、事件监听、启动动画循环
  }

  destroy() {
    // 取消动画帧、移除事件监听、释放资源
    if (this.animationId) cancelAnimationFrame(this.animationId);
  }

  resize() {
    // 响应式重算 canvas.width/height + DPR
  }
}

// 默认导出，方便 Hook import
export default MyCanvasEffect;
```

### 2.4 熔炼检查清单

| 检查项 | 要求 | 未通过的后果 |
|-------|------|------------|
| 零外部 CDN | 所有资源内联或本地 | 离线/内网无法加载 |
| prefers-reduced-motion | 包含 `@media` 媒体查询 | 无障碍问题 |
| DPR 适配 | 处理 `window.devicePixelRatio` | 高分屏模糊 |
| resize 监听 | 监听 `resize` 并 debounce | 窗口缩放后变形 |
| destroy 方法 | 可清理动画帧和事件 | Hook 卸载后内存泄漏 |
| 无全局污染 | 使用 class/module，不挂 window | 多实例冲突 |

---

## 第三阶段：安装 Skill（部署）

### 3.1 安装到用户级 Skill 目录

```bash
# Linux / WSL / macOS
mkdir -p ~/.kimi-code/skills/my-canvas-effect
cp -r my-canvas-effect/* ~/.kimi-code/skills/my-canvas-effect/

# 验证安装
ls ~/.kimi-code/skills/my-canvas-effect/SKILL.md
```

### 3.2 安装到项目级 Skill 目录（团队协作推荐）

```bash
# 在 Phoenix 项目根目录
cp -r my-canvas-effect .kimi-code/skills/

# 提交到版本控制
git add .kimi-code/skills/my-canvas-effect
git commit -m "feat: add my-canvas-effect skill"
```

### 3.3 安装到本手册包（归档）

```bash
# 若此 Skill 是 Obsidian 涅槃手册生态的一部分
cp -r my-canvas-effect /path/to/Obsidian涅槃手册_完整包/skills/
```

---

## 第四阶段：LiveView 集成（附魔）

### 4.1 创建 JS Hook

在 Phoenix 项目的 `assets/js/hooks/` 下创建对应 Hook：

```javascript
// assets/js/hooks/my_effect.js
import MyCanvasEffect from "../../vendor/my-canvas-effect/effect.js";

export const MyEffect = {
  mounted() {
    this.effect = new MyCanvasEffect(this.el, {
      // 可从 data-* 属性或 LiveView assign 读取配置
      particleCount: Number(this.el.dataset.particles) || 100,
      color: this.el.dataset.color || "#00E5FF",
    });
    this.effect.init();
  },

  updated() {
    // 若 LiveView push_event 更新了参数，在此响应
    if (this.effect && this.effect.resize) {
      this.effect.resize();
    }
  },

  destroyed() {
    if (this.effect) {
      this.effect.destroy();
      this.effect = null;
    }
  },
};
```

### 4.2 注册 Hook

```javascript
// assets/js/app.js
import { MyEffect } from "./hooks/my_effect";

let liveSocket = new LiveSocket("/live", Socket, {
  hooks: { MyEffect },
});
```

### 4.3 HEEx 模板引用

```heex
<%!-- lib/my_app_web/live/slide_deck_live.html.heex --%>
<div class="slide-deck relative overflow-hidden">
  <%!-- 背景层：z-index -1，鼠标事件穿透 --%>
  <canvas
    id="bg-effect"
    class="fixed inset-0 -z-10 pointer-events-none"
    phx-hook="MyEffect"
    data-particles="200"
    data-color="#00E5FF"
  ></canvas>

  <%!-- 内容层 --%>
  <div class="relative z-10">
    <%= @slide_content %>
  </div>
</div>
```

### 4.4 多背景切换策略

若需在不同幻灯片页切换不同背景效果：

1. **单 Hook 多实例**：在 `slide_deck_live.ex` 的 `handle_params/3` 中根据当前页号 `push_event/3` 切换背景配置
2. **条件渲染**：HEEx 中用 `case @slide_theme do` 渲染不同 `<canvas phx-hook="...">`
3. **CSS 变量驱动**：同一 Hook 读取 CSS 变量变化，动态调整配色

---

## 快速参考：已有 Skill 引用路径

| Skill 名称 | 安装位置 | 用途 |
|-----------|---------|------|
| `canvas-effect-aurora` | `~/.kimi-code/skills/canvas-effect-aurora` | 极光呼吸场 |
| `canvas-effect-firefly` | `~/.kimi-code/skills/canvas-effect-firefly` | 萤火虫星座 |
| `canvas-effect-geometric` | `~/.kimi-code/skills/canvas-effect-geometric` | 几何矩阵 |
| `canvas-effect-liquid` | `~/.kimi-code/skills/canvas-effect-liquid` | 液态光绘 |
| `canvas-effect-nebula` | `~/.kimi-code/skills/canvas-effect-nebula` | 星云粒子 |
| `ide-code-wave-bg` | `~/.kimi-code/skills/ide-code-wave-bg` | 代码字符波 |
| `phoenix_liveview_3d_block_wave` | 本包 `skills/` 或 `~/.kimi-code/skills/` | 3D 方块浪 |

---

> **炼金格言**：一次生成，处处复用；Skill 即契约，Hook 即桥梁。
