<template>
  <q-separator class="q-my-md" />
  <div class="calendar-widget q-px-md q-pb-md">
    <div class="event-calendar">
      <!-- EVENTS -->
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

      <!-- CALENDAR (без точек Quasar, с ручной обводкой и тултипом) -->
      <div ref="calendarWrapper" class="calendar-wrapper">
        <q-date
          v-model="selectedDate"
          flat
          bordered
          mask="YYYY-MM-DD"
          color="primary"
          class="custom-calendar"
          @update:model-value="onDateClick"
          @navigation="onNavigation"
        />
        <!-- Кастомный тултип -->
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
            <div class="text-weight-bold">{{ event.title }}</div>
            <div class="text-grey-3">{{ event.time }}</div>
            <div class="text-grey-5">{{ event.status }}</div>
          </div>
        </div>
      </div>
    </div>
  </div>
  <q-separator class="q-my-md" />
</template>

<script>
import { ref, onMounted, nextTick, computed } from "vue";
import axios from "axios";

export default {
  name: "MyLayout",
  setup() {
    // --- Данные календаря ---
    const selectedDate = ref(new Date().toISOString().split("T")[0]);
    const datePicker = ref([]);
    const calendarWrapper = ref(null);
    const currentMonth = ref(new Date().getMonth() + 1); // 1-12
    const currentYear = ref(new Date().getFullYear());
    const tooltip = ref({
      visible: false,
      x: 0,
      y: 0,
      events: [],
    });

    // --- События из API ---
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

    const selectedEvents = computed(() => {
      return eventsMap.value[selectedDate.value] || [];
    });

    const getEventsByDate = (date) => {
      return eventsMap.value[date] || [];
    };

    // --- Клик по дате ---
    const onDateClick = (date) => {
      selectedDate.value = date;
      const events = getEventsByDate(date);
      if (events.length === 1) {
        goToEvent(events[0].link);
      }
    };

    const goToEvent = (link) => {
      if (!link) return;
      window.location.href = link;
    };

    // --- Загрузка данных с сервера ---
    const fetchDatePickerInfo = async () => {
      try {
        const response = await axios.post(
          BACKEND_URL,
          new URLSearchParams({
            collection_code: "VTBLGetEventCalendarEvents",
          }).toString()
        );
        datePicker.value = response.data.results || [];
        await nextTick();
        bindTooltips(); // после загрузки данных навешиваем обводку и тултипы
      } catch (error) {
        console.error("Ошибка загрузки календаря", error);
      }
    };

    // --- Функция, которая добавляет обводку (event-day) и вешает тултипы ---
    const bindTooltips = async () => {
      await nextTick();
      const dayButtons = calendarWrapper.value?.querySelectorAll(
        ".q-date__calendar-item .q-btn"
      );
      if (!dayButtons) return;

      dayButtons.forEach((dayEl) => {
        // Сброс предыдущих состояний
        dayEl.classList.remove("event-day");
        dayEl.onmouseover = null;
        dayEl.onmouseout = null;

        // Ищем элемент с числом (у q-date внутри .q-btn__content)
        const content = dayEl.querySelector(".q-btn__content");
        if (!content) return;
        const dayText = content.innerText.trim();
        const day = parseInt(dayText, 10);
        if (isNaN(day)) return;

        // Формируем дату в формате YYYY-MM-DD
        const monthStr = String(currentMonth.value).padStart(2, "0");
        const dayStr = String(day).padStart(2, "0");
        const date = `${currentYear.value}-${monthStr}-${dayStr}`;

        const events = getEventsByDate(date);
        if (!events.length) return;

        // Добавляем класс для обводки
        dayEl.classList.add("event-day");

        // При наведении — показываем кастомный тултип
        dayEl.onmouseover = (event) => {
          const rect = dayEl.getBoundingClientRect();
          const wrapperRect = calendarWrapper.value.getBoundingClientRect();
          tooltip.value = {
            visible: true,
            x: rect.left - wrapperRect.left + rect.width / 2,
            y: rect.top - wrapperRect.top - 12,
            events: events,
          };
        };
        dayEl.onmouseout = () => {
          tooltip.value.visible = false;
        };
      });
    };

    // --- Обработчик смены месяца/года в календаре ---
    const onNavigation = async ({ month, year }) => {
      // month в Quasar приходит от 0 до 11, year — четырёхзначный
      currentMonth.value = month + 1;
      currentYear.value = year;
      await nextTick();
      bindTooltips(); // перепривязываем обводки и тултипы для нового месяца
    };

    // --- Вспомогательные функции (из вашего кода, сохранены) ---
    // (здесь должны быть fetchMenuLinks, fetchUsersInfo, mutObserver и т.д.)
    // Для краткости они не приведены, но вы их вставите обратно

    // Пример: мутация для padding (оставляем как было)
    const mutObserver = async () => {
      let observer = new MutationObserver(function (mutations) {
        mutations.forEach(function (mutationRecord) {
          document.getElementById("wt-body").style.paddingLeft =
            mutationRecord.target.style.paddingLeft;
        });
      });
      observer.observe(document.getElementsByClassName("q-page-container")[0], {
        attributes: true,
        attributeFilter: ["style"],
      });
    };

    onMounted(async () => {
      // await fetchMenuLinks();
      // await fetchUsersInfo();
      await fetchDatePickerInfo();
      await mutObserver();
      await nextTick();

      // Дополнительные стили для поиска и списка (из вашего кода)
      const searchQfieldNative = document.querySelectorAll(".q-field__native");
      searchQfieldNative.forEach((el) => {
        el.style.paddingLeft = "10px";
      });
      const listItemElements = document.querySelectorAll('[role="listitem"]');
      listItemElements.forEach((element) => {
        element.addEventListener("mouseenter", () => {
          element.style.backgroundColor = "rgba(255, 255, 255, 0.2)";
        });
        element.addEventListener("mouseleave", () => {
          element.style.backgroundColor = "";
        });
      });
    });

    // Возвращаем всё необходимое для шаблона
    return {
      selectedDate,
      selectedEvents,
      tooltip,
      onDateClick,
      goToEvent,
      calendarWrapper,
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

.events-block {
  padding: 16px;
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
}

.event-item {
  border-radius: 12px;
  margin-bottom: 6px;
}

.event-item :deep(.q-item__label) {
  color: white;
}

/* --- Центрирование календаря --- */
.calendar-wrapper {
  position: relative;
  display: flex;
  justify-content: center;
  overflow: visible;
}

.custom-calendar {
  background: transparent;
  color: white;
  width: auto;
  max-width: 100%;
}

/* Стилизация заголовка и навигации */
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

/* --- Обводка для дней с событиями (вместо точек) --- */
.custom-calendar :deep(.event-day .q-btn__content) {
  width: 34px !important;
  height: 34px !important;
  border: 2px solid rgba(255, 255, 255, 0.95);
  border-radius: 50%;
  transition: all 0.2s ease;
}

.custom-calendar :deep(.event-day:hover .q-btn__content) {
  background: rgba(255, 255, 255, 0.12);
}

/* Не ломаем активный (выбранный) день — убираем обводку, если она мешает */
.custom-calendar :deep(.q-date__calendar-item--active .q-btn__content) {
  border: none !important;
}

/* Стиль для сегодняшней даты */
.custom-calendar :deep(.q-date__today) {
  box-shadow: inset 0 0 0 2px white;
  border-radius: 50%;
}

/* --- Кастомный тултип --- */
.calendar-tooltip {
  position: absolute;
  transform: translate(-50%, -100%);
  z-index: 999999;
  min-width: 220px;
  max-width: 260px;
  padding: 12px;
  border-radius: 12px;
  background: rgba(35, 35, 35, 0.96);
  color: white;
  backdrop-filter: blur(12px);
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.25);
  pointer-events: none;
  font-size: 13px;
}

.tooltip-event:not(:last-child) {
  margin-bottom: 10px;
  padding-bottom: 10px;
  border-bottom: 1px solid rgba(255, 255, 255, 0.1);
}
</style>
