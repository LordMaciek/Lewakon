<script setup>
import { computed } from 'vue'

const props = defineProps({
  step: { type: Number, default: 0 },
})

// Seeded PRNG (mulberry32) + Box–Muller, so the "random" cloud is
// reproducible on every run/presentation instead of reshuffling each reload.
function mulberry32(seed) {
  return function () {
    seed |= 0
    seed = (seed + 0x6d2b79f5) | 0
    let t = Math.imul(seed ^ (seed >>> 15), 1 | seed)
    t = (t + Math.imul(t ^ (t >>> 7), 61 | t)) ^ t
    return ((t ^ (t >>> 14)) >>> 0) / 4294967296
  }
}
const rand = mulberry32(1337)
function gaussian() {
  const u = 1 - rand()
  const v = rand()
  return Math.sqrt(-2 * Math.log(u)) * Math.cos(2 * Math.PI * v)
}

// Cluster centers deliberately close together with a wide, overlapping
// spread — at step 0 this should read as one ambiguous cloud, not three
// obvious blobs. Coordinates clamped to stay inside the viewBox.
const CENTERS = [
  { x: 190, y: 190 },
  { x: 300, y: 140 },
  { x: 230, y: 250 },
]
const SPREAD = 40
const N_PER_CLUSTER = 12
const clamp = (v, lo, hi) => Math.min(hi, Math.max(lo, v))

const points = []
CENTERS.forEach((center, c) => {
  for (let i = 0; i < N_PER_CLUSTER; i++) {
    points.push({
      x: clamp(center.x + gaussian() * SPREAD, 25, 475),
      y: clamp(center.y + gaussian() * SPREAD, 15, 305),
      c,
    })
  }
})

const colors = ['#f97316', '#38bdf8', '#a78bfa'] // orange / sky / violet
const grayDot = '#94a3b8'

const centroids = computed(() =>
  [0, 1, 2].map((c) => {
    const g = points.filter((p) => p.c === c)
    return {
      x: g.reduce((s, p) => s + p.x, 0) / g.length,
      y: g.reduce((s, p) => s + p.y, 0) / g.length,
    }
  }),
)

// Rough "blob" boundary per cluster: push each point outward from the
// cluster centroid, sort by angle, connect. The gaussian blur filter on
// the path hides the jaggedness and makes it read as a soft cloud.
function hullPath(clusterIdx) {
  const pts = points.filter((p) => p.c === clusterIdx)
  const cx = pts.reduce((s, p) => s + p.x, 0) / pts.length
  const cy = pts.reduce((s, p) => s + p.y, 0) / pts.length
  const padded = pts
    .map((p) => {
      const dx = p.x - cx
      const dy = p.y - cy
      const len = Math.sqrt(dx * dx + dy * dy) || 1
      const pad = 30
      return {
        x: p.x + (dx / len) * pad,
        y: p.y + (dy / len) * pad,
        angle: Math.atan2(dy, dx),
      }
    })
    .sort((a, b) => a.angle - b.angle)
  return 'M ' + padded.map((p) => `${p.x},${p.y}`).join(' L ') + ' Z'
}

const visibleHulls = computed(() => (props.step >= 2 ? [0, 1, 2] : []))

const stepLabel = computed(() => {
  if (props.step >= 2) return 'STEP 03 — CLUSTER BOUNDARIES'
  if (props.step >= 1) return 'STEP 02 — CLUSTER ASSIGNMENT'
  return 'STEP 01 — RAW OBSERVATIONS'
})
</script>

<template>
  <div class="cluster-demo">
    <div class="step-label">{{ stepLabel }}</div>

    <svg viewBox="0 0 500 330" preserveAspectRatio="xMidYMid meet" class="chart">
      <defs>
        <filter id="cluster-glow" x="-60%" y="-60%" width="220%" height="220%">
          <feGaussianBlur stdDeviation="9" result="blur" />
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

      <!-- axes -->
      <line x1="15" y1="315" x2="485" y2="315" class="axis" />
      <line x1="15" y1="8" x2="15" y2="315" class="axis" />
      <text x="460" y="330" class="axis-label">var 1</text>
      <text x="2" y="18" class="axis-label">var 2</text>

      <!-- cluster halos, step 2 -->
      <transition-group name="hull" tag="g">
        <path
          v-for="c in visibleHulls"
          :key="'hull-' + c"
          :d="hullPath(c)"
          :fill="colors[c]"
          opacity="0.16"
          filter="url(#cluster-glow)"
        />
      </transition-group>

      <!-- observations -->
      <circle
        v-for="(p, i) in points"
        :key="i"
        :cx="p.x"
        :cy="p.y"
        r="6"
        :fill="step >= 1 ? colors[p.c] : grayDot"
        class="point"
        :style="{ transitionDelay: (i % 11) * 25 + 'ms', animationDelay: (i % 11) * 25 + 'ms' }"
      />

      <!-- centroids, step 1+ -->
      <g v-if="step >= 1">
        <g v-for="(c, i) in centroids" :key="'cen' + i" class="centroid" :style="{ animationDelay: '400ms' }">
          <line :x1="c.x - 9" :y1="c.y - 9" :x2="c.x + 9" :y2="c.y + 9" :stroke="colors[i]" stroke-width="2.5" />
          <line :x1="c.x - 9" :y1="c.y + 9" :x2="c.x + 9" :y2="c.y - 9" :stroke="colors[i]" stroke-width="2.5" />
        </g>
      </g>
    </svg>
  </div>
</template>

<style scoped>
.cluster-demo {
  width: 100%;
  height: 90%;
  max-height: 95%;
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
  transition: opacity 0.3s ease;
  flex-shrink: 0;
}

.chart {
  width: 100%;
  height: auto;
  max-width: min(90%, 620px);
  max-height: calc(100% - 2em);
  flex-shrink: 1;
}

.grid line {
  stroke: currentColor;
  stroke-width: 0.5;
  opacity: 0.12;
}

.axis {
  stroke: currentColor;
  stroke-width: 1;
  opacity: 0.4;
}

.axis-label {
  fill: currentColor;
  opacity: 0.5;
  font-size: 10px;
  font-family: var(--slidev-code-font-family, monospace);
}

.point {
  transition: fill 0.5s ease;
  transform-box: fill-box;
  transform-origin: center;
  animation: pop-in 0.45s ease backwards;
}

.centroid {
  transform-box: fill-box;
  transform-origin: center;
  animation: pop-in 0.4s ease backwards;
}

.hull-enter-active {
  transition: opacity 0.8s ease;
}
.hull-enter-from {
  opacity: 0;
}

@keyframes pop-in {
  from {
    opacity: 0;
    transform: scale(0);
  }
  to {
    opacity: 1;
    transform: scale(1);
  }
}
</style>
