const eventDates = computed(() => {

  return Object.keys(eventsMap.value)
    .map(date => date.replaceAll("-", "/"));

});

const getEventColor = (date) => {

  return "white";

};

