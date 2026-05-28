const attachTooltipListeners = async () => {

  await nextTick();

  const dayButtons =
    calendarWrapper.value?.querySelectorAll(
      ".q-date__calendar-item .q-btn"
    );

  if (!dayButtons) return;

  dayButtons.forEach((dayEl) => {

    dayEl.onmouseenter = null;
    dayEl.onmouseleave = null;

    const timestamp =
      dayEl.getAttribute("data-day");

    if (!timestamp) return;

    const normalizedDate =
      timestamp.replaceAll("/", "-");

    const events =
      getEventsByDate(normalizedDate);

    if (!events.length) return;

    dayEl.onmouseenter = () => {

      const rect =
        dayEl.getBoundingClientRect();

      const wrapperRect =
        calendarWrapper.value.getBoundingClientRect();

      const tooltipWidth = 260;

      let x =
        rect.left -
        wrapperRect.left +
        rect.width / 2;

      const minX =
        tooltipWidth / 2 + 12;

      const maxX =
        wrapperRect.width -
        tooltipWidth / 2 -
        12;

      if (x < minX) {
        x = minX;
      }

      if (x > maxX) {
        x = maxX;
      }

      tooltip.value = {
        visible: true,

        x,

        y:
          rect.top -
          wrapperRect.top -
          12,

        events,
      };
    };

    dayEl.onmouseleave = () => {
      tooltip.value.visible = false;
    };
  });
};
