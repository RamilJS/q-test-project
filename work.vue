<style scoped>

/*
===========================================================
CALENDAR WRAPPER
===========================================================
*/

.calendar-wrapper {

  position: relative;

  display: flex;

  justify-content: center;

  overflow: visible;

}

/*
===========================================================
CALENDAR
===========================================================
*/

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
DAY BUTTON
===========================================================
*/

.custom-calendar :deep(.q-date__calendar-item .q-btn) {

  width: 34px;

  height: 34px;

  min-width: 34px;

  min-height: 34px;

  border-radius: 50%;

  position: relative;

  overflow: hidden;

}

/*
===========================================================
EVENT DAY
===========================================================
*/

.custom-calendar :deep(.event-day) {

  border-radius: 50% !important;

  box-shadow:
    inset 0 0 0 2px rgba(255,255,255,.95);

  transition:
    background .2s ease,
    box-shadow .2s ease;

}

/*
===========================================================
EVENT HOVER
===========================================================
*/

.custom-calendar :deep(.event-day:hover) {

  background:
    rgba(255,255,255,.12);

}

/*
===========================================================
BUTTON CONTENT
===========================================================
*/

.custom-calendar :deep(.event-day .q-btn__content) {

  display: flex;

  align-items: center;

  justify-content: center;

  border-radius: 50%;

  position: relative;

  z-index: 2;

}

/*
===========================================================
ACTIVE DAY
===========================================================
*/

.custom-calendar :deep(.q-date__calendar-item--active .q-btn) {

  border-radius: 50% !important;

}

/*
===========================================================
TODAY
===========================================================
*/

.custom-calendar :deep(.q-date__today) {

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

  background:
    rgba(35,35,35,.96);

  color: white;

  backdrop-filter: blur(12px);

  box-shadow:
    0 8px 24px rgba(0,0,0,.25);

  pointer-events: none;

  font-size: 13px;

}

/*
===========================================================
TOOLTIP EVENTS
===========================================================
*/

.tooltip-event:not(:last-child) {

  margin-bottom: 10px;

  padding-bottom: 10px;

  border-bottom:
    1px solid rgba(255,255,255,.1);

}

</style>
