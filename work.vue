<template #day="scope">
  <div @click="console.log('scope.date:', scope.date)">
    {{ scope.day }}
  </div>
</template>

<template #day="scope">
  <div class="full-width full-height flex flex-center">
    <q-btn
      flat round dense size="sm"
      :label="scope.day"
      class="calendar-day-btn"
      :class="{
        'event-day': hasEvents(scope.date.replaceAll('/', '-')),  // ← фикс
      }"
      @click="onDateClick(scope.date.replaceAll('/', '-'))"       // ← фикс
    >
      <q-tooltip
        v-if="getEventsByDate(scope.date.replaceAll('/', '-')).length"  // ← фикс
        ...
      >

        .event-day {
  background: rgba(255, 255, 255, 0.25) !important;
  border: 2px solid white !important;
  border-radius: 50% !important;
  font-weight: 700;
}
