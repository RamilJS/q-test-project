import { ref, computed, onMounted, nextTick, onUnmounted } from "vue";

export default {
  setup() {
    const showGuide = ref(false);
    const guideTooltipStyle = ref({});
    const breadcrumbsTarget = ref(null);
    let guideTimeout = null;

    const cookieName = "breadcrumbs_guide_shown"; // 🔑 имя cookie

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
        position: "fixed",
        top: `${rect.bottom + 12}px`,
        left: `${rect.left}px`,
        zIndex: 2200,
      };
    };

    const startGuide = async () => {
      await nextTick();

      setTimeout(() => {
        const el = breadcrumbsTarget.value?.$el || breadcrumbsTarget.value;
        if (!el) return;

        addHighlight();
        positionTooltip();
        showGuide.value = true;

        guideTimeout = setTimeout(closeGuide, 5000);
      }, 100);
    };

    const closeGuide = () => {
      showGuide.value = false;
      removeHighlight();
      clearTimeout(guideTimeout);
      setCookie(cookieName, "1", 365); // ✅ записываем cookie
    };

    onMounted(() => {
      if (!getCookie(cookieName)) { // ✅ проверяем cookie
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
      breadcrumbsTarget,
    };
  },
};

