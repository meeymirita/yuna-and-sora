<script setup>
import { ref, onMounted, onBeforeUnmount, nextTick } from 'vue'
import { gsap } from 'gsap'
import dimensions from './assets/yuna-sora-dimensions.json'

const columns = ref([])
const active = ref(null)

function open(img) { active.value = img }
function close() { active.value = null }
function step(d) {
  if (!active.value) return
  const n = items.length
  const i = items.findIndex(it => it.id === active.value.id)
  active.value = items[(i + d + n) % n]
}
function onKey(e) {
  if (e.key === 'Escape') close()
  else if (e.key === 'ArrowRight') step(1)
  else if (e.key === 'ArrowLeft') step(-1)
}

const onEnter = (el, done) =>
  gsap.fromTo(el, { opacity: 0 }, { opacity: 1, duration: 0.3, onComplete: done })
const onImgEnter = (el, done) =>
  gsap.fromTo(el, { scale: 0.85, opacity: 0 }, { scale: 1, opacity: 1, duration: 0.4, ease: 'power3.out', onComplete: done })
const onLeave = (el, done) =>
  gsap.to(el, { opacity: 0, duration: 0.25, onComplete: done })
const colCount = ref(4)

const galleryImages = import.meta.glob('./assets/yuna-sora/*.png', {
  eager: true,
  query: '?url',
  import: 'default',
})

const items = Object.entries(galleryImages)
  .sort(([a], [b]) => a.localeCompare(b))
  .map(([path, src], i) => {
  const size = dimensions[path.split('/').at(-1)]
  const ratio = size.height / size.width
  return {
    id: i + 1,
    ratio,
    src,
    full: src,
    alt: `Иллюстрация Юны и Соры ${i + 1}`,
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
        <img :src="img.src" :alt="img.alt" draggable="false" />
      </div>
    </div>
    <Transition :css="false" @enter="onEnter" @leave="onLeave">
      <div v-if="active" class="lightbox" @click="close">
        <Transition :css="false" appear @enter="onImgEnter">
          <img :key="active.id" :src="active.full" :alt="active.alt" />
        </Transition>
        <button class="close" aria-label="Закрыть" @click.stop="close">✕</button>
        <button class="nav prev" aria-label="Назад" @click.stop="step(-1)">‹</button>
        <button class="nav next" aria-label="Вперёд" @click.stop="step(1)">›</button>
        <div class="counter">{{ active.id }} / {{ items.length }}</div>
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
  background: transparent;
}
.card img {
  width: 100%;
  height: 100%;
  object-fit: contain;
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
.nav {
  position: absolute;
  top: 50%;
  transform: translateY(-50%);
  width: 52px;
  height: 52px;
  border: 0;
  border-radius: 50%;
  background: rgba(255, 255, 255, 0.15);
  color: #fff;
  font-size: 32px;
  line-height: 1;
  padding-bottom: 4px;
  cursor: pointer;
}
.nav:hover { background: rgba(255, 255, 255, 0.3); }
.prev { left: 24px; }
.next { right: 24px; }
.counter {
  position: absolute;
  bottom: 20px;
  left: 50%;
  transform: translateX(-50%);
  color: #fff;
  font-size: 14px;
  opacity: 0.7;
}
.close:hover { background: rgba(255, 255, 255, 0.3); }
</style>
