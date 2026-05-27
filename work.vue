const bindTooltips = async () => {
  tooltip.value.visible = false;
  await nextTick();

  const dayButtons = calendarWrapper.value?.querySelectorAll(
    ".q-date__calendar-item .q-btn"
  );
  if (!dayButtons || dayButtons.length === 0) return;

  dayButtons.forEach((dayEl) => {
    // НЕ удаляем класс event-day глобально!
    // только убираем старые обработчики тултипов
    dayEl.onmouseover = null;
    dayEl.onmouseout = null;

    const content = dayEl.querySelector(".q-btn__content");
    if (!content) return;
    const dayText = content.innerText.trim();
    const day = parseInt(dayText, 10);
    if (isNaN(day)) return;

    // Получаем месяц и год из DOM (надёжно)
    const { month, year } = getCurrentMonthYearFromDOM();
    if (!month || !year) {
      // fallback на переменные
      var monthVar = currentMonth.value;
      var yearVar = currentYear.value;
    } else {
      var monthVar = month;
      var yearVar = year;
    }

    const monthStr = String(monthVar).padStart(2, "0");
    const dayStr = String(day).padStart(2, "0");
    const date = `${yearVar}-${monthStr}-${dayStr}`;

    const events = getEventsByDate(date);
    if (events.length) {
      dayEl.classList.add("event-day");
    } else {
      // Не удаляем класс, если его нет – ничего не делаем
      // (класс может быть от предыдущей даты, но это не страшно)
    }

    // Тултип вешаем всегда, даже если событий нет – но тогда тултип будет пустой, лучше проверять
    if (events.length) {
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
    }
  });
};
