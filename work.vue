<div class="calendar-wrapper">
  <q-date
    v-model="selectedDate"
    flat
    bordered
    minimal
    mask="YYYY-MM-DD"
    color="primary"
    class="custom-calendar"
    :events="eventDates"
    :event-color="getEventColor"
    @update:model-value="onDateClick"
  >
    <template #day="scope">
      <div class="day-cell">

        <q-tooltip
          v-if="hasEvents(scope.timestamp.date)"
          anchor="top middle"
          self="bottom middle"
        >
          <div
            v-for="event in getEventsByDate(scope.timestamp.date)"
            :key="event.id"
            class="tooltip-event"
          >
            <div class="text-weight-bold">
              {{ event.title }}
            </div>

            <div>
              {{ event.time }}
            </div>
          </div>
        </q-tooltip>

        {{ scope.day }}
      </div>
    </template>
  </q-date>
</div>

const getEventsByDate = (date) => {
  return eventsMap.value[date] || [];
};

const hasEvents = (date) => {
  return getEventsByDate(date).length > 0;
};

return {
  selectedDate,
  selectedEvents,
  onDateClick,
  goToEvent,

  eventDates,
  getEventColor,

  getEventsByDate,
  hasEvents
};

.day-cell {
  width: 100%;
  height: 100%;

  display: flex;
  align-items: center;
  justify-content: center;

  position: relative;
}

.tooltip-event:not(:last-child) {
  margin-bottom: 10px;
  padding-bottom: 10px;
  border-bottom: 1px solid rgba(255,255,255,.15);
}
