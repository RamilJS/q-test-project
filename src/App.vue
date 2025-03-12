<template>
  <div id="q-app">
    <router-view />
  </div>
</template>

.table-list {
  display: grid;
  grid-template-columns: repeat(6, 1fr);
  column-gap: 5px;
  padding: 0 0 10px;
  margin-bottom: 10px;
  border-bottom: 2px solid #ddd;
  list-style-type: none;
}

.table-row-item {
  display: flex;
  flex-wrap: wrap;
  flex-direction: column;
  max-width: 300px;
  width: 100%;
  box-sizing: border-box;
}

.table-row-name {
  display: block;
  width: 100%;
  box-sizing: border-box;

  /* Эти параметры обеспечивают перенос текста вниз */
  word-wrap: break-word;
  overflow-wrap: break-word;
  white-space: normal;
}
