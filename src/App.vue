<script setup>
import { ref, onMounted, onBeforeUnmount, nextTick } from 'vue'
import { gsap } from 'gsap'

const columns = ref([])
const active = ref(null)
const ageGateOpen = ref(true)
const ageRejected = ref(false)

function confirmAge() {
  localStorage.setItem('yuna-sora-age-confirmed', 'true')
  ageGateOpen.value = false
}

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
const SHAPES = { tall: 1.6, square: 1, mid: 1.25 }
const pattern = ['tall', 'square', 'mid', 'square', 'tall', 'tall', 'square', 'mid']

const galleryThumbs = import.meta.glob('./assets/yuna-sora-optimized/thumbs/*.webp', {
  eager: true,
  query: '?url',
  import: 'default',
})
const galleryFull = import.meta.glob('./assets/yuna-sora-optimized/full/*.webp', {
  eager: true,
  query: '?url',
  import: 'default',
})

const items = Object.entries(galleryThumbs)
  .sort(([a], [b]) => a.localeCompare(b))
  .map(([path, src], i) => {
  const shape = pattern[i % pattern.length]
  const fullPath = `./assets/yuna-sora-optimized/full/${path.split('/').at(-1)}`
  return {
    id: i + 1,
    ratio: SHAPES[shape],
    src,
    full: galleryFull[fullPath],
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
  ageGateOpen.value = localStorage.getItem('yuna-sora-age-confirmed') !== 'true'
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
        <img :src="img.src" :alt="img.alt" loading="lazy" decoding="async" draggable="false" />
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
  <div v-if="ageGateOpen" class="age-overlay">
    <section class="age-dialog" role="dialog" aria-modal="true" aria-labelledby="age-title">
      <template v-if="!ageRejected">
        <div class="age-badge">18+</div>
        <h1 id="age-title">Контент для взрослых</h1>
        <p>На сайте есть материалы, предназначенные только для совершеннолетних.</p>
        <p>Подтвердите, что вам исполнилось 18 лет, чтобы продолжить.</p>
        <div class="age-actions">
          <button class="age-continue" @click="confirmAge">Мне есть 18 — продолжить</button>
          <button class="age-leave" @click="ageRejected = true">Мне нет 18</button>
        </div>
      </template>
      <template v-else>
        <div class="age-badge">18+</div>
        <h1 id="age-title">Доступ закрыт</h1>
        <p>Этот сайт предназначен только для совершеннолетних.</p>
      </template>
    </section>
  </div>
</template>

<style scoped>
.wall {
  display: flex;
  gap: 8px;
  padding: 8px;
  align-items: flex-start;
}
.col {
  flex: 1;
  min-width: 0;
  display: flex;
  flex-direction: column;
  gap: 8px;
}
.card {
  width: 100%;
  border-radius: 16px;
  overflow: hidden;
  background: #f5eaf1;
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
.age-overlay {
  position: fixed;
  inset: 0;
  z-index: 200;
  display: grid;
  place-items: center;
  padding: 24px;
  background: rgba(8, 7, 12, 0.88);
  backdrop-filter: blur(12px);
}
.age-dialog {
  width: min(100%, 440px);
  padding: 36px;
  border: 1px solid rgba(255, 255, 255, 0.14);
  border-radius: 24px;
  background: #1b1720;
  color: #fff;
  text-align: center;
  box-shadow: 0 24px 80px rgba(0, 0, 0, 0.45);
}
.age-badge {
  display: grid;
  place-items: center;
  width: 58px;
  height: 58px;
  margin: 0 auto 20px;
  border: 1px solid rgba(255, 171, 211, 0.45);
  border-radius: 50%;
  color: #ffabd3;
  font-weight: 700;
}
.age-dialog h1 { margin-bottom: 14px; font-size: 24px; }
.age-dialog p { margin: 0 0 10px; color: rgba(255, 255, 255, 0.72); line-height: 1.55; }
.age-actions { display: grid; gap: 10px; margin-top: 24px; }
.age-actions button {
  min-height: 48px;
  border: 0;
  border-radius: 12px;
  font: inherit;
  cursor: pointer;
}
.age-continue { background: #f49ac7; color: #24131d; font-weight: 700; }
.age-continue:hover { background: #ffb5d9; }
.age-leave { background: transparent; color: rgba(255, 255, 255, 0.72); }
.age-leave:hover { background: rgba(255, 255, 255, 0.08); }
</style>
