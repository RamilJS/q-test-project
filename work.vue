const eventDates = computed(() => {

  return Object.keys(eventsMap.value)
    .filter(Boolean)
    .map(date => {

      const [year, month, day] =
        date.split("-");

      return `${year}/${month}/${day}`;

    });

});

console.log(eventDates.value);
