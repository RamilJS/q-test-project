import { ref, onMounted, onUnmounted, nextTick, computed } from "vue";
import axios from "axios";

export default {
  name: "MyLayout",
  setup() {
    const selectedDate = ref(new Date().toISOString().split("T")[0]);
    const datePicker = ref([]);
    const calendarWrapper = ref(null);
    const currentMonth = ref(new Date().getMonth() + 1);
    const currentYear = ref(new Date().getFullYear());
    const tooltip = ref({ visible: false, x: 0, y: 0, events: [] });

    // Храним ссылки на обработчики, чтобы удалять их корректно
    const boundHandlers = new WeakMap();
    let domObserver = null;
    let isBinding = false; // флаг — защита от рекурсии

    const calendarEvents = computed(() =>
      datePicker.value.map((event) => ({
        id: event.id,
        title: event.name,
        date: event.start_date ? event.start_date.split("T")[0] : "",
        time:
          event.start_date && event.finish_date
            ? `${event.start_date.slice(11, 16)} - ${event.finish_date.slice(11, 16)}`
            : "",
        link: event.link,
        status: event.status_name,
      }))
    );

    const eventsMap = computed(() => {
      const map = {};
      calendarEvents.value.forEach((event) => {
        if (!map[event.date]) map[event.date] = [];
        map[event.date].push(event);
      });
      return map;
    });

    const selectedEvents = computed(
      () => eventsMap.value[selectedDate.value] || []
    );

    const getEventsByDate = (date) => eventsMap.value[date] || [];

    const goToEvent = (link) => {
      if (link) window.location.href = link;
    };

    const onDateClick = (date) => {
      selectedDate.value = date;
      // НЕ вызываем bindTooltips здесь — MutationObserver сделает это сам
    };

    // ─── Ключевая функция ───────────────────────────────────────────────────
    const bindTooltips = () => {
      // Защита от рекурсии: MutationObserver видит наши же изменения DOM
      if (isBinding) return;
      isBinding = true;

      try {
        tooltip.value.visible = false;

        const dayButtons = calendarWrapper.value?.querySelectorAll(
          ".q-date__calendar-item .q-btn"
        );
        if (!dayButtons) return;

        dayButtons.forEach((dayEl) => {
          // Снимаем старые обработчики через WeakMap
          const old = boundHandlers.get(dayEl);
          if (old) {
            dayEl.removeEventListener("mouseover", old.over);
            dayEl.removeEventListener("mouseout", old.out);
          }

          dayEl.classList.remove("event-day");

          const content = dayEl.querySelector(".q-btn__content");
          if (!content) return;

          const day = parseInt(content.innerText.trim(), 10);
          if (isNaN(day)) return;

          const date = `${currentYear.value}-${String(currentMonth.value).padStart(2, "0")}-${String(day).padStart(2, "0")}`;
          const events = getEventsByDate(date);
          if (!events.length) return;

          dayEl.classList.add("event-day");

          // Создаём новые обработчики и сохраняем в WeakMap
          const overHandler = () => {
            const rect = dayEl.getBoundingClientRect();
            const wrapperRect = calendarWrapper.value.getBoundingClientRect();
            const tooltipWidth = 260;
            let x = rect.left - wrapperRect.left + rect.width / 2;
            x = Math.max(tooltipWidth / 2 + 12, Math.min(x, wrapperRect.width - tooltipWidth / 2 - 12));
            tooltip.value = {
              visible: true,
              x,
              y: rect.top - wrapperRect.top - 12,
              events,
            };
          };
          const outHandler = () => {
            tooltip.value.visible = false;
          };

          dayEl.addEventListener("mouseover", overHandler);
          dayEl.addEventListener("mouseout", outHandler);
          boundHandlers.set(dayEl, { over: overHandler, out: outHandler });
        });
      } finally {
        isBinding = false;
      }
    };
    // ───────────────────────────────────────────────────────────────────────

    // Запускаем MutationObserver на враппере: 
    // любое изменение DOM внутри календаря (клик, смена месяца) → переприменяем обводку
    const startDomObserver = () => {
      if (!calendarWrapper.value) return;
      domObserver = new MutationObserver(() => {
        // nextTick гарантирует, что Vue тоже закончил свой патч
        nextTick(bindTooltips);
      });
      domObserver.observe(calendarWrapper.value, {
        childList: true,
        subtree: true,
        attributes: true,
        attributeFilter: ["class"], // следим за изменением классов (active и т.д.)
      });
    };

    // onNavigation: обновляем currentMonth/currentYear
    // Quasar передаёт month 0-based → прибавляем 1
    const onNavigation = ({ month, year }) => {
      currentMonth.value = month + 1;
      currentYear.value = year;
      // bindTooltips вызовет MutationObserver автоматически после ре-рендера
    };

    const BACKEND_URL =
      window.location.origin + `/pp/Ext5/extjs_json_collection_data.html`;

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
        bindTooltips();
      } catch (error) {
        console.error("Ошибка загрузки календаря", error);
      }
    };

    const mutObserver = () => {
      const pageContainer = document.getElementsByClassName("q-page-container")[0];
      if (!pageContainer) return;
      const observer = new MutationObserver((mutations) => {
        mutations.forEach((m) => {
          document.getElementById("wt-body").style.paddingLeft =
            m.target.style.paddingLeft;
        });
      });
      observer.observe(pageContainer, {
        attributes: true,
        attributeFilter: ["style"],
      });
    };

    onMounted(async () => {
      await fetchDatePickerInfo();
      mutObserver();
      startDomObserver(); // ← запускаем наблюдатель за DOM календаря

      await nextTick();
      const listItemElements = document.querySelectorAll('[role="listitem"]');
      document.querySelectorAll(".q-field__native").forEach((el) => {
        el.style.paddingLeft = "10px";
      });
      listItemElements.forEach((el) => {
        el.addEventListener("mouseenter", () => {
          el.style.backgroundColor = "rgba(255, 255, 255, 0.2)";
        });
        el.addEventListener("mouseleave", () => {
          el.style.backgroundColor = "";
        });
      });
    });

    onUnmounted(() => {
      // Чистим observer при уничтожении компонента
      domObserver?.disconnect();
    });

    return {
      selectedDate,
      selectedEvents,
      tooltip,
      onDateClick,
      goToEvent,
      calendarWrapper,
      onNavigation,
    };
  },
};
