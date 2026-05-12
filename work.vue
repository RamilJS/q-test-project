<q-separator class="q-my-md" />

<div class="calendar-widget q-px-md q-pb-md">

  <div class="event-calendar">

    <q-date
      v-model="selectedDate"
      flat
      bordered
      color="primary"
      mask="YYYY-MM-DD"
      :events="eventDates"
    />

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
          class="text-white"
        >
          <q-item-section>

            <q-item-label>
              {{ event.title }}
            </q-item-label>

            <q-item-label caption class="text-grey-3">
              {{ event.time }}
            </q-item-label>

          </q-item-section>
        </q-item>

      </q-list>

    </div>

  </div>

</div>

<q-separator class="q-my-md" />


const selectedDate = ref('2026-05-12')

const events = ref([
  {
    id: 1,
    date: '2026-05-06',
    title: 'Вебинар',
    time: '14:00'
  },
  {
    id: 2,
    date: '2026-05-06',
    title: 'Планёрка',
    time: '16:30'
  },
  {
    id: 3,
    date: '2026-05-15',
    title: 'Обучение',
    time: '11:00'
  }
])

const eventDates = computed(() =>
  events.value.map(e => e.date)
)

const selectedEvents = computed(() =>
  events.value.filter(
    e => e.date === selectedDate.value
  )
)

return {
  ...
  selectedDate,
  events,
  eventDates,
  selectedEvents
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
