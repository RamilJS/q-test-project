const startGuide = async () => {
  await nextTick();

  // небольшая задержка для уверенности, что DOM дорисовался
  setTimeout(() => {
    addHighlight();
    positionTooltip();
    showGuide.value = true;
    guideTimeout = setTimeout(closeGuide, 5000);
  }, 100);
};

const positionTooltip = () => {
  const el = breadcrumbsTarget.value?.$el || breadcrumbsTarget.value;
  if (!el) {
    console.warn("Элемент для подсветки не найден");
    return;
  }
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
    if (!el) return; // если элемента нет, ничего не делаем

    addHighlight();
    positionTooltip();
    showGuide.value = true;

    guideTimeout = setTimeout(closeGuide, 5000);
  }, 100);
};

const closeGuide = () => {
  showGuide.value = false;
  removeHighlight(); // гарантированно убираем рамку
  clearTimeout(guideTimeout);
  setCookie(cookieName, "1", 365);
};
