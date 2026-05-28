<script>
import { ref, onMounted, nextTick, computed } from "vue";
import axios from "axios";

export default {
  name: "MyLayout",

  setup() {

    /*
    ===========================================================
    STATE
    ===========================================================
    */

    const selectedDate = ref(
      new Date().toISOString().split("T")[0]
    );

    const datePicker = ref([]);

    const calendarWrapper = ref(null);

    const tooltip = ref({
      visible: false,
      x: 0,
      y: 0,
      events: [],
    });

    /*
    ===========================================================
    EVENTS
    ===========================================================
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

          status: event.status_name,
        };

      });

    });

    /*
    ===========================================================
    EVENTS MAP
    ===========================================================
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
    ===========================================================
    SELECTED EVENTS
    ===========================================================
    */

    const selectedEvents = computed(() => {

      return (
        eventsMap.value[selectedDate.value] || []
      );

    });

    /*
    ===========================================================
    GET EVENTS BY DATE
    ===========================================================
    */

    const getEventsByDate = (date) => {

      return eventsMap.value[date] || [];

    };

    /*
    ===========================================================
    CLICK DATE
    ===========================================================
    */

    const onDateClick = async (date) => {

      selectedDate.value = date;

      await nextTick();

      bindTooltips();

    };

    /*
    ===========================================================
    GO TO EVENT
    ===========================================================
    */

    const goToEvent = (link) => {

      if (!link) return;

      window.location.href = link;

    };

    /*
    ===========================================================
    API
    ===========================================================
    */

    const BACKEND_URL =
      window.location.origin +
      `/pp/Ext5/extjs_json_collection_data.html`;

    /*
    ===========================================================
    FETCH EVENTS
    ===========================================================
    */

    const fetchDatePickerInfo = async () => {

      try {

        const response = await axios.post(
          BACKEND_URL,

          new URLSearchParams({
            collection_code:
              "VTBLGetEventCalendarEvents",
          }).toString()
        );

        datePicker.value =
          response.data.results || [];

        await nextTick();

        bindTooltips();

      } catch (error) {

        console.error(
          "Ошибка загрузки календаря",
          error
        );

      }

    };

    /*
    ===========================================================
    TOOLTIPS + CIRCLES
    ===========================================================
    */

    const bindTooltips = async () => {

      tooltip.value.visible = false;

      await nextTick();

      const calendarItems =
        calendarWrapper.value?.querySelectorAll(
          ".q-date__calendar-item"
        );

      if (!calendarItems?.length) return;

      calendarItems.forEach((item) => {

        const btn =
          item.querySelector(".q-btn");

        if (!btn) return;

        /*
        =====================================
        CLEAN
        =====================================
        */

        btn.classList.remove("event-day");

        btn.onmouseover = null;
        btn.onmouseout = null;

        /*
        =====================================
        DATE
        =====================================
        */

        const timestamp =
          item.getAttribute("data-day");

        if (!timestamp) return;

        const date =
          timestamp.replaceAll("/", "-");

        /*
        =====================================
        EVENTS
        =====================================
        */

        const events =
          getEventsByDate(date);

        if (!events.length) return;

        /*
        =====================================
        ADD CIRCLE
        =====================================
        */

        btn.classList.add("event-day");

        /*
        =====================================
        TOOLTIP
        =====================================
        */

        btn.onmouseover = () => {

          const rect =
            btn.getBoundingClientRect();

          const wrapperRect =
            calendarWrapper.value.getBoundingClientRect();

          const tooltipWidth = 260;

          let x =
            rect.left -
            wrapperRect.left +
            rect.width / 2;

          const minX =
            tooltipWidth / 2 + 12;

          const maxX =
            wrapperRect.width -
            tooltipWidth / 2 -
            12;

          if (x < minX) {
            x = minX;
          }

          if (x > maxX) {
            x = maxX;
          }

          tooltip.value = {
            visible: true,

            x,

            y:
              rect.top -
              wrapperRect.top -
              12,

            events,
          };

        };

        btn.onmouseout = () => {

          tooltip.value.visible = false;

        };

      });

    };

    /*
    ===========================================================
    MONTH NAVIGATION
    ===========================================================
    */

    const onNavigation = async () => {

      await nextTick();

      setTimeout(() => {

        bindTooltips();

      }, 0);

    };

    /*
    ===========================================================
    MOUNT
    ===========================================================
    */

    onMounted(async () => {

      await fetchDatePickerInfo();

    });

    /*
    ===========================================================
    RETURN
    ===========================================================
    */

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
</script>

<style scoped>

.calendar-wrapper {

  position: relative;

  display: flex;

  justify-content: center;

  overflow: visible;

}

.custom-calendar {

  background: transparent;

  color: white;

  max-width: 100%;

  margin-top: 10px;

  margin-bottom: 15px;

}

/*
===========================================================
HEADER
===========================================================
*/

.custom-calendar :deep(.q-date__header) {

  background: transparent;

  color: white;

}

.custom-calendar :deep(.q-date__navigation) {

  color: white;

}

.custom-calendar :deep(.q-date__calendar-weekdays) {

  background: rgba(255,255,255,.12);

  color: white;

  font-weight: 700;

}

/*
===========================================================
EVENT DAY
===========================================================
*/

.custom-calendar :deep(.event-day) {

  border: 2px solid rgba(255,255,255,.95) !important;

  border-radius: 50% !important;

  box-sizing: border-box !important;

  transition: all .2s ease;

}

.custom-calendar :deep(.event-day:hover) {

  background: rgba(255,255,255,.12);

}

.custom-calendar :deep(.event-day .q-btn__content) {

  width: 100%;

  height: 100%;

  display: flex;

  align-items: center;

  justify-content: center;

}

/*
===========================================================
TODAY
===========================================================
*/

.custom-calendar :deep(.q-date__today) {

  box-shadow:
    inset 0 0 0 2px white;

  border-radius: 50%;

}

/*
===========================================================
TOOLTIP
===========================================================
*/

.calendar-tooltip {

  position: absolute;

  transform: translate(-50%, -100%);

  z-index: 999999;

  min-width: 220px;

  max-width: 260px;

  padding: 12px;

  border-radius: 12px;

  background: rgba(35,35,35,.96);

  color: white;

  backdrop-filter: blur(12px);

  box-shadow:
    0 8px 24px rgba(0,0,0,.25);

  pointer-events: none;

  font-size: 13px;

}

.tooltip-event:not(:last-child) {

  margin-bottom: 10px;

  padding-bottom: 10px;

  border-bottom:
    1px solid rgba(255,255,255,.1);

}

</style>
