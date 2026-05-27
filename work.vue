const onNavigation = ({ month, year }) => {

  currentMonth.value = month + 1;
  currentYear.value = year;

  requestAnimationFrame(() => {

    requestAnimationFrame(() => {

      bindTooltips();

    });

  });
};
