<div class="new-card">

  <div class="breadcrumbs-container q-mt-sm q-gutter-sm row justify-between">
    <q-breadcrumbs class="q-mt-lg g-ml-lg text-primary">
      <q-breadcrumbs-el
        ref="breadcrumbsTarget"
        class="breadcrumb-item highlight-target"
        label="Личный кабинет Адаптации"
        :to="{ path: '/adaptation' }"
      />
      <q-breadcrumbs-el label="Задачи по изучению информации"/>
    </q-breadcrumbs>
  </div>

  <div>Тут остальная логика страницы</div>

  <!-- Оверлей -->
  <div v-if="showGuide" class="guide-overlay">
    <div class="guide-tooltip">
      <p>Если вы хотите вернуться назад, нажмите на «Личный кабинет»</p>
      <q-btn color="primary" label="ОК" @click="closeGuide" />
    </div>
  </div>
</div>



import { ref, onMounted, nextTick } from "vue";

export default {
  setup() {
    const showGuide = ref(false);
    const guideTimeout = ref(null);

    const setCookie = (name, value, days) => {
      const d = new Date();
      d.setTime(d.getTime() + days * 24 * 60 * 60 * 1000);
      document.cookie = `${name}=${value};expires=${d.toUTCString()};path=/`;
    };

    const getCookie = (name) => {
      const value = `; ${document.cookie}`;
      const parts = value.split(`; ${name}=`);
      if (parts.length === 2) return parts.pop().split(";").shift();
    };

    const startGuide = () => {
      showGuide.value = true;
      guideTimeout.value = setTimeout(() => {
        closeGuide();
      }, 5000);
    };

    const closeGuide = () => {
      showGuide.value = false;
      clearTimeout(guideTimeout.value);
      setCookie("breadcrumbs_guide_shown", "1", 365); // год
    };

    onMounted(async () => {
      await nextTick();
      if (!getCookie("breadcrumbs_guide_shown")) {
        startGuide();
      }
    });

    return { showGuide, closeGuide };
  }
};

.guide-overlay {
  position: fixed;
  inset: 0;
  background: rgba(0,0,0,0.7);
  z-index: 2000;
  display: flex;
  align-items: flex-start;
  justify-content: flex-start;
}

/* Подсветка */
.highlight-target {
  position: relative;
  z-index: 2100;
  box-shadow: 0 0 0 4px rgba(255, 215, 0, 0.9);
  border-radius: 6px;
}

/* Всплывающая подсказка */
.guide-tooltip {
  position: absolute;
  top: 80px; /* подгони под твой UI */
  left: 20px;
  max-width: 260px;
  background: white;
  color: black;
  border-radius: 8px;
  padding: 16px;
  z-index: 2200;
  box-shadow: 0 4px 12px rgba(0,0,0,0.3);
}

