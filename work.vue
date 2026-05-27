// Возвращает { month: 1..12, year: число } или null
const getDisplayedMonthFromDOM = () => {
  const viewEl = calendarWrapper.value?.querySelector('.q-date__view');
  if (!viewEl) return null;
  const text = viewEl.innerText.trim();
  // Пример: "Май 2025"
  const match = text.match(/([А-Яа-я]+)\s+(\d{4})/);
  if (!match) return null;
  const monthName = match[1].toLowerCase();
  const year = parseInt(match[2], 10);
  
  const monthsMap = {
    'января': 1, 'февраля': 2, 'марта': 3, 'апреля': 4, 'мая': 5, 'июня': 6,
    'июля': 7, 'августа': 8, 'сентября': 9, 'октября': 10, 'ноября': 11, 'декабря': 12
  };
  const month = monthsMap[monthName];
  if (!month) return null;
  return { month, year };
};

const bindTooltips = async () => {
  tooltip.value.visible = false;
  await nextTick();

  // Определяем, какой месяц и год сейчас реально показывает календарь
  const displayed = getDisplayedMonthFromDOM();
  if (!displayed) {
    console.warn('Не удалось определить месяц из DOM');
    return;
  }
  const { month: currentMonthReal, year: currentYearReal } = displayed;

  // Синхронизируем переменные (если нужно где-то ещё)
  currentMonth.value = currentMonthReal;
  currentYear.value = currentYearReal;

  const dayButtons = calendarWrapper.value?.querySelectorAll(
    ".q-date__calendar-item .q-btn"
  );
  if (!dayButtons || dayButtons.length === 0) return;

  dayButtons.forEach((dayEl) => {
    // Сброс
    dayEl.classList.remove("event-day");
    dayEl.onmouseover = null;
    dayEl.onmouseout = null;

    const content = dayEl.querySelector(".q-btn__content");
    if (!content) return;
    const dayText = content.innerText.trim();
    const day = parseInt(dayText, 10);
    if (isNaN(day)) return;

    // Формируем дату на основе РЕАЛЬНОГО месяца из DOM
    const monthStr = String(currentMonthReal).padStart(2, "0");
    const dayStr = String(day).padStart(2, "0");
    const date = `${currentYearReal}-${monthStr}-${dayStr}`;

    const events = getEventsByDate(date);
    if (!events.length) return;

    dayEl.classList.add("event-day");

    dayEl.onmouseover = (event) => {
      const rect = dayEl.getBoundingClientRect();
      const wrapperRect = calendarWrapper.value.getBoundingClientRect();
      const tooltipWidth = 260;
      let x = rect.left - wrapperRect.left + rect.width / 2;
      const minX = tooltipWidth / 2 + 12;
      const maxX = wrapperRect.width - tooltipWidth / 2 - 12;
      if (x < minX) x = minX;
      if (x > maxX) x = maxX;
      tooltip.value = {
        visible: true,
        x,
        y: rect.top - wrapperRect.top - 12,
        events,
      };
    };
    dayEl.onmouseout = () => {
      tooltip.value.visible = false;
    };
  });
};

const onNavigation = () => {
  // Не используем month, year из события – они могут быть неправильными из-за локали/багов.
  // Просто даём время на обновление DOM и перевешиваем обводку.
  setTimeout(() => {
    bindTooltips();
  }, 20);
};



        
