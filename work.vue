
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
/>
