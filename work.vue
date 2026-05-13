const calendarEvents = computed(() => {

  return datePicker.value.map((event) => ({

    id: event.id,

    title: event.name,

    date: event.start_date
      ? event.start_date.split("T")[0]
      : "",

    time: `${event.start_date.slice(11, 16)} - ${event.finish_date.slice(11, 16)}`,

    link: event.link,

    status: event.status_name,

    image: event.image_url,

  }));

});

const calendarEvents = computed(() => {

  return datePicker.value.map((event) => {

    const startDate = event.start_date || "";
    const finishDate = event.finish_date || "";

    return {

      id: event.id,

      title: event.name,

      date: startDate
        ? startDate.split("T")[0]
        : "",

      time: startDate && finishDate
        ? `${startDate.slice(11, 16)} - ${finishDate.slice(11, 16)}`
        : "",

      link: event.link,

      status: event.status_name,

      image: event.image_url,

    };

  });

});
