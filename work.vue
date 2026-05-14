const eventDates = computed(() => {
  return Object.keys(eventsMap.value).map(
    d => d.replaceAll("-", "/")
  );
});
