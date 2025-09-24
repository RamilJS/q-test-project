"2011-12-26T10:45:15+00:00"
<script setup>
import { computed } from "vue"

// допустим, task у тебя приходит через props или из API
const props = defineProps({
  task: {
    type: Object,
    required: true
  }
})

// показываем только первые 2 комментария
const limitedComments = computed(() => {
  return props.task?.commentsArray?.slice(0, 2) || []
})
</script>

<template>
  <div v-if="task.commentsArray" style="padding: 0; margin: 0">
    <ul
      v-for="comment in limitedComments"
      :key="comment.id"
      class="comments-list"
    >
      <li class="comments-item column">
        <div class="row">
          <p>{{ formatXmlDate(comment.date) }}</p>
          <p>{{ comment.person_fullname }}</p>
          <p>{{ comment.comment_text }}</p>
        </div>

        <q-btn @click="deleteComment(comment.id)">
          Удалить
        </q-btn>
      </li>
    </ul>
  </div>
</template>
