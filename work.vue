<template #day="scope">
  <div class="full-width full-height flex flex-center calendar-day-wrapper">
    <q-btn
      flat
      round
      dense
      size="sm"
      :label="scope.day"
      class="calendar-day-btn"
      :class="{ 'event-day': hasEvents(scope.date) }"
      @click="onDateClick(scope.date)"
    />
    <!-- Tooltip вынесен ИЗ кнопки, на уровень враппера -->
    <q-tooltip
      v-if="getEventsByDate(scope.date).length"
      anchor="top middle"
      self="bottom middle"
      class="calendar-tooltip"
      :delay="200"
    >
      <div
        v-for="event in getEventsByDate(scope.date)"
        :key="event.id"
        class="q-mb-sm"
      >
        <div class="text-weight-bold">{{ event.title }}</div>
        <div class="text-grey-3">{{ event.time }}</div>
        <div class="text-grey-5">{{ event.status }}</div>
      </div>
    </q-tooltip>
  </div>
</template>

/* Враппер должен занимать всё пространство ячейки */
.calendar-day-wrapper {
  position: relative;
  width: 100%;
  height: 100%;
}

.calendar-day-btn {
  color: white;
  width: 32px;
  height: 32px;
}

/* ВАЖНО: используем outline вместо border,
   чтобы не ломать layout кнопки,
   и !important чтобы перебить стили Quasar */
.event-day {
  outline: 2px solid rgba(255, 255, 255, 0.85) !important;
  outline-offset: -1px;
  border-radius: 50% !important;
  background: rgba(255, 255, 255, 0.1) !important;
}

.event-day:hover {
  background: rgba(255, 255, 255, 0.25) !important;
}

/* Если outline всё равно не видно — альтернатива через :deep */
:deep(.event-day .q-btn__wrapper) {
  border-radius: 50%;
  box-shadow: 0 0 0 2px rgba(255, 255, 255, 0.85);
}

.calendar-tooltip {
  max-width: 260px;
  font-size: 13px;
  background: #0c205a;
  color: white;
  border-radius: 8px;
  padding: 8px 12px;
}
