<template>
  <q-page class="q-pa-md">
    <!-- Хлебные крошки -->
    <div class="new-card">
      <div class="breadcrumbs-container q-mt-sm q-gutter-sm row justify-between">
        <q-breadcrumbs class="q-mt-lg g-ml-lg text-primary">
          <q-breadcrumbs-el
            class="breadcrumb-item"
            label="Личный кабинет Адаптации"
            :to="{ path: '/adaptation' }"
          />
          <q-breadcrumbs-el label="Задачи по изучению информации"/>
        </q-breadcrumbs>
      </div>
    </div>

    <!-- Оверлей -->
    <transition name="fade">
      <div v-if="showOverlay" class="overlay" @click.self="closeOverlay">
        <div class="overlay-dim"></div>

        <!-- подсветка -->
        <div class="overlay-highlight" :style="highlightStyle"></div>

        <!-- подсказка -->
        <div v-if="tooltipStyle" class="overlay-tooltip" :style="tooltipStyle">
          Если вы хотите вернуться назад,<br/>
          нажмите <b>Личный кабинет</b>
          <div class="q-mt-sm text-right">
            <q-btn color="primary" size="sm" label="OK" @click.stop="closeOverlay" />
          </div>
        </div>
      </div>
    </transition>
  </q-page>
</template>

<script>
import { ref, onMounted, nextTick, onBeforeUnmount } from 'vue'

export default {
  setup () {
    const showOverlay = ref(false)
    const highlightStyle = ref({})
    const tooltipStyle = ref(null)
    let timer = null

    function closeOverlay () {
      showOverlay.value = false
      if (timer) {
        clearTimeout(timer)
        timer = null
      }
    }

    onMounted(async () => {
      await nextTick()

      const el = document.querySelector('.breadcrumb-item')
      if (!el) return

      const rect = el.getBoundingClientRect()
      const pad = 6 // отступ вокруг подсветки

      highlightStyle.value = {
        top: `${rect.top + window.scrollY - pad}px`,
        left: `${rect.left + window.scrollX - pad}px`,
        width: `${rect.width + pad * 2}px`,
        height: `${rect.height + pad * 2}px`
      }

      tooltipStyle.value = {
        top: `${rect.bottom + window.scrollY + 10}px`,
        left: `${rect.left + window.scrollX}px`,
        minWidth: '220px'
      }

      showOverlay.value = true
      timer = setTimeout(closeOverlay, 3000)
    })

    onBeforeUnmount(() => {
      if (timer) clearTimeout(timer)
    })

    // важно: возвращаем всё, что используем в шаблоне
    return {
      showOverlay,
      highlightStyle,
      tooltipStyle,
      closeOverlay
    }
  }
}
</script>

<style scoped>
.overlay {
  position: fixed;
  inset: 0;
  z-index: 2000;
}
.overlay-dim {
  position: absolute;
  inset: 0;
  background: rgba(0,0,0,0.6);
}
.overlay-highlight {
  position: absolute;
  border: 3px solid #42a5f5;
  border-radius: 6px;
  box-shadow: 0 0 20px #42a5f5;
  z-index: 2002;
  pointer-events: none;
}
.overlay-tooltip {
  position: absolute;
  z-index: 2003;
  background: #fff;
  padding: 12px;
  border-radius: 6px;
  box-shadow: 0 6px 24px rgba(0,0,0,0.2);
}
.fade-enter-active,
.fade-leave-active { transition: opacity .25s; }
.fade-enter-from, .fade-leave-to { opacity: 0; }
</style>
