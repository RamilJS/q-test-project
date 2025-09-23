// src/composables/useGuideHighlight.js
import { ref, onMounted, onUnmounted, nextTick } from "vue";

export function useGuideHighlight(targetRef, options = {}) {
  const {
    cookieKey = "breadcrumbs_guide_shown",
    message = "Если вы хотите вернуться назад, нажмите на «Личный кабинет»",
    autoCloseMs = 5000,
  } = options;

  const showGuide = ref(false);
  const guideTooltipStyle = ref({});
  let guideTimeout = null;

  // --- cookies ---
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

  // --- подсветка ---
  const addHighlight = () => {
    const el = targetRef.value?.$el || targetRef.value;
    if (el) el.classList.add("highlight-target");
  };

  const removeHighlight = () => {
    const el = targetRef.value?.$el || targetRef.value;
    if (el) el.classList.remove("highlight-target");
  };

  // --- позиция подсказки ---
  const positionTooltip = () => {
    const el = targetRef.value?.$el || targetRef.value;
    if (!el) return;

    const rect = el.getBoundingClientRect();
    guideTooltipStyle.value = {
      position: "fixed",
      top: `${rect.bottom + 12}px`,
      left: `${rect.left}px`,
      zIndex: 2200,
    };
  };

  // --- запуск ---
  const startGuide = async () => {
    await nextTick();

    setTimeout(() => {
      const el = targetRef.value?.$el || targetRef.value;
      if (!el) return;

      addHighlight();
      positionTooltip();
      showGuide.value = true;

      guideTimeout = setTimeout(closeGuide, autoCloseMs);
    }, 100); // ждём, пока DOM устаканится
  };

  const closeGuide = () => {
    showGuide.value = false;
    removeHighlight();
    clearTimeout(guideTimeout);
    setCookie(cookieKey, "1", 365);
  };

  onMounted(() => {
    if (!getCookie(cookieKey)) {
      startGuide();
      window.addEventListener("resize", positionTooltip);
    }
  });

  onUnmounted(() => {
    window.removeEventListener("resize", positionTooltip);
    clearTimeout(guideTimeout);
  });

  return {
    showGuide,
    guideTooltipStyle,
    closeGuide,
    message,
  };
}
