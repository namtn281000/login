<template>
  <aside class="side-hud" :class="['side-hud--' + side, { 'is-lite': lite }]" aria-hidden="true">
    <!-- LEFT: data rain, radar, spectrum -->
    <template v-if="side === 'left'">
      <section class="panel" style="--i: 0">
        <header class="panel__head">
          <span>Data stream</span>
          <b>{{ rate }} GB/s</b>
        </header>
        <canvas ref="canvas" class="panel__canvas panel__canvas--rain" />
      </section>

      <section class="panel" style="--i: 1">
        <header class="panel__head">
          <span>Radar sweep</span>
          <b>{{ blips.length }} contacts</b>
        </header>
        <svg class="radar" viewBox="0 0 120 120">
          <defs>
            <linearGradient id="side-hud-sweep" x1="0" y1="0" x2="1" y2="0">
              <stop offset="0" stop-color="#5CCBFF" stop-opacity="0" />
              <stop offset="1" stop-color="#5CCBFF" stop-opacity="0.55" />
            </linearGradient>
          </defs>
          <circle class="radar__ring" cx="60" cy="60" r="18" />
          <circle class="radar__ring" cx="60" cy="60" r="36" />
          <circle class="radar__ring radar__ring--outer" cx="60" cy="60" r="56" />
          <circle class="radar__ticks" cx="60" cy="60" r="56" />
          <line class="radar__axis" x1="60" y1="4" x2="60" y2="116" />
          <line class="radar__axis" x1="4" y1="60" x2="116" y2="60" />
          <g class="radar__sweep">
            <path d="M60 60 L17.1 24 A56 56 0 0 1 60 4 Z" fill="url(#side-hud-sweep)" />
            <line x1="60" y1="60" x2="60" y2="4" stroke="#8FE9FF" stroke-width="1" />
          </g>
          <circle
            v-for="(b, i) in blips"
            :key="i"
            class="blip"
            :class="{ 'blip--amber': i % 3 === 1 }"
            :cx="b.x"
            :cy="b.y"
            r="2.3"
            :style="{ animationDelay: b.delay + 's' }"
          />
        </svg>
      </section>

      <section class="panel panel--opt" style="--i: 2">
        <header class="panel__head">
          <span>Spectrum</span>
          <b>Live</b>
        </header>
        <div class="eq">
          <i v-for="bar in eqBars" :key="bar.n" :style="bar.style" />
        </div>
      </section>
    </template>

    <!-- RIGHT: signal wave, link gauges, event log -->
    <template v-else>
      <section class="panel" style="--i: 0">
        <header class="panel__head">
          <span>Signal</span>
          <b>{{ latency }} ms</b>
        </header>
        <canvas ref="canvas" class="panel__canvas panel__canvas--wave" />
      </section>

      <section class="panel" style="--i: 1">
        <header class="panel__head">
          <span>Link status</span>
          <b>Stable</b>
        </header>
        <ul class="gauges">
          <li v-for="g in gauges" :key="g.k">
            <div class="gauge__row">
              <span>{{ g.k }}</span>
              <b>{{ g.v }}%</b>
            </div>
            <div class="seg"><i :class="{ hot: g.v > 90 }" :style="{ width: g.v + '%' }" /></div>
          </li>
        </ul>
      </section>

      <section class="panel panel--opt" style="--i: 2">
        <header class="panel__head">
          <span>Event log</span>
          <b>Live</b>
        </header>
        <transition-group name="log" tag="ul" class="log">
          <li v-for="line in log" :key="line.id">
            <span class="log__t">{{ line.t }}</span> {{ line.text }}
          </li>
        </transition-group>
      </section>
    </template>
  </aside>
</template>

<script>
const HEX = '0123456789ABCDEF'
const NODES = ['SIN', 'LHR', 'JFK', 'SYD', 'GRU', 'HND', 'DXB', 'SFO', 'FRA', 'JNB']
const EVENTS = ['SYNC OK', 'ACK', 'HANDSHAKE', 'AUTH OK', 'ROUTE UPD', 'KEY ROTATE', 'PING 14ms', 'HEARTBEAT']
const RADAR_PERIOD = 4 // seconds per sweep, keep in sync with the CSS animation

// Cheap deterministic hash in [0, 1)
const hash = (i, k = 1) => {
  const s = Math.sin(i * 12.9898 * k + k * 78.233) * 43758.5453
  return s - Math.floor(s)
}
const pad = (n) => (n < 10 ? '0' + n : '' + n)

export default {
  name: 'SideHud',

  props: {
    /** Which column to render */
    side: { type: String, default: 'left', validator: (v) => v === 'left' || v === 'right' },
    /** Stops all motion (used when the globe had to lower its quality) */
    lite: { type: Boolean, default: false }
  },

  data() {
    return {
      rate: '4.82',
      latency: 18,
      gauges: [
        { k: 'Uplink', v: 72 },
        { k: 'Throughput', v: 58 },
        { k: 'Integrity', v: 96 }
      ],
      log: []
    }
  },

  computed: {
    // Radar contacts: a = angle clockwise from the top (deg), r = distance from the centre.
    // delay syncs each blip's flash with the moment the sweep line passes over it.
    blips() {
      return [
        { a: 28, r: 30 }, { a: 95, r: 44 }, { a: 160, r: 22 },
        { a: 212, r: 48 }, { a: 285, r: 36 }, { a: 330, r: 14 }
      ].map((b) => {
        const rad = (b.a * Math.PI) / 180
        return {
          x: +(60 + b.r * Math.sin(rad)).toFixed(1),
          y: +(60 - b.r * Math.cos(rad)).toFixed(1),
          delay: +((b.a / 360) * RADAR_PERIOD - RADAR_PERIOD).toFixed(2)
        }
      })
    },
    eqBars() {
      return Array.from({ length: 24 }, (_, i) => ({
        n: i,
        style: {
          animationDuration: (0.7 + hash(i + 1, 5) * 1.3).toFixed(2) + 's',
          animationDelay: (-hash(i + 1, 6) * 2).toFixed(2) + 's'
        }
      }))
    }
  },

  watch: {
    lite() {
      this.run()
      this.startTimers()
    }
  },

  created() {
    // Non-reactive canvas / loop state
    this.ctx = null
    this.cw = 0
    this.ch = 0
    this.dpr = 1
    this.raf = 0
    this.t0 = 0
    this.lastDraw = 0
    this.timers = []
    this.logId = 0
    this.mq = window.matchMedia('(prefers-reduced-motion: reduce)')

    if (this.side === 'right') {
      for (let i = 0; i < 5; i++) this.pushLog()
    }
  },

  mounted() {
    this.ctx = this.$refs.canvas.getContext('2d')
    this.t0 = performance.now()

    this.ro = new ResizeObserver(this.onResize)
    this.ro.observe(this.$refs.canvas)
    if (this.mq.addEventListener) this.mq.addEventListener('change', this.onMotionChange)

    this.onResize()
    this.run()
    this.startTimers()
  },

  beforeDestroy() {
    cancelAnimationFrame(this.raf)
    this.stopTimers()
    if (this.ro) this.ro.disconnect()
    if (this.mq.removeEventListener) this.mq.removeEventListener('change', this.onMotionChange)
  },

  methods: {
    /* ---------- loop ---------- */
    onMotionChange() {
      this.run()
      this.startTimers()
    },

    onResize() {
      const rect = this.$refs.canvas.getBoundingClientRect()
      this.cw = Math.max(1, Math.round(rect.width))
      this.ch = Math.max(1, Math.round(rect.height))
      this.dpr = Math.min(window.devicePixelRatio || 1, 1.5)
      this.$refs.canvas.width = Math.round(this.cw * this.dpr)
      this.$refs.canvas.height = Math.round(this.ch * this.dpr)
      if (!this.raf) this.draw(2) // keep a still frame visible when not animating
    },

    run() {
      cancelAnimationFrame(this.raf)
      this.raf = 0
      if (this.mq.matches || this.lite) this.draw(2)
      else this.raf = requestAnimationFrame(this.tick)
    },

    // The side panels are secondary, so they draw at about 30 fps
    tick(now) {
      if (now - this.lastDraw >= 32) {
        this.lastDraw = now
        this.draw((now - this.t0) / 1000)
      }
      this.raf = requestAnimationFrame(this.tick)
    },

    draw(T) {
      if (!this.ctx || !this.cw) return
      if (this.side === 'left') this.drawRain(T)
      else this.drawWave(T)
    },

    /* ---------- timers (numbers and log) ---------- */
    startTimers() {
      this.stopTimers()
      if (this.mq.matches || this.lite) return
      this.timers.push(setInterval(this.tickStats, 1100))
      if (this.side === 'right') this.timers.push(setInterval(this.pushLog, 1500))
    },

    stopTimers() {
      this.timers.forEach((t) => clearInterval(t))
      this.timers = []
    },

    tickStats() {
      if (this.side === 'left') {
        this.rate = (3.2 + Math.random() * 3.4).toFixed(2)
        return
      }
      this.latency = 12 + Math.round(Math.random() * 14)
      this.gauges = this.gauges.map((g) => ({
        k: g.k,
        v: Math.round(Math.max(38, Math.min(99, g.v + (Math.random() - 0.5) * 26)))
      }))
    },

    pushLog() {
      const now = new Date()
      const a = NODES[Math.floor(Math.random() * NODES.length)]
      let b = NODES[Math.floor(Math.random() * NODES.length)]
      if (b === a) b = NODES[(NODES.indexOf(a) + 1) % NODES.length]
      const line = {
        id: ++this.logId,
        t: pad(now.getUTCHours()) + ':' + pad(now.getUTCMinutes()) + ':' + pad(now.getUTCSeconds()),
        text: a + '>' + b + ' ' + EVENTS[Math.floor(Math.random() * EVENTS.length)]
      }
      this.log = this.log.slice(-5).concat(line)
    },

    /* ---------- canvas: falling hex columns ---------- */
    drawRain(T) {
      const { ctx, cw, ch, dpr } = this
      ctx.setTransform(dpr, 0, 0, dpr, 0, 0)
      ctx.clearRect(0, 0, cw, ch)

      const CW = 12 // column width
      const RH = 13 // row height
      const TRAIL = 10
      const cols = Math.max(1, Math.floor(cw / CW))
      const rows = Math.ceil(ch / RH)
      const left = (cw - cols * CW) / 2 + CW / 2
      const tick = Math.floor(T * 3) // characters re-roll three times a second

      ctx.font = '10px "JetBrains Mono", ui-monospace, monospace'
      ctx.textAlign = 'center'
      ctx.textBaseline = 'top'

      for (let c = 0; c < cols; c++) {
        const speed = 3 + hash(c + 1, 1) * 7 // rows per second
        const cycle = rows + TRAIL + 4 + Math.floor(hash(c + 1, 2) * 10)
        const head = Math.floor(T * speed + hash(c + 1, 5) * cycle) % cycle
        const rgb = hash(c + 1, 4) < 0.14 ? '255,180,84' : '110,215,255'
        for (let k = 0; k < TRAIL; k++) {
          const r = head - k
          if (r < 0 || r >= rows) continue
          const a = k === 0 ? 1 : Math.pow(1 - k / TRAIL, 1.7) * 0.8
          const g = HEX[Math.floor(hash(c * 131 + r + 1, 7 + ((tick + c) % 5)) * 16)]
          ctx.fillStyle = k === 0 ? 'rgba(235,250,255,0.95)' : 'rgba(' + rgb + ',' + a.toFixed(2) + ')'
          ctx.fillText(g, left + c * CW, r * RH)
        }
      }
    },

    /* ---------- canvas: scrolling signal ---------- */
    drawWave(T) {
      const { ctx, cw, ch, dpr } = this
      ctx.setTransform(dpr, 0, 0, dpr, 0, 0)
      ctx.clearRect(0, 0, cw, ch)

      // grid (vertical lines drift left)
      ctx.lineWidth = 1
      ctx.strokeStyle = 'rgba(130,200,255,.09)'
      ctx.beginPath()
      for (let i = 1; i < 4; i++) {
        const y = Math.round((ch * i) / 4) + 0.5
        ctx.moveTo(0, y)
        ctx.lineTo(cw, y)
      }
      const gap = 24
      const shift = (T * 22) % gap
      for (let x = -shift; x < cw; x += gap) {
        const xr = Math.round(x) + 0.5
        ctx.moveTo(xr, 0)
        ctx.lineTo(xr, ch)
      }
      ctx.stroke()

      const mid = ch * 0.5
      const amp = ch * 0.36
      const n = Math.ceil(cw / 3) + 1
      const y1 = (x) => {
        const u = x + T * 58
        return mid - amp * (0.55 * Math.sin(u * 0.05) + 0.3 * Math.sin(u * 0.13 + 1.3) + 0.15 * Math.sin(u * 0.37 + T * 4))
      }
      const y2 = (x) => {
        const u = x + T * 35
        return mid + amp * 0.45 * (0.6 * Math.sin(u * 0.08 + 0.7) + 0.4 * Math.sin(u * 0.21))
      }

      // secondary trace (amber)
      ctx.beginPath()
      for (let i = 0; i < n; i++) {
        const x = Math.min(cw, i * 3)
        if (i) ctx.lineTo(x, y2(x))
        else ctx.moveTo(x, y2(x))
      }
      ctx.strokeStyle = 'rgba(255,180,84,.75)'
      ctx.lineWidth = 1
      ctx.stroke()

      // main trace (cyan) with an area fill
      ctx.beginPath()
      for (let i = 0; i < n; i++) {
        const x = Math.min(cw, i * 3)
        if (i) ctx.lineTo(x, y1(x))
        else ctx.moveTo(x, y1(x))
      }
      ctx.strokeStyle = 'rgba(130,230,255,.95)'
      ctx.lineWidth = 1.4
      ctx.stroke()
      ctx.lineTo(cw, ch)
      ctx.lineTo(0, ch)
      ctx.closePath()
      const g = ctx.createLinearGradient(0, 0, 0, ch)
      g.addColorStop(0, 'rgba(110,215,255,.22)')
      g.addColorStop(1, 'rgba(110,215,255,0)')
      ctx.fillStyle = g
      ctx.fill()

      // glowing head
      const hx = cw - 3
      const hy = y1(hx)
      const glow = ctx.createRadialGradient(hx, hy, 0, hx, hy, 9)
      glow.addColorStop(0, 'rgba(180,240,255,.9)')
      glow.addColorStop(1, 'rgba(110,215,255,0)')
      ctx.fillStyle = glow
      ctx.beginPath()
      ctx.arc(hx, hy, 9, 0, Math.PI * 2)
      ctx.fill()
      ctx.fillStyle = '#fff'
      ctx.beginPath()
      ctx.arc(hx, hy, 1.8, 0, Math.PI * 2)
      ctx.fill()
    }
  }
}
</script>

<style scoped>
.side-hud {
  --dx: -22px;
  position: absolute;
  top: 76px;
  bottom: 76px;
  z-index: 1;
  width: clamp(184px, 15.5vw, 244px);
  display: flex;
  flex-direction: column;
  justify-content: center;
  gap: 14px;
  pointer-events: none;
  color: var(--muted, #8d9fb9);
  font: 500 10px/1.4 var(--font-mono, 'JetBrains Mono', ui-monospace, Menlo, Consolas, monospace);
  letter-spacing: 0.12em;
  text-transform: uppercase;
  font-variant-numeric: tabular-nums;
}
.side-hud--left {
  left: clamp(16px, 2.4vw, 40px);
}
.side-hud--right {
  --dx: 22px;
  right: clamp(16px, 2.4vw, 40px);
}

.panel {
  position: relative;
  padding: 10px 12px 12px;
  border: 1px solid var(--line, rgba(150, 185, 230, 0.16));
  border-radius: 10px;
  background: rgba(7, 14, 26, 0.55);
  animation: panel-in 0.8s cubic-bezier(0.2, 0.7, 0.2, 1) both;
  animation-delay: calc(var(--i) * 0.14s + 0.25s);
}
/* HUD corner brackets */
.panel::before,
.panel::after {
  content: '';
  position: absolute;
  width: 12px;
  height: 12px;
  border: 0 solid var(--accent, #5ccbff);
}
.panel::before {
  top: -1px;
  left: -1px;
  border-top-width: 1px;
  border-left-width: 1px;
  border-top-left-radius: 10px;
}
.panel::after {
  right: -1px;
  bottom: -1px;
  border-right-width: 1px;
  border-bottom-width: 1px;
  border-bottom-right-radius: 10px;
}

.panel__head {
  display: flex;
  justify-content: space-between;
  gap: 8px;
  margin-bottom: 8px;
  font-size: 9.5px;
  white-space: nowrap;
}
.panel__head b {
  color: var(--accent, #5ccbff);
  font-weight: 500;
}

.panel__canvas {
  display: block;
  width: 100%;
}
.panel__canvas--rain { height: 170px; }
.panel__canvas--wave { height: 84px; }

/* ---------- radar ---------- */
.radar {
  display: block;
  width: 100%;
  max-width: 150px;
  margin: 0 auto;
}
.radar__ring {
  fill: none;
  stroke: rgba(110, 210, 255, 0.22);
  stroke-width: 0.7;
}
.radar__ring--outer { stroke: rgba(110, 210, 255, 0.4); }
.radar__ticks {
  fill: none;
  stroke: rgba(160, 236, 255, 0.45);
  stroke-width: 2;
  stroke-dasharray: 1 3.887;
}
.radar__axis {
  stroke: rgba(110, 210, 255, 0.16);
  stroke-width: 0.6;
}
.radar__sweep {
  transform-box: view-box;
  transform-origin: 60px 60px;
  animation: sweep 4s linear infinite;
}
.blip {
  fill: #6effd2;
  opacity: 0.75;
  animation: blip 4s linear infinite;
}
.blip--amber { fill: #ffb454; }

/* ---------- spectrum ---------- */
.eq {
  display: flex;
  align-items: flex-end;
  gap: 3px;
  height: 36px;
}
.eq i {
  flex: 1;
  height: 100%;
  border-radius: 1px;
  background: linear-gradient(180deg, #8fe4ff, #2a8fd0);
  opacity: 0.85;
  transform-origin: bottom;
  animation: eq 1.2s ease-in-out infinite alternate;
}

/* ---------- gauges ---------- */
.gauges {
  display: grid;
  gap: 10px;
  margin: 0;
  padding: 0;
  list-style: none;
}
.gauge__row {
  display: flex;
  justify-content: space-between;
  margin-bottom: 4px;
  font-size: 9.5px;
}
.gauge__row b {
  color: var(--fg, #e9f0fa);
  font-weight: 500;
}
.seg {
  height: 6px;
  background: rgba(130, 200, 255, 0.1);
  -webkit-mask-image: repeating-linear-gradient(90deg, #000 0 5px, transparent 5px 7px);
  mask-image: repeating-linear-gradient(90deg, #000 0 5px, transparent 5px 7px);
}
.seg i {
  display: block;
  height: 100%;
  background: linear-gradient(90deg, #3aa8e8, #6ee6ff);
  transition: width 0.9s cubic-bezier(0.3, 0.7, 0.2, 1);
}
.seg i.hot {
  background: linear-gradient(90deg, #ffb454, #ff8a5b);
}

/* ---------- log ---------- */
.log {
  position: relative;
  height: 90px;
  margin: 0;
  padding: 0;
  overflow: hidden;
  list-style: none;
  font-size: 9.5px;
  letter-spacing: 0.04em;
}
.log li {
  width: 100%;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
.log__t { color: var(--faint, #70829f); }
.log-enter-active,
.log-leave-active,
.log-move {
  transition: all 0.45s ease;
}
.log-enter { opacity: 0; transform: translateX(10px); }
.log-leave-to { opacity: 0; transform: translateY(-8px); }
.log-leave-active { position: absolute; }

/* ---------- motion ---------- */
@keyframes panel-in {
  from { opacity: 0; transform: translateX(var(--dx)); }
  to { opacity: 1; transform: none; }
}
@keyframes sweep {
  to { transform: rotate(360deg); }
}
@keyframes blip {
  0% { opacity: 1; }
  70% { opacity: 0.12; }
  100% { opacity: 0.08; }
}
@keyframes eq {
  from { transform: scaleY(0.12); }
  to { transform: scaleY(1); }
}

.is-lite *,
.is-lite *::before,
.is-lite *::after {
  animation-play-state: paused !important;
}

@media (prefers-reduced-motion: reduce) {
  .side-hud *,
  .side-hud *::before,
  .side-hud *::after {
    animation: none !important;
    transition: none !important;
  }
  .eq i { transform: scaleY(0.55); }
}

/* Short windows: keep only the first two panels per side */
@media (max-height: 720px) {
  .panel--opt { display: none; }
}
</style>