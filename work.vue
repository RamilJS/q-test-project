const getCurrentMonthYearFromDOM = () => {
  const wrapper = calendarWrapper.value;
  if (!wrapper) return { month: null, year: null };
  
  // Пробуем найти заголовок – у Quasar это может быть .q-date__view или .q-date__title
  let viewEl = wrapper.querySelector('.q-date__view');
  if (!viewEl) viewEl = wrapper.querySelector('.q-date__title');
  if (!viewEl) {
    // Если не нашли – пробуем найти любой элемент с текстом, похожим на "Май 2025"
    const allText = wrapper.querySelectorAll('.q-date__header span, .q-date__header div');
    for (let el of allText) {
      const text = el.innerText.trim();
      if (/\d{4}/.test(text)) {
        viewEl = el;
        break;
      }
    }
  }
  if (!viewEl) {
    console.warn('Не удалось найти заголовок месяца в календаре');
    return { month: null, year: null };
  }
  
  const text = viewEl.innerText.trim();
  // Пример: "Май 2025" или "May 2025"
  const match = text.match(/([А-Яа-яA-Za-z]+)\s+(\d{4})/);
  if (!match) return { month: null, year: null };
  
  const monthName = match[1].toLowerCase();
  const year = parseInt(match[2], 10);
  
  const monthsMap = {
    'января': 1, 'январь': 1, 'january': 1, 'jan': 1,
    'февраля': 2, 'февраль': 2, 'february': 2, 'feb': 2,
    'марта': 3, 'март': 3, 'march': 3, 'mar': 3,
    'апреля': 4, 'апрель': 4, 'april': 4, 'apr': 4,
    'мая': 5, 'май': 5, 'may': 5,
    'июня': 6, 'июнь': 6, 'june': 6, 'jun': 6,
    'июля': 7, 'июль': 7, 'july': 7, 'jul': 7,
    'августа': 8, 'август': 8, 'august': 8, 'aug': 8,
    'сентября': 9, 'сентябрь': 9, 'september': 9, 'sep': 9,
    'октября': 10, 'октябрь': 10, 'october': 10, 'oct': 10,
    'ноября': 11, 'ноябрь': 11, 'november': 11, 'nov': 11,
    'декабря': 12, 'декабрь': 12, 'december': 12, 'dec': 12
  };
  const month = monthsMap[monthName];
  if (!month) return { month: null, year: null };
  
  return { month, year };
};





const onNavigation = async ({ month, year }) => {
  // month от 0 до 11, переводим в 1..12 (для синхронизации переменных, если они где-то ещё нужны)
  currentMonth.value = month + 1;
  currentYear.value = year;
  
  // Ждём, пока Quasar перерисует DOM
  await nextTick();
  requestAnimationFrame(() => {
    bindTooltips();
  });
};
