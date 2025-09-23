<template>
  <q-page class="q-pa-md">

    <!-- Твои хлебные крошки -->
    <div class="new-card">
      <div class="breadcrumbs-container q-mt-sm q-gutter-sm row justify-between">
        <q-breadcrumbs class="q-mt-lg g-ml-lg text-primary">
          <q-breadcrumbs-el
            ref="highlightTarget"
            class="breadcrumb-item"
            label="Личный кабинет Адаптации"
            :to="{ path: '/adaptation' }"
          />
          <q-breadcrumbs-el label="Задачи по изучению информации"/>
        </q-breadcrumbs>
      </div>
    </div>

    <!-- Оверлей с подсветкой -->
    <transition name="fade">
      <div v-if="showOverlay" class="overlay">
        <div class="overlay-highlight" :style="highlightStyle"></div>

        <div class="overlay-controls">
          <q-btn color="primary" label="OK" @click="closeOverlay" />
        </div>
      </div>
    </transition>

  </q-page>
</template>

<script setup>
import { ref, onMounted, nextTick } from "vue"

const showOverlay = ref(false)
const highlightTarget = ref(null)
const highlightStyle = ref({})

const closeOverlay = () => {
  showOverlay.value = false
}

onMounted(async () => {
  await nextTick()
  const el = highlightTarget.value.$el // q-breadcrumbs-el
  if (el) {
    const rect = el.getBoundingClientRect()
    highlightStyle.value = {
      top: rect.top + window.scrollY + "px",
      left: rect.left + window.scrollX + "px",
      width: rect.width + "px",
      height: rect.height + "px"
    }
    showOverlay.value = true
    // Автоматически скрыть через 3 секунды
    setTimeout(() => {
      closeOverlay()
    }, 3000)
  }
})
</script>

<style>
.overlay {
  position: fixed;
  inset: 0;
  background: rgba(0,0,0,0.6);
  z-index: 2000;
}

.overlay-highlight {
  position: absolute;
  border: 3px solid #42a5f5;
  border-radius: 6px;
  box-shadow: 0 0 20px #42a5f5;
  background: transparent;
}

.overlay-controls {
  position: absolute;
  bottom: 30px;
  right: 30px;
}

.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.3s;
}
.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}
</style>

<template>
  <q-page class="q-pa-md">

    <!-- Хлебные крошки -->
    <div class="new-card">
      <div class="breadcrumbs-container q-mt-sm q-gutter-sm row justify-between">
        <q-breadcrumbs class="q-mt-lg g-ml-lg text-primary">
          <q-breadcrumbs-el
            ref="highlightTarget"
            class="breadcrumb-item"
            label="Личный кабинет Адаптации"
            :to="{ path: '/adaptation' }"
          />
          <q-breadcrumbs-el label="Задачи по изучению информации"/>
        </q-breadcrumbs>
      </div>
    </div>

    <!-- Оверлей с подсветкой -->
    <transition name="fade">
      <div v-if="showOverlay" class="overlay">
        <div class="overlay-highlight" :style="highlightStyle"></div>

        <!-- Подсказка -->
        <div
          v-if="tooltipStyle"
          class="overlay-tooltip bg-white text-dark shadow-4 q-pa-md rounded-borders"
          :style="tooltipStyle"
        >
          Если вы хотите вернуться назад,<br />
          нажмите <b>Личный кабинет</b>
          <div class="q-mt-sm text-right">
            <q-btn color="primary" size="sm" label="OK" @click="closeOverlay" />
          </div>
        </div>
      </div>
    </transition>

  </q-page>
</template>

<script setup>
import { ref, onMounted, nextTick } from "vue"

const showOverlay = ref(false)
const highlightTarget = ref(null)
const highlightStyle = ref({})
const tooltipStyle = ref(null)

const closeOverlay = () => {
  showOverlay.value = false
}

onMounted(async () => {
  await nextTick()
  const el = highlightTarget.value.$el
  if (el) {
    const rect = el.getBoundingClientRect()

    // Подсветка
    highlightStyle.value = {
      top: rect.top + window.scrollY + "px",
      left: rect.left + window.scrollX + "px",
      width: rect.width + "px",
      height: rect.height + "px"
    }

    // Позиция подсказки (над элементом)
    tooltipStyle.value = {
      top: rect.top + window.scrollY - 70 + "px", // над элементом
      left: rect.left + "px",
      minWidth: rect.width + 80 + "px"
    }

    showOverlay.value = true

    // Автоматически скрыть через 3 секунды
    setTimeout(() => {
      closeOverlay()
    }, 3000)
  }
})
</script>

<style>
.overlay {
  position: fixed;
  inset: 0;
  background: rgba(0,0,0,0.6);
  z-index: 2000;
}

.overlay-highlight {
  position: absolute;
  border: 3px solid #42a5f5;
  border-radius: 6px;
  box-shadow: 0 0 20px #42a5f5;
  background: transparent;
}

.overlay-tooltip {
  position: absolute;
  z-index: 2001;
}

.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.3s;
}
.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}
</style>


