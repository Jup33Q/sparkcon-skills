---
name: elixir-phx-liveview-ecosystem
description: >
  Comprehensive Elixir ecosystem skill covering OTP design patterns, ETS in-memory storage,
  macro-based metaprogramming, Ecto with SQLite3, Phoenix LiveView modern patterns,
  HEEx component architecture, Tailwind CSS + DaisyUI integration, and Membrane multimedia
  framework. Use when building Elixir/Phoenix applications, writing LiveView components,
  configuring Ecto repositories, implementing GenServer/Supervisor hierarchies, writing macros,
  or working with real-time streaming (WebRTC, RTMP) via Membrane. Targets Elixir 1.17+,
  Phoenix 1.7+, LiveView 1.0+, OTP 27+, Ecto 3.12+, Tailwind 3.4+, DaisyUI 4+.
---

# Elixir / Phoenix LiveView / Membrane Ecosystem

## Table of Contents

| Topic | File | When to Read |
|-------|------|--------------|
| OTP: GenServer, Supervisor, Application, Distributed | `references/otp.md` | Building process trees, fault-tolerant systems, or distributed clusters |
| ETS: In-memory key-value storage | `references/ets.md` | Caching, session storage, rate limiting, shared process state |
| Metaprogramming: Macros, quote/unquote, __using__ | `references/metaprogramming.md` | Writing DSLs, code generation, compile-time transformations |
| Ecto + SQLite3: Schema, queries, migrations | `references/ecto-sqlite.md` | Database layer, SQLite-first Phoenix projects, CRUD operations |
| LiveView Core: Mount, handle_event, assigns, JS | `references/liveview-core.md` | Interactive real-time UI, stateful components, server-rendered SPA |
| HEEx + Tailwind + DaisyUI: Component styling | `references/liveview-ui-heex.md` | HTML templating, responsive design, prebuilt UI components |
| Membrane: Multimedia streaming framework | `references/membrane.md` | Audio/video processing, WebRTC, RTMP, streaming pipelines |

## Environment Defaults

- **Elixir**: 1.17+
- **OTP**: 27+
- **Phoenix**: 1.7+ (with `phx.new --database sqlite3`)
- **Phoenix LiveView**: 1.0+
- **Ecto**: 3.12+
- **Tailwind CSS**: 3.4+ (via `tailwindcss-rust` or CLI)
- **DaisyUI**: 4+
- **Membrane Core**: 1.1+

## New Project Setup

Create a new Phoenix project with SQLite as default database:

```bash
mix archive.install hex phx_new
mix phx.new my_app --database sqlite3 --live
# Or with Tailwind + Esbuild preconfigured (Phoenix 1.7 default):
mix phx.new my_app --database sqlite3 --live --assets esbuild,tailwind
```

After creation, add DaisyUI to assets:

```javascript
// assets/tailwind.config.js
module.exports = {
  content: ["../lib/*_web.ex", "../lib/*_web/**/*.*ex"],
  theme: { extend: {} },
  plugins: [require("daisyui")],
  daisyui: { themes: ["light", "dark", "cupcake"] }
}
```

```bash
cd assets && npm install -D daisyui@latest
```

## Progressive Reading Guide

1. **New to Elixir/Phoenix**: Start with `otp.md` → `liveview-core.md` → `ecto-sqlite.md` → `liveview-ui-heex.md`
2. **Building UI-heavy app**: `liveview-core.md` → `liveview-ui-heex.md` → `ets.md` (for caching)
3. **Writing libraries/DSLs**: `otp.md` (supervision tree) → `metaprogramming.md` → `liveview-core.md` (for `use MyAppWeb, :live_view` patterns)
4. **Media streaming app**: `membrane.md` → `otp.md` (pipeline supervision) → `liveview-core.md` (for player UI)
