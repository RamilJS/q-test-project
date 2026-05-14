<template>
  <q-separator class="q-my-md" />
  <div class="calendar-widget q-px-md q-pb-md">
    <div class="event-calendar">

      <!-- СОБЫТИЯ -->
      <div class="events-block">
        <div class="events-title">События</div>

        <div v-if="selectedEvents.length === 0" class="no-events">
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
              <q-item-label caption class="text-grey-5">
                {{ event.status }}
              </q-item-label>
            </q-item-section>
          </q-item>
        </q-list>
      </div>

      <!-- КАЛЕНДАРЬ -->
      <q-date
        v-model="selectedDate"
        flat
        bordered
        mask="YYYY-MM-DD"
        color="primary"
        class="custom-calendar"
        :events="eventDates"
        :event-color="eventColor"
        @update:model-value="onDateClick"
      />

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
    const selectedDate = ref(new Date().toISOString().split("T")[0]);
    const datePicker = ref([]);

    const calendarEvents = computed(() => {
      return datePicker.value.map((event) => {
        const startDate = event.start_date || "";
        const finishDate = event.finish_date || "";
        return {
          id: event.id,
          title: event.name,
          date: startDate ? startDate.split("T")[0] : "",
          time:
            startDate && finishDate
              ? `${startDate.slice(11, 16)} - ${finishDate.slice(11, 16)}`
              : "",
          link: event.link,
          status: event.status_name,
          image: event.image_url,
        };
      });
    });

    const eventsMap = computed(() => {
      const map = {};
      calendarEvents.value.forEach((event) => {
        if (!map[event.date]) map[event.date] = [];
        map[event.date].push(event);
      });
      return map;
    });

    const eventDates = computed(() => Object.keys(eventsMap.value));

    const eventColor = (date) => {
      return eventsMap.value[date] ? "#ffd54f" : "";
    };

    const getEventsByDate = (date) => eventsMap.value[date] || [];

    const hasEvents = (date) => getEventsByDate(date).length > 0;

    const onDateClick = (date) => {
      selectedDate.value = date;
      const events = getEventsByDate(date);
      if (events.length === 1) {
        goToEvent(events[0].link);
      }
    };

    const goToEvent = (link) => {
      window.location.href = link;
    };

    const selectedEvents = computed(
      () => eventsMap.value[selectedDate.value] || []
    );

    const fetchDatePickerInfo = async () => {
      try {
        const response = await axios.post(
          BACKEND_URL,
          new URLSearchParams({
            collection_code: "VTBLGetEventCalendarEvents",
          }).toString()
        );
        datePicker.value = response.data.results;
      } catch (error) {
        console.error("Ошибка загрузки данных календаря", error);
      }
    };

    onMounted(async () => {
      await fetchDatePickerInfo();
    });

    return {
      selectedDate,
      datePicker,
      calendarEvents,
      eventsMap,
      eventDates,
      eventColor,
      selectedEvents,
      getEventsByDate,
      hasEvents,
      onDateClick,
      goToEvent,
    };
  },
};
</script>

<style scoped>
.calendar-widget {
  margin-top: 16px;
}

.event-calendar {
  background: linear-gradient(180deg, #1da1f2 0%, #1687d9 100%);
  border-radius: 18px;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.15);
}

/* ── События ── */
.events-block {
  padding: 16px 16px 10px;
  background: rgba(255, 255, 255, 0.08);
  border-bottom: 1px solid rgba(255, 255, 255, 0.15);
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
  color: rgba(255, 255, 255, 0.75);
  padding: 20px 0;
  font-size: 14px;
}

.event-item {
  border-radius: 12px;
  margin-bottom: 6px;
  transition: background 0.2s ease, transform 0.2s ease;
}

.event-item:hover {
  background: rgba(255, 255, 255, 0.12);
  transform: translateY(-1px);
}

.event-item :deep(.q-item__label) {
  color: white;
}

/* ── Календарь ── */
.custom-calendar {
  background: transparent;
  color: white;
  margin: 15px auto;
}

.custom-calendar :deep(.q-date__header) {
  background: transparent;
  color: white;
}

.custom-calendar :deep(.q-date__navigation) {
  color: white;
}

.custom-calendar :deep(.q-date__calendar-weekdays) {
  background: rgba(255, 255, 255, 0.12);
  color: white;
  font-weight: 700;
}

/* Цвет цифр дней */
.custom-calendar :deep(.q-date__calendar-item .q-btn) {
  color: white;
}

/* Сегодня */
.custom-calendar :deep(.q-date__today .q-btn) {
  box-shadow: inset 0 0 0 2px white;
  border-radius: 50%;
}

/* Выбранная дата */
.custom-calendar :deep(.q-date__calendar-item--selected .q-btn) {
  background: white !important;
  color: #1da1f2 !important;
  border-radius: 50%;
}

/* ── Дни с событиями — кружок и точка ── */

/* Кружок вокруг дня с событием */
.custom-calendar :deep(.q-date__event + .q-btn),
.custom-calendar :deep(.q-btn + .q-date__event) {
  border: 2px solid rgba(255, 255, 255, 0.9);
  border-radius: 50%;
}

/* Точка под днём с событием */
.custom-calendar :deep(.q-date__event) {
  background: #ffd54f !important;
  box-shadow: 0 0 8px rgba(255, 213, 79, 0.8);
  width: 6px;
  height: 6px;
  border-radius: 50%;
}
</style>
