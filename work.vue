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

      <!-- КАЛЕНДАРЬ (без кастомного слота #day) -->
      <q-date
        v-model="selectedDate"
        flat
        bordered
        mask="YYYY-MM-DD"
        color="primary"
        class="custom-calendar"
        :options="dateOptions"
        @update:model-value="onDateSelected"
      />
    </div>
  </div>
  <q-separator class="q-my-md" />
</template>

<script>
import { ref, onMounted, computed, watch, nextTick } from "vue";
import axios from "axios";

export default {
  name: "MyLayout",
  setup() {
    const selectedDate = ref(new Date().toISOString().split("T")[0]);
    const datePicker = ref([]);

    // Преобразование событий из API
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

    // Группировка событий по датам
    const eventsMap = computed(() => {
      const map = {};
      calendarEvents.value.forEach((event) => {
        if (!map[event.date]) map[event.date] = [];
        map[event.date].push(event);
      });
      return map;
    });

    // Массив дат, на которые есть события
    const eventDates = computed(() => Object.keys(eventsMap.value));

    // Получить события по дате
    const getEventsByDate = (date) => eventsMap.value[date] || [];

    // Проверка, есть ли события на дату
    const hasEvents = (date) => getEventsByDate(date).length > 0;

    // options для q-date: добавляем класс event-day на даты с событиями
    const dateOptions = (date) => {
      // date приходит в формате 'YYYY/MM/DD'
      const formatted = date.split("/").join("-");
      return hasEvents(formatted) ? "event-day" : "";
    };

    // Выбранные события для текущей даты
    const selectedEvents = computed(() => eventsMap.value[selectedDate.value] || []);

    // Переход по ссылке события
    const goToEvent = (link) => {
      window.location.href = link;
    };

    // Обработчик выбора даты (клик по дню)
    const onDateSelected = (date) => {
      const events = getEventsByDate(date);
      if (events.length === 1) {
        goToEvent(events[0].link);
      }
    };

    // Загрузка данных с сервера
    const fetchDatePickerInfo = async () => {
      try {
        const url = BACKEND_URL; // должен быть определён глобально или импортирован
        const params = {
          collection_code: "VTBLGetEventCalendarEvents",
        };
        const response = await axios.post(url, new URLSearchParams(params).toString());
        datePicker.value = response.data.results;
      } catch (error) {
        console.error("Ошибка загрузки данных календаря", error);
      }
    };

    // ---- Добавляем нативные тултипы (title) для дней с событиями ----
    // Функция обновления title у ячеек календаря
    const updateNativeTooltips = () => {
      // Ищем все ячейки дней внутри .custom-calendar
      const dayCells = document.querySelectorAll(".custom-calendar .q-date__calendar-item");
      dayCells.forEach((cell) => {
        // Получаем дату из data-атрибута, который q-date устанавливает на ячейку
        const dateAttr = cell.getAttribute("data-date");
        if (dateAttr) {
          // формат data-date: YYYY/MM/DD
          const dateStr = dateAttr.split("/").join("-");
          const events = getEventsByDate(dateStr);
          if (events.length) {
            // Формируем текст для title
            const tooltipText = events
              .map((e) => `${e.title} (${e.time})`)
              .join("\n");
            cell.setAttribute("title", tooltipText);
          } else {
            cell.removeAttribute("title");
          }
        }
      });
    };

    // Следим за изменением списка событий или переключением месяца и обновляем тултипы
    watch(
      [calendarEvents, () => selectedDate.value],
      async () => {
        await nextTick();
        updateNativeTooltips();
      },
      { deep: true, immediate: true }
    );

    // Также обновляем при любом перерендере календаря (смена месяца)
    const observer = new MutationObserver(() => {
      updateNativeTooltips();
    });

    onMounted(async () => {
      await fetchDatePickerInfo();
      await nextTick();
      updateNativeTooltips();
      // Наблюдаем за изменениями DOM внутри календаря (смена месяца, перерисовка)
      const calendarEl = document.querySelector(".custom-calendar");
      if (calendarEl) {
        observer.observe(calendarEl, { childList: true, subtree: true });
      }
    });

    return {
      selectedDate,
      datePicker,
      calendarEvents,
      eventsMap,
      eventDates,
      selectedEvents,
      getEventsByDate,
      hasEvents,
      onDateSelected,
      goToEvent,
      dateOptions,
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

/* ========== КАЛЕНДАРЬ ========== */
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

/* Стили для дня с событиями */
.custom-calendar :deep(.q-date__calendar-item .event-day) {
  position: relative;
  border: 2px solid rgba(255, 255, 255, 0.95);
  border-radius: 50%;
  background: transparent !important;
  color: white !important;
}

/* Точка-маркер под числом */
.custom-calendar :deep(.event-day::after) {
  content: "";
  position: absolute;
  bottom: 2px;
  left: 50%;
  transform: translateX(-50%);
  width: 6px;
  height: 6px;
  border-radius: 50%;
  background: #ffd54f;
  box-shadow: 0 0 8px rgba(255, 213, 79, 0.8);
}

/* Выбранная дата */
.custom-calendar :deep(.q-date__calendar-item .q-btn--active) {
  background: white !important;
  color: #1da1f2 !important;
  font-weight: 700;
}

/* Сегодняшняя дата */
.custom-calendar :deep(.q-date__today) {
  box-shadow: inset 0 0 0 2px white;
  border-radius: 50%;
}

/* Ховер на дне */
.custom-calendar :deep(.q-date__calendar-item .q-btn:hover) {
  background: rgba(255, 255, 255, 0.15);
  transform: scale(1.05);
}
</style>
