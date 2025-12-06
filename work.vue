const showPopup = ref(false);

const openDatePopup = () => {
  showPopup.value = true;

  nextTick(() => {
    positionQDatePopup();
  });
};

const positionQDatePopup = () => {
  setTimeout(() => {
    const menu = document.querySelector(".q-menu.q-position-engine");
    const input = document.querySelector(".my-date-input"); // класс зададим вручную

    if (!menu || !input) return;

    const rect = input.getBoundingClientRect();

    // Выставляем фиксированную позицию
    menu.style.position = "fixed";
    menu.style.top = rect.top + "px";
    menu.style.left = rect.left - menu.offsetWidth - 10 + "px"; 
    // ^^^ 10px — расстояние между input и calendar

  }, 10);
};

<q-input
  class="my-date-input"
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
      @click="openDatePopup"
    />
  </template>
</q-input>

<q-popup-proxy
  v-model="showPopup"
  transition-show="scale"
  transition-hide="scale"
>
  <q-date
    v-model="newTaskActualDate"
    mask="DD.MM.YYYY"
    :locale="ruLocale"
    auto-close
  />
</q-popup-proxy>
