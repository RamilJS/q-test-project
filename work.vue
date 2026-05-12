<q-separator class="q-my-md" />

<div class="calendar-widget q-px-md q-pb-md">
  <template>
  <div class="event-calendar">

    <q-date
      v-model="selectedDate"
      minimal
      flat
      color="primary"
      text-color="white"
      event-color="white"
      :events="eventDates"
      mask="YYYY-MM-DD"
    />

    <div class="events-block q-mt-md">

      <div
        v-if="selectedEvents.length === 0"
        class="no-events"
      >
        Сегодня нет событий
      </div>

      <q-list v-else dense>

        <q-item
          v-for="event in selectedEvents"
          :key="event.id"
        >
          <q-item-section>
            <q-item-label>
              {{ event.title }}
            </q-item-label>

            <q-item-label caption>
              {{ event.time }}
            </q-item-label>
          </q-item-section>
        </q-item>

      </q-list>

    </div>

  </div>
</template>
</div>

<q-separator class="q-my-md" />


<script setup>
import { ref, computed } from 'vue'

const selectedDate = ref('2026-05-08')

const events = ref([
  {
    id: 1,
    date: '2026-05-06',
    title: 'Вебинар',
    time: '14:00'
  },
  {
    id: 2,
    date: '2026-05-07',
    title: 'Обучение',
    time: '11:00'
  },
  {
    id: 3,
    date: '2026-05-15',
    title: 'Совещание',
    time: '16:30'
  }
])

const eventDates = computed(() =>
  events.value.map(e => e.date)
)

const selectedEvents = computed(() =>
  events.value.filter(
    e => e.date === selectedDate.value
  )
)
</script>


.event-calendar {
  border-radius: 12px;
  overflow: hidden;
  background: #1da1f2;
}

.event-calendar .q-date {
  background: #1da1f2;
  color: white;
}

.event-calendar .q-date__header {
  background: #1da1f2;
  color: white;
}

.event-calendar .q-date__calendar-item--event {
  border: 2px solid rgba(255,255,255,0.7);
  border-radius: 50%;
}

.events-block {
  background: #1da1f2;
  color: white;
  min-height: 80px;
  padding: 16px;
  border-top: 1px solid rgba(255,255,255,0.2);
}

.no-events {
  text-align: center;
  font-size: 16px;
}
