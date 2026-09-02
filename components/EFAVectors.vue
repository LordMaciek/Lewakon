<script setup>
import { computed } from 'vue'

const props = defineProps({
  step: { type: Number, default: 0 },
})

function mulberry32(seed) {
  return function () {
    seed |= 0
    seed = (seed + 0x6d2b79f5) | 0
    let t = Math.imul(seed ^ (seed >>> 15), 1 | seed)
    t = (t + Math.imul(t ^ (t >>> 7), 61 | t)) ^ t
    return ((t ^ (t >>> 14)) >>> 0) / 4294967296
  }
}
const rand = mulberry32(77)

// Same conceptual items as the factor simulator: loadings on two
// underlying dimensions, plus a bit of angular noise so at step 0 the
// two bundles aren't blindingly obvious before coloring.
const RAW = [
  { label: 'Synonyms', v: 0.85, q: 0.05, group: 0 },
  { label: 'Analogies', v: 0.75, q: 0.15, group: 0 },
  { label: 'Reading', v: 0.80, q: 0.00, group: 0 },
  { label: 'Vocabulary', v: 0.70, q: 0.10, group: 0 },
  { label: 'Arithmetic', v: 0.05, q: 0.80, group: 1 },
  { label: 'Number series', v: 0.10, q: 0.85, group: 1 },
  { label: 'Geometry', v: 0.00, q: 0.75, group: 1 },
  { label: 'Algebra', v: 0.15, q: 0.80, group: 1 },
]

const NOISE = 0.34
const items = RAW.map((it) => ({
  ...it,
  v: it.v + (rand() - 0.5) * NOISE,
  q: it.q + (rand() - 0.5) * NOISE,
}))

const CENTER = { x: 250, y: 175 }
const SCALE = 135

const arrows = computed(() =>
  items.map((it) => {
    const angle = Math.atan2(it.q, it.v)
    const mag = Math.min(1, Math.sqrt(it.v * it.v + it.q * it.q))
    return {
      label: it.label,
      group: it.group,
      x2: CENTER.x + Math.cos(angle) * mag * SCALE,
      y2: CENTER.y - Math.sin(angle) * mag * SCALE,
      angle,
    }
  }),
)

const colors = ['#22d3ee', '#fb923c'] // vivid cyan / vivid orange
const gray = '#94a3b8'

const MARGIN_X = [20, 480]
const MARGIN_Y = [20, 310]
const clamp = (v, lo, hi) => Math.min(hi, Math.max(lo, v))
function clampPoint(x, y) {
  return { x: clamp(x, MARGIN_X[0], MARGIN_X[1]), y: clamp(y, MARGIN_Y[0], MARGIN_Y[1]) }
}

function circularMeanAngle(group) {
  const subset = arrows.value.filter((a) => a.group === group)
  const sx = subset.reduce((s, a) => s + Math.cos(a.angle), 0)
  const sy = subset.reduce((s, a) => s + Math.sin(a.angle), 0)
  return Math.atan2(sy, sx)
}

const AXIS_RADIUS = 145
const LABEL_RADIUS = 158

const factorAxes = computed(() =>
  [0, 1].map((g) => {
    const angle = circularMeanAngle(g)
    const tip = clampPoint(CENTER.x + Math.cos(angle) * AXIS_RADIUS, CENTER.y - Math.sin(angle) * AXIS_RADIUS)
    const label = clampPoint(CENTER.x + Math.cos(angle) * LABEL_RADIUS, CENTER.y - Math.sin(angle) * LABEL_RADIUS)
    return {
      x1: CENTER.x - Math.cos(angle) * 20,
      y1: CENTER.y + Math.sin(angle) * 20,
      x2: tip.x,
      y2: tip.y,
      labelX: label.x,
      labelY: label.y,
    }
  }),
)

const showGroups = computed(() => props.step >= 1)
const showAxes = computed(() => props.step >= 2)

const stepLabel = computed(() => {
  if (props.step >= 2) return 'STEP 03 — UNDERLYING FACTOR AXES'
  if (props.step >= 1) return 'STEP 02 — GROUPING BY DIRECTION'
  return 'STEP 01 — VARIABLES AS VECTORS'
})
</script>

<template>
  <div class="vec-demo">
    <div class="step-label">{{ stepLabel }}</div>

    <svg viewBox="0 0 500 330" preserveAspectRatio="xMidYMid meet" class="chart">
      <defs>
        <marker id="arrow-gray" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
          <path d="M0,0 L10,5 L0,10 Z" :fill="gray" />
        </marker>
        <marker id="arrow-0" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
          <path d="M0,0 L10,5 L0,10 Z" :fill="colors[0]" />
        </marker>
        <marker id="arrow-1" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
          <path d="M0,0 L10,5 L0,10 Z" :fill="colors[1]" />
        </marker>
        <filter id="axis-glow" x="-60%" y="-60%" width="220%" height="220%">
          <feGaussianBlur stdDeviation="4" result="blur" />
          <feMerge>
            <feMergeNode in="blur" />
            <feMergeNode in="SourceGraphic" />
          </feMerge>
        </filter>
        <filter id="vector-glow" x="-80%" y="-80%" width="260%" height="260%">
          <feGaussianBlur stdDeviation="2.5" result="blur" />
          <feMerge>
            <feMergeNode in="blur" />
            <feMergeNode in="SourceGraphic" />
          </feMerge>
        </filter>
      </defs>

      <!-- background grid -->
      <g class="grid">
        <line v-for="i in 12" :key="'v' + i" :x1="i * 42" y1="0" :x2="i * 42" y2="330" />
        <line v-for="i in 9" :key="'h' + i" x1="0" :y1="i * 37" x2="500" :y2="i * 37" />
      </g>

      <!-- unit circle for reference -->
      <circle :cx="CENTER.x" :cy="CENTER.y" :r="SCALE" class="unit-circle" />

      <!-- factor axes, step 2 -->
      <g v-if="showAxes">
        <g v-for="(ax, i) in factorAxes" :key="'axis' + i">
          <line
            :x1="ax.x1" :y1="ax.y1" :x2="ax.x2" :y2="ax.y2"
            :stroke="colors[i]" stroke-width="2.5" stroke-dasharray="2 4"
            filter="url(#axis-glow)" class="axis-line"
          />
          <text :x="ax.labelX" :y="ax.labelY" :fill="colors[i]" class="axis-tag">
            factor {{ i + 1 }}
          </text>
        </g>
      </g>

      <!-- origin -->
      <circle :cx="CENTER.x" :cy="CENTER.y" r="3" class="origin" />

      <!-- variable vectors -->
      <line
        v-for="(a, i) in arrows" :key="i"
        :x1="CENTER.x" :y1="CENTER.y" :x2="a.x2" :y2="a.y2"
        :stroke="showGroups ? colors[a.group] : gray"
        :stroke-width="showGroups ? 2.5 : 2"
        :marker-end="`url(#arrow-${showGroups ? a.group : 'gray'})`"
        :filter="showGroups ? 'url(#vector-glow)' : null"
        class="vector"
        :style="{ transitionDelay: (i * 40) + 'ms' }"
      />
      <text
        v-for="(a, i) in arrows" :key="'lbl' + i"
        :x="a.x2 + (a.x2 > CENTER.x ? 6 : -6)"
        :y="a.y2 + (a.y2 > CENTER.y ? 12 : -6)"
        :text-anchor="a.x2 > CENTER.x ? 'start' : 'end'"
        class="vec-label"
      >
        {{ arrows[i].label }}
      </text>
    </svg>
  </div>
</template>

<style scoped>
.vec-demo {
  width: 100%;
  height: 80%;
  max-height: 100%;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 0.6em;
  font-family: var(--slidev-code-font-family, monospace);
  box-sizing: border-box;
  overflow: hidden;
}

.step-label {
  font-size: 0.75rem;
  letter-spacing: 0.15em;
  opacity: 0.6;
  flex-shrink: 0;
}

.chart {
  width: 100%;
  height: auto;
  max-width: min(90%, 620px);
  max-height: calc(100% - 2em);
}

.grid line {
  stroke: currentColor;
  stroke-width: 0.5;
  opacity: 0.1;
}

.unit-circle {
  fill: none;
  stroke: currentColor;
  stroke-width: 1;
  opacity: 0.15;
}

.origin {
  fill: currentColor;
  opacity: 0.5;
}

.vector {
  transition: stroke 0.5s ease, stroke-width 0.4s ease;
}

.vec-label {
  fill: currentColor;
  opacity: 0.55;
  font-size: 9px;
  font-family: var(--slidev-code-font-family, monospace);
}

.axis-tag {
  font-size: 11px;
  letter-spacing: 0.05em;
  text-anchor: middle;
  text-transform: uppercase;
}

.axis-line {
  animation: fade-in 0.6s ease;
}

@keyframes fade-in {
  from {
    opacity: 0;
  }
  to {
    opacity: 1;
  }
}
</style>
