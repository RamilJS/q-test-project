<template>
  <div id="q-app">
    <router-view />
  </div>
</template


value = false, 2000); // Убираем сообщение через 2 секунды
            } catch (err) {
                console.error("Ошибка при копировании: ", err);
            }
        };

        return { copied, copyToClipboard };
    }
}).use(Quasar).mount("#app");
</script>

</body>
</html>

<li class="table-row-item" style="position: relative;">
  <p style="margin-bottom: 10px;">Название документа</p>

  <span
    v-if="true"
    style="
      position: absolute;
      bottom: 100%;
      left: 50%;
      transform: translateX(-50%);
      padding: 5px;
      background: white;
      color: #0c205a;
      font-size: 12px;
      z-index: 999;
      border: 1px solid #ccc;
      border-radius: 4px;
      white-space: nowrap;
      box-shadow: 0 2px 5px rgba(0, 0, 0, 0.2);
    "
  >
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
      @click="copyToClipboard(infoTask.path)"
      class="info-task-link"
    >
      Копировать ссылку для ознакомления
    </q-btn>
