<q-item
  v-for="event in selectedEvents"
  :key="event.id"
  clickable
  v-ripple
  class="event-item"
  @click="goToEvent(event.link)"
>
  <q-item-section>
    <q-item-label class="text-weight-medium">
      {{ event.title }}
    </q-item-label>

    <q-item-label caption class="text-grey-3">
      {{ event.time }}
    </q-item-label>
  </q-item-section>

  <q-item-section side>
    <q-icon
      name="fa-solid fa-circle-arrow-right"
      size="24px"
      color="white"
    />
  </q-item-section>

  <q-tooltip>Перейти к мероприятию</q-tooltip>
</q-item>
