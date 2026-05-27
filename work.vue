const timestamp =
  dayEl.getAttribute("data-day");

if (!timestamp) return;

if (
  !timestamp.startsWith(
    `${currentYear.value}/`
  )
) return;

/*
=========================================================
NORMALIZE DATE
=========================================================
*/

const date =
  timestamp.replaceAll("/", "-");

  И ЕЩЁ

У тебя сейчас:

dayEl.classList.remove("event-day");

НО ты НЕ чистишь старые event handlers корректно.

ДОБАВЬ В bindTooltips()

В САМОМ НАЧАЛЕ:

tooltip.value.visible = false;
