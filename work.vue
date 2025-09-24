onMounted(() => {
  if (!getCookie(cookieName)) {
    startGuide();
    window.addEventListener("resize", positionTooltip);
  } else {
    // ✅ гарантированно убираем рамку, если гид уже был показан
    removeHighlight();
  }
});

onUnmounted(() => {
  window.removeEventListener("resize", positionTooltip);
  clearTimeout(guideTimeout);
  removeHighlight(); // ✅ на всякий случай
});
