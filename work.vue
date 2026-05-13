<template #day="scope">

  <div
    class="calendar-day-wrapper"
    @click="onDateClick(scope.date)"
  >

    <div
      class="calendar-day"
      :class="{
        'event-day': hasEvents(scope.date),
        'selected-event-day':
          selectedDate === scope.date
      }"
    >
      {{ scope.day }}

      <!-- DOT -->
      <div
        v-if="hasEvents(scope.date)"
        class="event-dot"
      />
    </div>

    <!-- TOOLTIP -->
    <q-tooltip
      v-if="hasEvents(scope.date)"
      anchor="top middle"
      self="bottom middle"
      :offset="[0, 8]"
      class="calendar-tooltip"
    >
      <div
        v-for="event in getEventsByDate(scope.date)"
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

  </div>

</template>


/* =========================================================
   DAY WRAPPER
========================================================= */

.calendar-day-wrapper {
  width: 100%;
  height: 100%;

  display: flex;
  align-items: center;
  justify-content: center;

  cursor: pointer;

  position: relative;
}

/* =========================================================
   DAY
========================================================= */

.calendar-day {
  width: 34px;
  height: 34px;

  border-radius: 50%;

  display: flex;
  align-items: center;
  justify-content: center;

  position: relative;

  transition:
    all 0.2s ease;

  color: white;

  font-size: 14px;
}

/* =========================================================
   HOVER
========================================================= */

.calendar-day:hover {
  background:
    rgba(255,255,255,0.15);

  transform: scale(1.08);
}

/* =========================================================
   EVENT DAY
========================================================= */

.event-day::after {
  content: "";

  position: absolute;

  inset: 0;

  border-radius: 50%;

  border: 2px solid white;

  pointer-events: none;
}

/* =========================================================
   SELECTED
========================================================= */

.selected-event-day {
  background: white;
  color: #1da1f2;
  font-weight: 700;
}

/* =========================================================
   DOT
========================================================= */

.event-dot {
  position: absolute;

  bottom: 2px;

  width: 6px;
  height: 6px;

  border-radius: 50%;

  background: #ffd54f;

  box-shadow:
    0 0 8px rgba(255,213,79,0.8);
}

/* =========================================================
   TOOLTIP
========================================================= */

.calendar-tooltip {
  z-index: 99999;

  max-width: 260px;

  border-radius: 10px;

  font-size: 13px;

  backdrop-filter: blur(12px);
}

.tooltip-event:not(:last-child) {
  margin-bottom: 10px;
  padding-bottom: 10px;

  border-bottom:
    1px solid rgba(255,255,255,0.1);
}

на край
:deep(.q-date__calendar-item) {
  overflow: visible !important;
}

:deep(.q-date__calendar-days-container) {
  overflow: visible !important;
}
