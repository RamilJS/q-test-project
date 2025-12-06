<q-input
  ref="dateInput"
  v-model="newTaskActualDate"
  label="Дата выполнения"
  outlined
  style="max-width: 200px"
  readonly
>
  <template #append>
    <q-icon
      name="event"
      class="cursor-pointer"
      @click="openPopup"
    />
  </template>
</q-input>

<q-popup-proxy
  v-model="showPopup"
  transition-show="scale"
  transition-hide="scale"
  auto-close
  ref="popup"
>
  <q-date
    v-model="newTaskActualDate"
    mask="DD.MM.YYYY"
    :locale="ruLocale"
    @update:model-value="closePopup"
  />
</q-popup-proxy>

const showPopup = ref(false);
const popup = ref(null);

function openPopup() {
  showPopup.value = true;

  // ждём рендера popup
  nextTick(() => {
    moveCalendarLeft();
  });
}

function closePopup() {
  showPopup.value = false;
}

// ─────────────────────
// Сдвигаем q-date влево
// ─────────────────────
function moveCalendarLeft() {
  const el = popup.value?.$el;
  if (!el) return;

  // Сдвигаем на 250px влево
  el.style.setProperty("--q-pe-left", "-250px");
}
