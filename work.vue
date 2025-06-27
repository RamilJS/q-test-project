<q-tabs
  v-model="activeCoworkersTab"
  dense
  class="text-primary bg-grey-2 shadow-1 rounded-borders q-pa-sm"
  active-color="primary"
  indicator-color="deep-purple"
  align="left"
  narrow-indicator
>
  <q-tab
    name="list"
    label="Список рекомендованных коллег"
    class="q-px-md text-bold text-dark"
    style="font-size: 16px;"
  />
  <q-tab
    name="recommendation"
    label="Форма рекомендации коллег"
    class="q-px-md text-bold text-dark"
    style="font-size: 16px;"
  />
</q-tabs>

// Материальный стиль с активной заливкой
<q-tabs
  v-model="activeCoworkersTab"
  class="bg-white text-dark shadow-2 rounded-borders"
  active-color="white"
  indicator-color="transparent"
  dense
>
  <q-tab
    name="list"
    label="Список"
    :class="activeCoworkersTab === 'list' ? 'bg-primary text-white shadow-1' : ''"
    class="q-mx-sm q-px-md q-py-sm rounded-borders"
  />
  <q-tab
    name="recommendation"
    label="Рекомендация"
    :class="activeCoworkersTab === 'recommendation' ? 'bg-primary text-white shadow-1' : ''"
    class="q-mx-sm q-px-md q-py-sm rounded-borders"
  />
</q-tabs>

// Минималистичный underline с жирным текстом
<q-tabs
  v-model="activeCoworkersTab"
  class="text-grey-8 q-mb-md"
  indicator-color="blue"
  active-color="blue-10"
  dense
  align="justify"
  narrow-indicator
>
  <q-tab
    name="list"
    label="Список"
    class="text-bold"
    style="font-size: 16px;"
  />
  <q-tab
    name="recommendation"
    label="Рекомендация"
    class="text-bold"
    style="font-size: 16px;"
  />
</q-tabs>


