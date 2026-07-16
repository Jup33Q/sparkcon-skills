---
name: canvas-effects-catalog
description: >
  Obsidian 涅槃手册生态中所有可用交互化背景特效的速查目录。
  按效果类型、技术栈、适用场景分类，方便快速选型与引用。
  所有 Skill 均已验证可在 Phoenix LiveView + Tailwind CSS 环境下运行。
tags: ["phoenix", "liveview", "canvas", "catalog", "background", "effects"]
---

# Canvas 交互背景效果速查目录

## 目录结构说明

本目录下所有 Skill 均遵循统一接口规范（参见 `Interactive_Background_Alchemist_SKILL.md`），可直接通过 Phoenix LiveView JS Hook 挂载。

---

## 2D 粒子/流体类

### canvas-effect-aurora — 极光呼吸场
- **路径**: `~/.kimi-code/skills/canvas-effect-aurora`
- **技术**: Canvas 2D, 36 条径向光带, 正弦波动
- **交互**: 鼠标移动导致光带弯曲 + 中心 glow 偏移
- **配色**: 深蓝 `#06080F` + 青色 `#00E5FF`
- **适用**: 科技风幻灯片、深色主题首页、沉浸式背景
- **性能**: 极低 CPU 占用，无粒子系统

### canvas-effect-nebula — 星云粒子
- **路径**: `~/.kimi-code/skills/canvas-effect-nebula`
- **技术**: Canvas 2D, 数百粒子, 物理引擎
- **交互**: 鼠标吸引 + 点击爆炸 + 边缘反弹
- **配色**: 可配置，默认紫/蓝渐变
- **适用**: 数据可视化大屏、星空主题、梦幻氛围
- **性能**: 中等，建议粒子数 ≤ 300

### canvas-effect-firefly — 萤火虫星座
- **路径**: `~/.kimi-code/skills/canvas-effect-firefly`
- **技术**: Canvas 2D, 发光粒子, 连线算法
- **交互**: 粒子漂移 + 脉冲发光 + 临近连线成星座
- **配色**: 暖黄/绿光点，暗背景
- **适用**: 自然主题、夜间氛围、有机感设计
- **性能**: 中等，连线距离阈值建议 ≤ 100px

### canvas-effect-liquid — 液态光绘
- **路径**: `~/.kimi-code/skills/canvas-effect-liquid`
- **技术**: Canvas 2D, Bezier 曲线, 拖尾渐变
- **交互**: 鼠标绘制光轨，松手后缓慢消散
- **配色**: 可配置荧光色
- **适用**: 创意工具展示、涂鸦风页面、艺术主题
- **性能**: 高（曲线数量随绘制增长，需定期清理）

---

## 2D 几何/矩阵类

### canvas-effect-geometric — 几何矩阵
- **路径**: `~/.kimi-code/skills/canvas-effect-geometric`
- **技术**: Canvas 2D, 网格形状, 矩阵变换
- **交互**: 鼠标靠近时形状旋转/缩放/发光
- **配色**: 极简黑白或单色调
- **适用**: 建筑/设计类主题、极简主义、结构感页面
- **性能**: 高，形状数与网格密度相关

---

## 代码/字符类

### ide-code-wave-bg — 代码字符波
- **路径**: `~/.kimi-code/skills/ide-code-wave-bg`
- **技术**: Canvas 2D, 字符网格, 语法高亮着色
- **交互**: 字符随正弦波流动，鼠标靠近时字符躲避
- **配色**: Atom One Dark 风格（关键字、字符串、注释等分色）
- **适用**: 开发者作品集、技术博客、IDE 工具页
- **性能**: 中等，字符密度与画布尺寸成正比
- **特色**: 支持注入自定义代码字符串作为显示内容

---

## 3D WebGL 类

### Phoenix_LiveView_3D_Block_Wave — 3D 方块波浪
- **路径**: 本包 `skills/Phoenix_LiveView_3D_Block_Wave_SKILL.md`
- **技术**: Three.js, InstancedMesh, 自定义顶点/片元着色器
- **交互**: 鼠标位置驱动波浪相位，GPU 计算
- **配色**: 可配置形状（方块/四面体/蜂窝）+ 渐变色
- **适用**: 高冲击力首页、发布会幻灯片、品牌展示
- **性能**: 高（需 WebGL，2500+ 实例流畅）
- **依赖**: `three` npm 包

---

## 幻灯片专用类

### make-marp-html-bubble — Marp 气泡拼布背景
- **路径**: `~/.kimi-code/skills/make-marp-html-bubble` (项目级) 或本包同级
- **技术**: Canvas 2D, 气泡物理, 拼布局算法
- **交互**: 气泡漂浮、碰撞、鼠标扰动
- **配色**: 半透明渐变气泡
- **适用**: Marp 导出 HTML 的幻灯片背景增强
- **性能**: 中等
- **说明**: 作用于 Marp CLI 导出的 HTML，非 LiveView 项目内使用

---

## 选型决策树

```
需要 3D 冲击力？
├── 是 → Phoenix_LiveView_3D_Block_Wave
└── 否 → 需要文字/代码元素？
    ├── 是 → ide-code-wave-bg
    └── 否 → 偏好粒子/流体？
        ├── 是 → 需鼠标吸引/爆炸？
        │   ├── 是 → canvas-effect-nebula
        │   └── 否 → canvas-effect-firefly
        └── 否 → 偏好光带/流动？
            ├── 是 → canvas-effect-aurora
            └── 否 → 偏好几何/结构？
                ├── 是 → canvas-effect-geometric
                └── 否 → canvas-effect-liquid（自由绘制）
```

---

## 组合使用建议

| 场景 | 推荐组合 |
|------|---------|
| 科技产品发布 | `aurora` 全局背景 + `nebula` 数据页背景 |
| 开发者大会 | `ide-code-wave-bg` 首页 + `geometric` 内容页 |
| 自然/环保主题 | `firefly` 全局背景 + `liquid` 互动环节 |
| 品牌高端展示 | `3D_Block_Wave` 首页 + `aurora` 过渡页 |
| 教学/培训课件 | `bubble` (Marp 专用) 或 `aurora` (LiveView) |

---

> **维护提示**：新增 Skill 后请同步更新本目录的「已有 Skill 引用路径」和「选型决策树」章节。
