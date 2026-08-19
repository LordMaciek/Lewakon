<script setup>
import { ref, computed } from 'vue'

const props = defineProps({
  step: { type: Number, default: 99 },
})

// Two latent traits the audience never "sees" directly — only through
// their joint effect on the observed tests below.
const verbal = ref(0)
const quant = ref(0)

const verbalColor = '#38bdf8' // sky
const quantColor = '#f97316' // orange

// Fixed loadings (how strongly each test depends on each trait) plus a
// small fixed idiosyncratic noise term per item — generated once with a
// seeded PRNG so the "measurement error" doesn't reshuffle on every drag,
// only the latent-driven part moves.
function mulberry32(seed) {
  return function () {
    seed |= 0
    seed = (seed + 0x6d2b79f5) | 0
    let t = Math.imul(seed ^ (seed >>> 15), 1 | seed)
    t = (t + Math.imul(t ^ (t >>> 7), 61 | t)) ^ t
    return ((t ^ (t >>> 14)) >>> 0) / 4294967296
  }
}
const rand = mulberry32(2024)

const items = [
  { label: 'Synonyms', v: 0.85, q: 0.05 },
  { label: 'Analogies', v: 0.75, q: 0.15 },
  { label: 'Reading comp.', v: 0.80, q: 0.00 },
  { label: 'Vocabulary', v: 0.70, q: 0.10 },
  { label: 'Arithmetic', v: 0.05, q: 0.80 },
  { label: 'Number series', v: 0.10, q: 0.85 },
  { label: 'Geometry', v: 0.00, q: 0.75 },
  { label: 'Algebra', v: 0.15, q: 0.80 },
].map((it) => ({ ...it, noise: (rand() - 0.5) * 0.9 }))

function hexLerp(hexA, hexB, t) {
  const a = [1, 3, 5].map((i) => parseInt(hexA.slice(i, i + 2), 16))
  const b = [1, 3, 5].map((i) => parseInt(hexB.slice(i, i + 2), 16))
  const c = a.map((v, i) => Math.round(v + (b[i] - v) * t))
  return `rgb(${c[0]}, ${c[1]}, ${c[2]})`
}

const observed = computed(() =>
  items.map((it) => {
    const raw = it.v * verbal.value + it.q * quant.value + it.noise
    const clamped = Math.max(-3, Math.min(3, raw))
    const ratio = it.v / (it.v + it.q || 1) // which trait this item "belongs" to
    return {
      label: it.label,
      value: clamped,
      pct: (clamped / 3) * 50, // % of half-height, for the bar
      color: hexLerp(quantColor, verbalColor, ratio),
    }
  }),
)

const showAnnotation = computed(() => props.step >= 1)
const showGrouping = computed(() => props.step >= 2)
</script>

<template>
  <div class="factor-sim">
    <div class="sliders">
      <div class="slider-block">
        <label class="slider-label" :style="{ color: verbalColor }">
          verbal ability
          <span class="value">{{ verbal >= 0 ? '+' : '' }}{{ verbal.toFixed(2) }}</span>
        </label>
        <input
          v-model.number="verbal"
          type="range"
          min="-3"
          max="3"
          step="0.05"
          class="slider"
          :style="{ '--accent': verbalColor }"
        />
      </div>
      <div class="slider-block">
        <label class="slider-label" :style="{ color: quantColor }">
          quant reasoning
          <span class="value">{{ quant >= 0 ? '+' : '' }}{{ quant.toFixed(2) }}</span>
        </label>
        <input
          v-model.number="quant"
          type="range"
          min="-3"
          max="3"
          step="0.05"
          class="slider"
          :style="{ '--accent': quantColor }"
        />
      </div>
    </div>

    <div class="bars">
      <div v-for="(o, i) in observed" :key="i" class="bar-col">
        <div class="bar-track">
          <div class="baseline" />
          <div
            class="bar-fill"
            :style="{
              height: Math.abs(o.pct) + '%',
              bottom: o.pct >= 0 ? '50%' : 50 + o.pct + '%',
              backgroundColor: o.color,
            }"
          />
        </div>
        <div class="bar-value">{{ o.value >= 0 ? '+' : '' }}{{ o.value.toFixed(1) }}</div>
        <div class="bar-label">{{ o.label }}</div>
      </div>
    </div>

    <div v-if="showGrouping" class="grouping">
      <div class="group-tag" :style="{ color: verbalColor, borderColor: verbalColor }">verbal factor</div>
      <div class="group-tag" :style="{ color: quantColor, borderColor: quantColor }">quant factor</div>
    </div>

    <div v-if="showAnnotation" class="annotation">
      moving one hidden trait shifts several tests at once — that's the whole idea of a common factor
    </div>
  </div>
</template>

<style scoped>
.factor-sim {
  width: 100%;
  height: 90%;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 1.2em;
  font-family: var(--slidev-code-font-family, monospace);
  box-sizing: border-box;
  padding: 0.5em;
}

.sliders {
  width: 100%;
  max-width: 560px;
  display: flex;
  flex-direction: column;
  gap: 0.6em;
}

.slider-block {
  display: flex;
  flex-direction: column;
  gap: 0.2em;
}

.slider-label {
  font-size: 0.8rem;
  letter-spacing: 0.08em;
  display: flex;
  justify-content: space-between;
  text-transform: uppercase;
  opacity: 0.9;
}

.value {
  opacity: 0.7;
  font-variant-numeric: tabular-nums;
}

.slider {
  -webkit-appearance: none;
  width: 100%;
  height: 4px;
  border-radius: 2px;
  background: color-mix(in srgb, var(--accent) 35%, transparent);
  outline: none;
}
.slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  width: 16px;
  height: 16px;
  border-radius: 50%;
  background: var(--accent);
  cursor: pointer;
  box-shadow: 0 0 8px var(--accent);
}
.slider::-moz-range-thumb {
  width: 16px;
  height: 16px;
  border: none;
  border-radius: 50%;
  background: var(--accent);
  cursor: pointer;
  box-shadow: 0 0 8px var(--accent);
}

.bars {
  width: 100%;
  max-width: 620px;
  display: flex;
  justify-content: space-between;
  align-items: flex-end;
  gap: 0.4em;
}

.bar-col {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.3em;
}

.bar-track {
  position: relative;
  width: 60%;
  height: 130px;
}

.baseline {
  position: absolute;
  top: 50%;
  left: -6px;
  right: -6px;
  height: 1px;
  background: currentColor;
  opacity: 0.3;
}

.bar-fill {
  position: absolute;
  width: 100%;
  border-radius: 3px;
  transition: height 0.15s ease, bottom 0.15s ease, background-color 0.4s ease;
}

.bar-value {
  font-size: 0.65rem;
  opacity: 0.6;
  font-variant-numeric: tabular-nums;
}

.bar-label {
  font-size: 0.6rem;
  opacity: 0.55;
  text-align: center;
  line-height: 1.1;
  min-height: 2.2em;
}

.grouping {
  display: flex;
  gap: 1.5em;
}

.group-tag {
  font-size: 0.7rem;
  letter-spacing: 0.1em;
  text-transform: uppercase;
  padding: 0.2em 0.7em;
  border: 1px solid;
  border-radius: 999px;
  opacity: 0.9;
}

.annotation {
  font-size: 0.85rem;
  opacity: 0.7;
  text-align: center;
  max-width: 34em;
  line-height: 1.4;
}
</style>
