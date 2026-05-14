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
            left: tooltip.x + 'px',
            top: tooltip.y + 'px'
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
QDATE EVENTS FORMAT
IMPORTANT: YYYY/MM/DD
=========================================================
*/

const eventDates = computed(() => {

  return Object.keys(eventsMap.value)
    .map((date) => {

      return date.replaceAll("-", "/");
    });
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

  return (
    eventsMap.value[date] || []
  );
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

  /*
  =======================================================
  AUTO OPEN IF ONLY ONE EVENT
  =======================================================
  */

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
FETCH
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
TOOLTIPS + BORDERS
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
    REMOVE PREVIOUS STATE
    =====================================================
    */

    dayEl.classList.remove(
      "event-day"
    );

    dayEl.removeAttribute(
      "title"
    );

    dayEl.onmouseover = null;
    dayEl.onmouseout = null;

    /*
    =====================================================
    GET DAY
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
    ADD BORDER
    =====================================================
    */

    dayEl.classList.add(
      "event-day"
    );

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
    TOOLTIP SHOW
    =====================================================
    */

    dayEl.onmouseover = () => {

      const rect =
        dayEl.getBoundingClientRect();

      const wrapperRect =
        calendarWrapper.value.getBoundingClientRect();

      tooltip.value = {

        visible: true,

        x:
          rect.left -
          wrapperRect.left +
          rect.width / 2,

        y:
          rect.top -
          wrapperRect.top - 12,

        events
      };
    };

    /*
    =====================================================
    TOOLTIP HIDE
    =====================================================
    */

    dayEl.onmouseout = () => {

      tooltip.value.visible = false;
    };
  });
};

/*
=========================================================
MONTH NAVIGATION
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

  background:
    linear-gradient(
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

  overflow: visible;
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
QUASAR EVENT DOT
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
EVENT DAY BORDER
=========================================================
*/

.custom-calendar
  :deep(.event-day .q-btn__content) {

  width: 34px !important;
  height: 34px !important;

  border:
    2px solid rgba(255,255,255,0.95);

  border-radius: 50%;

  transition: all 0.2s ease;
}

/*
=========================================================
HOVER
=========================================================
*/

.custom-calendar
  :deep(.event-day:hover .q-btn__content) {

  background:
    rgba(255,255,255,0.12);
}

/*
=========================================================
DO NOT BREAK ACTIVE DAY
=========================================================
*/

.custom-calendar
  :deep(.q-date__calendar-item--active .q-btn__content) {

  border: none !important;
}

/*
=========================================================
TODAY STYLE
=========================================================
*/

.custom-calendar
  :deep(.q-date__today) {

  box-shadow:
    inset 0 0 0 2px white;

  border-radius: 50%;
}

/*
=========================================================
TOOLTIP
=========================================================
*/

.calendar-tooltip {

  position: absolute;

  transform:
    translate(-50%, -100%);

  z-index: 999999;

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
