<q-date
  v-model="selectedDate"
  flat
  bordered
  color="primary"
  mask="YYYY-MM-DD"
>

  <!-- КАСТОМНЫЙ ДЕНЬ -->
  <template #day="scope">

    <div class="calendar-day-wrapper">

      <div
        class="calendar-day"
        :class="{
          'event-day': hasEvents(scope.date),
          'today-day': scope.today
        }"
        @click="onDateClick(scope.date)"
      >

        {{ scope.day }}

        <!-- ТОЧКА СОБЫТИЯ -->
        <div
          v-if="hasEvents(scope.date)"
          class="event-dot"
        />

        <!-- TOOLTIP -->
        <q-tooltip
          v-if="hasEvents(scope.date)"
          class="calendar-tooltip"
          anchor="top middle"
          self="bottom middle"
          :offset="[0, 10]"
        >

          <div
            v-for="event in getEventsByDate(scope.date)"
            :key="event.id"
            class="tooltip-event"
          >

            <div class="text-weight-bold">
              {{ event.title }}
            </div>

            <div class="tooltip-time">
              {{ event.time }}
            </div>

            <div class="tooltip-status">
              {{ event.status }}
            </div>

          </div>

        </q-tooltip>

      </div>

    </div>

  </template>

</q-date>

import { ref, computed } from "vue";

const selectedDate = ref(
  new Date().toISOString().split("T")[0]
);

const datePicker = ref([]);

const calendarEvents = computed(() => {

  return datePicker.value.map((event) => ({

    id: event.id,

    title: event.name,

    date: event.start_date.split("T")[0],

    time: new Date(event.start_date)
      .toLocaleTimeString("ru-RU", {
        hour: "2-digit",
        minute: "2-digit",
      }),

    link: event.link,

    status: event.status_name,

    image: event.image_url,

  }));

});

const eventsMap = computed(() => {

  const map = {};

  calendarEvents.value.forEach((event) => {

    if (!map[event.date]) {
      map[event.date] = [];
    }

    map[event.date].push(event);

  });

  return map;

});

const getEventsByDate = (date) => {
  return eventsMap.value[date] || [];
};

const hasEvents = (date) => {
  return getEventsByDate(date).length > 0;
};

const selectedEvents = computed(() => {
  return eventsMap.value[selectedDate.value] || [];
});

const onDateClick = (date) => {

  selectedDate.value = date;

  const events = getEventsByDate(date);

  if (!events.length) {
    return;
  }

  window.location.href = events[0].link;

};

const goToEvent = (link) => {
  window.location.href = link;
};


.calendar-widget {
  margin-top: 16px;
}

.event-calendar {
  background: #1da1f2;

  border-radius: 12px;

  overflow: hidden;

  display: flex;
  flex-direction: column;
}

.event-calendar .q-date {
  background: #1da1f2;

  color: white;

  margin: 0 auto;

  margin-top: 19px;
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

/* TODAY */
.event-calendar .q-date__today {
  border: 2px solid white;

  border-radius: 50%;
}

/* DAY WRAPPER */
.calendar-day-wrapper {
  width: 100%;
  height: 100%;

  display: flex;

  align-items: center;
  justify-content: center;
}

/* DAY */
.calendar-day {
  width: 34px;
  height: 34px;

  border-radius: 50%;

  display: flex;

  align-items: center;
  justify-content: center;

  position: relative;

  cursor: pointer;

  transition: all 0.2s ease;

  color: white;

  font-size: 14px;
}

/* HOVER */
.calendar-day:hover {
  background: rgba(255,255,255,0.15);

  transform: scale(1.05);
}

/* EVENT DAY */
.event-day {
  border: 2px solid white;

  background: rgba(255,255,255,0.08);

  box-shadow:
    0 0 0 2px rgba(255,255,255,0.08),
    0 0 8px rgba(255,255,255,0.3);
}

/* TODAY */
.today-day {
  border: 2px solid #ffd54f;

  background: rgba(255,255,255,0.18);
}

/* EVENT DOT */
.event-dot {
  width: 6px;
  height: 6px;

  border-radius: 50%;

  background: #ffd54f;

  position: absolute;

  bottom: 2px;
}

/* EVENTS BLOCK */
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

/* TOOLTIP */
.calendar-tooltip {
  background: #0c205a;

  color: white;

  border-radius: 10px;

  padding: 10px 12px;

  max-width: 260px;

  font-size: 13px;

  box-shadow: 0 4px 20px rgba(0,0,0,0.35);
}

.tooltip-event:not(:last-child) {
  margin-bottom: 12px;

  padding-bottom: 12px;

  border-bottom: 1px solid rgba(255,255,255,0.1);
}

.tooltip-time {
  color: #b8d7ff;

  margin-top: 4px;
}

.tooltip-status {
  color: #ffd54f;

  margin-top: 2px;
}
