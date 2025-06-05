<script>
import { ref, onMounted, onUnmounted, nextTick, computed } from "vue";
import axios from "axios";

export default {
  setup() {
    const activeTab = ref("all");
    const toastVisible = ref(false);
    const toastMessage = ref("");

    const expandedProcesses = ref({});  // { [processId]: true/false }
    const cabinetData = ref([]);        // Здесь должны быть ваши данные
    let observer;

    // Computed: все ли процессы раскрыты?
    const isAllExpanded = computed(() => {
      const ids = cabinetData.value.map(p => p.id);
      return ids.length > 0 && ids.every(id => expandedProcesses.value[id]);
    });

    // Загрузка состояний из sessionStorage
    const loadExpandedStates = () => {
      const savedStates = sessionStorage.getItem("expandedProcesses");
      if (savedStates) {
        expandedProcesses.value = JSON.parse(savedStates);
      }
    };

    // Сохранение состояний в sessionStorage
    const saveExpandedStates = () => {
      sessionStorage.setItem(
        "expandedProcesses",
        JSON.stringify(expandedProcesses.value)
      );
    };

    // Раскрыть все / Свернуть все
    const toggleProcessList = () => {
      const newState = !isAllExpanded.value;
      cabinetData.value.forEach((process) => {
        expandedProcesses.value[process.id] = newState;
      });
      saveExpandedStates();
    };

    // Индивидуальное переключение одного процесса
    const toggleProcess = (processId) => {
      expandedProcesses.value[processId] = !expandedProcesses.value[processId];
      saveExpandedStates();
    };

    // Подгрузка данных
    const fetchCabinetData = async () => {
      // Заглушка — замените своей логикой
      const response = await axios.get("/api/processes");
      cabinetData.value = response.data;
    };

    const fetchUsersData = async () => {};
    const fetchMaterialsData = async () => {};
    const fetchCommentListData = async () => {};
    const fetchCoworkers = async () => {};
    const changeStepperStyle = () => {};
    const changeFontSizeStyle = () => {};
    const changeAnotherFontSizeStyle = () => {};
    const changeLeftLineStyle = () => {};
    const changeQItemLabelSize = () => {};
    const showToast = (msg) => {
      toastMessage.value = msg;
      toastVisible.value = true;
      setTimeout(() => (toastVisible.value = false), 2000);
    };

    onMounted(async () => {
      loadExpandedStates();         // 1. Восстанавливаем состояния

      await fetchCabinetData();     // 2. Получаем список процессов

      nextTick(() => {
        // 3. Устанавливаем default значение false, если не было сохранено
        cabinetData.value.forEach((process) => {
          if (!(process.id in expandedProcesses.value)) {
            expandedProcesses.value[process.id] = false;
          }
        });
        saveExpandedStates();
      });

      fetchMaterialsData();
      fetchCommentListData();
      fetchUsersData();
      fetchCoworkers();

      // MutationObserver — стили и прочее
      observer = new MutationObserver(async (mutationsList) => {
        await nextTick();
        mutationsList.forEach(() => {
          changeStepperStyle();
          changeFontSizeStyle();
          changeAnotherFontSizeStyle();
          changeLeftLineStyle();
          changeQItemLabelSize();
        });
      });

      observer.observe(document.querySelector(".content-container"), {
        childList: true,
        subtree: true,
        attributes: true,
      });
    });

    onUnmounted(() => {
      observer?.disconnect();
    });

    return {
      activeTab,
      toastVisible,
      toastMessage,
      expandedProcesses,
      toggleProcessList,
      toggleProcess,
      isAllExpanded,
      cabinetData
    };
  },
};
</script>
