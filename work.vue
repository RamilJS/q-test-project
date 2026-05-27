ЗАМЕНИ ВЕСЬ ЭТОТ БЛОК:
const timestamp = dayEl.getAttribute("data-day");

if (!timestamp) return;

if (
  !timestamp.startsWith(
    `${currentYear.value}/`
  )
) return;

const date =
  timestamp.replaceAll("/", "-");
ВОТ НА ЭТО:

const parentDay = dayEl.closest(".q-date__calendar-item");

if (!parentDay) return;

const timestamp = parentDay.getAttribute("data-day");

if (!timestamp) return;

const date = timestamp.replaceAll("/", "-");


ЗАМЕНИ:
currentMonth.value = month + 1;
НА:
currentMonth.value = month;
