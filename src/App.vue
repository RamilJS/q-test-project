
<li
  class="table-row-item"
  style="position: relative; overflow: hidden; max-width: 100%;"
>
  <p
    style="
      margin-bottom: 10px;
      word-wrap: break-word;
      overflow-wrap: break-word;
      white-space: normal;
      max-width: 100%;
    "
  >
    {{ infoTask.pathText }}
  </p>

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
    <span v-else style="word-wrap: break-word; overflow-wrap: break-word; white-space: normal;">
      {{ infoTask.pathText }}
    </span>
  </div>
</li>
