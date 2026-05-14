

.custom-calendar :deep(.event-day) {
  border: 2px solid rgba(255, 255, 255, 0.95) !important;

  border-radius: 50% !important;

  width: 34px !important;
  height: 34px !important;

  min-width: 34px !important;
  min-height: 34px !important;

  transition: all 0.2s ease;
}



.custom-calendar :deep(.event-day:hover) {
  background: rgba(255, 255, 255, 0.12);
}


.custom-calendar :deep(.event-day .q-btn__content) {
  width: 100%;
  height: 100%;

  border-radius: 50%;
}

const tooltipWidth = 260;

let x =
  rect.left -
  wrapperRect.left +
  rect.width / 2;

const minX = tooltipWidth / 2 + 12;

const maxX =
  wrapperRect.width -
  tooltipWidth / 2 - 12;

/*
=========================================================
LEFT EDGE
=========================================================
*/

if (x < minX) {
  x = minX;
}

/*
=========================================================
RIGHT EDGE
=========================================================
*/

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
