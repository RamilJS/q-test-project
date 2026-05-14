

.custom-calendar
  :deep(.event-day .q-btn__content) {

  width: 34px !important;
  height: 34px !important;

  border: 2px solid white !important;

  border-radius: 50%;

  transition: all 0.2s ease;
}



.custom-calendar
  :deep(.event-day:hover .q-btn__content) {

  background:
    rgba(255,255,255,0.12);
}



.custom-calendar
  :deep(.q-date__today .q-btn__content) {

  border:
    2px solid rgba(255,255,255,0.45);

  border-radius: 50%;
}


.custom-calendar
  :deep(.q-date__calendar-item--active .q-btn__content) {

  background: white !important;

  color: #1976d2 !important;

  border: none !important;

  border-radius: 50%;
}
