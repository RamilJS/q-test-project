import { ref, onMounted, nextTick } from "vue";

export default {
  setup() {
    const showGuide = ref(false);
    const guideTooltipStyle = ref({});
    const breadcrumbsTarget = ref(null);
    let guideTimeout = null;

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

    const addHighlight = () => {
      const el = breadcrumbsTarget.value?.$el || breadcrumbsTarget.value;
      if (el) el.classList.add("highlight-target");
    };

    const removeHighlight = () => {
      const el = breadcrumbsTarget.value?.$el || breadcrumbsTarget.value;
      if (el) el.classList.remove("highlight-target");
    };

    const positionTooltip = () => {
      const el = breadcrumbsTarget.value?.$el || breadcrumbsTarget.value;
      if (!el) return;
      const rect = el.getBoundingClientRect();

      guideTooltipStyle.value = {
        position: "fixed", // fixed, чтобы не зависеть от scroll контейнеров
        top: `${rect.bottom + 8}px`,
        left: `${rect.left}px`,
        zIndex: 2200,
      };
    };

    const startGuide = async () => {
      await nextTick();
      addHighlight();
      positionTooltip();
      showGuide.value = true;
      guideTimeout = setTimeout(closeGuide, 5000);
    };

    const closeGuide = () => {
      showGuide.value = false;
      removeHighlight();
      clearTimeout(guideTimeout);
      setCookie("breadcrumbs_guide_shown", "1", 365);
    };

    onMounted(() => {
      if (!getCookie("breadcrumbs_guide_shown")) {
        startGuide();
      }
    });

    return { showGuide, guideTooltipStyle, closeGuide, breadcrumbsTarget };
  },
};

.guide-overlay {
  position: fixed;
  inset: 0;
  background: rgba(0,0,0,0.7);
  z-index: 2000;
}

/* Подсветка элемента */
.highlight-target {
  box-shadow: 0 0 0 4px rgba(255, 215, 0, 0.9);
  border-radius: 6px;
}

/* Всплывающая подсказка */
.guide-tooltip {
  background: white;
  color: black;
  border-radius: 8px;
  padding: 16px;
  box-shadow: 0 4px 12px rgba(0,0,0,0.3);
}
