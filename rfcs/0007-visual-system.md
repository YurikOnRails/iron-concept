# RFC 0007 — Visual System Architecture

**Status:** Draft
**Created:** 2026-03-24

---

## Summary

This RFC defines how IRON renders SCADA screens — from simple dashboards
built with ready-made widgets to full P&ID mnemonic diagrams with animated
equipment symbols. The visual system has three layers, each serving
a different user and a different complexity level. LiveView handles 95%
of runtime rendering. React is used only for the visual editor (a design-time
tool, not a runtime dependency).

---

## Motivation

### The gap in current documentation

IRON's architecture documents describe the data path in detail: sensors → edge →
NATS → TimescaleDB → browser. But they do not explain how the **operator's screen**
is actually built. For a SCADA system, this is the most visible part of the product
and the first thing every evaluator will judge.

### Why this is hard

A SCADA screen is not a web page. It is an interactive mnemonic diagram of a plant:

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│       ┌─────┐    ┌──────┐    ┌─────┐                           │
│       │TANK │    │PUMP  │    │VALVE│                            │
│       │     │───▸│ ◉    │───▸│ ◆   │───▸  to reactor           │
│       │87.5°│    │RUN   │    │ 42% │                            │
│       │ ●OK │    │ ●OK  │    │ ●OK │                            │
│       └─────┘    └──────┘    └─────┘                            │
│                                                                 │
│   [TREND: temperature last 1h ▁▂▃▅▆▇█▇▆▅]   [ALARMS: 0 active]│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

An operator must understand the plant state in 2 seconds by looking at the picture.
Not by reading a table. The visual representation is a safety tool — it prevents
misunderstanding.

### What LiveView can and cannot do

LiveView excels at:
- Tables with live-updating tag values
- Alarm panels, trend charts, lists
- Forms, RBAC-protected controls
- Grid-based dashboards from components

LiveView is not designed for:
- Drag-and-drop placement of SVG symbols on a free-form canvas
- Complex vector editing (draw pipes, align, group, snap-to-grid)
- Canvas interactions that require high-frequency mouse tracking

The solution is not "LiveView vs React." It is a layered system where each
technology handles what it does best.

---

## The Three-Layer Visual System

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Layer 3 — SVG Editor (React island)          DESIGN-TIME ONLY │
│  Drag-and-drop mnemonic builder                                │
│  Phase 3 · For integrators and engineers                       │
│  Outputs: SVG files with data-iron-tag attributes              │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Layer 2 — SVG Mimics (LiveView + phx-hook)          RUNTIME   │
│  Static SVG + dynamic data overlay                             │
│  Phase 2 · For engineers (YAML + Inkscape)                     │
│  Input: SVG files drawn in any editor                          │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Layer 1 — Widget Dashboards (pure LiveView)         RUNTIME   │
│  Grid of ready-made components                                 │
│  Phase 1 · For everyone (YAML or mouse)                        │
│  Covers 70% of real-world use cases                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

Each layer builds on the one below. A single screen can combine all three:
a grid dashboard with a trend chart (Layer 1), an embedded mnemonic SVG (Layer 2),
and a custom widget built in the visual editor (Layer 3).

---

## Layer 1 — Widget Dashboards (Pure LiveView)

### What it is

A grid-based dashboard assembled from a library of pre-built LiveView components.
No JavaScript. No SVG knowledge. Configuration in YAML or via a drag-and-drop
grid constructor in the browser.

### Who it serves

- **Level 1 users (mouse only):** drag widgets onto a grid, bind to tags, done
- **Level 2 users (YAML):** write dashboard config as code, commit to Git

### Dashboard configuration

```yaml
# config/dashboards/reactor_01.yaml
title: "Reactor 01 — Overview"
layout: grid
columns: 4
rows: 3
refresh: realtime    # LiveView PubSub, not polling

widgets:
  - type: gauge
    tag: reactor_01.temperature
    position: [0, 0]
    size: [1, 1]
    config:
      min: 0
      max: 200
      unit: "°C"
      zones:
        green: [0, 150]
        yellow: [150, 180]
        red: [180, 200]

  - type: trend
    tags:
      - reactor_01.temperature
      - reactor_01.pressure
    position: [1, 0]
    size: [2, 2]
    config:
      timerange: 1h
      y_axes:
        - label: "Temperature"
          unit: "°C"
          range: [0, 200]
        - label: "Pressure"
          unit: "bar"
          range: [0, 10]

  - type: status
    tag: pump_01.running
    position: [0, 1]
    size: [1, 1]
    config:
      labels:
        true: "RUNNING"
        false: "STOPPED"
      colors:
        true: green
        false: gray

  - type: command_button
    tag: pump_01.start_cmd
    position: [3, 0]
    size: [1, 1]
    config:
      label: "Start Pump"
      requires_confirm: true
      requires_role: operator

  - type: alarm_panel
    filter: "reactor_01.*"
    position: [0, 2]
    size: [4, 1]
```

### Widget library (Phase 1–2)

```
Display widgets (READ path):
  gauge           — circular gauge with colored zones
  numeric         — large number with unit and quality indicator
  status          — boolean ON/OFF with label and color
  trend           — time-series chart (TimescaleDB continuous aggregates)
  sparkline       — compact inline trend
  bar             — horizontal/vertical bar with fill level
  tank_level      — tank symbol with animated fill
  alarm_panel     — filterable alarm list with acknowledge button
  table           — tag table with sortable columns

Control widgets (WRITE path):
  command_button  — sends a command, requires role + confirm
  setpoint        — numeric input for analog output
  selector        — dropdown for mode selection (auto/manual/off)
  slider          — analog control with limits

Layout widgets:
  group           — visual grouping with title
  mimic_embed     — embeds a Layer 2 SVG mnemonic inside the grid
  nav_link        — navigation to another dashboard
```

### How it works technically

Each widget is a LiveView function component that subscribes to its tag(s)
via Phoenix PubSub on mount:

```elixir
defmodule IronWeb.Widgets.Gauge do
  use IronWeb, :live_component

  def mount(socket) do
    if connected?(socket) do
      Phoenix.PubSub.subscribe(Iron.PubSub, "tag:#{socket.assigns.tag}")
    end
    {:ok, socket}
  end

  def handle_info({:tag_update, %{value: value, quality: quality}}, socket) do
    {:noreply, assign(socket, value: value, quality: quality)}
  end

  def render(assigns) do
    ~H"""
    <div class={["iron-gauge", quality_class(@quality)]}>
      <svg viewBox="0 0 200 200">
        <!-- gauge arc, needle, value display -->
        <text x="100" y="160" text-anchor="middle" class="text-2xl">
          <%= format_value(@value, @config.unit) %>
        </text>
      </svg>
      <span class="iron-quality-badge"><%= @quality %></span>
    </div>
    """
  end
end
```

The dashboard renderer reads `config/dashboards/*.yaml` and composes
the grid from these components:

```elixir
defmodule IronWeb.DashboardLive do
  use IronWeb, :live_view

  def mount(%{"id" => dashboard_id}, _session, socket) do
    dashboard = Iron.Dashboards.load(dashboard_id)  # reads YAML
    {:ok, assign(socket, dashboard: dashboard)}
  end

  def render(assigns) do
    ~H"""
    <div class="iron-grid" style={"grid-template-columns: repeat(#{@dashboard.columns}, 1fr)"}>
      <%= for widget <- @dashboard.widgets do %>
        <div style={grid_position(widget.position, widget.size)}>
          <.live_component
            module={widget_module(widget.type)}
            id={widget_id(widget)}
            tag={widget.tag}
            config={widget.config}
          />
        </div>
      <% end %>
    </div>
    """
  end
end
```

### Trend charts

Trend charts deserve special attention — they are the most data-intensive widget.

**Runtime rendering:** LiveView pushes data points to the client. The chart itself
is rendered by a lightweight JS library (e.g., uPlot — 45KB, no dependencies,
handles 1M+ points). Mounted as a `phx-hook`:

```javascript
// assets/js/hooks/trend_chart.js
Hooks.TrendChart = {
  mounted() {
    this.chart = new uPlot(this.chartOpts(), this.el)
    this.handleEvent("trend-data", ({ points }) => {
      this.chart.setData(points)
    })
  }
}
```

**Data source:** TimescaleDB continuous aggregates provide pre-computed
downsampled data. A 30-day trend query returns in 3–8ms — fast enough
for LiveView to push on mount without noticeable delay.

```
Zoom level       Data source                   Points
──────────────────────────────────────────────────────
Last 1 hour      Raw hypertable                ~3,600
Last 24 hours    1-minute aggregate            ~1,440
Last 7 days      5-minute aggregate            ~2,016
Last 30 days     1-hour aggregate              ~720
Last 1 year      1-day aggregate               ~365
```

---

## Layer 2 — SVG Mnemonic Diagrams (LiveView + phx-hook)

### What it is

A full P&ID-style mnemonic diagram drawn as SVG, with live data overlaid
by a lightweight JavaScript hook. The SVG is static (the drawing).
The data is dynamic (tag values injected by LiveView).

### Who it serves

- **Level 2 users:** draw SVG in Inkscape/Figma, add data-attributes, commit to Git
- **Level 3 users:** create custom SVG components with complex animations
- **AI-assisted:** LLMs generate SVG with correct `data-iron-tag` attributes from
  natural language descriptions

### The binding contract

Any SVG element can be bound to an IRON tag using `data-iron-*` attributes:

```svg
<!-- assets/mimics/reactor_01.svg -->
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1200 800">

  <!-- Static: plant drawing, labels, pipes (unchanged at runtime) -->
  <rect class="pipe" x="100" y="300" width="400" height="20" fill="#666"/>
  <text x="350" y="60" class="title">Reactor 01</text>

  <!-- Dynamic: text value bound to a tag -->
  <text data-iron-tag="reactor_01.temperature"
        data-iron-format="%.1f °C"
        x="340" y="220"
        class="tag-value">--.-</text>

  <!-- Dynamic: color changes based on boolean tag -->
  <circle data-iron-tag="pump_01.running"
          data-iron-class-true="fill-green-500"
          data-iron-class-false="fill-gray-400"
          cx="150" cy="300" r="25"/>

  <!-- Dynamic: CSS animation toggled by tag -->
  <g data-iron-tag="pump_01.running"
     data-iron-animate-true="spin 2s linear infinite"
     data-iron-animate-false="none">
    <circle cx="150" cy="300" r="20" fill="none" stroke="#333" stroke-width="2"/>
    <line x1="150" y1="280" x2="150" y2="300" stroke="#333" stroke-width="2"/>
  </g>

  <!-- Dynamic: fill height proportional to tag value (tank level) -->
  <rect data-iron-tag="tank_01.level"
        data-iron-fill="vertical"
        data-iron-range="[0, 100]"
        x="500" y="100" width="100" height="300"
        class="tank-fill"/>

  <!-- Dynamic: pipe color based on flow (gradient) -->
  <rect data-iron-tag="flow_01.rate"
        data-iron-color-map="[[0, '#666'], [50, '#3b82f6'], [100, '#22c55e']]"
        x="100" y="300" width="400" height="20"/>

  <!-- Dynamic: visibility controlled by tag -->
  <g data-iron-tag="valve_01.alarm"
     data-iron-visible-true="true"
     data-iron-visible-false="false">
    <text x="600" y="400" fill="red" class="alarm-flash">⚠ ALARM</text>
  </g>

  <!-- Interactive: click sends command (WRITE path) -->
  <rect data-iron-command="pump_01.start_cmd"
        data-iron-command-value="true"
        data-iron-requires-role="operator"
        x="120" y="350" width="60" height="30" rx="4"
        class="command-button" cursor="pointer"/>
  <text x="150" y="370" text-anchor="middle" class="button-label">START</text>

</svg>
```

### Supported data-iron-* attributes

| Attribute | Purpose | Example |
|---|---|---|
| `data-iron-tag` | Bind element to a tag | `"reactor_01.temperature"` |
| `data-iron-format` | Printf-style format for text | `"%.1f °C"` |
| `data-iron-class-true/false` | CSS class toggled by boolean | `"fill-green-500"` |
| `data-iron-animate-true/false` | CSS animation toggled by boolean | `"spin 2s linear infinite"` |
| `data-iron-fill` | Fill direction for level display | `"vertical"` or `"horizontal"` |
| `data-iron-range` | Value range for fill/color mapping | `"[0, 100]"` |
| `data-iron-color-map` | Value-to-color gradient mapping | `"[[0,'#666'],[100,'#0f0']]"` |
| `data-iron-visible-true/false` | Show/hide based on boolean | `"true"` / `"false"` |
| `data-iron-quality` | Show quality badge | `"badge"` or `"border"` |
| `data-iron-command` | Click sends command (WRITE path) | `"pump_01.start_cmd"` |
| `data-iron-command-value` | Value to send on click | `"true"` |
| `data-iron-requires-role` | Minimum role for command | `"operator"` |

### How it works technically

```
┌────────────────────────────────────────────────────────┐
│  Browser                                               │
│                                                        │
│  ┌─────────────────────────────┐                       │
│  │  SVG DOM                    │                       │
│  │  (static drawing +          │  ◄── IronMimic hook   │
│  │   data-iron-* elements)     │      updates DOM      │
│  └─────────────────────────────┘      on pushEvent     │
│                                          ▲             │
└──────────────────────────────────────────│─────────────┘
                                           │ WebSocket
┌──────────────────────────────────────────│─────────────┐
│  iron-web (Phoenix)                      │             │
│                                          │             │
│  MimicLive (LiveView)                    │             │
│    │                                     │             │
│    ├─ mount: load SVG, extract tags      │             │
│    ├─ subscribe to each tag via PubSub   │             │
│    └─ on tag_update: push_event ─────────┘             │
│         to IronMimic hook with                         │
│         {tag, value, quality}                          │
│                                                        │
└────────────────────────────────────────────────────────┘
```

**LiveView (server side):**

```elixir
defmodule IronWeb.MimicLive do
  use IronWeb, :live_view

  def mount(%{"id" => mimic_id}, _session, socket) do
    mimic = Iron.Mimics.load(mimic_id)   # loads SVG file
    tags = Iron.Mimics.extract_tags(mimic) # parses data-iron-tag attributes

    if connected?(socket) do
      Enum.each(tags, fn tag ->
        Phoenix.PubSub.subscribe(Iron.PubSub, "tag:#{tag}")
      end)
    end

    {:ok, assign(socket, mimic_svg: mimic.svg, tags: tags)}
  end

  def handle_info({:tag_update, %{tag: tag, value: value, quality: quality}}, socket) do
    push_event(socket, "tag-update", %{tag: tag, value: value, quality: quality})
    {:noreply, socket}
  end

  def render(assigns) do
    ~H"""
    <div id="iron-mimic"
         phx-hook="IronMimic"
         phx-update="ignore">
      <%= raw(@mimic_svg) %>
    </div>
    """
  end
end
```

**JavaScript hook (client side, ~80 lines):**

```javascript
// assets/js/hooks/iron_mimic.js
Hooks.IronMimic = {
  mounted() {
    this.handleEvent("tag-update", ({ tag, value, quality }) => {
      this.el.querySelectorAll(`[data-iron-tag="${tag}"]`).forEach(el => {
        this.updateElement(el, value, quality)
      })
    })
  },

  updateElement(el, value, quality) {
    const format = el.dataset.ironFormat

    // Text value display
    if (format && el.tagName === "text") {
      el.textContent = this.formatValue(value, format)
    }

    // Boolean class toggle
    const classTrue = el.dataset.ironClassTrue
    const classFalse = el.dataset.ironClassFalse
    if (classTrue || classFalse) {
      el.setAttribute("class", value ? classTrue : classFalse)
    }

    // Animation toggle
    const animTrue = el.dataset.ironAnimateTrue
    const animFalse = el.dataset.ironAnimateFalse
    if (animTrue || animFalse) {
      el.style.animation = value ? animTrue : animFalse
    }

    // Fill level (vertical/horizontal)
    const fill = el.dataset.ironFill
    if (fill) {
      const range = JSON.parse(el.dataset.ironRange || "[0,100]")
      const pct = ((value - range[0]) / (range[1] - range[0])) * 100
      if (fill === "vertical") {
        el.setAttribute("height", `${pct}%`)
      } else {
        el.setAttribute("width", `${pct}%`)
      }
    }

    // Visibility toggle
    const visTrue = el.dataset.ironVisibleTrue
    if (visTrue !== undefined) {
      el.style.display = value ? "" : "none"
    }

    // Quality indicator
    el.dataset.ironQualityState = quality
  },

  formatValue(value, format) {
    if (typeof value === "boolean") return value ? "ON" : "OFF"
    if (typeof value === "number") {
      const match = format.match(/%\.(\d+)f(.*)/)
      if (match) return value.toFixed(parseInt(match[1])) + match[2]
    }
    return String(value)
  }
}
```

### The SVG workflow

```
Step 1 — Draw                    Step 2 — Bind                  Step 3 — Deploy
─────────────────               ──────────────                 ─────────────────
Inkscape / Figma /              Add data-iron-tag              iron validate
draw.io / AI                    attributes to                  iron dev
                                SVG elements
Create plant layout             (manually or via               See live data on
with equipment symbols          iron bind command)             mnemonic diagram
         │                              │                              │
         ▼                              ▼                              ▼
  reactor_01.svg               reactor_01.svg               http://localhost:4000
  (static drawing)         (drawing + data bindings)        /mimic/reactor_01
```

**AI-assisted SVG generation:**

```bash
# Engineer describes the process to AI
# AI generates SVG with correct data-iron-tag attributes
# iron validate checks that all referenced tags exist

iron validate --mimics
# ✅ reactor_01.svg — 12 bindings, all tags exist
# ⚠️  reactor_01.svg — pump_02.running bound but not in alarm config
# ❌ tank_line.svg  — tag "tank_03.level" does not exist in config/tags/
```

### Standard symbol library

IRON ships with SVG symbols for common industrial equipment:

```
assets/symbols/
  pumps/
    centrifugal.svg      — with data-iron-tag placeholder for running state
    positive_disp.svg
  valves/
    gate.svg             — with position animation (0–100%)
    ball.svg
    control.svg
  tanks/
    vertical.svg         — with level fill animation
    horizontal.svg
  motors/
    motor.svg            — with rotation animation
  pipes/
    horizontal.svg
    vertical.svg
    elbow_90.svg
    tee.svg
  instruments/
    temperature.svg
    pressure.svg
    flow.svg
    level.svg
  indicators/
    quality_badge.svg    — GOOD/UNCERTAIN/BAD
    alarm_flash.svg
```

Engineers compose mimics by placing these symbols in Inkscape and connecting
them with pipe segments. Each symbol already has the correct `data-iron-*`
attribute placeholders — the engineer only fills in the tag name.

---

## Layer 3 — Visual SVG Editor (React Island)

### What it is

A browser-based drag-and-drop mnemonic editor. The engineer drags equipment
symbols from a palette, places them on a canvas, draws pipes between them,
and binds each symbol to a tag. The output is an SVG file with `data-iron-tag`
attributes — the same format that Layer 2 consumes.

### Who it serves

- **Level 1 users:** build mnemonic diagrams without any external tool
- **System integrators:** rapid screen development during client meetings

### Why React (and only here)

A visual editor is a fundamentally different application from a real-time dashboard:

| Property | SCADA runtime (Layers 1–2) | Visual editor (Layer 3) |
|---|---|---|
| Users | Many operators simultaneously | One engineer at a time |
| State | Server-authoritative (tag values) | Client-authoritative (canvas state) |
| Updates | Push from server (PubSub) | Local, high-frequency mouse events |
| DOM | Reactive diffs (LiveView) | Imperative manipulation (drag/drop/resize) |
| Latency tolerance | 50–200ms acceptable | Must feel instant (<16ms) |

LiveView sends every interaction to the server and waits for a diff response.
For a drag operation generating 60 mouse events per second, this creates
unacceptable latency. The editor must maintain local state.

React is used **only for the editor**. The operator never sees React.
The runtime display of mimics is pure LiveView + a small JS hook (Layer 2).

### Architecture

```
┌──────────────────────────────────────────────────────────┐
│  Browser — Editor Mode (engineer only)                   │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │  React Island (phx-hook mount)                     │  │
│  │                                                    │  │
│  │  ┌──────────┐  ┌──────────────────────────────┐   │  │
│  │  │ Symbol   │  │  Canvas                       │   │  │
│  │  │ Palette  │  │                               │   │  │
│  │  │          │  │  [drag symbols, draw pipes,   │   │  │
│  │  │ ⬡ Pump  │  │   resize, align, group]       │   │  │
│  │  │ ◇ Valve │  │                               │   │  │
│  │  │ □ Tank  │  │  Local state, 60fps            │   │  │
│  │  │ ○ Motor │  │                               │   │  │
│  │  └──────────┘  └──────────────────────────────┘   │  │
│  │                                                    │  │
│  │  ┌──────────────────────────────────────────────┐  │  │
│  │  │ Property Panel                               │  │  │
│  │  │ Selected: pump_01                            │  │  │
│  │  │ Bind to tag: [reactor_01.pump_01.running ▾]  │  │  │
│  │  │ Animation: [rotate when true           ▾]   │  │  │
│  │  │ Label: [Pump 01                          ]   │  │  │
│  │  └──────────────────────────────────────────────┘  │  │
│  └────────────────────────────────────────────────────┘  │
│         │                                                │
│         │ pushEvent("save-mimic", svg_with_bindings)     │
│         ▼                                                │
│  LiveView receives SVG → saves to config/mimics/ → Git  │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### LiveView ↔ React communication

```elixir
# LiveView host for the editor
defmodule IronWeb.EditorLive do
  use IronWeb, :live_view

  def mount(%{"id" => mimic_id}, _session, socket) do
    mimic = Iron.Mimics.load(mimic_id)
    tags = Iron.Tags.list_all()  # available tags for binding
    symbols = Iron.Symbols.list_all()  # symbol library

    {:ok, assign(socket, mimic: mimic, tags: tags, symbols: symbols)}
  end

  # React pushes the edited SVG back to LiveView
  def handle_event("save-mimic", %{"svg" => svg, "id" => id}, socket) do
    Iron.Mimics.save(id, svg)  # writes to config/mimics/
    {:noreply, put_flash(socket, :info, "Mnemonic saved")}
  end

  def render(assigns) do
    ~H"""
    <div id="mimic-editor"
         phx-hook="MimicEditor"
         data-mimic={Jason.encode!(@mimic)}
         data-tags={Jason.encode!(@tags)}
         data-symbols={Jason.encode!(@symbols)}>
    </div>
    """
  end
end
```

```javascript
// React component mounted via phx-hook
Hooks.MimicEditor = {
  mounted() {
    const mimic = JSON.parse(this.el.dataset.mimic)
    const tags = JSON.parse(this.el.dataset.tags)
    const symbols = JSON.parse(this.el.dataset.symbols)

    // Mount React editor into this element
    this.root = createRoot(this.el)
    this.root.render(
      <MimicEditorApp
        initialMimic={mimic}
        availableTags={tags}
        availableSymbols={symbols}
        onSave={(svg) => this.pushEvent("save-mimic", { svg, id: mimic.id })}
      />
    )
  },

  destroyed() {
    this.root.unmount()
  }
}
```

### Editor features (Phase 3)

```
Basic (Phase 3 start):
  ✓ Drag symbols from palette to canvas
  ✓ Select, move, resize symbols
  ✓ Property panel: bind symbol to tag
  ✓ Save as SVG with data-iron-tag attributes
  ✓ Preview: switch to runtime mode (LiveView takes over)

Advanced (Phase 3 end):
  ✓ Draw pipes between symbols (auto-routing)
  ✓ Snap-to-grid, alignment guides
  ✓ Group/ungroup
  ✓ Undo/redo (Ctrl+Z)
  ✓ Copy/paste between mimics
  ✓ Symbol editor: create custom symbols
  ✓ Templates: pre-built screen layouts (pump station, boiler room, etc.)
```

---

## How the Layers Map to User Levels

```
                    Layer 1             Layer 2              Layer 3
                    Widget Dashboard    SVG Mimic            SVG Editor
────────────────────────────────────────────────────────────────────────
Level 1             Grid constructor    —                    Drag-and-drop
(Mouse only)        in browser                               editor in browser

Level 2             YAML config         SVG in Inkscape +    —
(YAML)              in Git              data-iron-tag attrs

Level 3             Custom LiveView     Custom phx-hook +    Extend editor
(Code)              components          SVG animations       with new symbols
────────────────────────────────────────────────────────────────────────
Technology          Pure LiveView       LiveView + JS hook   React island
                                        (~80 lines)          (phx-hook)

Runtime JS          0 lines             ~80 lines            0 (editor only,
on operator screen                                           not runtime)
────────────────────────────────────────────────────────────────────────
Phase               Phase 1             Phase 2              Phase 3
```

---

## Spec-Driven Integration

The visual system is fully integrated with IRON's spec-driven approach:

### iron validate --mimics

```bash
iron validate --mimics

# Checks all SVG files in config/mimics/ and config/dashboards/
✅ dashboards/reactor_01.yaml — 8 widgets, all tags exist
✅ mimics/reactor_01.svg — 14 bindings, all tags exist
⚠️  mimics/pump_station.svg — pump_03.running bound but tag has no alarms
❌ mimics/tank_line.svg — data-iron-tag="tank_03.level" references
   nonexistent tag (did you mean "tank_01.level"?)
❌ dashboards/overview.yaml — widget references tag "boiler.temp"
   but no such tag in config/tags/
```

### iron generate dashboard

```bash
# Generate a default dashboard from tag configuration
iron generate dashboard reactor_01

# Creates config/dashboards/reactor_01.yaml with:
#   - gauge for each analog tag
#   - status indicator for each digital tag
#   - trend chart for all analog tags
#   - alarm panel filtered to reactor_01.*
#   - command buttons for each command in config/commands/reactor_01.yaml
```

### iron generate mimic (AI-assisted)

```bash
# Generate an SVG mnemonic from tag configuration + description
iron generate mimic reactor_01 --description "Chemical reactor with
  inlet pump, heating jacket, temperature sensor, outlet valve,
  and product tank"

# AI generates reactor_01.svg with:
#   - Equipment symbols from the standard library
#   - Pipe connections between components
#   - data-iron-tag attributes for all reactor_01.* tags
#   - Layout matching the described process flow

iron validate --mimics
# ✅ mimics/reactor_01.svg — 12 bindings, all tags exist
```

---

## Trade-offs

### What this design gives up

**No real-time collaborative editing.** The SVG editor (Layer 3) is single-user.
Two engineers cannot edit the same mimic simultaneously. This is acceptable —
mnemonic design is not a collaborative task in practice. Version control (Git)
handles merge conflicts if needed.

**SVG, not Canvas/WebGL.** SVG scales poorly above ~5,000 DOM elements.
A very large mnemonic (entire refinery on one screen) may need to be split
into linked screens. In practice, operators work with one process area
at a time — screens with 100–500 elements, which SVG handles easily.

**No proprietary symbol format.** IRON uses standard SVG. This means symbols
from other SCADA systems (WinCC, Ignition) cannot be imported directly.
However, SVG is the universal vector format — conversion from other formats
is straightforward and can be automated.

**Trend charts use a JS library.** uPlot is a runtime JavaScript dependency
for Layer 1 trend charts. This is the one exception to the "no JavaScript
in runtime" principle. The alternative — server-rendered chart images — would
be worse in every dimension (latency, interactivity, bandwidth). uPlot is
45KB, has zero dependencies, and handles 1M+ data points.

### What this design preserves

**LiveView for 95% of runtime rendering.** The operator's screen is served
by LiveView. Real-time updates flow through PubSub. Reconnection, state
recovery, and diff-based DOM updates are handled by the framework.
The JS hook for Layer 2 is ~80 lines of DOM manipulation — not a framework.

**Spec-driven.** Dashboards are YAML. Mimics are SVG with data-attributes.
Both are text files in Git. Both are validated by `iron validate`.
Both can be generated by AI and checked by a deterministic tool.

**READ/WRITE separation.** Display elements (Layer 1 + Layer 2) use the READ
path only. Command widgets and `data-iron-command` elements go through the
WRITE path — authentication, RBAC check, audit log, confirmation dialog.
A rendering bug cannot send a command.

---

## Alternatives Considered

### Full React SPA (rejected)

The Ignition approach: entire frontend is a custom Java/JS application.

Rejected because:
- Two separate applications (API + SPA) instead of one (LiveView)
- Real-time sync requires manual WebSocket management
- Doubles the skill set required (Elixir + React instead of mostly Elixir)
- Contradicts "developer and engineer are equal" — React is harder
  for non-developers than LiveView templates

### Canvas/WebGL rendering (rejected)

Games-style rendering: everything drawn on a `<canvas>` element.

Rejected because:
- Accessibility: canvas content is invisible to screen readers
- Selection/interaction requires reimplementing what the browser gives for free with SVG
- Text rendering is worse than SVG/HTML
- Overkill for 100–500 element screens (SVG handles this fine)
- Would be considered for 10,000+ element screens (not IRON's target)

### Grafana embedding (rejected)

Use Grafana for dashboards and trends.

Rejected because:
- Grafana is an observability tool, not a SCADA tool
- No mnemonic diagrams, no command buttons, no alarm management
- Adds a separate system to deploy, configure, and maintain
- Useful as a complementary tool (mentioned in Level 3 docs) but not as the primary UI

### Server-rendered SVG with LiveView only (rejected)

No JS hook. LiveView re-renders the entire SVG on every tag update.

Rejected because:
- A mnemonic with 200 bound elements updating at 10Hz generates 2,000
  LiveView diffs per second. Each diff includes the full SVG path data.
- The JS hook approach pushes only `{tag, value, quality}` — 3 fields.
  The hook updates the DOM directly. 100x less data over the WebSocket.

---

## Implementation Phases

```
Phase 1 (Months 1–6):
  ✓ Widget library: gauge, numeric, status, trend, alarm_panel, command_button
  ✓ Dashboard YAML configuration
  ✓ Grid constructor in browser (drag widgets, bind tags)
  ✓ uPlot integration for trend charts

Phase 2 (Months 6–12):
  ✓ IronMimic phx-hook for SVG binding
  ✓ data-iron-* attribute specification (this document)
  ✓ Standard symbol library (20+ equipment types)
  ✓ iron validate --mimics
  ✓ iron generate dashboard
  ✓ Widget library expansion: tank_level, bar, sparkline, slider, setpoint

Phase 3 (Months 12–24):
  ✓ React SVG editor (basic: drag, place, bind, save)
  ✓ Pipe drawing with auto-routing
  ✓ Symbol editor (create custom symbols)
  ✓ Templates for common process types
  ✓ iron generate mimic (AI-assisted)
  ✓ Editor features: undo/redo, snap, align, group
```

---

## Open Questions

**Symbol standardization:** should IRON adopt ISA-5.1 (P&ID symbols) as the
default symbol style, or create a modern flat/outline style that is more
readable on screens? ISA-5.1 is the industry standard but was designed for
paper drawings, not 1920×1080 monitors.

**Mobile rendering:** mnemonic diagrams designed for 1920×1080 do not work
on a phone. Should IRON auto-generate simplified mobile views from the same
mnemonic, or require separate mobile dashboards? The widget dashboard (Layer 1)
adapts to mobile naturally. SVG mimics do not.

**Animation performance:** CSS animations (rotation, pulsing, fill level) are
efficient. Complex animations (fluid flow through pipes, particle effects)
may require `requestAnimationFrame` in the JS hook. Where is the line between
"useful industrial visualization" and "unnecessary eye candy"?

**Offline mimic viewing:** should mimics work offline (service worker caching
the SVG and last-known tag values)? Useful for operators in areas with
poor connectivity, but adds complexity.
