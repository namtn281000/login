<template>
  <div ref="wrap" class="earth-globe" :style="{ '--cy': centerY * 100 + '%' }">
    <canvas
      ref="canvas"
      class="earth-globe__canvas"
      role="img"
      aria-label="Rotating Earth with a scanning grid, network links and satellites on orbit rings"
    />

    <div v-if="showHud" class="earth-globe__hud" aria-hidden="true">
      <div class="hud-row">
        <span class="hud-live"><i />Orbit view</span>
        <span>{{ satelliteCount }} satellites · {{ nodeCount }} nodes</span>
      </div>
      <div class="hud-row">
        <span>{{ coordText }}</span>
        <span>UTC {{ clockText }}</span>
      </div>
    </div>
  </div>
</template>

<script>
import { LAND_MASK_B64, MASK_W, MASK_H } from '../js/landMask.js'

const TAU = Math.PI * 2
const DEG = Math.PI / 180
const RING_N = 200 // points per orbit ring
const TRAIL = 18 // segments in a satellite trail
const TRAIL_STEP = 0.05 // radians between trail segments
const LONG_STEPS = 48 // points per long-haul arc
const LINK_STEPS = 14 // points per short network link
const BEACONS_PER_RING = 6
const LIGHT = { x: -0.45, y: 0.5, z: 0.74 } // light direction (view space), from the upper left

// rgb triplets used for glows, blinking nodes and comets
const COLOR = {
  cyan: '130,222,255',
  mint: '110,255,210',
  amber: '255,180,84',
  white: '225,246,255'
}
const NODE_COLORS = [COLOR.amber, COLOR.cyan, COLOR.mint] // nodes cycle through these
const SPARK_COLORS = ['120,235,255', '255,190,100', '150,255,210'] // blinking dots on the surface

// r = radius (in globe radii), inc = inclination, node = rotation around the vertical axis,
// sats: a = start angle, v = angular speed (rad/s, sign = direction), c = rgb colour
const RING_DEFS = [
  {
    r: 1.3, inc: 1.12, node: 0.35, dash: null,
    sats: [{ a: 0.6, v: 0.42, c: COLOR.white, label: 'SAT-01' }]
  },
  {
    r: 1.54, inc: -0.62, node: -0.7, dash: null,
    sats: [
      { a: 2.0, v: -0.27, c: COLOR.cyan },
      { a: 5.2, v: -0.27, c: COLOR.mint, label: 'SAT-03' }
    ]
  },
  {
    r: 1.8, inc: 0.22, node: 0.9, dash: [0.5, 8],
    sats: [
      { a: 3.4, v: 0.17, c: COLOR.amber, label: 'SAT-04' },
      { a: 0.2, v: 0.17, c: COLOR.cyan }
    ]
  }
]

// [code, lat, lon] - network nodes
const NODE_DEFS = [
  ['SIN', 1.36, 103.99], ['LHR', 51.47, -0.45], ['JFK', 40.64, -73.78], ['SYD', -33.94, 151.18],
  ['GRU', -23.43, -46.47], ['HND', 35.55, 139.78], ['DXB', 25.25, 55.36], ['SFO', 37.62, -122.38],
  ['CDG', 49.01, 2.55], ['FRA', 50.03, 8.57], ['JNB', -26.14, 28.25], ['NBO', -1.32, 36.93],
  ['CAI', 30.12, 31.41], ['DEL', 28.56, 77.1], ['BOM', 19.09, 72.87], ['HKG', 22.31, 113.91],
  ['ICN', 37.46, 126.44], ['PEK', 40.08, 116.58], ['LAX', 33.94, -118.41], ['ORD', 41.98, -87.9],
  ['YYZ', 43.68, -79.63], ['MEX', 19.44, -99.07], ['BOG', 4.7, -74.14], ['SCL', -33.39, -70.79],
  ['EZE', -34.82, -58.54], ['LOS', 6.58, 3.32], ['SVO', 55.97, 37.41], ['SGN', 10.82, 106.66],
  ['HAN', 21.22, 105.81], ['PER', -31.94, 115.97]
]
const LABELED = ['SIN', 'HND', 'LHR', 'JFK', 'DXB', 'SYD', 'GRU', 'SFO', 'DEL', 'JNB'] // nodes that get a callout

// [from, to, animation offset, comet colour] - long-haul arcs with a moving comet
const ARC_DEFS = [
  ['SIN', 'LHR', 0.0, COLOR.cyan], ['JFK', 'HND', 0.35, COLOR.white], ['SFO', 'SIN', 0.8, COLOR.cyan],
  ['DXB', 'SYD', 1.15, COLOR.mint], ['GRU', 'CDG', 0.55, COLOR.cyan], ['LAX', 'ICN', 1.4, COLOR.amber],
  ['JNB', 'DEL', 0.2, COLOR.mint], ['YYZ', 'FRA', 0.95, COLOR.white]
]

// Cheap deterministic hash in [0, 1)
const hash = (i, k = 1) => {
  const s = Math.sin(i * 12.9898 * k + k * 78.233) * 43758.5453
  return s - Math.floor(s)
}

export default {
  name: 'EarthGlobe',

  props: {
    /** Globe rotation speed (rad/s) */
    spinSpeed: { type: Number, default: 0.11 },
    /** How far the north pole leans toward the viewer (rad) */
    tilt: { type: Number, default: 0.42 },
    /** Longitude (degrees east) centred on the screen at t = 0 */
    startLongitude: { type: Number, default: 100 },
    showHud: { type: Boolean, default: true },
    /** Latitude / longitude grid on the globe */
    showGrid: { type: Boolean, default: true },
    /** Network links, data packets and long-haul arcs */
    showNetwork: { type: Boolean, default: true },
    /** Horizontal scan line sweeping over the globe */
    showScan: { type: Boolean, default: true },
    /** Upper limit for the canvas pixel ratio (retina screens would otherwise draw 4x the pixels) */
    maxDpr: { type: Number, default: 1.5 },
    /** Starting quality: 0 = all effects ... 3 = lightest */
    quality: { type: Number, default: 0 },
    /** Step the quality down automatically when the frame rate stays low (emits `quality-change`) */
    autoQuality: { type: Boolean, default: true },
    /** Scales the globe radius (1 = fit the container) */
    scale: { type: Number, default: 1 },
    /** Vertical centre of the globe as a fraction of the container height */
    centerY: { type: Number, default: 0.5 },
    /** Callout labels are not drawn inside this band around the horizontal centre (px each side),
     *  so labels do not run underneath a card placed on top of the globe */
    clearHalfWidth: { type: Number, default: 0 }
  },

  data() {
    // Only what the template needs is reactive. Canvas state lives on `this` (see created()).
    return {
      coordText: '',
      clockText: '00:00:00'
    }
  },

  computed: {
    satelliteCount() {
      return RING_DEFS.reduce((total, ring) => total + ring.sats.length, 0)
    },
    nodeCount() {
      return NODE_DEFS.length
    }
  },

  watch: {
    tilt() {
      this.applyTilt()
      this.buildRings()
    },
    scale() {
      this.onResize()
    },
    centerY() {
      this.onResize()
    }
  },

  created() {
    // Non-reactive scene state: assigned here instead of data() so Vue does not
    // walk the typed arrays or re-render on every animation frame.
    this.ctx = null
    this.raf = 0
    this.t0 = 0
    this.lastHud = 0
    this.cw = 0
    this.ch = 0
    this.dpr = 1
    this.R = 0
    this.cx = 0
    this.cy = 0
    this.spacing = 9
    this.n = 0
    this.dots = null
    this.starField = []
    this.meteors = []
    this.nextMeteor = 3

    // quality / frame-rate watchdog
    this.qLevel = Math.max(0, Math.min(3, Math.round(this.quality)))
    this.fx = this.qualityFx(this.qLevel)
    this.inView = true
    this.lastFrame = 0
    this.frameSum = 0
    this.frameCount = 0
    this.slowWindows = 0
    this.warmupUntil = 0

    this.mask = this.decodeMask()
    this.applyTilt()
    this.nodes = this.buildNodes()
    this.arcs = this.buildLongArcs()
    this.links = this.buildLinks()
    this.graticule = this.buildGraticule()
    this.rings = RING_DEFS.map((def) => ({
      ...def,
      px: new Float32Array(RING_N),
      py: new Float32Array(RING_N),
      pz: new Float32Array(RING_N),
      beacons: []
    }))

    this.coordText = this.formatCoords(this.spinAt(0))
    this.updateClock()
  },

  mounted() {
    this.ctx = this.$refs.canvas.getContext('2d')
    this.t0 = performance.now()
    this.warmupUntil = this.t0 + 1500 // ignore the first frames (JIT, fonts, layout)

    this.mq = window.matchMedia('(prefers-reduced-motion: reduce)')
    if (this.mq.addEventListener) this.mq.addEventListener('change', this.start)

    this.ro = new ResizeObserver(this.onResize)
    this.ro.observe(this.$refs.wrap)

    // Stop drawing while the globe is scrolled out of view
    if ('IntersectionObserver' in window) {
      this.io = new IntersectionObserver((entries) => {
        this.inView = entries[entries.length - 1].isIntersecting
        this.start()
      })
      this.io.observe(this.$refs.wrap)
    }

    this.onResize()
    this.start()
    this.clockTimer = setInterval(this.updateClock, 1000)
  },

  beforeDestroy() {
    cancelAnimationFrame(this.raf)
    clearInterval(this.clockTimer)
    if (this.ro) this.ro.disconnect()
    if (this.io) this.io.disconnect()
    if (this.mq && this.mq.removeEventListener) this.mq.removeEventListener('change', this.start)
  },

  methods: {
    /* ---------- setup ---------- */
    decodeMask() {
      const bin = atob(LAND_MASK_B64)
      const bytes = new Uint8Array(bin.length)
      for (let i = 0; i < bin.length; i++) bytes[i] = bin.charCodeAt(i)
      return bytes
    },

    isLand(lat, lon) {
      const i = Math.min(MASK_W - 1, Math.max(0, Math.floor((lon + 180) / (360 / MASK_W))))
      const j = Math.min(MASK_H - 1, Math.max(0, Math.floor((90 - lat) / (180 / MASK_H))))
      const k = j * MASK_W + i
      return (this.mask[k >> 3] >> (7 - (k & 7))) & 1
    },

    applyTilt() {
      this.cosT = Math.cos(this.tilt)
      this.sinT = Math.sin(this.tilt)
    },

    buildNodes() {
      return NODE_DEFS.map(([code, lat, lon]) => {
        const la = lat * DEG
        const lo = lon * DEG
        return {
          code,
          lat,
          lon,
          x: Math.cos(la) * Math.sin(lo),
          y: Math.sin(la),
          z: Math.cos(la) * Math.cos(lo),
          sx: 0,
          sy: 0,
          sz: 0
        }
      })
    },

    // Great-circle arc between two unit vectors, lifted above the surface
    makeArc(A, B, steps, liftBase, liftScale, extra) {
      const dot = Math.max(-1, Math.min(1, A.x * B.x + A.y * B.y + A.z * B.z))
      const om = Math.acos(dot)
      const so = Math.sin(om) || 1
      const lift = liftBase + liftScale * (om / Math.PI)
      const pts = new Float32Array((steps + 1) * 3)
      for (let k = 0; k <= steps; k++) {
        const s = k / steps
        const wa = Math.sin((1 - s) * om) / so
        const wb = Math.sin(s * om) / so
        const h = 1 + lift * Math.sin(Math.PI * s)
        pts[k * 3] = (A.x * wa + B.x * wb) * h
        pts[k * 3 + 1] = (A.y * wa + B.y * wb) * h
        pts[k * 3 + 2] = (A.z * wa + B.z * wb) * h
      }
      return Object.assign(
        { pts, steps, sx: new Float32Array(steps + 1), sy: new Float32Array(steps + 1), sz: new Float32Array(steps + 1) },
        extra
      )
    },

    buildLongArcs() {
      const byCode = {}
      this.nodes.forEach((nd) => { byCode[nd.code] = nd })
      return ARC_DEFS.map(([from, to, off, c]) =>
        this.makeArc(byCode[from], byCode[to], LONG_STEPS, 0.06, 0.22, { off, c })
      )
    },

    // Each node links to its two nearest neighbours (if closer than 55 degrees)
    buildLinks() {
      const nodes = this.nodes
      const seen = new Set()
      const links = []
      nodes.forEach((a, i) => {
        nodes
          .map((b, j) => ({ j, d: j === i ? -2 : a.x * b.x + a.y * b.y + a.z * b.z }))
          .sort((p, q) => q.d - p.d)
          .slice(0, 2)
          .forEach(({ j, d }) => {
            const key = i < j ? i + '-' + j : j + '-' + i
            if (seen.has(key) || d < Math.cos(55 * DEG)) return
            seen.add(key)
            links.push(
              this.makeArc(a, nodes[j], LINK_STEPS, 0.025, 0.08, {
                ph: hash(links.length + 1),
                speed: 0.22 + 0.4 * hash(links.length + 1, 2),
                c: NODE_COLORS[links.length % 3]
              })
            )
          })
      })
      return links
    },

    // Latitude circles and meridians every 15 degrees (poles above 75 degrees left empty)
    buildGraticule() {
      const lines = []
      const add = (pts, major) => {
        const count = pts.length / 3
        lines.push({
          pts: Float32Array.from(pts),
          steps: count - 1,
          major,
          sx: new Float32Array(count),
          sy: new Float32Array(count),
          sz: new Float32Array(count)
        })
      }
      for (let lat = -75; lat <= 75; lat += 15) {
        const pts = []
        const la = lat * DEG
        for (let k = 0; k <= 96; k++) {
          const lo = (k / 96) * TAU
          pts.push(Math.cos(la) * Math.sin(lo), Math.sin(la), Math.cos(la) * Math.cos(lo))
        }
        add(pts, lat === 0)
      }
      for (let lon = 0; lon < 360; lon += 15) {
        const pts = []
        const lo = lon * DEG
        for (let lat = -75; lat <= 75; lat += 3) {
          const la = lat * DEG
          pts.push(Math.cos(la) * Math.sin(lo), Math.sin(la), Math.cos(la) * Math.cos(lo))
        }
        add(pts, lon % 90 === 0)
      }
      return lines
    },

    // Evenly spread dots on a sphere (golden-angle spiral), tagged land / ocean / spark
    buildGlobe() {
      const { R, spacing } = this
      const n = Math.max(900, Math.min(14000, Math.round((4 * Math.PI * R * R) / (spacing * spacing))))
      const dots = {
        gx: new Float32Array(n),
        gy: new Float32Array(n),
        gz: new Float32Array(n),
        gland: new Uint8Array(n),
        gsp: new Uint8Array(n), // 1 = blinking "spark" dot
        gph: new Float32Array(n), // random phase 0..1
        ox: new Float32Array(n),
        oy: new Float32Array(n),
        oa: new Float32Array(n),
        os: new Float32Array(n),
        osc: new Float32Array(n) // scan-line intensity
      }
      const golden = Math.PI * (3 - Math.sqrt(5))
      for (let i = 0; i < n; i++) {
        const y = 1 - (2 * (i + 0.5)) / n
        const r = Math.sqrt(1 - y * y)
        const th = golden * i
        const x = Math.cos(th) * r
        const z = Math.sin(th) * r
        dots.gx[i] = x
        dots.gy[i] = y
        dots.gz[i] = z
        const land = this.isLand((Math.asin(y) * 180) / Math.PI, (Math.atan2(x, z) * 180) / Math.PI)
        dots.gland[i] = land
        dots.gph[i] = hash(i + 1)
        dots.gsp[i] = hash(i + 1, 3) < (land ? 0.07 : 0.012) ? 1 : 0
      }
      this.n = n
      this.dots = dots
    },

    // Point on an orbit ring in globe radii, after inclination, node rotation and the view tilt
    orbitPoint(ring, th) {
      const x = Math.cos(th) * ring.r
      const z = Math.sin(th) * ring.r
      const y1 = -z * Math.sin(ring.inc)
      const z1 = z * Math.cos(ring.inc)
      const cn = Math.cos(ring.node)
      const sn = Math.sin(ring.node)
      const x2 = x * cn + z1 * sn
      const z2 = -x * sn + z1 * cn
      return [x2, y1 * this.cosT - z2 * this.sinT, y1 * this.sinT + z2 * this.cosT]
    },

    buildRings() {
      const { cx, cy, R } = this
      this.rings.forEach((ring, ri) => {
        for (let i = 0; i < RING_N; i++) {
          const p = this.orbitPoint(ring, (i / RING_N) * TAU)
          ring.px[i] = cx + p[0] * R
          ring.py[i] = cy - p[1] * R
          ring.pz[i] = p[2]
        }
        // blinking beacons parked on the ring
        ring.beacons = []
        for (let j = 0; j < BEACONS_PER_RING; j++) {
          const p = this.orbitPoint(ring, (j / BEACONS_PER_RING) * TAU + ri * 0.7)
          ring.beacons.push({
            x: cx + p[0] * R,
            y: cy - p[1] * R,
            z: p[2],
            ph: j * 1.3 + ri * 2.1,
            c: j % 2 ? COLOR.amber : COLOR.mint
          })
        }
      })
    },

    buildStars() {
      const { cw, ch } = this
      let seed = 7
      const rnd = () => {
        seed = (seed * 16807) % 2147483647
        return (seed - 1) / 2147483646
      }
      const count = Math.round((cw * ch) / 4200)
      const stars = []
      for (let i = 0; i < count; i++) {
        const pick = rnd()
        stars.push({
          x: rnd() * cw,
          y: rnd() * ch,
          r: rnd() < 0.88 ? 0.6 : 1.1,
          a: 0.25 + rnd() * 0.6,
          ph: rnd() * TAU,
          sp: 0.6 + rnd() * 2.6,
          c: pick < 0.76 ? '190,220,255' : pick < 0.9 ? '140,235,255' : '255,214,150',
          sparkle: rnd() < 0.05 // 4-point glint
        })
      }
      this.starField = stars
    },

    onResize() {
      if (!this.ctx) return
      const rect = this.$refs.wrap.getBoundingClientRect()
      this.cw = Math.max(1, Math.round(rect.width))
      this.ch = Math.max(1, Math.round(rect.height))
      this.dpr = Math.min(window.devicePixelRatio || 1, this.fx.dprCap)
      const canvas = this.$refs.canvas
      canvas.width = Math.round(this.cw * this.dpr)
      canvas.height = Math.round(this.ch * this.dpr)
      this.cx = this.cw / 2
      this.cy = this.ch * this.centerY
      // Leave room for the outer ring horizontally and the steepest ring vertically
      const roomY = Math.min(this.cy, this.ch - this.cy)
      this.R = Math.max(56, Math.min(this.cw / 2 / 1.85, (roomY - 36) / 1.4) * this.scale)
      // Dot spacing grows with the globe so a bigger globe does not mean many more dots (about 7,800 up to R = 350);
      // lower quality levels space the dots out further
      this.spacing = Math.max(5.5, Math.min(14, this.R * 0.04)) * this.fx.spacingMul
      this.buildGlobe()
      this.buildRings()
      this.buildStars()
      if (this.mq && this.mq.matches) this.renderFrame(2, performance.now())
    },

    /* ---------- animation loop ---------- */
    start() {
      cancelAnimationFrame(this.raf)
      this.raf = 0
      this.lastFrame = 0 // restart the frame-rate window
      this.frameSum = 0
      this.frameCount = 0
      if (this.mq.matches) this.renderFrame(2, performance.now()) // single still frame
      else if (this.inView) this.raf = requestAnimationFrame(this.animate)
    },

    animate(now) {
      this.renderFrame((now - this.t0) / 1000, now)
      if (this.autoQuality) this.watchFrameRate(now)
      this.raf = requestAnimationFrame(this.animate)
    },

    /* ---------- quality ---------- */
    // What each quality level turns off (0 = everything on, 3 = lightest)
    qualityFx(q) {
      return {
        dprCap: [this.maxDpr, 1.25, 1, 1][q],
        spacingMul: [1, 1.25, 1.55, 1.9][q],
        grid: q < 1,
        network: q < 2,
        scan: q < 2,
        callouts: q < 2,
        glow: q < 1,
        sparks: q < 3,
        beacons: q < 3,
        starStep: q < 3 ? 1 : 2
      }
    },

    setQuality(level) {
      this.qLevel = level
      this.fx = this.qualityFx(level)
      this.$emit('quality-change', level)
      this.onResize() // new pixel ratio and dot density
    },

    // Average frame time over ~45 frames (or 1.5 s, whichever comes first, so a very slow
    // device is detected quickly); two slow windows in a row lower the quality one step
    watchFrameRate(now) {
      const dt = now - this.lastFrame
      this.lastFrame = now
      if (dt <= 0 || dt > 500 || now < this.warmupUntil) {
        this.frameSum = 0
        this.frameCount = 0
        return
      }
      this.frameSum += dt
      this.frameCount++
      if (this.frameCount < 45 && this.frameSum < 1500) return
      const avg = this.frameSum / this.frameCount
      this.frameSum = 0
      this.frameCount = 0
      this.slowWindows = avg > 40 ? this.slowWindows + 1 : 0 // slower than 25 fps
      if (this.slowWindows >= 2 && this.qLevel < 3) {
        this.slowWindows = 0
        this.warmupUntil = now + 1200 // let the rebuild settle before measuring again
        this.setQuality(this.qLevel + 1)
      }
    },

    spinAt(T) {
      return (-this.startLongitude * Math.PI) / 180 + T * this.spinSpeed
    },

    renderFrame(T, now) {
      const { ctx, cw, ch, dpr } = this
      if (!ctx || !cw || !ch || !this.dots) return
      ctx.setTransform(dpr, 0, 0, dpr, 0, 0)
      ctx.clearRect(0, 0, cw, ch)

      const spin = this.spinAt(T)
      this.drawStars(T)
      this.drawMeteors(T)
      this.drawAtmosphere()
      this.drawRings(false, T)
      if (this.fx.beacons) this.drawBeacons(false, T)
      this.drawSats(false, T) // behind the globe
      this.drawGlobe(spin, T)
      if (this.showNetwork && this.fx.network) this.drawNetwork(spin, T)
      this.drawNodes(spin, T)
      if (this.fx.callouts) this.drawCallouts()
      this.drawReticle(T)
      this.drawRings(true, T)
      if (this.fx.beacons) this.drawBeacons(true, T)
      this.drawSats(true, T) // in front of the globe
      this.updateHud(spin, now)
    },

    /* ---------- background ---------- */
    drawStars(T) {
      const { ctx } = this
      const stars = this.starField
      for (let si = 0; si < stars.length; si += this.fx.starStep) {
        const s = stars[si]
        const tw = 0.5 + 0.5 * Math.sin(T * s.sp + s.ph)
        const a = s.a * (0.35 + 0.65 * tw)
        ctx.fillStyle = 'rgba(' + s.c + ',' + a.toFixed(2) + ')'
        ctx.beginPath()
        ctx.arc(s.x, s.y, s.r, 0, TAU)
        ctx.fill()
        if (s.sparkle) {
          const len = 3 + 7 * tw
          ctx.strokeStyle = 'rgba(' + s.c + ',' + (a * 0.9).toFixed(2) + ')'
          ctx.lineWidth = 0.8
          ctx.beginPath()
          ctx.moveTo(s.x - len, s.y)
          ctx.lineTo(s.x + len, s.y)
          ctx.moveTo(s.x, s.y - len)
          ctx.lineTo(s.x, s.y + len)
          ctx.stroke()
        }
      }
    },

    drawMeteors(T) {
      const { ctx, cw, ch } = this
      if (T > this.nextMeteor && !(this.mq && this.mq.matches)) {
        const speed = 520 + Math.random() * 320
        const ang = 2.55 + Math.random() * 0.3 // heading down-left
        this.meteors.push({
          t: T,
          x0: cw * (0.35 + Math.random() * 0.65),
          y0: ch * Math.random() * 0.4,
          vx: Math.cos(ang) * speed,
          vy: Math.sin(ang) * speed,
          len: 80 + Math.random() * 90
        })
        this.nextMeteor = T + 2.5 + Math.random() * 4.5
      }
      const life = 0.9
      this.meteors = this.meteors.filter((m) => T - m.t < life)
      for (const m of this.meteors) {
        const age = T - m.t
        const x = m.x0 + m.vx * age
        const y = m.y0 + m.vy * age
        const k = Math.hypot(m.vx, m.vy)
        const tx = x - (m.vx / k) * m.len
        const ty = y - (m.vy / k) * m.len
        const a = 1 - age / life
        const g = ctx.createLinearGradient(x, y, tx, ty)
        g.addColorStop(0, 'rgba(235,248,255,' + (a * 0.9).toFixed(2) + ')')
        g.addColorStop(1, 'rgba(120,200,255,0)')
        ctx.strokeStyle = g
        ctx.lineWidth = 1.3
        ctx.beginPath()
        ctx.moveTo(x, y)
        ctx.lineTo(tx, ty)
        ctx.stroke()
      }
    },

    drawAtmosphere() {
      const { ctx, cx, cy, R } = this
      const g = ctx.createRadialGradient(cx, cy, R * 0.96, cx, cy, R * 1.34)
      g.addColorStop(0, 'rgba(92,203,255,.34)')
      g.addColorStop(0.35, 'rgba(60,150,255,.12)')
      g.addColorStop(1, 'rgba(40,110,255,0)')
      ctx.fillStyle = g
      ctx.beginPath()
      ctx.arc(cx, cy, R * 1.34, 0, TAU)
      ctx.fill()
    },

    /* ---------- orbit rings, beacons, satellites ---------- */
    strokeRing(ring, front) {
      const { ctx } = this
      ctx.beginPath()
      let pen = false
      for (let i = 0; i <= RING_N; i++) {
        const k = i % RING_N
        if ((ring.pz[k] >= 0) === front) {
          if (pen) ctx.lineTo(ring.px[k], ring.py[k])
          else {
            ctx.moveTo(ring.px[k], ring.py[k])
            pen = true
          }
        } else pen = false
      }
      ctx.stroke()
    },

    drawRings(front, T) {
      const { ctx } = this
      this.rings.forEach((ring, ri) => {
        // colour slowly drifts between cyan and mint, alpha flickers a little
        const m = 0.5 + 0.5 * Math.sin(T * 0.6 + ri * 1.7)
        const rgb = Math.round(130 - 20 * m) + ',' + Math.round(222 + 33 * m) + ',' + Math.round(255 - 45 * m)
        const flick = 0.5 + 0.5 * Math.sin(T * 2.6 + ri * 2.0)
        ctx.setLineDash(ring.dash || [])
        ctx.lineCap = ring.dash ? 'round' : 'butt'
        if (front && !ring.dash) {
          ctx.strokeStyle = 'rgba(' + rgb + ',' + (0.05 + 0.04 * flick).toFixed(3) + ')'
          ctx.lineWidth = 5
          this.strokeRing(ring, true)
        }
        ctx.strokeStyle = 'rgba(' + rgb + ',' + (front ? 0.5 + 0.18 * flick : 0.16).toFixed(3) + ')'
        ctx.lineWidth = ring.dash ? (front ? 2.2 : 1.6) : front ? 1.3 : 1
        this.strokeRing(ring, front)
      })
      ctx.setLineDash([])
      ctx.lineCap = 'butt'
    },

    drawBeacons(front, T) {
      const { ctx } = this
      ctx.globalCompositeOperation = 'lighter'
      for (const ring of this.rings) {
        for (const b of ring.beacons) {
          if ((b.z >= 0) !== front) continue
          const pulse = Math.pow(Math.max(0, Math.sin(T * 2.2 + b.ph)), 4)
          if (pulse < 0.05) continue
          ctx.fillStyle = 'rgba(' + b.c + ',' + (pulse * (front ? 0.2 : 0.08)).toFixed(3) + ')'
          ctx.beginPath()
          ctx.arc(b.x, b.y, 7, 0, TAU)
          ctx.fill()
          ctx.fillStyle = 'rgba(' + b.c + ',' + (pulse * (front ? 0.95 : 0.4)).toFixed(3) + ')'
          ctx.beginPath()
          ctx.arc(b.x, b.y, 1.9, 0, TAU)
          ctx.fill()
        }
      }
      ctx.globalCompositeOperation = 'source-over'
    },

    drawSats(front, T) {
      const { ctx, cx, cy, R } = this
      ctx.font = '500 10px "JetBrains Mono", ui-monospace, monospace'
      for (const ring of this.rings) {
        for (const sat of ring.sats) {
          const th = sat.a + sat.v * T
          const dir = sat.v >= 0 ? 1 : -1

          // trail
          let prev = this.orbitPoint(ring, th)
          for (let j = 1; j <= TRAIL; j++) {
            const cur = this.orbitPoint(ring, th - dir * j * TRAIL_STEP)
            if (((prev[2] + cur[2]) / 2 >= 0) === front) {
              const f = 1 - j / (TRAIL + 1)
              ctx.strokeStyle = 'rgba(' + sat.c + ',' + (Math.pow(f, 1.6) * (front ? 0.85 : 0.3)).toFixed(3) + ')'
              ctx.lineWidth = 0.6 + 2.2 * f
              ctx.beginPath()
              ctx.moveTo(cx + prev[0] * R, cy - prev[1] * R)
              ctx.lineTo(cx + cur[0] * R, cy - cur[1] * R)
              ctx.stroke()
            }
            prev = cur
          }

          // head
          const p = this.orbitPoint(ring, th)
          if ((p[2] >= 0) === front) {
            const x = cx + p[0] * R
            const y = cy - p[1] * R
            const g = ctx.createRadialGradient(x, y, 0, x, y, 13)
            g.addColorStop(0, 'rgba(' + sat.c + ',' + (front ? 0.9 : 0.35) + ')')
            g.addColorStop(1, 'rgba(' + sat.c + ',0)')
            ctx.fillStyle = g
            ctx.beginPath()
            ctx.arc(x, y, 13, 0, TAU)
            ctx.fill()
            ctx.fillStyle = '#fff'
            ctx.beginPath()
            ctx.arc(x, y, front ? 2.4 : 1.8, 0, TAU)
            ctx.fill()
            if (front && sat.label) {
              ctx.fillStyle = 'rgba(' + sat.c + ',.9)'
              ctx.textAlign = 'left'
              ctx.fillText(sat.label, x + 10, y - 9)
            }
          }
        }
      }
    },

    /* ---------- globe ---------- */
    drawGlobe(spin, T) {
      const { ctx, cx, cy, R, n, cosT, sinT, spacing } = this
      const { gx, gy, gz, gland, gsp, gph, ox, oy, oa, os, osc } = this.dots
      const cs = Math.cos(spin)
      const sn = Math.sin(spin)
      const scanOn = this.showScan && this.fx.scan
      const sparksOn = this.fx.sparks
      const scanY = -1.05 + 2.1 * ((T * 0.18) % 1) // view-space height of the scan line, sweeps bottom to top

      // sphere body
      let g = ctx.createRadialGradient(cx - 0.35 * R, cy - 0.4 * R, R * 0.08, cx, cy, R)
      g.addColorStop(0, '#14416B')
      g.addColorStop(0.55, '#0A2342')
      g.addColorStop(1, '#050F1F')
      ctx.fillStyle = g
      ctx.beginPath()
      ctx.arc(cx, cy, R, 0, TAU)
      ctx.fill()

      // rotate the dots: spin around the vertical axis, then tilt toward the viewer
      for (let i = 0; i < n; i++) {
        const x = gx[i]
        const y = gy[i]
        const z = gz[i]
        const x1 = x * cs + z * sn
        const z1 = -x * sn + z * cs
        const y2 = y * cosT - z1 * sinT
        const z2 = y * sinT + z1 * cosT
        if (z2 <= 0.02) {
          oa[i] = 0
          continue
        }
        const light = x1 * LIGHT.x + y2 * LIGHT.y + z2 * LIGHT.z
        oa[i] = (0.2 + 0.8 * (light > 0 ? light : 0)) * (z2 < 0.3 ? z2 / 0.3 : 1)
        ox[i] = cx + x1 * R
        oy[i] = cy - y2 * R
        os[i] = 0.55 + 0.45 * z2
        let sc = 0
        if (scanOn) {
          const d = (y2 - scanY) * 18
          sc = Math.exp(-d * d)
          if (y2 < scanY) sc += Math.exp(-(scanY - y2) * 6) * 0.22 // fading wake behind the line
        }
        osc[i] = sc
      }

      // ocean dots
      ctx.fillStyle = '#4BA3E3'
      const side = spacing * 0.15
      for (let i = 0; i < n; i++) {
        if (oa[i] > 0 && !gland[i] && !(sparksOn && gsp[i])) {
          const s = side * os[i]
          ctx.globalAlpha = Math.min(1, oa[i] * 0.34 + osc[i] * 0.5)
          ctx.fillRect(ox[i] - s / 2, oy[i] - s / 2, s, s)
        }
      }

      // land dots (gentle twinkle)
      ctx.fillStyle = '#A6EBFF'
      const lr = spacing * 0.165
      for (let i = 0; i < n; i++) {
        if (oa[i] > 0 && gland[i] && !(sparksOn && gsp[i])) {
          const tw = 0.86 + 0.14 * Math.sin(T * 1.7 + gph[i] * TAU)
          ctx.globalAlpha = Math.min(1, oa[i] * tw + osc[i] * 0.5)
          ctx.beginPath()
          ctx.arc(ox[i], oy[i], lr * os[i], 0, TAU)
          ctx.fill()
        }
      }

      // blinking spark dots: change colour and flash
      ctx.globalCompositeOperation = 'lighter'
      for (let i = 0; i < n; i++) {
        if (sparksOn && oa[i] > 0 && gsp[i]) {
          const pulse = Math.pow(Math.max(0, Math.sin(T * 3.2 + gph[i] * 40)), 2)
          const col = SPARK_COLORS[Math.floor(T * 0.7 + gph[i] * 7) % 3]
          const a = Math.min(1, oa[i] * (0.25 + 0.75 * pulse))
          const r = lr * os[i] * (1.1 + pulse * 0.9)
          ctx.globalAlpha = 1
          ctx.fillStyle = 'rgba(' + col + ',' + a.toFixed(3) + ')'
          ctx.beginPath()
          ctx.arc(ox[i], oy[i], r, 0, TAU)
          ctx.fill()
          if (this.fx.glow && pulse > 0.35) {
            ctx.fillStyle = 'rgba(' + col + ',' + (pulse * oa[i] * 0.22).toFixed(3) + ')'
            ctx.beginPath()
            ctx.arc(ox[i], oy[i], r * 3.4, 0, TAU)
            ctx.fill()
          }
        }
      }

      // dots lit by the scan line
      if (scanOn) {
        ctx.fillStyle = '#EAFBFF'
        for (let i = 0; i < n; i++) {
          if (oa[i] > 0 && osc[i] > 0.2) {
            ctx.globalAlpha = Math.min(1, osc[i])
            ctx.beginPath()
            ctx.arc(ox[i], oy[i], lr * os[i] * 1.25, 0, TAU)
            ctx.fill()
          }
        }
      }
      ctx.globalAlpha = 1
      ctx.globalCompositeOperation = 'source-over'

      if (this.showGrid && this.fx.grid) this.drawGraticule(spin, T)

      // night-side shading
      g = ctx.createRadialGradient(cx - 0.4 * R, cy - 0.45 * R, R * 0.55, cx - 0.1 * R, cy - 0.1 * R, R * 1.35)
      g.addColorStop(0, 'rgba(2,8,18,0)')
      g.addColorStop(1, 'rgba(2,8,18,.62)')
      ctx.fillStyle = g
      ctx.beginPath()
      ctx.arc(cx, cy, R, 0, TAU)
      ctx.fill()

      // atmosphere rim
      g = ctx.createRadialGradient(cx, cy, R * 0.82, cx, cy, R)
      g.addColorStop(0, 'rgba(92,203,255,0)')
      g.addColorStop(0.7, 'rgba(92,203,255,.08)')
      g.addColorStop(1, 'rgba(120,220,255,.55)')
      ctx.fillStyle = g
      ctx.beginPath()
      ctx.arc(cx, cy, R, 0, TAU)
      ctx.fill()
      ctx.strokeStyle = 'rgba(160,230,255,.45)'
      ctx.lineWidth = 1
      ctx.beginPath()
      ctx.arc(cx, cy, R, 0, TAU)
      ctx.stroke()

      if (scanOn) this.drawScanLine(scanY)
    },

    // Rotate and project a polyline (arc or graticule line) into screen space
    projectLine(line, spin) {
      const { cx, cy, R, cosT, sinT } = this
      const cs = Math.cos(spin)
      const sn = Math.sin(spin)
      const p = line.pts
      for (let k = 0; k <= line.steps; k++) {
        const x = p[k * 3]
        const y = p[k * 3 + 1]
        const z = p[k * 3 + 2]
        const x1 = x * cs + z * sn
        const z1 = -x * sn + z * cs
        line.sx[k] = cx + x1 * R
        line.sy[k] = cy - (y * cosT - z1 * sinT) * R
        line.sz[k] = y * sinT + z1 * cosT
      }
    },

    drawGraticule(spin, T) {
      const { ctx } = this
      const minor = new Path2D()
      const major = new Path2D()
      for (const line of this.graticule) {
        this.projectLine(line, spin)
        const path = line.major ? major : minor
        let pen = false
        for (let k = 0; k <= line.steps; k++) {
          if (line.sz[k] > 0) {
            if (pen) path.lineTo(line.sx[k], line.sy[k])
            else {
              path.moveTo(line.sx[k], line.sy[k])
              pen = true
            }
          } else pen = false
        }
      }
      const flick = 0.5 + 0.5 * Math.sin(T * 1.3)
      ctx.lineWidth = 0.6
      ctx.strokeStyle = 'rgba(110,210,255,' + (0.075 + 0.04 * flick).toFixed(3) + ')'
      ctx.stroke(minor)
      ctx.lineWidth = 0.9
      ctx.strokeStyle = 'rgba(140,230,255,' + (0.22 + 0.08 * flick).toFixed(3) + ')'
      ctx.stroke(major)
    },

    // Horizontal chord across the disc at the scan height
    drawScanLine(scanY) {
      const { ctx, cx, cy, R } = this
      if (Math.abs(scanY) >= 1) return
      const half = Math.sqrt(1 - scanY * scanY) * R
      const y = cy - scanY * R
      const g = ctx.createLinearGradient(cx - half, y, cx + half, y)
      g.addColorStop(0, 'rgba(190,245,255,0)')
      g.addColorStop(0.15, 'rgba(190,245,255,.85)')
      g.addColorStop(0.85, 'rgba(190,245,255,.85)')
      g.addColorStop(1, 'rgba(190,245,255,0)')
      ctx.globalCompositeOperation = 'lighter'
      ctx.strokeStyle = g
      ctx.lineWidth = 7
      ctx.globalAlpha = 0.14
      ctx.beginPath()
      ctx.moveTo(cx - half, y)
      ctx.lineTo(cx + half, y)
      ctx.stroke()
      ctx.globalAlpha = 1
      ctx.lineWidth = 1.2
      ctx.beginPath()
      ctx.moveTo(cx - half, y)
      ctx.lineTo(cx + half, y)
      ctx.stroke()
      ctx.globalCompositeOperation = 'source-over'
    },

    /* ---------- network ---------- */
    drawNetwork(spin, T) {
      const { ctx } = this
      ctx.lineCap = 'round'

      // short links with flickering light and a data packet each
      for (const link of this.links) {
        this.projectLine(link, spin)
        const flick = 0.07 + 0.1 * (0.5 + 0.5 * Math.sin(T * 1.8 + link.ph * TAU))
        ctx.strokeStyle = 'rgba(120,215,255,' + flick.toFixed(3) + ')'
        ctx.lineWidth = 0.8
        ctx.beginPath()
        let pen = false
        for (let k = 0; k <= link.steps; k++) {
          if (link.sz[k] > 0) {
            if (pen) ctx.lineTo(link.sx[k], link.sy[k])
            else {
              ctx.moveTo(link.sx[k], link.sy[k])
              pen = true
            }
          } else pen = false
        }
        ctx.stroke()

        const f = ((T * link.speed + link.ph) % 1) * link.steps
        const k = Math.floor(f)
        const u = f - k
        if (link.sz[k] > 0.05 && link.sz[k + 1] > 0.05) {
          const x = link.sx[k] + (link.sx[k + 1] - link.sx[k]) * u
          const y = link.sy[k] + (link.sy[k + 1] - link.sy[k]) * u
          ctx.fillStyle = 'rgba(' + link.c + ',.95)'
          ctx.beginPath()
          ctx.arc(x, y, 1.5, 0, TAU)
          ctx.fill()
        }
      }

      // long-haul arcs with a moving comet
      for (const arc of this.arcs) {
        this.projectLine(arc, spin)
        ctx.strokeStyle = 'rgba(130,222,255,.16)'
        ctx.lineWidth = 1
        ctx.beginPath()
        let pen = false
        for (let k = 0; k <= arc.steps; k++) {
          if (arc.sz[k] > 0) {
            if (pen) ctx.lineTo(arc.sx[k], arc.sy[k])
            else {
              ctx.moveTo(arc.sx[k], arc.sy[k])
              pen = true
            }
          } else pen = false
        }
        ctx.stroke()

        const head = (T * 0.32 + arc.off) % 1.6
        const tail = 0.38
        for (let k = 0; k < arc.steps; k++) {
          const s = (k + 1) / arc.steps
          if (s > head || s < head - tail || arc.sz[k] <= 0 || arc.sz[k + 1] <= 0) continue
          const a = 1 - (head - s) / tail
          ctx.strokeStyle = 'rgba(' + arc.c + ',' + (a * 0.95).toFixed(3) + ')'
          ctx.lineWidth = 1.2 + a * 0.9
          ctx.beginPath()
          ctx.moveTo(arc.sx[k], arc.sy[k])
          ctx.lineTo(arc.sx[k + 1], arc.sy[k + 1])
          ctx.stroke()
        }
      }
      ctx.lineCap = 'butt'
    },

    // Blinking nodes (colour cycles), with expanding ping rings on every third one
    drawNodes(spin, T) {
      const { ctx, cx, cy, R, cosT, sinT } = this
      const cs = Math.cos(spin)
      const sn = Math.sin(spin)
      ctx.globalCompositeOperation = 'lighter'
      ctx.lineWidth = 1
      for (let i = 0; i < this.nodes.length; i++) {
        const c = this.nodes[i]
        const x1 = c.x * cs + c.z * sn
        const z1 = -c.x * sn + c.z * cs
        const y2 = c.y * cosT - z1 * sinT
        const z2 = c.y * sinT + z1 * cosT
        c.sx = cx + x1 * R
        c.sy = cy - y2 * R
        c.sz = z2
        if (z2 < 0.06) continue
        const col = NODE_COLORS[Math.floor(T * 0.6 + i * 1.3) % 3]
        const a = Math.min(1, z2 * 2) * (0.55 + 0.45 * Math.sin(T * 4 + i * 2.1))
        if (this.fx.glow) {
          ctx.fillStyle = 'rgba(' + col + ',' + (a * 0.13).toFixed(3) + ')'
          ctx.beginPath()
          ctx.arc(c.sx, c.sy, 6, 0, TAU)
          ctx.fill()
        }
        ctx.fillStyle = 'rgba(' + col + ',' + a.toFixed(3) + ')'
        ctx.beginPath()
        ctx.arc(c.sx, c.sy, 1.8, 0, TAU)
        ctx.fill()
        if (i % 3 === 0) {
          const ph = (T * 0.45 + i * 0.37) % 1
          ctx.strokeStyle = 'rgba(' + col + ',' + ((1 - ph) * 0.6 * z2).toFixed(3) + ')'
          ctx.beginPath()
          ctx.arc(c.sx, c.sy, 2 + ph * 13, 0, TAU)
          ctx.stroke()
        }
      }
      ctx.globalCompositeOperation = 'source-over'
    },

    // Leader line + code + coordinates next to the closest labelled nodes
    drawCallouts() {
      const { ctx, cx, R } = this
      if (R < 110) return
      const shown = this.nodes
        .filter((nd) => LABELED.indexOf(nd.code) !== -1 && nd.sz > 0.55)
        .sort((a, b) => b.sz - a.sz)
        .slice(0, 4)
      ctx.font = '500 9.5px "JetBrains Mono", ui-monospace, monospace'
      const taken = [] // label boxes already drawn, to avoid overlaps
      for (const nd of shown) {
        const dirx = nd.sx >= cx ? 1 : -1
        const ex = nd.sx + dirx * 16
        const ey = nd.sy - 16
        const text =
          nd.code + '  ' + Math.abs(nd.lat).toFixed(1) + (nd.lat >= 0 ? 'N ' : 'S ') +
          Math.abs(nd.lon).toFixed(1) + (nd.lon >= 0 ? 'E' : 'W')
        const w = ctx.measureText(text).width + 6
        const box = { x0: Math.min(ex, ex + dirx * w) - 6, x1: Math.max(ex, ex + dirx * w) + 6, y0: ey - 18, y1: ey + 6 }
        if (this.clearHalfWidth && box.x1 > cx - this.clearHalfWidth && box.x0 < cx + this.clearHalfWidth) continue
        if (taken.some((t) => box.x0 < t.x1 && box.x1 > t.x0 && box.y0 < t.y1 && box.y1 > t.y0)) continue
        taken.push(box)
        ctx.strokeStyle = 'rgba(150,235,255,.55)'
        ctx.lineWidth = 0.8
        ctx.beginPath()
        ctx.moveTo(nd.sx, nd.sy)
        ctx.lineTo(ex, ey)
        ctx.lineTo(ex + dirx * w, ey)
        ctx.stroke()
        ctx.fillStyle = 'rgba(205,240,255,.9)'
        ctx.textAlign = dirx > 0 ? 'left' : 'right'
        ctx.fillText(text, ex + dirx * 3, ey - 4)
      }
      ctx.textAlign = 'left'
    },

    // Flat HUD ring around the globe: tick marks and three rotating arc segments
    drawReticle(T) {
      const { ctx, cx, cy, R } = this
      const rot = T * 0.06
      const shortTicks = new Path2D()
      const longTicks = new Path2D()
      const count = 120
      for (let i = 0; i < count; i++) {
        const a = rot + (i * TAU) / count
        const long = i % 10 === 0
        const r0 = R * 1.07
        const r1 = r0 + R * (long ? 0.04 : 0.02)
        const path = long ? longTicks : shortTicks
        path.moveTo(cx + Math.cos(a) * r0, cy + Math.sin(a) * r0)
        path.lineTo(cx + Math.cos(a) * r1, cy + Math.sin(a) * r1)
      }
      ctx.lineWidth = 1
      ctx.strokeStyle = 'rgba(130,222,255,.2)'
      ctx.stroke(shortTicks)
      ctx.strokeStyle = 'rgba(160,236,255,.5)'
      ctx.stroke(longTicks)

      ctx.lineCap = 'round'
      ctx.lineWidth = 2
      const arcs = [[0.35, 0.9], [-0.5, 0.6], [0.22, 0.45]]
      arcs.forEach(([speed, span], i) => {
        const start = T * speed + i * 2.1
        ctx.strokeStyle = 'rgba(' + (i === 1 ? COLOR.amber : COLOR.cyan) + ',' + (0.35 + 0.2 * Math.sin(T * 2 + i)).toFixed(2) + ')'
        ctx.beginPath()
        ctx.arc(cx, cy, R * 1.135, start, start + span)
        ctx.stroke()
      })
      ctx.lineCap = 'butt'
    },

    /* ---------- HUD ---------- */
    formatCoords(spin) {
      const lon = (((((-spin * 180) / Math.PI) % 360) + 540) % 360) - 180
      const lat = (this.tilt * 180) / Math.PI
      return (
        `LAT ${Math.abs(lat).toFixed(1)}° ${lat >= 0 ? 'N' : 'S'} · ` +
        `LON ${Math.abs(lon).toFixed(1)}° ${lon >= 0 ? 'E' : 'W'}`
      )
    },

    updateHud(spin, now) {
      if (now - this.lastHud < 150) return
      this.lastHud = now
      this.coordText = this.formatCoords(spin)
    },

    updateClock() {
      this.clockText = new Date().toISOString().slice(11, 19)
    }
  }
}
</script>

<style scoped>
.earth-globe {
  --muted: #8d9fb9;
  --live: #46e6a6;
  position: relative;
  width: 100%;
  height: 100%;
  overflow: hidden;
  background:
    radial-gradient(ellipse 70% 60% at 50% var(--cy, 52%), rgba(40, 120, 200, 0.26), transparent 70%),
    radial-gradient(ellipse 120% 80% at 85% 0%, rgba(60, 224, 200, 0.08), transparent 60%),
    linear-gradient(180deg, #07101c, #040910);
}

.earth-globe__canvas {
  position: absolute;
  inset: 0;
  display: block;
  width: 100%;
  height: 100%;
}

.earth-globe__hud {
  position: absolute;
  inset: 0;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  padding: 18px clamp(16px, 2.2vw, 28px);
  pointer-events: none;
  color: var(--muted);
  font: 500 11px/1.4 'JetBrains Mono', ui-monospace, 'SFMono-Regular', Menlo, Consolas, monospace;
  letter-spacing: 0.12em;
  text-transform: uppercase;
  font-variant-numeric: tabular-nums;
}

.hud-row {
  display: flex;
  flex-wrap: wrap;
  justify-content: space-between;
  gap: 4px 16px;
}
.hud-row span {
  white-space: nowrap;
}

.hud-live {
  display: inline-flex;
  align-items: center;
  gap: 8px;
}
.hud-live i {
  width: 6px;
  height: 6px;
  border-radius: 50%;
  background: var(--live);
  animation: hud-pulse 2s ease-out infinite;
}

@keyframes hud-pulse {
  0% { box-shadow: 0 0 0 0 rgba(70, 230, 166, 0.55); }
  70%, 100% { box-shadow: 0 0 0 8px rgba(70, 230, 166, 0); }
}

@media (prefers-reduced-motion: reduce) {
  .hud-live i { animation: none; }
}

@media (max-width: 900px) {
  .earth-globe__hud {
    padding: 14px 16px;
    font-size: 10px;
  }
}
</style>