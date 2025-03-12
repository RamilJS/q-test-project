<template>
  <div id="q-app">
    <router-view />
  </div>
</template>

import { ref, onMounted, onBeforeUnmount, watch, nextTick } from "vue";

export default {
  setup() {
    // Данные
    const cabinetData = ref([]);
    const coworkersData = ref([]);
    const materialsData = ref([]);
    const commentListData = ref([]);
    const usersData = ref([]);

    // Состояния
    const allExpanded = ref(false);
    const expandedProcesses = ref({});
    const observer = ref(null);

    // Инициализация при загрузке
    onMounted(() => {
      fetchCabinetData();
      fetchMaterialsData();
      fetchCommentListData();
      fetchUsersData();
      fetchCoworkers();

      // Восстановление состояния из sessionStorage
      const savedAllExpanded = sessionStorage.getItem("allExpanded");
      if (savedAllExpanded !== null) {
        allExpanded.value = JSON.parse(savedAllExpanded);
      }

      const savedExpandedProcesses = sessionStorage.getItem("expandedProcesses");
      if (savedExpandedProcesses !== null) {
        expandedProcesses.value = JSON.parse(savedExpandedProcesses);
      } else {
        // Если нет сохраненного состояния, инициализируем значениями allExpanded
        cabinetData.value.forEach((process) => {
          expandedProcesses.value[process.id] = allExpanded.value;
        });
      }

      nextTick(() => {
        changeStyles();
      });

      observer.value = new MutationObserver(async () => {
        await nextTick();
        changeStyles();
      });

      observer.value.observe(document.querySelector(".content-container"), {
        childList: true,
        subtree: true,
        attributes: true,
      });
    });

    // Очистка observer при размонтировании компонента
    onBeforeUnmount(() => {
      if (observer.value) {
        observer.value.disconnect();
      }
    });

    // Вотчер: следим за изменениями expandedProcesses
    watch(
      expandedProcesses,
      (newVal) => {
        sessionStorage.setItem("expandedProcesses", JSON.stringify(newVal));
      },
      { deep: true }
    );

    // Тоггл открытия/закрытия всех процессов
    const toggleProcessList = () => {
      const newExpandedState = !allExpanded.value;

      cabinetData.value.forEach((process) => {
        expandedProcesses.value[process.id] = newExpandedState;
      });

      allExpanded.value = newExpandedState;

      // Сохраняем состояния
      sessionStorage.setItem("allExpanded", JSON.stringify(allExpanded.value));
      sessionStorage.setItem("expandedProcesses", JSON.stringify(expandedProcesses.value));
    };

    // Фейковые функции для данных
    const fetchCabinetData = () => {
      // Имитация запроса данных
      cabinetData.value = [
        { id: 1, title: "Процесс 1" },
        { id: 2, title: "Процесс 2" },
        { id: 3, title: "Процесс 3" },
      ];

      // Если expandedProcesses пустой, инициализируем его после загрузки данных
      if (Object.keys(expandedProcesses.value).length === 0) {
        cabinetData.value.forEach((process) => {
          expandedProcesses.value[process.id] = allExpanded.value;
        });
      }
    };

    const fetchCoworkers = () => {
      coworkersData.value = []; // или какие-то данные
    };

    const fetchMaterialsData = () => {
      materialsData.value = []; // или какие-то данные
    };

    const fetchCommentListData = () => {
      commentListData.value = []; // или какие-то данные
    };

    const fetchUsersData = () => {
      usersData.value = []; // или какие-то данные
    };

    // Функция обработки стилей после изменений DOM
    const changeStyles = () => {
      const container = document.querySelector(".content-container");
      if (!container) return;

      const processItems = container.querySelectorAll(".adaptation-process-item");
      const materialsBlock = document.querySelector(".materials__block");

      processItems.forEach((processItem) => {
        const qItem = processItem.querySelector(".q-expansion-item__container");
        if (!qItem) return;

        const qItemStyle = window.getComputedStyle(qItem);
        const qItemMarginTop = parseInt(qItemStyle.marginTop, 10);
        const qItemMarginBottom = parseInt(qItemStyle.marginBottom, 10);

        const qItemHeight = qItem.offsetHeight + qItemMarginTop + qItemMarginBottom;

        const itemOffsetTop = processItem.offsetTop;

        processItem.style.setProperty("--progress-line-offset-top", `${itemOffsetTop}px`);
        processItem.style.setProperty("--progress-line-offset-bottom", `${itemOffsetTop + qItemHeight}px`);
      });

      if (materialsBlock) {
        const materialsBlockOffsetTop = materialsBlock.offsetTop;
        materialsBlock.style.setProperty("--progress-line-offset-top", `${materialsBlockOffsetTop}px`);
      }
    };

    return {
      cabinetData,
      coworkersData,
      materialsData,
      commentListData,
      usersData,
      expandedProcesses,
      allExpanded,
      toggleProcessList,
      fetchCabinetData,
      fetchMaterialsData,
      fetchCommentListData,
      fetchUsersData,
      fetchCoworkers,
      changeStyles,
    };
  },
};
