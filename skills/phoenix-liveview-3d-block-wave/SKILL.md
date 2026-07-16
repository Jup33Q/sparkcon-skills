# Skill: Interactive 3D Block Wave (Tetrahedral/Honeycomb) for Phoenix LiveView

## 1. 概述

在 Phoenix LiveView 应用中实现全屏交互式 3D 方块波浪（或四面体/蜂窝状）背景特效。通过 Three.js 实例化渲染 + 自定义顶点着色器实现高性能 GPU 动画，并通过 LiveView JS Hook 桥接鼠标交互。同时提供完整的页面 Magic Move / Shared Element 过渡方案。

## 2. 核心架构

```
LiveView 页面 (HEEx)
    └── Layout 容器 (固定定位，z-index: -1)
        └── <div id="wave-canvas" phx-hook="BlockWave3D">
            └── Three.js Canvas (WebGL)
                ├── InstancedMesh (2500+ 实例)
                ├── Vertex Shader (波浪 + 鼠标交互)
                └── Fragment Shader (颜色渐变)
    └── 页面内容层 (z-index: 10, 半透明背景)
        └── <div phx-hook="PageTransition">
            └── 内容组件 (带 Magic Move FLIP 动画)
```

## 3. 依赖

```elixir
# mix.exs
{:phoenix_live_view, "~> 1.0"}
{:live_flip, "~> 0.1.0", optional: true}  # 可选：FLIP 动画
{:live_motion, "~> 0.3", optional: true}   # 可选：声明式动画
```

```javascript
# assets/package.json
{
  "dependencies": {
    "three": "^0.170.0"
  }
}
```

## 4. LiveView Layout (HEEx)

```elixir
# lib/my_app_web/components/layouts/root.html.heex
<!DOCTYPE html>
<html lang="zh-CN">
  <head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title><%= assigns[:page_title] || "MyApp" %></title>
    <link phx-track-static rel="stylesheet" href={~p"/assets/app.css"}>
    <script defer phx-track-static type="text/javascript" src={~p"/assets/app.js"}></script>
  </head>
  <body class="bg-gray-900 text-white">
    <!-- 3D 波浪背景层 - 全局共享，不随页面切换重建 -->
    <div 
      id="wave-canvas" 
      class="fixed inset-0 z-0"
      phx-hook="BlockWave3D"
      phx-update="ignore"
      data-shape="tetrahedral"
      data-grid-size="40"
      data-wave-speed="1.2"
      data-mouse-radius="3.0"
      data-color-primary="#4fc3f7"
      data-color-secondary="#e94560"
    ></div>

    <!-- 页面内容层 -->
    <main class="relative z-10 min-h-screen">
      <%= @inner_content %>
    </main>
  </body>
</html>
```

> **关键**：`phx-update="ignore"` 防止 LiveView DOM diff 覆盖 Three.js 渲染的 canvas。

## 5. JavaScript Hook 实现

```javascript
// assets/js/hooks/block_wave_3d.js
import * as THREE from 'three';

const BlockWave3D = {
  mounted() {
    this.initThree();
    this.initScene();
    this.initGrid();
    this.initMouseInteraction();
    this.animate();

    window.addEventListener('resize', this.onResize.bind(this));
  },

  destroyed() {
    window.removeEventListener('resize', this.onResize.bind(this));
    if (this.animationId) cancelAnimationFrame(this.animationId);
    this.renderer.dispose();
    this.geometry.dispose();
    this.material.dispose();
  },

  initThree() {
    this.container = this.el;
    this.width = window.innerWidth;
    this.height = window.innerHeight;

    this.renderer = new THREE.WebGLRenderer({ 
      antialias: true, 
      alpha: true 
    });
    this.renderer.setSize(this.width, this.height);
    this.renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    this.renderer.domElement.style.width = '100%';
    this.renderer.domElement.style.height = '100%';
    this.container.appendChild(this.renderer.domElement);

    this.scene = new THREE.Scene();
    this.camera = new THREE.PerspectiveCamera(
      75, 
      this.width / this.height, 
      0.1, 
      1000
    );
    this.camera.position.set(0, 15, 20);
    this.camera.lookAt(0, 0, 0);
  },

  initScene() {
    this.shape = this.el.dataset.shape || 'cube';
    this.gridSize = parseInt(this.el.dataset.gridSize || '50');
    this.waveSpeed = parseFloat(this.el.dataset.waveSpeed || '1.2');
    this.mouseRadius = parseFloat(this.el.dataset.mouseRadius || '3.0');
    this.colorPrimary = new THREE.Color(this.el.dataset.colorPrimary || '#4fc3f7');
    this.colorSecondary = new THREE.Color(this.el.dataset.colorSecondary || '#e94560');
  },

  initGrid() {
    const count = this.gridSize * this.gridSize;
    const spacing = 1.2;

    let geometry;
    switch(this.shape) {
      case 'tetrahedral':
        geometry = new THREE.TetrahedronGeometry(0.6, 0);
        break;
      case 'hexagonal':
        geometry = new THREE.CylinderGeometry(0.5, 0.5, 0.8, 6);
        break;
      default:
        geometry = new THREE.BoxGeometry(0.8, 0.8, 0.8);
    }

    this.geometry = geometry;

    this.material = new THREE.ShaderMaterial({
      uniforms: {
        uTime: { value: 0 },
        uMouse: { value: new THREE.Vector2(0, 0) },
        uMouseTrail: { value: new THREE.Vector2(0, 0) },
        uColorPrimary: { value: this.colorPrimary },
        uColorSecondary: { value: this.colorSecondary },
        uWaveSpeed: { value: this.waveSpeed },
        uMouseRadius: { value: this.mouseRadius }
      },
      vertexShader: this.getVertexShader(),
      fragmentShader: this.getFragmentShader(),
      side: THREE.DoubleSide
    });

    this.mesh = new THREE.InstancedMesh(geometry, this.material, count);
    this.scene.add(this.mesh);

    const dummy = new THREE.Object3D();
    this.positions = [];
    let i = 0;

    for (let x = 0; x < this.gridSize; x++) {
      for (let z = 0; z < this.gridSize; z++) {
        let posX, posZ;

        if (this.shape === 'hexagonal') {
          posX = (x - this.gridSize / 2) * spacing * 1.5;
          posZ = (z + (x % 2) * 0.5) * spacing * Math.sqrt(3);
        } else {
          posX = (x - this.gridSize / 2) * spacing;
          posZ = (z - this.gridSize / 2) * spacing;
        }

        this.positions.push({ x: posX, z: posZ, index: i });
        dummy.position.set(posX, 0, posZ);
        dummy.updateMatrix();
        this.mesh.setMatrixAt(i++, dummy.matrix);
      }
    }

    this.mesh.instanceMatrix.needsUpdate = true;
  },

  initMouseInteraction() {
    this.mouse = new THREE.Vector2(0, 0);
    this.mouseTrail = new THREE.Vector2(0, 0);
    this.raycaster = new THREE.Raycaster();
    this.hitPlane = new THREE.Plane(new THREE.Vector3(0, 1, 0), 0);

    this.container.addEventListener('mousemove', (e) => {
      const rect = this.container.getBoundingClientRect();
      const ndcX = ((e.clientX - rect.left) / rect.width) * 2 - 1;
      const ndcY = -((e.clientY - rect.top) / rect.height) * 2 + 1;

      this.raycaster.setFromCamera(new THREE.Vector2(ndcX, ndcY), this.camera);
      const intersects = new THREE.Vector3();
      this.raycaster.ray.intersectPlane(this.hitPlane, intersects);

      if (intersects) {
        this.mouse.set(intersects.x, intersects.z);
      }
    });
  },

  getVertexShader() {
    return `
      uniform float uTime;
      uniform vec2 uMouse;
      uniform vec2 uMouseTrail;
      uniform float uWaveSpeed;
      uniform float uMouseRadius;

      float sdSegment(vec2 p, vec2 a, vec2 b) {
        vec2 pa = p - a;
        vec2 ba = b - a;
        float h = clamp(dot(pa, ba) / dot(ba, ba), 0.0, 1.0);
        return length(pa - ba * h);
      }

      void main() {
        vec4 instancePos = instanceMatrix[3];

        float toCenter = length(instancePos.xz);
        float waveY = sin(uTime * uWaveSpeed * 2.0 + toCenter * 0.5) * 0.3;

        float mouseDist = sdSegment(instancePos.xz, uMouse, uMouseTrail);
        float mouseEffect = smoothstep(uMouseRadius, 1.0, mouseDist);

        vec3 transformed = position;

        float scale = 1.0 + (1.0 - mouseEffect) * 2.0;
        transformed *= scale;
        transformed.y += waveY - (1.0 - mouseEffect) * 2.0;

        float angle = uTime * uWaveSpeed + toCenter * 0.4 + (1.0 - mouseEffect) * 3.14;
        float c = cos(angle);
        float s = sin(angle);

        mat3 rotY = mat3(
          c, 0.0, s,
          0.0, 1.0, 0.0,
          -s, 0.0, c
        );
        transformed = rotY * transformed;

        vec4 mvPosition = vec4(transformed, 1.0);
        mvPosition = instanceMatrix * mvPosition;
        mvPosition = modelViewMatrix * mvPosition;
        gl_Position = projectionMatrix * mvPosition;
      }
    `;
  },

  getFragmentShader() {
    return `
      uniform vec3 uColorPrimary;
      uniform vec3 uColorSecondary;
      uniform float uTime;

      void main() {
        float mixFactor = sin(gl_FragCoord.y * 0.01 + uTime) * 0.5 + 0.5;
        vec3 color = mix(uColorPrimary, uColorSecondary, mixFactor);
        gl_FragColor = vec4(color, 0.9);
      }
    `;
  },

  animate() {
    this.animationId = requestAnimationFrame(this.animate.bind(this));
    const time = performance.now() * 0.001;

    this.mouseTrail.x += (this.mouse.x - this.mouseTrail.x) * 0.1;
    this.mouseTrail.y += (this.mouse.y - this.mouseTrail.y) * 0.1;

    this.material.uniforms.uTime.value = time;
    this.material.uniforms.uMouse.value.copy(this.mouse);
    this.material.uniforms.uMouseTrail.value.copy(this.mouseTrail);

    this.renderer.render(this.scene, this.camera);
  },

  onResize() {
    this.width = window.innerWidth;
    this.height = window.innerHeight;
    this.camera.aspect = this.width / this.height;
    this.camera.updateProjectionMatrix();
    this.renderer.setSize(this.width, this.height);
  }
};

export default BlockWave3D;
```

## 6. 注册 Hook

```javascript
// assets/js/app.js
import { Socket } from 'phoenix';
import { LiveSocket } from 'phoenix_live_view';
import BlockWave3D from './hooks/block_wave_3d';
import PageTransition from './hooks/page_transition';

let Hooks = {};
Hooks.BlockWave3D = BlockWave3D;
Hooks.PageTransition = PageTransition;

let csrfToken = document.querySelector("meta[name='csrf-token']").getAttribute("content");
let liveSocket = new LiveSocket("/live", Socket, { 
  hooks: Hooks, 
  params: { _csrf_token: csrfToken } 
});
liveSocket.connect();
```

## 7. 页面过渡方案 (Magic Move 风格)

### 7.1 方案 A: LiveFlip FLIP 动画 (推荐用于组件级 Magic Move)

LiveFlip 通过 Web Animations API 实现 FLIP（First, Last, Invert, Play）动画，让元素在位置/尺寸变化时平滑过渡。

```elixir
# mix.exs
{:live_flip, "~> 0.1.0"}
```

```elixir
# lib/my_app_web/live/dashboard_live.ex
defmodule MyAppWeb.DashboardLive do
  use MyAppWeb, :live_view
  import LiveFlip

  def mount(_params, _session, socket) do
    {:ok, assign(socket, cards: [
      %{id: "card-1", title: "Stats", value: "1,234", expanded: false},
      %{id: "card-2", title: "Revenue", value: "$56K", expanded: false}
    ])}
  end

  def handle_event("toggle_card", %{"id" => id}, socket) do
    cards = Enum.map(socket.assigns.cards, fn card ->
      if card.id == id do
        %{card | expanded: !card.expanded}
      else
        %{card | expanded: false}
      end
    end)
    {:noreply, assign(socket, cards: cards)}
  end

  def render(assigns) do
    ~H"""
    <div class="max-w-7xl mx-auto px-4 py-8">
      <h1 class="text-4xl font-bold text-white mb-8">Dashboard</h1>

      <div class={"grid gap-6 #{if @cards |> Enum.any?(& &1.expanded), do: 'grid-cols-1', else: 'grid-cols-1 md:grid-cols-3'}">
        <%= for card <- @cards do %>
          <.flip_wrap id={card.id}>
            <div 
              phx-click="toggle_card"
              phx-value-id={card.id}
              class={"bg-white/10 backdrop-blur-md rounded-xl p-6 cursor-pointer transition-all duration-500 #{if card.expanded, do: 'col-span-full min-h-[400px]', else: 'min-h-[200px]'}">
              <h2 class="text-xl font-semibold"><%= card.title %></h2>
              <p class="text-3xl font-bold mt-4"><%= card.value %></p>
              <%= if card.expanded do %>
                <div class="mt-6">
                  <p>详细内容区域...</p>
                </div>
              <% end %>
            </div>
          </.flip_wrap>
        <% end %>
      </div>
    </div>
    """
  end
end
```

### 7.2 方案 B: LiveMotion 声明式动画 (推荐用于复杂进入/退出动画)

LiveMotion 提供类似 Framer Motion 的声明式动画 API，支持 mount/unmount 动画。

```elixir
# mix.exs
{:live_motion, "~> 0.3"}
```

```elixir
# lib/my_app_web/live/page_live.ex
defmodule MyAppWeb.PageLive do
  use MyAppWeb, :live_view

  def render(assigns) do
    ~H"""
    <div class="min-h-screen">
      <!-- 页面进入动画 -->
      <LiveMotion.motion
        id="page-header"
        initial={[opacity: 0, y: 50, scale: 0.9]}
        animate={[opacity: 1, y: 0, scale: 1]}
        transition={[duration: 0.7, easing: "easeOut"]}
      >
        <h1 class="text-5xl font-bold text-white mb-4">Welcome</h1>
      </LiveMotion.motion>

      <!-- 交错进入的子元素 -->
      <div class="grid grid-cols-3 gap-6 mt-8">
        <%= for {item, index} <- Enum.with_index([1, 2, 3, 4, 5, 6]) do %>
          <LiveMotion.motion
            id={"item-#{item}"}
            initial={[opacity: 0, y: 30, scale: 0.8]}
            animate={[opacity: 1, y: 0, scale: 1]}
            transition={[
              duration: 0.5, 
              easing: "easeOut",
              delay: index * 0.1  # 交错延迟
            ]}
            exit={[opacity: 0, y: -30, scale: 0.8]}
          >
            <div class="bg-white/10 backdrop-blur-md rounded-xl p-6">
              <p>Item <%= item %></p>
            </div>
          </LiveMotion.motion>
        <% end %>
      </div>
    </div>
    """
  end
end
```

### 7.3 方案 C: 原生 LiveView.JS 过渡 (无外部依赖，最轻量)

使用 `Phoenix.LiveView.JS` 的 `transition/3` 和 `show/hide` 函数实现。

```elixir
# lib/my_app_web/helpers/transition_helpers.ex
defmodule MyAppWeb.TransitionHelpers do
  import Phoenix.LiveView.Helpers
  alias Phoenix.LiveView.JS

  @doc """
  页面进入过渡：从下方滑入 + 淡入 + 缩放
  """
  def page_enter(js \\ %JS{}) do
    js
    |> JS.transition(
      {"transition-all duration-700 ease-out", 
       "opacity-0 translate-y-12 scale-95", 
       "opacity-100 translate-y-0 scale-100"},
      time: 700
    )
  end

  @doc """
  页面退出过渡：向上滑出 + 淡出 + 缩小
  """
  def page_leave(js \\ %JS{}) do
    js
    |> JS.transition(
      {"transition-all duration-500 ease-in", 
       "opacity-100 translate-y-0 scale-100", 
       "opacity-0 -translate-y-12 scale-95"},
      time: 500
    )
  end

  @doc """
  Magic Move 风格：元素位置/尺寸变化的 FLIP 过渡
  注意：需要配合 CSS transition 使用
  """
  def magic_move_flip(js \\ %JS{}, selector) do
    js
    |> JS.transition(
      {"transition-all duration-500 cubic-bezier(0.22, 1, 0.36, 1)", 
       "", 
       ""},
      to: selector,
      time: 500
    )
  end

  @doc """
  卡片展开/收起动画
  """
  def card_expand(js \\ %JS{}, selector, expanded) do
    if expanded do
      js
      |> JS.add_class("col-span-full min-h-[400px]", to: selector)
      |> JS.remove_class("min-h-[200px]", to: selector)
    else
      js
      |> JS.remove_class("col-span-full min-h-[400px]", to: selector)
      |> JS.add_class("min-h-[200px]", to: selector)
    end
  end
end
```

```elixir
# lib/my_app_web/live/page_live.html.heex
<!-- 页面进入动画 -->
<div 
  id="page-content"
  class="transition-all duration-700 opacity-0 translate-y-12 scale-95"
  phx-mounted={JS.remove_class("opacity-0 translate-y-12 scale-95")}
>
  <h1>Page Content</h1>
</div>

<!-- 或使用 JS.show 实现更精确的进入动画 -->
<div
  id="page-content-2"
  style="display: none"
  class="opacity-0 translate-y-8"
  phx-mounted={JS.show(
    transition: {"ease-out duration-700", "opacity-0 translate-y-8", "opacity-100 translate-y-0"},
    time: 700
  )}
>
  <h1>Alternative Enter Animation</h1>
</div>
```

> **注意**：`JS.show` 配合 `style="display: none"` 和初始 class 是实现可靠进入动画的正确方式。`JS.transition` 单独使用时在 LiveView 的 DOM patching 机制下可能时机不对。

### 7.4 方案 D: 自定义 Hook 实现完整页面过渡 (SPA 风格)

监听 LiveView 的 `phx:page-loading-start` 和 `phx:page-loading-stop` 事件，实现类似 SPA 的页面切换动画。

```javascript
// assets/js/hooks/page_transition.js
const PageTransition = {
  mounted() {
    this.container = this.el;

    // 监听 LiveView 页面加载事件
    this.onPageLoadingStart = (e) => {
      if (e.detail?.kind === "redirect") {
        this.container.classList.add("page-leaving");
      }
    };

    this.onPageLoadingStop = (e) => {
      this.container.classList.remove("page-leaving");
      this.container.classList.add("page-entering");

      // 动画完成后移除类
      setTimeout(() => {
        this.container.classList.remove("page-entering");
      }, 700);
    };

    window.addEventListener("phx:page-loading-start", this.onPageLoadingStart);
    window.addEventListener("phx:page-loading-stop", this.onPageLoadingStop);

    // 初始进入动画
    this.container.classList.add("page-entering");
    setTimeout(() => {
      this.container.classList.remove("page-entering");
    }, 700);
  },

  destroyed() {
    window.removeEventListener("phx:page-loading-start", this.onPageLoadingStart);
    window.removeEventListener("phx:page-loading-stop", this.onPageLoadingStop);
  }
};

export default PageTransition;
```

```css
/* assets/css/app.css */

/* 页面过渡动画 */
.page-transition-wrapper {
  transition: all 0.7s cubic-bezier(0.22, 1, 0.36, 1);
}

.page-entering {
  animation: pageEnter 0.7s cubic-bezier(0.22, 1, 0.36, 1) forwards;
}

.page-leaving {
  animation: pageLeave 0.5s cubic-bezier(0.55, 0, 1, 0.45) forwards;
}

@keyframes pageEnter {
  from {
    opacity: 0;
    transform: translateY(40px) scale(0.96);
    filter: blur(4px);
  }
  to {
    opacity: 1;
    transform: translateY(0) scale(1);
    filter: blur(0);
  }
}

@keyframes pageLeave {
  from {
    opacity: 1;
    transform: translateY(0) scale(1);
    filter: blur(0);
  }
  to {
    opacity: 0;
    transform: translateY(-40px) scale(0.96);
    filter: blur(4px);
  }
}

/* Magic Move 元素过渡 */
.magic-move {
  transition: all 0.5s cubic-bezier(0.22, 1, 0.36, 1);
  will-change: transform, opacity, width, height, grid-column;
}

/* 3D 波浪容器 */
#wave-canvas {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  z-index: 0;
  pointer-events: all;
}

#wave-canvas canvas {
  display: block;
}

/* 页面内容层 - 半透明毛玻璃效果 */
.page-content {
  position: relative;
  z-index: 10;
  background: rgba(26, 26, 46, 0.85);
  backdrop-filter: blur(10px);
  min-height: 100vh;
}

/* 响应式 */
@media (max-width: 768px) {
  #wave-canvas {
    display: none;
  }
}
```

```elixir
# lib/my_app_web/live/app_live.ex
defmodule MyAppWeb.AppLive do
  use MyAppWeb, :live_view

  def render(assigns) do
    ~H"""
    <div class="app-container">
      <nav class="fixed top-0 z-50 w-full bg-gray-900/80 backdrop-blur-md border-b border-white/10">
        <div class="max-w-7xl mx-auto px-4 py-4 flex gap-6">
          <.link navigate={~p"/"} class="text-white hover:text-blue-400 transition">Home</.link>
          <.link navigate={~p"/dashboard"} class="text-white hover:text-blue-400 transition">Dashboard</.link>
          <.link navigate={~p"/settings"} class="text-white hover:text-blue-400 transition">Settings</.link>
        </div>
      </nav>

      <div 
        id="page-wrapper"
        class="page-content page-transition-wrapper pt-20"
        phx-hook="PageTransition"
      >
        <%= @inner_content %>
      </div>
    </div>
    """
  end
end
```

## 8. 完整应用示例

```elixir
# router.ex
scope "/", MyAppWeb do
  pipe_through :browser

  live_session :default, 
    layout: {MyAppWeb.Layouts, :app},
    on_mount: [{MyAppWeb.UserAuth, :mount_current_user}] do

    live "/", HomeLive, :index
    live "/dashboard", DashboardLive, :index
    live "/settings", SettingsLive, :index
  end
end
```

```elixir
# lib/my_app_web/live/home_live.ex
defmodule MyAppWeb.HomeLive do
  use MyAppWeb, :live_view

  def render(assigns) do
    ~H"""
    <div class="max-w-7xl mx-auto px-4 py-12">
      <LiveMotion.motion
        id="hero"
        initial={[opacity: 0, y: 60]}
        animate={[opacity: 1, y: 0]}
        transition={[duration: 0.8, easing: "easeOut"]}
      >
        <h1 class="text-6xl font-bold text-white mb-6">Welcome Home</h1>
        <p class="text-xl text-gray-300 mb-8">Interactive 3D wave background</p>
      </LiveMotion.motion>

      <div class="grid grid-cols-1 md:grid-cols-2 gap-8 mt-12">
        <.flip_wrap id="feature-1">
          <div class="magic-move bg-white/10 backdrop-blur-md rounded-2xl p-8 hover:bg-white/20 transition-colors">
            <h2 class="text-2xl font-semibold mb-4">Feature 1</h2>
            <p>Content with Magic Move animation</p>
          </div>
        </.flip_wrap>

        <.flip_wrap id="feature-2">
          <div class="magic-move bg-white/10 backdrop-blur-md rounded-2xl p-8 hover:bg-white/20 transition-colors">
            <h2 class="text-2xl font-semibold mb-4">Feature 2</h2>
            <p>Content with Magic Move animation</p>
          </div>
        </.flip_wrap>
      </div>
    </div>
    """
  end
end
```

## 9. 性能优化建议

1. **3D 波浪性能**：
   - 移动端 grid-size ≤ 30，桌面端 ≤ 50
   - 使用 `requestAnimationFrame` 时间戳控制帧率
   - `phx-update="ignore"` 防止 LiveView DOM diff 干扰 Three.js

2. **页面过渡性能**：
   - 使用 `will-change: transform, opacity` 提示浏览器优化
   - 避免在动画期间触发重排（避免改变 width/height，优先用 transform）
   - LiveFlip 使用 Web Animations API，比 CSS transition 更精确控制

3. **LiveView 优化**：
   - 使用 `live_session` 共享 layout，避免全页面重建
   - `push_patch` 代替 `push_redirect` 减少 LiveView 进程重启
   - 使用 `JS` 命令的 `blocking: false` 选项防止动画阻塞交互

## 10. 参数配置表

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `data-shape` | string | `cube` | 几何体：cube / tetrahedral / hexagonal |
| `data-grid-size` | int | `50` | 网格大小（总实例数 = grid-size²） |
| `data-wave-speed` | float | `1.2` | 波浪速度 |
| `data-mouse-radius` | float | `3.0` | 鼠标影响半径 |
| `data-color-primary` | string | `#4fc3f7` | 主色调 |
| `data-color-secondary` | string | `#e94560` | 次色调 |

## 11. 参考资源

- LiveFlip (FLIP 动画): https://github.com/probably-not/live-flip
- LiveMotion (声明式动画): https://hexdocs.pm/live_motion/
- LiveView 页面过渡: https://www.mave.io/blog/page-transitions-with-phoenix-liveview/
- LiveView JS 过渡: https://alembic.com.au/blog/improve-ux-with-liveview-page-transitions
- LiveView.JS 文档: https://hexdocs.pm/phoenix_live_view/Phoenix.LiveView.JS.html
- Shared Element Transitions API: https://www.smashingmagazine.com/2022/10/ui-animations-shared-element-transitions-api-part1/
