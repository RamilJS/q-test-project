const applyCalendarStyles = () => {

  nextTick(() => {

    // все дни календаря
    document
      .querySelectorAll(".calendar-day-btn")
      .forEach((el) => {

        const hasEvent =
          el.getAttribute("data-event") === "true";

        if (hasEvent) {

          el.style.border =
            "2px solid rgba(255,255,255,0.9)";

          el.style.borderRadius = "50%";

          el.style.background =
            "rgba(255,255,255,0.12)";

        }

      });

  });

};

<q-btn
  flat
  round
  dense
  size="sm"
  :label="scope.day"
  class="calendar-day-btn"
  :data-event="hasEvents(scope.date)"
  @click="onDateClick(scope.date)"
>

  <q-tooltip
    v-if="hasEvents(scope.date)"
    class="bg-dark text-white"
  >

    <div
      v-for="event in getEventsByDate(scope.date)"
      :key="event.id"
      class="q-mb-sm"
    >
      <div class="text-weight-bold">
        {{ event.title }}
      </div>

      <div>
        {{ event.time }}
      </div>

      <div class="text-grey-4">
        {{ event.status }}
      </div>
    </div>

  </q-tooltip>

</q-btn>

onMounted(async () => {

  await fetchDatePickerInfo();

  applyCalendarStyles();

});
watch(selectedDate, () => {

  applyCalendarStyles();

});
