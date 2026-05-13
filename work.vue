const calendarEvents = computed(() => {

  return datePicker.value.map((event) => ({

    id: event.id,

    title: event.name,

    date: event.start_date
      ? event.start_date.split("T")[0]
      : "",

    time: `${new Date(event.start_date).toLocaleTimeString(
      "ru-RU",
      {
        hour: "2-digit",
        minute: "2-digit",
      }
    )} - ${new Date(event.finish_date).toLocaleTimeString(
      "ru-RU",
      {
        hour: "2-digit",
        minute: "2-digit",
      }
    )}`,

    link: event.link,

    status: event.status_name,

    image: event.image_url,

  }));

});
