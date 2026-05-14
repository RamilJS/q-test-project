<!-- КАЛЕНДАРЬ -->
<q-date
  v-model="selectedDate"
  flat
  bordered
  mask="YYYY-MM-DD"
  color="primary"
  class="custom-calendar"
  @update:model-value="onDateClick"   <!-- ← вот сюда -->
>
  <template #day="scope">
    <div class="calendar-day-wrapper">
      <div
        class="calendar-day"
        :class="{
          'event-day': hasEvents(scope.date),
          'selected-event-day': selectedDate === scope.date
        }"
      >
        {{ scope.day }}
        <div v-if="hasEvents(scope.date)" class="event-dot" />
      </div>

      <q-tooltip ...>
        ...
      </q-tooltip>
    </div>
  </template>
</q-date>

const onDateClick = (date) => {
  console.log('clicked date:', date); // теперь должно выводиться
  const events = getEventsByDate(date);
  if (events.length === 1) {
    goToEvent(events[0].link);
  }
};
