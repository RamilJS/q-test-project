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

      <!-- CALENDAR -->
      <div
        ref="calendarWrapper"
        class="calendar-wrapper"
      >
        <q-date
          v-model="selectedDate"
          flat
          bordered
          mask="YYYY-MM-DD"
          color="primary"
          class="custom-calendar"
          :events="eventDates"
          event-color="amber"
          @update:model-value="onDateClick"
          @navigation="onNavigation"
        />

        <!-- TOOLTIP -->
        <div
          v-if="tooltip.visible"
          class="calendar-tooltip"
          :style="{
            top: tooltip.y + 'px',
            left: tooltip.x + 'px'
          }"
        >
          <div
            v-for="event in tooltip.events"
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
        </div>
      </div>

    </div>
  </div>

  <q-separator class="q-my-md" />
</template>

<script setup>
import {
  ref,
  computed,
  onMounted,
  nextTick
} from "vue";

import axios from "axios";

/*
=========================================================
STATE
=========================================================
*/

const selectedDate = ref(
  new Date().toISOString().split("T")[0]
);

const datePicker = ref([]);

const calendarWrapper = ref(null);

const currentMonth = ref(
  new Date().getMonth() + 1
);

const currentYear = ref(
  new Date().getFullYear()
);

const tooltip = ref({
  visible: false,
  x: 0,
  y: 0,
  events: []
});

/*
=========================================================
EVENTS
=========================================================
*/

const calendarEvents = computed(() => {
  return datePicker.value.map((event) => {
    const startDate =
      event.start_date || "";

    const finishDate =
      event.finish_date || "";

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

      status: event.status_name
    };
  });
});

/*
=========================================================
EVENT MAP
=========================================================
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
=========================================================
IMPORTANT !!!
QDATE EVENTS NEED YYYY/MM/DD
NOT YYYY-MM-DD
=========================================================
*/

const eventDates = computed(() => {
  return Object.keys(eventsMap.value).map(
    (date) => date.replaceAll("-", "/")
  );
});

/*
=========================================================
SELECTED EVENTS
=========================================================
*/

const selectedEvents = computed(() => {
  return (
    eventsMap.value[
      selectedDate.value
    ] || []
  );
});

const getEventsByDate = (date) => {
  return eventsMap.value[date] || [];
};

/*
=========================================================
CLICK DATE
=========================================================
*/

const onDateClick = (date) => {
  selectedDate.value = date;

  const events =
    getEventsByDate(date);

  // если одно событие -> переход
  if (events.length === 1) {
    goToEvent(events[0].link);
  }
};

/*
=========================================================
GO TO EVENT
=========================================================
*/

const goToEvent = (link) => {
  if (!link) return;

  window.location.href = link;
};

/*
=========================================================
FETCH EVENTS
=========================================================
*/

const BACKEND_URL =
  window.location.origin +
  "/pp/Ext5/extjs_json_collection_data.html";

const fetchDatePickerInfo =
  async () => {
    try {
      const response =
        await axios.post(
          BACKEND_URL,

          new URLSearchParams({
            collection_code:
              "VTBLGetEventCalendarEvents"
          }).toString()
        );

      datePicker.value =
        response.data.results || [];

      await nextTick();

      bindTooltips();
    }
    catch (error) {
      console.error(
        "Ошибка загрузки календаря",
        error
      );
    }
  };

/*
=========================================================
TOOLTIPS + HIGHLIGHT
=========================================================
*/

const bindTooltips = async () => {
  await nextTick();

  const dayButtons =
    calendarWrapper.value?.querySelectorAll(
      ".q-date__calendar-item .q-btn"
    );

  if (!dayButtons) return;

  dayButtons.forEach((dayEl) => {

    /*
    =====================================================
    GET DAY NUMBER
    =====================================================
    */

    const block =
      dayEl.querySelector(".block");

    const dayText =
      block?.innerText?.trim();

    const day = Number(dayText);

    if (!day) return;

    /*
    =====================================================
    BUILD DATE
    =====================================================
    */

    const month = String(
      currentMonth.value
    ).padStart(2, "0");

    const date =
      `${currentYear.value}-${month}-${String(day).padStart(2, "0")}`;

    const events =
      getEventsByDate(date);

    if (!events.length) return;

    /*
    =====================================================
    NATIVE TOOLTIP
    =====================================================
    */

    dayEl.setAttribute(
      "title",

      events
        .map(
          (e) =>
            `${e.title} (${e.time})`
        )
        .join("\n")
    );

    /*
    =====================================================
    CUSTOM HIGHLIGHT
    =====================================================
    */

    dayEl.classList.add(
      "event-day"
    );

    /*
    =====================================================
    CLEAN OLD LISTENERS
    =====================================================
    */

    dayEl.onmouseenter = null;
    dayEl.onmouseleave = null;

    /*
    =====================================================
    SHOW TOOLTIP
    =====================================================
    */

    dayEl.onmouseenter = () => {
      const rect =
        dayEl.getBoundingClientRect();

      tooltip.value = {
        visible: true,

        x:
          rect.left +
          rect.width / 2,

        y:
          rect.top - 10,

        events
      };
    };

    /*
    =====================================================
    HIDE TOOLTIP
    =====================================================
    */

    dayEl.onmouseleave = () => {
      tooltip.value.visible = false;
    };
  });
};

/*
=========================================================
MONTH CHANGE
=========================================================
*/

const onNavigation = async ({
  month,
  year
}) => {
  currentMonth.value = month;
  currentYear.value = year;

  await nextTick();

  bindTooltips();
};

/*
=========================================================
MOUNT
=========================================================
*/

onMounted(async () => {
  await fetchDatePickerInfo();
});
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
    0 10px 30px rgba(0,0,0,0.15);
}

/*
=========================================================
EVENTS
=========================================================
*/

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
}

.event-item {
  border-radius: 12px;

  margin-bottom: 6px;
}

.event-item
  :deep(.q-item__label) {
  color: white;
}

/*
=========================================================
CALENDAR
=========================================================
*/

.calendar-wrapper {
  position: relative;
}

.custom-calendar {
  background: transparent;

  color: white;

  margin: 15px auto;
}

.custom-calendar
  :deep(.q-date__header) {
  background: transparent;

  color: white;
}

.custom-calendar
  :deep(.q-date__navigation) {
  color: white;
}

.custom-calendar
  :deep(.q-date__calendar-weekdays) {
  background:
    rgba(255,255,255,0.12);

  color: white;

  font-weight: 700;
}

/*
=========================================================
EVENT DOT FROM QUASAR
=========================================================
*/

.custom-calendar
  :deep(.q-date__event) {
  width: 8px !important;

  height: 8px !important;

  border-radius: 50%;
}

/*
=========================================================
CUSTOM EVENT BORDER
=========================================================
*/

.custom-calendar
  :deep(.event-day) {
  position: relative !important;

  border:
    2px solid white !important;

  border-radius: 50% !important;
}

.custom-calendar
  :deep(.event-day::after) {
  content: "";

  position: absolute;

  bottom: 2px;
  left: 50%;

  transform:
    translateX(-50%);

  width: 6px;
  height: 6px;

  border-radius: 50%;

  background: #ffd54f;

  box-shadow:
    0 0 8px rgba(255,213,79,0.8);
}

/*
=========================================================
TOOLTIP
=========================================================
*/

.calendar-tooltip {
  position: fixed;

  transform:
    translate(-50%, -100%);

  z-index: 99999;

  min-width: 220px;
  max-width: 260px;

  padding: 12px;

  border-radius: 12px;

  background:
    rgba(35,35,35,0.96);

  color: white;

  backdrop-filter: blur(12px);

  box-shadow:
    0 8px 24px rgba(0,0,0,0.25);

  pointer-events: none;
}

.tooltip-event:not(:last-child) {
  margin-bottom: 10px;

  padding-bottom: 10px;

  border-bottom:
    1px solid rgba(255,255,255,0.1);
}
</style>
