    </ul>
  </div>
</template>
<div v-if="fakeCommentsArray" class="q-pa-md flex flex-center">
  <ul class="comments-list">
    <li
      v-for="comment in fakeCommentsArray"
      :key="comment.id"
      class="comments-item q-mb-sm"
    >
      <q-card flat bordered class="q-pa-sm">
        <div class="row items-center justify-between">
          <div class="col-auto">
            <p class="text-bold q-mr-sm">{{ formatXmlDate(comment.date) }}</p>
            <p class="text-bold q-mr-sm">{{ comment.person_fullname }}</p>
            <p class="q-mr-sm">{{ comment.comment_text }}</p>
          </div>
          <q-btn
            @click="deleteComment(comment.id)"
            size="sm"
            color="negative"
            flat
            dense
            label="Удалить"
          />
        </div>
      </q-card>
    </li>
  </ul>
</div>
.comments-list {
  list-style: none;
  padding: 0;
  margin: 0 auto;
  max-width: 700px; /* можно подогнать под вкус */
}


