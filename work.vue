<template>
  <q-separator class="q-my-md" />

  <div class="calendar-widget q-px-md q-pb-md">
    <div class="event-calendar">

      <!-- EVENTS -->
      <div class="events-block">
        <div class="events-title">
          События
        </div>

        <div
          v-if="selectedEvents.length === 0"
          class="no-events"
        >
          На выбранную дату событий нет
        </div>

        <q-list v-else dense>
          <q-item
            v-for="event in selectedEvents"
            :key="event.id"
            clickable
            v-ripple
            class="event-item"
            @click="goToEvent(event.link)"
          >
            <q-item-section>
              <q-item-label class="text-weight-medium">
                {{ event.title }}
              </q-item-label>

              <q-item-label caption class="text-grey-3">
                {{ event.time }}
              </q-item-label>
            </q-item-section>
          </q-item>
        </q-list>
      </div>

      <!-- CALENDAR -->
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
        />
      </div>
    </div>
  </div>

  <q-separator class="q-my-md" />
</template>

<script>
import { ref, onMounted, computed } from "vue";
import axios from "axios";

export default {
  name: "MyLayout",

  setup() {

    // =========================
    // STATE
    // =========================

    const selectedDate = ref(
      new Date().toISOString().split("T")[0]
    );

    const datePicker = ref([]);

    const BACKEND_URL =
      window.location.origin +
      `/pp/Ext5/extjs_json_collection_data.html`;

    // =========================
    // EVENTS
    // =========================

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

          time:
            startDate && finishDate
              ? `${startDate.slice(11, 16)} - ${finishDate.slice(11, 16)}`
              : "",

          link: event.link,
        };
      });
    });

    // =========================
    // MAP
    // =========================

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

    // =========================
    // DATES FOR Q-DATE EVENTS
    // FORMAT MUST BE YYYY/MM/DD
    // =========================

    const eventDates = computed(() => {

      return Object.keys(eventsMap.value)
        .map((date) => {

          return date.replaceAll("-", "/");

        });

    });

    // =========================
    // SELECTED EVENTS
    // =========================

    const selectedEvents = computed(() => {

      return eventsMap.value[selectedDate.value] || [];

    });

    // =========================
    // CLICK DATE
    // =========================

    const onDateClick = (date) => {

      selectedDate.value = date;

    };

    // =========================
    // OPEN EVENT
    // =========================

    const goToEvent = (link) => {

      if (!link) return;

      window.location.href = link;

    };

    // =========================
    // EVENT COLOR
    // =========================

    const getEventColor = (date) => {

      const normalized = date.replaceAll("/", "-");

      // если дата выбрана — белый кружок
      if (normalized === selectedDate.value) {
        return "white";
      }

      // обычные события
      return "amber";

    };

    // =========================
    // FETCH
    // =========================

    const fetchDatePickerInfo = async () => {

      try {

        const response = await axios.post(
          BACKEND_URL,

          new URLSearchParams({
            collection_code: "VTBLGetEventCalendarEvents",
          }).toString()
        );

        datePicker.value = response.data.results || [];

      } catch (error) {

        console.error(
          "Ошибка загрузки календаря",
          error
        );

      }
    };

    // =========================
    // MOUNT
    // =========================

    onMounted(async () => {

      await fetchDatePickerInfo();

    });

    return {
      selectedDate,
      selectedEvents,
      eventDates,

      onDateClick,
      goToEvent,

      getEventColor,
    };
  },
};
</script>

<style scoped>

.calendar-widget {
  margin-top: 16px;
}

.event-calendar {
  background: linear-gradient(
    180deg,
    #1da1f2 0%,
    #1687d9 100%
  );

  border-radius: 18px;

  overflow: hidden;

  display: flex;
  flex-direction: column;

  box-shadow:
    0 10px 30px rgba(0, 0, 0, 0.15);
}

/* =========================
   EVENTS BLOCK
========================= */

.events-block {
  padding: 16px;

  background:
    rgba(255, 255, 255, 0.08);

  border-bottom:
    1px solid rgba(255,255,255,0.15);

  min-height: 120px;
}

.events-title {
  color: white;

  font-size: 16px;

  font-weight: 700;

  margin-bottom: 12px;
}

.no-events {
  text-align: center;

  color:
    rgba(255,255,255,0.75);

  padding: 20px 0;
}

.event-item {
  border-radius: 12px;

  margin-bottom: 6px;
}

.event-item :deep(.q-item__label) {
  color: white;
}

/* =========================
   CALENDAR
========================= */

.calendar-wrapper {
  display: flex;
  justify-content: center;

  padding:
    10px 0 15px;
}

.custom-calendar {
  background: transparent;

  color: white;

  max-width: 100%;
}

/* HEADER */

.custom-calendar :deep(.q-date__header) {
  background: transparent;
  color: white;
}

.custom-calendar :deep(.q-date__navigation) {
  color: white;
}

.custom-calendar :deep(.q-date__calendar-weekdays) {
  background:
    rgba(255,255,255,0.12);

  color: white;

  font-weight: 700;
}

/* =========================
   EVENT DOTS
========================= */

/* размер точки события */

.custom-calendar :deep(.q-date__event) {

  width: 8px !important;
  height: 8px !important;

  border-radius: 50%;

  bottom: 4px;
}

/* =========================
   TODAY
========================= */

.custom-calendar :deep(.q-date__today) {

  box-shadow:
    inset 0 0 0 2px white;

  border-radius: 50%;
}

/* =========================
   SELECTED DATE
========================= */

.custom-calendar :deep(.bg-primary) {

  background:
    rgba(255,255,255,0.18) !important;

  border:
    2px solid white;

  color: white !important;

  border-radius: 50%;
}

</style>
