.calendar-day-btn {
  color: white;
  width: 34px;
  height: 34px;
  min-width: 34px;

  position: relative;
  transition: all 0.2s ease;
}

/* ДАТА С СОБЫТИЕМ */
.event-day {
  background: rgba(255,255,255,0.12);
  border: 2px solid #ffffff !important;
  border-radius: 50%;
  box-shadow:
    0 0 0 2px rgba(255,255,255,0.15),
    0 0 10px rgba(255,255,255,0.35);
}

/* ХОВЕР */
.event-day:hover {
  transform: scale(1.08);
  background: rgba(255,255,255,0.22);
}

/* СЕГОДНЯ */
.event-calendar .q-date__today {
  border: 2px solid #ffd54f !important;
  border-radius: 50%;
}

/* ЕСЛИ СЕГОДНЯ + СОБЫТИЕ */
.event-day.q-date__today {
  border: 2px solid #ffd54f !important;
  background: rgba(255,255,255,0.22);
}

<q-btn
  flat
  round
  dense
  size="sm"
  :label="scope.day"
  class="calendar-day-btn"
  :class="{
    'event-day': hasEvents(scope.date),
    'q-date__today': scope.today
  }"
  @click="onDateClick(scope.date)"
>
