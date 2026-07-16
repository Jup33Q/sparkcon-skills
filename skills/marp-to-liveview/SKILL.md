# Marp to LiveView Slides — 技能文件

将 Marp CLI 导出的 HTML 幻灯片优化并升级为 Phoenix LiveView 项目的完整技能。

## 触发条件

当用户提到以下场景时激活此技能：
- Marp HTML 导出优化
- Marp 幻灯片转 LiveView
- 静态幻灯片升级动态
- Phoenix 幻灯片/演示系统
- 类似 "将 marp 导出的 html slider 优化并放入 phx + liveview 项目"

## 问题诊断清单

在给出方案前，先用以下清单诊断用户的 Marp HTML 文件问题：

### P1: CSS 重复内联（高优先级）
- **症状**: 每个 `<section>` 都有 `style="..."` 和 `data-style="..."` 重复内联 CSS
- **检测**: `grep -c 'style=' file.html` 值远大于 slide 数量
- **影响**: 文件膨胀 3-5 倍，渲染性能下降
- **修复**: 使用 Phoenix Component + 共享 CSS class

### P2: SVG foreignObject 包装（中优先级）
- **症状**: 每个 slide 被 `<svg><foreignObject>` 包裹
- **检测**: `grep -c '<svg data-marpit-svg'` 
- **影响**: 额外 DOM 层级，文本选择异常
- **修复**: 使用纯 HTML `<div class="slide">`

### P3: 外部 CDN 依赖（高优先级）
- **症状**: twemoji 等使用外部 CDN URL
- **检测**: `grep -c 'cdn.jsdelivr.net.*twemoji'`
- **影响**: 离线/内网无法显示
- **修复**: LiveView Component 内联 SVG emoji

### P4: 无 prefers-reduced-motion（中优先级）
- **症状**: 渐变动画无媒体查询保护
- **检测**: `grep 'gradient-shift'` 且无 `@media (prefers-reduced-motion)`
- **影响**: 无障碍问题，持续 GPU 消耗
- **修复**: 添加 CSS 媒体查询

### P5: 表格可读性差（低优先级）
- **症状**: 奇偶行颜色反差过大
- **检测**: 观察表格样式
- **修复**: 柔和斑马纹 `rgba(128,128,128,0.04)`

### P6: 无数据可视化（中优先级）
- **症状**: 数据仅纯表格展示
- **修复**: ECharts Hook 自动渲染图表

### P7: 无 URL 同步（中优先级）
- **症状**: slide 状态不在 URL 中
- **修复**: `handle_params/3 + push_patch/2`

## 迁移架构

采用 **"bare 模板优先"** 策略：

```
┌─────────────────────────────────────────────┐
│                  Parser Layer                │
│    Marpit API / Earmark → {html, css, comments} │
├─────────────────────────────────────────────┤
│                  State Layer                 │
│    SlideDeckLive → socket.assigns           │
├─────────────────────────────────────────────┤
│               Presentation Layer             │
│    SlideComponents → HEEx templates         │
├─────────────────────────────────────────────┤
│              Interaction Layer               │
│    SlideTouch Hook → 键盘/触摸/滚轮         │
├─────────────────────────────────────────────┤
│             Visualization Layer (可选)       │
│    ECharts Hook → bar/line/pie              │
├─────────────────────────────────────────────┤
│            Collaboration Layer (可选)        │
│    PubSub + Presence → 多人同步             │
└─────────────────────────────────────────────┘
```

## 核心文件模板

### 1. slide_deck_live.ex
```elixir
defmodule YourAppWeb.SlideDeckLive do
  use YourAppWeb, :live_view

  @impl true
  def mount(_params, _session, socket) do
    {:ok, assign(socket, slide: 1, total_slides: 1, 
                         transitioning: false, theme: "dark", direction: :next)}
  end

  @impl true
  def handle_params(params, _uri, socket) do
    slide = String.to_integer(params["slide"] || "1")
    theme = params["theme"] || "dark"
    {:noreply, assign(socket, slide: clamp(slide, 1, socket.assigns.total_slides),
                             theme: theme)}
  end

  # 键盘导航
  def handle_event("keydown", %{"key" => key}, socket) do
    case key do
      "ArrowRight" -> go_slide(socket, socket.assigns.slide + 1)
      "ArrowLeft"  -> go_slide(socket, socket.assigns.slide - 1)
      " "          -> go_slide(socket, socket.assigns.slide + 1)
      "Home"       -> go_slide(socket, 1)
      "End"        -> go_slide(socket, socket.assigns.total_slides)
      _            -> {:noreply, socket}
    end
  end

  # 触摸导航
  def handle_event("swipe", %{"direction" => dir}, socket) do
    case dir do
      "left"  -> go_slide(socket, socket.assigns.slide + 1)
      "right" -> go_slide(socket, socket.assigns.slide - 1)
      _       -> {:noreply, socket}
    end
  end

  defp go_slide(socket, target) do
    new_slide = clamp(target, 1, socket.assigns.total_slides)
    if new_slide == socket.assigns.slide or socket.assigns.transitioning do
      {:noreply, socket}
    else
      direction = if new_slide > socket.assigns.slide, do: :next, else: :prev
      Process.send_after(self(), :clear_transition, 500)
      
      {:noreply, 
       socket
       |> assign(slide: new_slide, direction: direction, transitioning: true)
       |> push_patch(to: ~p"/slides/#{new_slide}?theme=#{socket.assigns.theme}")}
    end
  end

  def handle_info(:clear_transition, socket) do
    {:noreply, assign(socket, transitioning: false)}
  end

  defp clamp(v, min, _) when v < min, do: min
  defp clamp(v, _, max) when v > max, do: max
  defp clamp(v, _, _), do: v
end
```

### 2. SlideTouch Hook (assets/js/hooks/slide_touch.js)
```javascript
const SlideTouch = {
  mounted() {
    this.startX = 0
    this.minSwipe = 50
    
    this.el.addEventListener('touchstart', e => {
      this.startX = e.touches[0].clientX
    }, { passive: true })
    
    this.el.addEventListener('touchend', e => {
      const dx = e.changedTouches[0].clientX - this.startX
      if (Math.abs(dx) > this.minSwipe) {
        this.pushEvent('swipe', { direction: dx < 0 ? 'left' : 'right' })
      }
    }, { passive: true })
  }
}
export default SlideTouch
```

### 3. Emoji 组件（零 CDN）
```elixir
def emoji(%{name: name} = assigns) do
  svgs = %{
    "package" => ~s|<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M21 8a2 2 0 0 0-1-1.73l-7-4a2 2 0 0 0-2 0l-7 4A2 2 0 0 0 3 8v8a2 2 0 0 0 1 1.73l7 4a2 2 0 0 0 2 0l7-4A2 2 0 0 0 21 16Z"/></svg>|,
    # 添加更多...
  }
  
  ~H"""
  <span class="emoji-icon" role="img" aria-label={@name}>
    <%= Phoenix.HTML.raw(Map.get(svgs, @name, "")) %>
  </span>
  """
end
```

## CSS 关键规则

```css
/* 主题变量 */
:root {
  --slide-bg: #1e293b;
  --slide-fg: #e2e8f0;
  --slide-primary: #38bdf8;
  --slide-accent: #f59e0b;
}

/* GPU 加速动画 */
@keyframes slideInRight {
  from { transform: translateX(60px); opacity: 0; }
  to   { transform: translateX(0); opacity: 1; }
}

/* 必须：尊重用户偏好 */
@media (prefers-reduced-motion: reduce) {
  .slide { animation: none !important; transform: none !important; opacity: 1 !important; }
}

/* 柔和斑马纹 */
tbody tr:nth-child(even) { background: rgba(128, 128, 128, 0.04); }
tbody tr:hover { background: rgba(56, 189, 248, 0.08); }
```

## 性能检查清单

- [ ] bare 模板作为迁移起点
- [ ] CSS 全 class 管理，零 style= 内联
- [ ] Emoji SVG 内联，零外部 CDN
- [ ] 动画仅用 transform + opacity
- [ ] prefers-reduced-motion 已支持
- [ ] 大量 slide 使用 Streams
- [ ] OSC hover 显示（非始终显示）
- [ ] 进度条 CSS transition（非 JS）

## 协作扩展

添加实时协作需 3 步：

```elixir
# 1. mount 中订阅
def mount(_, _, socket) do
  if connected?(socket) do
    Phoenix.PubSub.subscribe(YourApp.PubSub, "presentation:#{id}")
    Phoenix.Presence.track(self(), "presentation:#{id}", %{slide: 1, role: :viewer})
  end
  {:ok, socket}
end

# 2. 演讲者广播
def handle_event("next_slide", _, socket) do
  Phoenix.PubSub.broadcast(YourApp.PubSub, 
    "presentation:#{socket.assigns.presentation_id}",
    %{event: "slide_changed", payload: %{slide: new_slide}})
  # ...
end

# 3. 观众接收
def handle_info(%{event: "slide_changed", payload: %{slide: s}}, socket) do
  {:noreply, assign(socket, slide: s)}
end
```

## 文件清单

完整迁移包含以下文件：

| 文件 | 路径 | 用途 |
|------|------|------|
| slide_deck_live.ex | lib/*/live/ | 核心 LiveView |
| slide_deck.html.heex | lib/*/live/ | 主模板 |
| slide_components.ex | lib/*/components/ | 组件库 |
| slide_touch_hook.js | assets/js/hooks/ | 触摸/键盘 Hook |
| echarts_hook.js | assets/js/hooks/ | 图表 Hook（可选）|
| slide_deck.css | assets/css/ | 样式系统 |
