<script setup>
import { ref, onMounted, onBeforeUnmount, nextTick } from 'vue'
import { gsap } from 'gsap'

const columns = ref([])
const active = ref(null)

function open(img) { active.value = img }
function close() { active.value = null }
function onKey(e) { if (e.key === 'Escape') close() }

const onEnter = (el, done) =>
  gsap.fromTo(el, { opacity: 0 }, { opacity: 1, duration: 0.3, onComplete: done })
const onImgEnter = (el, done) =>
  gsap.fromTo(el, { scale: 0.85, opacity: 0 }, { scale: 1, opacity: 1, duration: 0.4, ease: 'power3.out', onComplete: done })
const onLeave = (el, done) =>
  gsap.to(el, { opacity: 0, duration: 0.25, onComplete: done })
const colCount = ref(4)

// соотношения высота/ширина: длинная (tall), квадратная (square), средняя
const SHAPES = { tall: 1.6, square: 1, mid: 1.25 }
// порядок форм — длинная, квадратная, и т.д.
const pattern = ['tall', 'square', 'mid', 'square', 'tall', 'tall', 'square', 'mid']

const items = Array.from({ length: 40 }, (_, i) => {
  const shape = pattern[i % pattern.length]
  const ratio = SHAPES[shape]
  const w = 500
  const h = Math.round(w * ratio)
  return {
    id: i + 1,
    shape,
    ratio,
    src: `https://picsum.photos/seed/yuna${i + 1}/${w}/${h}`,
    full: `https://picsum.photos/seed/yuna${i + 1}/1600/${Math.round(1600 * ratio)}`,
  }
})

function calcColumns() {
  const w = window.innerWidth
  return w < 600 ? 2 : w < 900 ? 3 : w < 1300 ? 4 : 5
}

// каждую следующую картинку кладём в самую короткую колонку
function layout() {
  colCount.value = calcColumns()
  const cols = Array.from({ length: colCount.value }, () => ({ h: 0, list: [] }))
  for (const it of items) {
    const c = cols.reduce((a, b) => (b.h < a.h ? b : a))
    c.list.push(it)
    c.h += it.ratio
  }
  columns.value = cols.map(c => c.list)
}

let timer
function onResize() {
  clearTimeout(timer)
  timer = setTimeout(() => {
    if (calcColumns() !== colCount.value) layout()
  }, 150)
}

onMounted(async () => {
  layout()
  await nextTick()
  gsap.from('.card', { opacity: 0, y: 40, duration: 0.7, ease: 'power2.out', stagger: 0.04 })
  window.addEventListener('resize', onResize)
  window.addEventListener('keydown', onKey)
})
onBeforeUnmount(() => {
  window.removeEventListener('resize', onResize)
  window.removeEventListener('keydown', onKey)
})
</script>

<template>
  <main class="wall">
    <div v-for="(col, i) in columns" :key="colCount + '-' + i" class="col">
      <div
        v-for="img in col"
        :key="img.id"
        class="card"
        :style="{ aspectRatio: '1 / ' + img.ratio }"
        @click="open(img)"
      >
        <img :src="img.src" :alt="'photo ' + img.id" draggable="false" />
      </div>
    </div>
    <Transition :css="false" @enter="onEnter" @leave="onLeave">
      <div v-if="active" class="lightbox" @click="close">
        <Transition :css="false" appear @enter="onImgEnter">
          <img :src="active.full" :alt="'photo ' + active.id" />
        </Transition>
        <button class="close" aria-label="Закрыть" @click.stop="close">✕</button>
      </div>
    </Transition>
  </main>
</template>

<style scoped>
.wall {
  display: flex;
  gap: 16px;
  padding: 16px;
  align-items: flex-start;
}
.col {
  flex: 1;
  min-width: 0;
  display: flex;
  flex-direction: column;
  gap: 16px;
}
.card {
  width: 100%;
  border-radius: 16px;
  overflow: hidden;
  background: #222;
}
.card img {
  width: 100%;
  height: 100%;
  object-fit: cover;
  display: block;
  transition: transform 0.4s;
}
.card { cursor: zoom-in; }
.card:hover img { transform: scale(1.06); }
.lightbox {
  position: fixed;
  inset: 0;
  z-index: 100;
  background: rgba(0, 0, 0, 0.92);
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: zoom-out;
}
.lightbox img {
  max-width: 100vw;
  max-height: 100vh;
  object-fit: contain;
}
.close {
  position: absolute;
  top: 20px;
  right: 24px;
  width: 44px;
  height: 44px;
  border: 0;
  border-radius: 50%;
  background: rgba(255, 255, 255, 0.15);
  color: #fff;
  font-size: 20px;
  cursor: pointer;
}
.close:hover { background: rgba(255, 255, 255, 0.3); }
</style>
