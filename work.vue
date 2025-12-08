<!-- Назначенные встречи -->
<div class="employee-adaptation-container text-info-item column">
  <div class="text-info-wrapper">
    <p class="text-info-title text-grey q-mb-sm">Встречи по адаптации</p>
    <div class="row" style="margin-bottom:8px">
      <p v-if="employee.events.length > 0" class="text-info-value text-secondary">
        {{ pluralizeMeetings(employee.events.length) }}
      </p>
      <q-btn v-if="employee.events.length > 0" color="grey" icon="fa-solid fa-info" size="6px" push round class="metting-modal-button" style="margin-bottom:auto;margin-left:10px;">
        <q-tooltip class="bg-white text-black">
          <ul class="meeting-list q-mb-none q-pa-none">
            <li v-for="event in employee.events" :key="event.date_start" class="meeting-item">
              {{ event.date_start }}
            </li>
          </ul>
        </q-tooltip>
      </q-btn>
    </div>
  </div>
  
  <div class="text-info-wrapper meeting-modal column" style="margin-top:auto;">
    <div class="metting-link-wrapper column">
      <q-btn v-if="false" color="primary" text-color="white" label="НАЗНАЧИТЬ ВСТРЕЧУ" size="sm" padding="sm" @click="createNewMeeting(employee.person.person_id)" />
      
      <!-- Форма создания новой встречи -->
      <q-dialog v-if="meetingModalId === employee.person.person_id" v-model="newMeetingModalOpen">
        <q-card class="q-pa-md" style="min-width:600px;max-width:800px;box-shadow: 0 4px 12px rgba(0,0,0,0.2);">
          <q-card-section class="q-pb-none">
            <div class="text-h6 q-mb-md text-primary">Назначение встречи по адаптации</div>
          </q-card-section>
          
          <q-separator spaced />
          
          <q-card-section>
            <div class="q-gutter-md column">
              <!-- Блок с выбором сотрудников - ОБНОВЛЯЕМ -->
              <div>
                <div class="row items-center q-gutter-sm q-mb-sm">
                  <div class="text-subtitle2" style="width:80px;">Дополнительные участники встречи:</div>
                  <q-input
                    dense
                    filled
                    v-model="newMeetingEmailDisplay"
                    readonly
                    placeholder="Нажмите для выбора сотрудников"
                    class="col"
                  />
                  <q-btn dense flat round icon="add" @click="collaboratorDialogOpen = true" />
                </div>
                
                <!-- Отображение выбранных сотрудников как чипсы -->
                <div v-if="selectedCollaborators.length > 0" class="q-mt-sm">
                  <q-chip
                    v-for="(collaborator, index) in selectedCollaborators"
                    :key="collaborator.id"
                    removable
                    @remove="removeCollaborator(index)"
                    color="primary"
                    text-color="white"
                    class="q-ma-xs"
                  >
                    <div class="row items-center">
                      <span>{{ collaborator.name }}</span>
                      <span class="q-ml-xs text-caption">({{ collaborator.email }})</span>
                    </div>
                  </q-chip>
                </div>
              </div>
              
              <!-- Диалоговое окно поиска сотрудника -->
              <q-dialog v-model="collaboratorDialogOpen">
                <q-card style="min-width:400px;max-width:600px;">
                  <q-card-section>
                    <div class="text-h6">Выбор сотрудника</div>
                  </q-card-section>
                  
                  <q-card-section class="q-pt-none">
                    <div class="row q-col-gutter-md q-mb-sm">
                      <div class="col">
                        <q-input
                          v-model="collaboratorSearch"
                          placeholder="Поиск сотрудника..."
                          outlined
                          dense
                          @keyup.enter="fetchCollaboratorSearch"
                        />
                      </div>
                      <div>
                        <q-btn color="primary" label="Найти" @click="fetchCollaboratorSearch" />
                      </div>
                    </div>
                    
                    <q-list bordered>
                      <q-item
                        v-for="item in collaboratorListData"
                        :key="item.person_id"
                        clickable
                        @click="selectCollaborator(item)"
                        :class="{'bg-green-1': newMeetingEmail.includes(item.email)}"
                      >
                        <q-item-section>
                          <q-item-label>{{ item.fullname }}</q-item-label>
                          <q-item-label caption>{{ item.email }}</q-item-label>
                        </q-item-section>
                        <q-item-section side v-if="newMeetingEmail.includes(item.email)">
                          <q-icon name="check" color="green" />
                        </q-item-section>
                      </q-item>
                    </q-list>
                    
                    <div class="row justify-between items-center q-mt-sm">
                      <q-btn
                        flat
                        label="Назад"
                        :disable="collaboratorPage <= 1"
                        @click="collaboratorPage-- && fetchCollaboratorList()"
                      />
                      <q-btn
                        flat
                        label="Вперёд"
                        @click="collaboratorPage++ && fetchCollaboratorList()"
                      />
                    </div>
                  </q-card-section>
                  
                  <q-card-actions align="right">
                    <q-btn flat label="Закрыть" v-close-popup />
                  </q-card-actions>
                </q-card>
              </q-dialog>
              
              <!-- Остальные поля формы без изменений -->
              <div class="row items-center q-gutter-sm">
                <div class="text-subtitle2" style="width:80px;">Тема:</div>
                <q-input dense filled v-model="newMeetingTopic" placeholder="Встреча по адаптации" class="col" />
              </div>
              
              <div class="row items-center q-gutter-sm">
                <div class="text-subtitle2" style="width:80px;">Место:</div>
                <q-input dense filled v-model="newMeetingLocation" placeholder="Укажите место" class="col" />
              </div>
              
              <div class="row items-center q-gutter-md">
                <div class="text-subtitle2" style="width:130px;">Время начала:</div>
                <q-input dense filled v-model="newMeetingDateStart" type="date" style="max-width:160px;" />
                <q-input dense filled v-model="newMeetingTimeStart" type="time" style="max-width:100px;" />
              </div>
              
              <div class="row items-center q-gutter-md">
                <div class="text-subtitle2" style="width:130px;">Окончание:</div>
                <q-input dense filled v-model="newMeetingDateEnd" type="date" style="max-width:160px;" />
                <q-input dense filled v-model="newMeetingTimeEnd" type="time" style="max-width:100px;" />
              </div>
            </div>
          </q-card-section>
          
          <q-card-actions align="right">
            <q-btn flat label="Закрыть" color="primary" @click="handleDialogClose" />
            <q-btn label="Отправить" color="primary" flat @click="postNewMeeting(employee.id)" />
          </q-card-actions>
        </q-card>
      </q-dialog>
    </div>
  </div>
</div>
<script>
import { ref, onMounted, onUnmounted, watch, computed } from "vue";

export default {
  setup() {
    // ... ваш существующий код ...
    
    // ВАЖНО: оставляем newMeetingEmail как ref
    const newMeetingEmail = ref([]);
    
    // Добавляем ref для хранения объектов сотрудников
    const selectedCollaborators = ref([]);
    
    // УДАЛЯЕМ старый computed и создаем новый для отображения email
    const newMeetingEmailDisplay = computed({
      get() {
        // Возвращаем email через точку с запятой
        return newMeetingEmail.value.join("; ");
      },
      set(value) {
        // Оставляем для обратной совместимости
        newMeetingEmail.value = value.split(";").map(e => e.trim()).filter(e => e);
      }
    });
    
    // Обновляем функцию selectCollaborator
    const selectCollaborator = (item) => {
      // Проверяем, что email есть и его еще нет в массиве
      if (item.email && !newMeetingEmail.value.includes(item.email)) {
        // Добавляем email в массив для отправки
        newMeetingEmail.value.push(item.email);
        
        // Сохраняем объект сотрудника для отображения в чипсах
        selectedCollaborators.value.push({
          id: item.person_id || item.id,
          name: item.fullname || item.name,
          email: item.email,
          ...item
        });
      }
      collaboratorDialogOpen.value = false;
      collaboratorSearch.value = '';
    };
    
    // Обновляем функцию removeCollaborator
    const removeCollaborator = (index) => {
      // Удаляем email из массива
      const removedCollaborator = selectedCollaborators.value[index];
      const emailIndex = newMeetingEmail.value.indexOf(removedCollaborator.email);
      
      if (emailIndex !== -1) {
        newMeetingEmail.value.splice(emailIndex, 1);
      }
      
      // Удаляем объект сотрудника
      selectedCollaborators.value.splice(index, 1);
    };
    
    // Обновляем resetMeetingForm
    const resetMeetingForm = () => {
      newMeetingDateStart.value = '';
      newMeetingTimeStart.value = '';
      newMeetingDateEnd.value = '';
      newMeetingTimeEnd.value = '';
      newMeetingEmail.value = [];
      newMeetingTopic.value = 'Встреча по адаптации';
      newMeetingLocation.value = '';
      collaboratorDialogOpen.value = false;
      selectedCollaborators.value = []; // Очищаем объекты сотрудников
      collaboratorSearch.value = "";
    };
    
    // ... остальной ваш код без изменений ...
    
    return {
      // ... все ваши существующие переменные ...
      selectedCollaborators, // ДОБАВЛЯЕМ в return
      newMeetingEmailDisplay, // Уже есть у вас, оставляем
      removeCollaborator, // ДОБАВЛЯЕМ в return
      // ... остальные ваши методы ...
    };
  }
};
</script>



