<template>
  <div id="q-app">
    <router-view />
  </div>
</template>

const toggleExpansion = (refVar, key, value) => {
  refVar.value = value;
  sessionStorage.setItem(key, JSON.stringify(value));
};

<q-expansion-item
  :model-value="isExpansionOpen1"
  @update:model-value="val => toggleExpansion(isExpansionOpen1, 'isExpansionOpen1', val)"
  ...
>
