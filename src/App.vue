
<q-btn
  class="show-all-comments"
  color="primary"
  size="sm"
  padding="xs"
  style="margin-top: 5px;"
  label="Раскрыть полностью"
  @click="openHistoryModal(infoTask.id)"
/>

<q-dialog
  v-if="historyModalId === infoTask.id"
  v-model="isHistoryModalOpen"
>
  <q-card style="min-width: 600px; max-width: 90vw;">
    <q-card-section class="text-h6">
      История комментариев
    </q-card-section>

    <q-card-section style="max-height: 60vh; overflow-y: auto;">
      <div v-if="infoTask.comments && infoTask.comments.length">
        <div
          v-for="(comment, idx) in infoTask.comments"
          :key="idx"
          class="q-mb-md"
        >
          <q-card flat bordered>
            <q-card-section>
              <div class="text-subtitle2">
                {{ comment.author }} <span class="text-caption">({{ comment.date }})</span>
              </div>
              <div>{{ comment.text }}</div>
            </q-card-section>
          </q-card>
        </div>
      </div>
      <div v-else class="text-grey">
        Комментариев пока нет
      </div>
    </q-card-section>

    <q-card-actions align="right">
      <q-btn flat label="Закрыть" color="grey" v-close-popup />
    </q-card-actions>
  </q-card>
</q-dialog>

import { ref } from 'vue'

const isHistoryModalOpen = ref(false)
const historyModalId = ref(null)

function openHistoryModal(taskId) {
  historyModalId.value = taskId
  isHistoryModalOpen.value = true
}
