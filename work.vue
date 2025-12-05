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
      @click="showDatePopup = true"
    />
  </template>
</q-input>

<q-popup-proxy
  v-model="showDatePopup"
  :anchor="refEl"
  transition-show="scale"
  transition-hide="scale"
  auto-close
>
  <q-date
    v-model="newTaskActualDate"
    mask="DD.MM.YYYY"
    :locale="ruLocale"
    @update:model-value="showDatePopup = false"
  />
</q-popup-proxy>

const showDatePopup = ref(false);
const dateInput = ref(null);

const refEl = computed(() => dateInput.value?.$el);
