const onDateClick = (date) => {
  console.log('clicked date:', date);
  console.log('eventsMap keys:', Object.keys(eventsMap.value));
  console.log('events for this date:', getEventsByDate(date));
};

const fetchDatePickerInfo = async () => {
  try {
    const response = await axios.post(
      url,
      new URLSearchParams(params).toString()
    );
    datePicker.value = response.data.results;
    console.log('raw results:', response.data.results);        // что пришло с сервера
    console.log('calendarEvents:', calendarEvents.value);      // после маппинга
    console.log('eventsMap:', eventsMap.value);                // итоговая карта
  } catch (error) {
    console.error("Ошибка загрузки данных календаря", error);
  }
};
