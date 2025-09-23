const startGuide = async () => {
  await nextTick();       // дожидаемся отрисовки Vue
  addHighlight();         // подсветка элемента
  positionTooltip();      // вычисляем координаты
  showGuide.value = true;

  guideTimeout = setTimeout(closeGuide, 5000);
};

onMounted(() => {
  if (!getCookie("breadcrumbs_guide_shown")) {
    startGuide();
    window.addEventListener("resize", positionTooltip);
  }
});

onUnmounted(() => {
  window.removeEventListener("resize", positionTooltip);
});
