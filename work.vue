const selectedEvents = computed(() => {
  return eventsMap.value[selectedDate.value] || []
})

<!-- КАЛЕНДАРЬ -->
<div class="calendar-widget q-px-md q-pb-md">

  <div class="event-calendar">

    <q-date
      v-model="selectedDate"
      flat
      bordered
      color="primary"
      mask="YYYY-MM-DD"
      :events="eventDates"
    >

      <!-- КАСТОМНЫЙ РЕНДЕР ДНЯ -->
      <template #day="scope">

        <div class="full-width full-height flex flex-center">

          <q-btn
            flat
            round
            dense
            size="sm"
            :label="scope.day"
            class="calendar-day-btn"
            :class="{
              'event-day': hasEvents(scope.date)
            }"
            @click="onDateClick(scope.date)"
          >

            <!-- TOOLTIP -->
            <q-tooltip
              v-if="getEventsByDate(scope.date).length"
              anchor="top middle"
              self="bottom middle"
              class="calendar-tooltip"
            >

              <div
                v-for="event in getEventsByDate(scope.date)"
                :key="event.id"
                class="q-mb-md"
              >

                <div class="text-weight-bold">
                  {{ event.title }}
                </div>

                <div class="text-grey-3">
                  {{ event.time }}
                </div>

                <div class="text-grey-5">
                  {{ event.status }}
                </div>

              </div>

            </q-tooltip>

          </q-btn>

        </div>

      </template>

    </q-date>

    <!-- БЛОК СОБЫТИЙ -->
    <div class="events-block q-mt-md">

      <div
        v-if="selectedEvents.length === 0"
        class="no-events"
      >
        Сегодня нет событий
      </div>

      <q-list v-else dense>

        <q-item
          v-for="event in selectedEvents"
          :key="event.id"
          clickable
          v-ripple
          class="text-white"
          @click="goToEvent(event.link)"
        >

          <q-item-section>

            <q-item-label>
              {{ event.title }}
            </q-item-label>

            <q-item-label
              caption
              class="text-grey-3"
            >
              {{ event.time }}
            </q-item-label>

          </q-item-section>

        </q-item>

      </q-list>

    </div>

  </div>

</div>


const selectedDate = ref(
  new Date().toISOString().split('T')[0]
)

const datePicker = ref([])


// ЗАГРУЗКА СОБЫТИЙ
const fetchDatePickerInfo = async () => {

  try {

    const url = BACKEND_URL;

    const params = {
      collection_code: "VTBLGetEventCalendarEvents",
    };

    const response = await axios.post(
      url,
      new URLSearchParams(params).toString()
    );

    // ВАЖНО — весь массив
    datePicker.value = response.data.results;

  } catch (error) {

    console.error(
      "Ошибка загрузки данных календаря",
      error
    );

  }

};

const calendarEvents = computed(() => {

  return datePicker.value.map(event => ({

    id: event.id,

    title: event.name,

    date: event.start_date.split('T')[0],

    time: new Date(event.start_date)
      .toLocaleTimeString('ru-RU', {
        hour: '2-digit',
        minute: '2-digit'
      }),

    link: event.link,

    status: event.status_name,

    image: event.image_url

  }))

})

const eventsMap = computed(() => {

  const map = {}

  calendarEvents.value.forEach(event => {

    if (!map[event.date]) {
      map[event.date] = []
    }

    map[event.date].push(event)

  })

  return map

})

const eventDates = computed(() => {
  return Object.keys(eventsMap.value)
})

const getEventsByDate = (date) => {
  return eventsMap.value[date] || []
}

const hasEvents = (date) => {
  return getEventsByDate(date).length > 0
}

const onDateClick = (date) => {

  selectedDate.value = date

  const events = getEventsByDate(date)

  if (!events.length) return

  // Переход по первому событию
  window.location.href = events[0].link

}

const goToEvent = (link) => {
  window.location.href = link
}

return {

  ...

  selectedDate,
  datePicker,

  calendarEvents,
  eventsMap,

  eventDates,
  selectedEvents,

  getEventsByDate,
  hasEvents,

  onDateClick,
  goToEvent

}

.calendar-widget {
  margin-top: 16px;
}

.event-calendar {
  background: #1da1f2;
  border-radius: 12px;
  overflow: hidden;
}

.event-calendar .q-date {
  background: #1da1f2;
  color: white;
}

.event-calendar .q-date__navigation {
  color: white;
}

.event-calendar .q-date__calendar-weekdays {
  background: white;
  color: #0c205a;
  font-weight: 700;
}

.event-calendar .q-date__calendar-item div {
  color: white;
}

.event-calendar .q-date__today {
  border: 2px solid white;
  border-radius: 50%;
}

.calendar-day-btn {
  color: white;
  width: 32px;
  height: 32px;
}

.event-day {
  border: 2px solid rgba(255,255,255,0.8);
  border-radius: 50%;
}

.event-day:hover {
  background: rgba(255,255,255,0.2);
}

.events-block {
  background: #1da1f2;
  color: white;
  padding: 12px;
  border-top: 1px solid rgba(255,255,255,0.2);
  min-height: 80px;
}

.no-events {
  text-align: center;
  font-size: 16px;
}

.calendar-tooltip {
  max-width: 260px;
  font-size: 13px;
}
