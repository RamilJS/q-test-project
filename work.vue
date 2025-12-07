<span
  v-if="material.materialName === 'Перейти к форме рекомендации коллег'"
  class="process-link cursor-pointer text-primary link-look"
  @click="openCoworkersModal"
>
  {{ material.materialName }}
</span>

.link-look {
  text-decoration: underline;          /* подчёркивание */
  color: #007bff;                      /* цвет ссылки (или используй var(--q-primary)) */
  cursor: pointer;                     /* рука */
}

.link-look:hover {
  text-decoration: underline; /* подчёркивание остаётся */
  opacity: 0.8;               /* лёгкий hover-эффект */
}

.link-look:active {
  opacity: 0.6;
}

// или
.link-look {
  text-decoration: underline;
  color: var(--q-primary);
  cursor: pointer;
}
.link-look {
  text-decoration: underline;
  cursor: pointer;
  color: var(--q-primary);
  font-weight: 500;
}

.link-look:hover {
  color: #005bbb; /* чуть темнее — как у реальных ссылок */
}
