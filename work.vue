const dayButtons =
  calendarWrapper.value?.querySelectorAll(
    ".q-date__calendar-item .q-btn"
  );

dayButtons.forEach((el) => {
  console.log(el.outerHTML);
});
