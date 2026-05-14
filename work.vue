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
      <div class="calendar-wrapper">
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
        />

        <!-- TOOLTIP -->
        <!-- Без slot #day -->
        <div
          v-if="tooltip.visible"
          class="calendar-tooltip-fixed"
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

<script>
import {
  ref,
  computed,
  onMounted,
  onBeforeUnmount,
  nextTick
} from "vue";

import axios from "axios";

export default {
  name: "MyLayout",

  setup() {
    const selectedDate = ref(
      new Date().toISOString().split("T")[0]
    );

    const datePicker = ref([]);

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
          image: event.image_url
        };
      });
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

    /*
    =========================================================
    DATES WITH EVENTS
    =========================================================
    */

    const eventDates = computed(() => {
      return Object.keys(eventsMap.value);
    });

    const getEventsByDate = (date) => {
      return eventsMap.value[date] || [];
    };

    const hasEvents = (date) => {
      return getEventsByDate(date).length > 0;
    };

    /*
    =========================================================
    SELECTED EVENTS
    =========================================================
    */

    const selectedEvents = computed(() => {
      return eventsMap.value[selectedDate.value] || [];
    });

    /*
    =========================================================
    CLICK DATE
    =========================================================
    */

    const onDateClick = (date) => {
      selectedDate.value = date;

      const events = getEventsByDate(date);

      if (!events.length) {
        return;
      }

      // если событие одно -> переход
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

    const fetchDatePickerInfo = async () => {
      try {
        const url = BACKEND_URL;

        const params = {
          collection_code: "VTBLGetEventCalendarEvents"
        };

        const response = await axios.post(
          url,
          new URLSearchParams(params).toString()
        );

        datePicker.value = response.data.results || [];

        await nextTick();

        bindCalendarTooltips();
      }
      catch (error) {
        console.error(
          "Ошибка загрузки данных календаря",
          error
        );
      }
    };

    /*
    =========================================================
    TOOLTIPS WITHOUT SLOT
    =========================================================
    */

    const hideTooltip = () => {
      tooltip.value.visible = false;
    };

    const showTooltip = (event, date) => {
      const events = getEventsByDate(date);

      if (!events.length) {
        return;
      }

      const rect = event.target.getBoundingClientRect();

      tooltip.value = {
        visible: true,
        x: rect.left + rect.width / 2,
        y: rect.top - 10,
        events
      };
    };

    const bindCalendarTooltips = () => {
      nextTick(() => {
        const dayButtons = document.querySelectorAll(
          ".custom-calendar .q-date__calendar-item div"
        );

        dayButtons.forEach((el) => {
          const text = el.textContent?.trim();

          if (!text || isNaN(Number(text))) {
            return;
          }

          const currentMonth =
            document.querySelector(
              ".custom-calendar .q-date__header-title-label"
            )?.textContent || "";

          const currentYear = new Date().getFullYear();

          const date = buildDateFromCalendar(
            currentMonth,
            currentYear,
            text
          );

          if (!date || !hasEvents(date)) {
            return;
          }

          // визуально выделяем
          el.classList.add("calendar-event-day");

          // стандартный tooltip
          const events = getEventsByDate(date);

          el.setAttribute(
            "title",
            events
              .map((e) => `${e.title} (${e.time})`)
              .join("\n")
          );

          // custom hover tooltip
          el.addEventListener("mouseenter", (e) => {
            showTooltip(e, date);
          });

          el.addEventListener("mouseleave", () => {
            hideTooltip();
          });
        });
      });
    };

    /*
    =========================================================
    BUILD DATE
    =========================================================
    */

    const buildDateFromCalendar = (
      monthLabel,
      year,
      day
    ) => {
      const months = {
        January: "01",
        February: "02",
        March: "03",
        April: "04",
        May: "05",
        June: "06",
        July: "07",
        August: "08",
        September: "09",
        October: "10",
        November: "11",
        December: "12",

        Январь: "01",
        Февраль: "02",
        Март: "03",
        Апрель: "04",
        Май: "05",
        Июнь: "06",
        Июль: "07",
        Август: "08",
        Сентябрь: "09",
        Октябрь: "10",
        Ноябрь: "11",
        Декабрь: "12"
      };

      const month = Object.keys(months).find((m) =>
        monthLabel.includes(m)
      );

      if (!month) {
        return null;
      }

      return `${year}-${months[month]}-${String(day).padStart(2, "0")}`;
    };

    /*
    =========================================================
    MOUNT
    =========================================================
    */

    onMounted(async () => {
      await fetchDatePickerInfo();

      // rebinding after month switch
      const observer = new MutationObserver(() => {
        bindCalendarTooltips();
      });

      const calendar = document.querySelector(
        ".custom-calendar"
      );

      if (calendar) {
        observer.observe(calendar, {
          childList: true,
          subtree: true
        });
      }

      onBeforeUnmount(() => {
        observer.disconnect();
      });
    });

    return {
      selectedDate,
      selectedEvents,
      tooltip,
      eventDates,
      onDateClick,
      goToEvent
    };
  }
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

  box-shadow: 0 10px 30px rgba(0,0,0,0.15);
}

.events-block {
  padding: 16px;

  background: rgba(255,255,255,0.08);

  border-bottom: 1px solid rgba(255,255,255,0.15);

  min-height: 120px;
  padding-bottom: 10px;
}

.events-title {
  color: white;
  font-size: 16px;
  font-weight: 700;
  margin-bottom: 12px;
}

.no-events {
  text-align: center;
  color: rgba(255,255,255,0.75);
  padding: 20px 0;
  font-size: 14px;
}

.event-item {
  border-radius: 12px;
  margin-bottom: 6px;

  transition:
    background 0.2s ease,
    transform 0.2s ease;
}

.event-item:hover {
  background: rgba(255,255,255,0.12);
  transform: translateY(-1px);
}

.event-item :deep(.q-item__label) {
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

.custom-calendar :deep(.q-date__header) {
  background: transparent;
  color: white;
}

.custom-calendar :deep(.q-date__navigation) {
  color: white;
}

.custom-calendar :deep(.q-date__calendar-weekdays) {
  background: rgba(255,255,255,0.12);
  color: white;
  font-weight: 700;
}

/*
=========================================================
EVENT DAY
=========================================================
*/

.custom-calendar :deep(.calendar-event-day) {
  position: relative;

  border: 2px solid rgba(255,255,255,0.95);
  border-radius: 50%;

  transition: all 0.2s ease;
}

.custom-calendar :deep(.calendar-event-day:hover) {
  transform: scale(1.08);
  background: rgba(255,255,255,0.12);
}

.custom-calendar :deep(.calendar-event-day::after) {
  content: "";

  position: absolute;

  width: 6px;
  height: 6px;

  border-radius: 50%;

  background: #ffd54f;

  bottom: 2px;
  left: 50%;

  transform: translateX(-50%);

  box-shadow: 0 0 8px rgba(255,213,79,0.8);
}

.custom-calendar :deep(.q-date__today) {
  box-shadow: inset 0 0 0 2px white;
  border-radius: 50%;
}

/*
=========================================================
TOOLTIP
=========================================================
*/

.calendar-tooltip-fixed {
  position: fixed;

  transform: translate(-50%, -100%);

  z-index: 99999;

  min-width: 220px;
  max-width: 260px;

  padding: 12px;

  border-radius: 12px;

  background: rgba(35,35,35,0.96);

  color: white;

  backdrop-filter: blur(12px);

  box-shadow:
    0 8px 24px rgba(0,0,0,0.25);

  pointer-events: none;
}

.tooltip-event:not(:last-child) {
  margin-bottom: 10px;
  padding-bottom: 10px;

  border-bottom: 1px solid rgba(255,255,255,0.1);
}
</style>
