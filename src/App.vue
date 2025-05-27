<li class="table-row-item" style="position: relative">
  <p style="margin-bottom: 10px">Название документа</p>

  <span v-if="copiedState[infoTask.id]" class="copied-message">
    Ссылка успешно скопирована!
  </span>

  <div>
    <q-btn
      v-if="infoTask.path !== ''"
      flat
      unelevated
      size="sm"
      padding="none"
      color="secondary"
      @click="copyToClipboard(infoTask.id, infoTask.path)"
      class="info-task-link"
    >
      Копировать ссылку для ознакомления
    </q-btn>
    <span v-else>{{ infoTask.pathText }}</span>
  </div>
</li>
