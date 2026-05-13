<template>
  <q-separator class="q-my-md" />

  <div class="calendar-widget q-px-md q-pb-md">
    <div class="event-calendar">

      <!-- СОБЫТИЯ -->
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
            <q-item-section avatar>
              <q-avatar
                rounded
                size="42px"
              >
                <img
                  :src="event.image"
                  alt=""
                />
              </q-avatar>
            </q-item-section>

            <q-item-section>
              <q-item-label class="text-weight-medium">
                {{ event.title }}
              </q-item-label>

              <q-item-label
                caption
                class="text-grey-3"
              >
                {{ event.time }}
              </q-item-label>

              <q-item-label
                caption
                class="text-grey-5"
              >
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
      >
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
                'event-day': hasEvents(scope.timestamp.date),
                'selected-event-day':
                  selectedDate === scope.timestamp.date
              }"
              @click="onDateClick(scope.timestamp.date)"
            >
              <!-- ТОЧКА СОБЫТИЯ -->
              <div
                v-if="hasEvents(scope.timestamp.date)"
                class="event-dot"
              />

              <!-- TOOLTIP -->
              <q-tooltip
                v-if="getEventsByDate(scope.timestamp.date).length"
                anchor="top middle"
                self="bottom middle"
                class="calendar-tooltip"
              >
                <div
                  v-for="event in getEventsByDate(scope.timestamp.date)"
                  :key="event.id"
                  class="tooltip-event"
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
    </div>
  </div>

  <q-separator class="q-my-md" />
</template>

<script>
import {
  ref,
  computed,
  onMounted
} from "vue";

import axios from "axios";
import { useRouter } from "vue-router";

export default {
  name: "MyLayout",

  setup() {
    const router = useRouter();

    const selectedDate = ref(
      new Date().toISOString().split("T")[0]
    );

    const datePicker = ref([]);

    /*
    |--------------------------------------------------------------------------
    | CALENDAR EVENTS
    |--------------------------------------------------------------------------
    */

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

          status: event.status_name,

          image: event.image_url,
        };
      });
    });

    /*
    |--------------------------------------------------------------------------
    | EVENTS MAP
    |--------------------------------------------------------------------------
    */

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

    /*
    |--------------------------------------------------------------------------
    | HELPERS
    |--------------------------------------------------------------------------
    */

    const getEventsByDate = (date) => {
      return eventsMap.value[date] || [];
    };

    const hasEvents = (date) => {
      return getEventsByDate(date).length > 0;
    };

    /*
    |--------------------------------------------------------------------------
    | SELECTED EVENTS
    |--------------------------------------------------------------------------
    */

    const selectedEvents = computed(() => {
      return eventsMap.value[selectedDate.value] || [];
    });

    /*
    |--------------------------------------------------------------------------
    | NAVIGATION
    |--------------------------------------------------------------------------
    */

    const goToEvent = (link) => {
      router.push(link);
    };

    /*
    |--------------------------------------------------------------------------
    | DATE CLICK
    |--------------------------------------------------------------------------
    */

    const onDateClick = (date) => {
      selectedDate.value = date;

      const events = getEventsByDate(date);

      if (!events.length) return;

      // Если событие одно — сразу переход
      if (events.length === 1) {
        goToEvent(events[0].link);
      }
    };

    /*
    |--------------------------------------------------------------------------
    | API
    |--------------------------------------------------------------------------
    */

    const BACKEND_URL =
      window.location.origin +
      `/pp/Ext5/extjs_json_collection_data.html`;

    const fetchDatePickerInfo = async () => {
      try {
        const response = await axios.post(
          BACKEND_URL,
          new URLSearchParams({
            collection_code:
              "VTBLGetEventCalendarEvents",
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

    /*
    |--------------------------------------------------------------------------
    | MOUNT
    |--------------------------------------------------------------------------
    */

    onMounted(async () => {
      await fetchDatePickerInfo();
    });

    return {
      selectedDate,
      datePicker,
      calendarEvents,
      eventsMap,
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

/* =========================================================
   WRAPPER
========================================================= */

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
    0 10px 30px rgba(0,0,0,0.15);
}

/* =========================================================
   EVENTS BLOCK
========================================================= */

.events-block {
  padding: 16px;

  background:
    rgba(255,255,255,0.08);

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

  font-size: 14px;
}

/* =========================================================
   EVENT ITEM
========================================================= */

.event-item {
  border-radius: 12px;

  margin-bottom: 6px;

  transition:
    background 0.2s ease,
    transform 0.2s ease;
}

.event-item:hover {
  background:
    rgba(255,255,255,0.12);

  transform: translateY(-1px);
}

.event-item :deep(.q-item__label) {
  color: white;
}

/* =========================================================
   CALENDAR
========================================================= */

.custom-calendar {
  background: transparent;
  color: white;
}

/* HEADER */

.custom-calendar :deep(.q-date__header) {
  background: transparent;
  color: white;
}

/* NAVIGATION */

.custom-calendar :deep(.q-date__navigation) {
  color: white;
}

/* WEEK DAYS */

.custom-calendar
  :deep(.q-date__calendar-weekdays) {

  background:
    rgba(255,255,255,0.12);

  color: white;

  font-weight: 700;
}

/* DAY */

.calendar-day-btn {
  position: relative;

  width: 34px;
  height: 34px;

  color: white;

  transition:
    background 0.2s ease,
    transform 0.2s ease;
}

.calendar-day-btn:hover {
  background:
    rgba(255,255,255,0.15);

  transform: scale(1.05);
}

/* =========================================================
   EVENT DAY
========================================================= */

.event-day {
  border:
    2px solid rgba(255,255,255,0.95);

  border-radius: 50%;
}

/* =========================================================
   SELECTED DAY
========================================================= */

.selected-event-day {
  background: white !important;
  color: #1da1f2 !important;
}

/* =========================================================
   TODAY
========================================================= */

.custom-calendar
  :deep(.q-date__today) {

  box-shadow:
    inset 0 0 0 2px white;

  border-radius: 50%;
}

/* =========================================================
   EVENT DOT
========================================================= */

.event-dot {
  position: absolute;

  bottom: 3px;

  width: 6px;
  height: 6px;

  border-radius: 50%;

  background: #ffd54f;

  box-shadow:
    0 0 6px rgba(255,213,79,0.8);
}

/* =========================================================
   TOOLTIP
========================================================= */

.calendar-tooltip {
  max-width: 260px;

  font-size: 13px;

  border-radius: 10px;
}

.tooltip-event:not(:last-child) {
  margin-bottom: 10px;
  padding-bottom: 10px;

  border-bottom:
    1px solid rgba(255,255,255,0.1);
}

</style>
