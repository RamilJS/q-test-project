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
