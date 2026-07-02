


var iEventID, aFormFields, sNextStep, oForm, sFormTitle, oEvent, oEventTE, oOptions, sSubmitType;
 
oMailTemplate = new Object();
oMailTemplate.manager = "Уважаемый(ая) ##фио_руководителя##!<br/><br/>При бюджетировании внешнего обучения на <b>##текущий_год##</b> год и/или при постановке целей по развитию на <b>##текущий_год##</b> год для Вашего сотрудника ##фио_сотрудника## было запланировано участие в обучающем мероприятии <b>\"##название_мероприятия##\"</b>.<br/><br/>Данный курс запланирован к проведению в следующие даты: <br/>##список_дат##<br/>Ориентировочная стоимость – <b>##ориентировочная_стоимость##</b> Итоговая стоимость будет зависеть от количества участников в группе. <br/><br/>Этапы согласования:<br/>##список_согласования##<br/>Ссылка на заявку: ##ссылка_на_заявку##<br/><br/><b>Если участие сотрудника в данном обучении неактуально</b>, то нажмите кнопку <br/>##кнопка_отклонения##.<br/><br/><b>Если участие сотрудника в данном обучении актуально</b>, то нажмите кнопку <br/>##кнопка_подтверждения##, либо проигнорируйте письмо (в этом случае участие будет считаться согласованным Вами и через ##количество_дней_руководителя## рабочих дня Учебный портал переведет заявку на следующий этап).<br/><br/><br/>В случае вопросов, пожалуйста, обращайтесь к куратору курса: ##отвественный_сотрудник##<br/>Не отвечайте на это письмо. Для согласования задачи используйте ссылки в письме.";
oMailTemplate.manager_remember = "Уважаемый(ая) ##фио_руководителя##!<br/><br/>Напоминаем вам, что при бюджетировании внешнего обучения на <b>##текущий_год##</b> год и/или при постановке целей по развитию на <b>##текущий_год##</b> год для Вашего сотрудника ##фио_сотрудника## было запланировано участие в обучающем мероприятии <b>\"##название_мероприятия##\"</b>.<br/><br/>Данный курс запланирован к проведению в следующие даты: <br/>##список_дат##<br/>Ориентировочная стоимость – <b>##ориентировочная_стоимость##</b> Итоговая стоимость будет зависеть от количества участников в группе. <br/><br/>Этапы согласования:<br/>##список_согласования##<br/>Ссылка на заявку: ##ссылка_на_заявку##<br/><br/><b>Если участие сотрудника в данном обучении неактуально</b>, то нажмите кнопку <br/>##кнопка_отклонения##.<br/><br/><b>Если участие сотрудника в данном обучении актуально</b>, то нажмите кнопку <br/>##кнопка_подтверждения##.<br/><br/><br/>В случае вопросов, пожалуйста, обращайтесь к куратору курса: ##отвественный_сотрудник##<br/>Не отвечайте на это письмо. Для согласования задачи используйте ссылки в письме.";
oMailTemplate.mir_code_manager  = "Уважаемый(ая) ##фио_руководителя_мир_кода##!<br/><br/>При бюджетировании внешнего обучения на <b>##текущий_год##</b> год и/или при постановке целей по развитию на <b>##текущий_год##</b> год для Вашего сотрудника ##фио_сотрудника## было запланировано участие в обучающем мероприятии <b>\"##название_мероприятия##\"</b>.<br/><br/>Данный курс запланирован к проведению в следующие даты: <br/>##список_дат##<br/>Ориентировочная стоимость – <b>##ориентировочная_стоимость##</b> Итоговая стоимость будет зависеть от количества участников в группе. <br/><br/>Этапы согласования:<br/>##список_согласования##<br/>Ссылка на заявку: ##ссылка_на_заявку##<br/><br/><b>Если участие сотрудника в данном обучении неактуально</b>, то нажмите кнопку <br/>##кнопка_отклонения##.<br/><br/><b>Если участие сотрудника в данном обучении актуально</b>, то нажмите кнопку <br/>##кнопка_подтверждения##, либо проигнорируйте письмо (в этом случае участие будет считаться согласованным Вами и через ##количество_дней_руководителя_мир_кода## рабочих дня Учебный портал переведет заявку на следующий этап).<br/><br/><br/>В случае вопросов, пожалуйста, обращайтесь к куратору курса: ##отвественный_сотрудник##<br/>Не отвечайте на это письмо. Для согласования задачи используйте ссылки в письме.";
oMailTemplate.mir_code_manager_remember  = "Уважаемый(ая) ##фио_руководителя_мир_кода##!<br/><br/>Напоминаем вам, что при бюджетировании внешнего обучения на <b>##текущий_год##</b> год и/или при постановке целей по развитию на <b>##текущий_год##</b> год для Вашего сотрудника ##фио_сотрудника## было запланировано участие в обучающем мероприятии <b>\"##название_мероприятия##\"</b>.<br/><br/>Данный курс запланирован к проведению в следующие даты: <br/>##список_дат##<br/>Ориентировочная стоимость – <b>##ориентировочная_стоимость##</b> Итоговая стоимость будет зависеть от количества участников в группе. <br/><br/>Этапы согласования:<br/>##список_согласования##<br/>Ссылка на заявку: ##ссылка_на_заявку##<br/><br/><b>Если участие сотрудника в данном обучении неактуально</b>, то нажмите кнопку <br/>##кнопка_отклонения##.<br/><br/><b>Если участие сотрудника в данном обучении актуально</b>, то нажмите кнопку <br/>##кнопка_подтверждения##.<br/><br/><br/>В случае вопросов, пожалуйста, обращайтесь к куратору курса: ##отвественный_сотрудник##<br/>Не отвечайте на это письмо. Для согласования задачи используйте ссылки в письме.";
oMailTemplate.collaborator = "Добрый день, ##фио_сотрудника##! <br/><br/>Ваше участие в мероприятии \"##название_мероприятия##\" планировалось в процессе бюджетирования внешнего обучения на ##текущий_год## год и/или при постановке целей по развитию на ##текущий_год## год в рамках УЭД.<br/><br/>Данное письмо подтверждает, что Ваш руководитель/владелец MIR кода подтвердил/(и) возможность Вашего участия в данном мероприятии. <br/><br/>Данный курс запланирован к проведению в следующие даты: <br/>##список_дат##<br/>Вам необходимо выбрать подходящие даты обучения и пройти регистрацию в течение <b>3-х рабочих дней с момента получения данного письма</b>.<br/><br/>После регистрации (можно выбрать только 1 группу) мероприятие будет автоматически добавлено в Ваш календарь.<br/><br/>Для выбора дат и регистрации перейдите по ссылке: ##ссылка_на_заявку##<br/><br/>Если Вы не планируете принимать участие в мероприятии, пожалуйста, нажмите на кнопку: <br/>##кнопка_отклонения##. <br/><br/>В случае вопросов, пожалуйста, обращайтесь к куратору курса: ##отвественный_сотрудник##<br/><br/><br/>Не отвечайте на это письмо. Для согласования задачи используйте ссылки в письме.";
function getParam(sName, sDefault) {
               var sValue = PARAMETERS.GetOptProperty(sName);
    //alert(sValue);
               if ( sDefault != undefined && (sValue == undefined || sValue == "") ) {
                              sValue = sDefault;
               }
               return sValue;
}
function getFormField(sName, sDefault) {
    var sValue = ArrayOptFind(aFormFields, ("This.name == " + XQueryLiteral(sName)));
    sValue = (sValue != undefined ? sValue.value : sValue);
    if ( sDefault != undefined && (sValue == undefined || sValue == "") ) {
                              sValue = sDefault;
               }
    return sValue;
}
function getFormFieldDefault(sName, sDefault) {
    var sValue = ArrayOptFind(aFormFieldsDef, ("This.name == " + XQueryLiteral(sName)));
    sValue = (sValue != undefined ? sValue.value : sValue);
    if ( sDefault != undefined && (sValue == undefined || sValue == "") ) {
                              sValue = sDefault;
               }
    return sValue;
}
 
 
aFormFields = ParseJson( getParam("form_fields", "[]") );
aFormFieldsDef = ParseJson( getParam("form_fields_default", "[]") );
 
iEventID = OptInt(getFormField("event_id", getFormFieldDefault("event_id", "step_0")), 0);
sSubmitType = getFormField("__submit_type__", getFormFieldDefault("__submit_type__", "step_0"));
 
oOptions = new Object();
oOptions.id = iEventID;
oOptions.name = "";
oOptions.next_step = "step_0";
oOptions.cost = 0;
oOptions.duration = 0;
oOptions.duration_days = 0;
oOptions.person_num = 0;
oOptions.place = "";
oOptions.type = "Очно"
oOptions.title = "Создание нового мероприятия"
oOptions.desc = ""
oOptions.dates = [];
oOptions.select_collaborator_ids = "";
oOptions.main_collaborator_id = "";
oOptions.ed_org = "";
oOptions.auto_accept_manager_days_count = 0;
oOptions.auto_accept_mir_code_manager_days_count = 0;
oOptions.manager_mail = oMailTemplate.manager;
oOptions.manager_mail_remember = oMailTemplate.manager_remember;
oOptions.mir_code_manager_mail = oMailTemplate.mir_code_manager;
oOptions.mir_code_manager_mail_remember = oMailTemplate.mir_code_manager_remember;
oOptions.collaborator_mail = oMailTemplate.collaborator;
if (ArrayOptFirstElem(aFormFields) != undefined){
               try {
                              oOptions.next_step = String(ArrayOptFind(aFormFields, " This.name == 'next_step' ").value);
               }
               catch(_e) {
                              oOptions.next_step = "step_0";
               }
}
 
if (iEventID != 0) {
    oEventTE = tools.open_doc(iEventID).TopElem;
    if (oEventTE.Name == "education_method") {
        oOptions.id = Int(iEventID);
        oOptions.name = String(oEventTE.name);
        oOptions.desc = String(oEventTE.desc);
        oOptions.title = "Редактирование мероприятия";
        oOptions.cost = String(oEventTE.cost);
        oOptions.duration = String(oEventTE.duration);
        oOptions.duration_days = String(oEventTE.duration_days);
        oOptions.person_num = String(oEventTE.person_num);
        oOptions.place = String(oEventTE.custom_elems.ObtainChildByKey("f_vtbl_place").value);
        oOptions.type = String(oEventTE.custom_elems.ObtainChildByKey("f_vtbl_type").value);
        oOptions.dates = String(oEventTE.custom_elems.ObtainChildByKey("f_vtbl_dates").value);
        oOptions.select_collaborator_ids = String(oEventTE.custom_elems.ObtainChildByKey("f_vtbl_collaborators").value);
        oOptions.main_collaborator_id = String(oEventTE.custom_elems.ObtainChildByKey("f_vtbl_main_collaborator").value);
        oOptions.ed_org = String(oEventTE.custom_elems.ObtainChildByKey("f_vtbl_ed_org").value);
        oOptions.dates = ParseJson(oOptions.dates != "" ? oOptions.dates : "[]");
        oOptions.auto_accept_manager_days_count = String(OptInt(oEventTE.custom_elems.ObtainChildByKey("f_vtbl_auto_accept_manager_days_count").value, 0));
        oOptions.auto_accept_mir_code_manager_days_count = String(OptInt(oEventTE.custom_elems.ObtainChildByKey("f_vtbl_auto_accept_mir_code_manager_days_count").value, 0));
        oOptions.manager_mail = String(oEventTE.custom_elems.ObtainChildByKey("f_vtbl_manager_mail").value);
        oOptions.manager_mail = (oOptions.manager_mail != "" ? oOptions.manager_mail : oMailTemplate.manager );
 
        oOptions.manager_mail_remember = String(oEventTE.custom_elems.ObtainChildByKey("f_vtbl_manager_mail_remember").value);
        oOptions.manager_mail_remember = (oOptions.manager_mail_remember != "" ? oOptions.manager_mail_remember: oMailTemplate.manager_remember );
 
        oOptions.mir_code_manager_mail = String(oEventTE.custom_elems.ObtainChildByKey("f_vtbl_mir_code_manager_mail").value);
        oOptions.mir_code_manager_mail = (oOptions.mir_code_manager_mail != "" ? oOptions.mir_code_manager_mail : oMailTemplate.mir_code_manager );
 
        oOptions.mir_code_manager_mail_remember = String(oEventTE.custom_elems.ObtainChildByKey("f_vtbl_mir_code_manager_mail_remember").value);
        oOptions.mir_code_manager_mail_remember = (oOptions.mir_code_manager_mail_remember != "" ? oOptions.mir_code_manager_mail_remember : oMailTemplate.mir_code_manager_remember );
 
        oOptions.collaborator_mail = String(oEventTE.custom_elems.ObtainChildByKey("f_vtbl_collaborator_mail").value);
        oOptions.collaborator_mail = (oOptions.collaborator_mail != "" ? oOptions.collaborator_mail : oMailTemplate.collaborator );
    }
}
 
oForm = new Object();
oForm.command = "display_form";
oForm.height = 320;
oForm.title = oOptions.title
oForm.message = "";
oForm.form_fields = [{
    name: "next_step",
    type: "hidden",
    value: oOptions.next_step,
    mandatory: false,
    visibility: [false]
},{
    name: "event_id",
    type: "hidden",
    value: oOptions.id,
    mandatory: false,
    visibility: [false]
},{
    name: "name",
    label: "Название мероприятия",
    type: "string",
    value: oOptions.name,
    placeholder: '111',
    mandatory: false,
    visibility: false
},{
    name: "ed_org",
    label: "Обучающая организация",
    type: "string",
    value: oOptions.ed_org,
    placeholder: '111',
    mandatory: false,
    visibility: false
},{
    name: "hour",
    label: "Количество академических часов",
    type: "integer",
    validation: "number",
    error_message: "Введите целое число академических часов",
    column: 1,
    value: oOptions.duration,
    mandatory: false,
    visibility: false
},{
    name: "day",
    label: "Количество дней",
    type: "integer",
    validation: "number",
    error_message: "Введите целое число количества дней",
    column: 2,
    value: oOptions.duration_days,
    mandatory: false,
    visibility: false
},{
    name: "event_type",
    label: "Форма проведения",
    type: "select",
    value: oOptions.type,
    validation: "nonempty",
    error_message: "Укажите форму проведения",
    entries: [{
        name: "Очно",
        value: "Очно"
    }, {
        name: "Онлайн",
        value: "Онлайн"
    }],
    mandatory: false,
    visibility: false
},{
    name: "group_cost",
    label: "Стоимость за группу",
    type: "real",
    validation: "number",
    error_message: "Введите стоимость за группу, обратите внимание, что пробелы не допускаются",
    column: 1,
    value: oOptions.cost,
    mandatory: false,
    visibility: false
},{
    name: "max_persons",
    label: "Максимальное количество участников",
    type: "integer",
    validation: "number",
    error_message: "Введите максимальное количество участников",
    column: 2,
    value: oOptions.person_num,
    mandatory: false,
    visibility: false
},{
    name: "place",
    label: "Место проведения",
    type: "string",
    value: oOptions.place,
    mandatory: false,
    visibility: false
},{
    name: "auto_accept_manager_days_count",
    label: "Количество дней до автосогласования руководителя (Если 0, то автосогласование не работает)",
    type: "integer",
    validation: "number",
    error_message: "Введите количество дней до автосогласования руководителя",
    value: oOptions.auto_accept_manager_days_count,
    column: 1,
    mandatory: false,
    visibility: false
},{
    name: "auto_accept_mir_code_manager_days_count",
    label: "Количество дней до автосогласования владельца Мир-кода (Если 0, то автосогласование не работает)",
    type: "integer",
    validation: "number",
    error_message: "Введите количество дней до автосогласования владельца Мир-кода",
    value: oOptions.auto_accept_mir_code_manager_days_count,
    column: 2,
    mandatory: false,
    visibility: false
},{
    name: "main_collaborator_id",
    label: "Выберите отвественного сотрудника",
    title: "Выберите сотрудника",
    type: "foreign_elem",
    value: oOptions.main_collaborator_id,
    mandatory: false,
    multiple: false,
    catalog: "collaborator",
    query_qual: "$elem/is_dismiss=false()"
},{
    name: "desc",
    label: "Краткое описание",
    type: "text",
    richtext: true,
    value: oOptions.desc,
    page: "page_two",
    mandatory: false,
    visibility: false
},{
    name: "person_find",
    type: "text",
    label: "Вставьте ФИО или email для поиска новых сотрудников. По одному на строчку",
    value: "",
    page: "page_two",
    mandatory: false,
    visibility: false
},{
    name: "select_collaborator_ids",
    label: "Выберите сотрудников из списка",
    title: "Выберите сотрудников",
    type: "foreign_elem",
    value: oOptions.select_collaborator_ids,
    mandatory: false,
    multiple: true,
    catalog: "collaborator",
    query_qual: "$elem/is_dismiss=false()"
},{
    name: "date_paragraph",
    type: "paragraph",
    value: "Внесите до 5 возможных дат проведения",
    page: "page_two",
    mandatory: false,
    visibility: false
}];
for (i = 1; i <=5; i++) {
    oForm.form_fields.push({
        name: ("date_" + i + "_paragraph"),
        type: "paragraph",
        value: "Дата " + i + ":",
        page: "page_two",
        mandatory: false,
        visibility: false
    });
    oForm.form_fields.push({
        name: ("date_" + i + "_start"),
        label: "Дата начала",
        type: "date",
        validation: "",
        error_message: "Введите дату",
        value: (ArrayCount(oOptions.dates) >= i ? oOptions.dates[i-1].start_date : ""),
        column: 1,
        mandatory: false,
        page: "page_two",
        mandatory: false,
        visibility: false
    });
    oForm.form_fields.push({
        name: ("date_" + i + "_finish"),
        label: "Дата окончания",
        type: "date",
        validation: "",
        error_message: "Введите дату",
        value: (ArrayCount(oOptions.dates) >= i ? oOptions.dates[i-1].finish_date : ""),
        column: 2,
        mandatory: false,
        page: "page_two",
        mandatory: false,
        visibility: false
    });
    oForm.form_fields.push({
        name: ("date_" + i + "_hour_start"),
        label: "Время начало в формате 'чч:мм'",
        type: "string",
        validation: "nonempty",
        column: 1,
        value: (ArrayCount(oOptions.dates) >= i ? oOptions.dates[i-1].start_hour : "10:00"),
        mandatory: false,
        page: "page_two",
        mandatory: false,
        visibility: false
    });
    oForm.form_fields.push({
        name: ("date_" + i + "_hour_finish"),
        label: "Время окончания в формате 'чч:мм'",
        type: "string",
        validation: "nonempty",
        column: 2,
        value: (ArrayCount(oOptions.dates) >= i ? oOptions.dates[i-1].finish_hour : "18:00"),
        mandatory: false,
        page: "page_two",
        mandatory: false,
        visibility: false
    });
    oForm.form_fields.push({
        name: ("date_" + i + "_event_type"),
        label: "Форма проведения",
        type: "select",
         value: (ArrayCount(oOptions.dates) >= i ? oOptions.dates[i-1].event_type : "Очно"),
        entries: [{
            name: "Очно",
            value: "Очно"
        }, {
            name: "Онлайн",
            value: "Онлайн" }
        ],
        mandatory: false,
        page: "page_two",
        mandatory: false,
        visibility: false
    });
}
oForm.form_fields.push({
    name: "start_requests_question",
    type: "paragraph",
    value: "Вы действительно хотите запустить процесс согласования? После старта возможность редактирования основных данных карточки мероприятия будет недоступна.",
    page: "page_two",
    mandatory: false,
    visibility: false
},{
    name: "start_save_question",
    type: "paragraph",
    value: "Вы изменили состав участников мероприятия, удалив сотрудников, по которым уже был запущен процесс согласования.<br/>Заявки следующих сотрудников будут удалены: <ul>##person_fullnames##</ul>.<br/>Вы подтверждаете данное действие?",
    page: "page_two",
    mandatory: false,
    visibility: false
},{
    name: "edit_mail_question",
    type: "paragraph",
    value: "Перед запуском проверьте и, при необходимости, исправьте шаблоны отправляемых писем.",
    page: "page_two",
    mandatory: false,
    visibility: false
},{
    name: "manager_mail",
    label: "Шаблон письма для руководителя",
    type: "text",
    richtext: true,
    value: oOptions.manager_mail,
    page: "page_two",
    mandatory: false,
    visibility: false
},{
    name: "mir_code_manager_mail",
    label: "Шаблон письма для владельца Мир-кода",
    type: "text",
    richtext: true,
    value: oOptions.mir_code_manager_mail,
    page: "page_two",
    mandatory: false,
    visibility: false
},{
    name: "collaborator_mail",
    label: "Шаблон письма для сотрудника",
    type: "text",
    richtext: true,
    value: oOptions.collaborator_mail,
    page: "page_two",
    mandatory: false,
    visibility: false
},{
    name: "select_remember_option",
    label: "Выберите, кому вы хотите отправить напоминания о необходимости согласовать потребность в обучении",
    type: "select",
    value: "0",
    entries: [{
        name: "Руководителям",
        value: "0"
    }, {
        name: "Владельцам Мир-Кодов",
        value: "1"
    }],
    mandatory: false,
    page: "page_two",
    mandatory: false,
    visibility: false
},{
    name: "manager_mail_remember",
    label: "Шаблон письма-напоминания для руководителя",
    type: "text",
    richtext: true,
    value: oOptions.manager_mail_remember,
    page: "page_two",
    mandatory: false,
    visibility: false
},{
    name: "mir_code_manager_mail_remember",
    label: "Шаблон письма-напоминания для владельца Мир-кода",
    type: "text",
    richtext: true,
    value: oOptions.mir_code_manager_mail_remember,
    page: "page_two",
    mandatory: false,
    visibility: false
},{
    name: "edit_mail_help",
    type: "paragraph",
    value: "Список возможных автоподстановок:<p style=\"font-size: 10px\"><br/><b>##фио_сотрудника##</b> - ФИО сотрудника, который хочет обучаться<br/><b>##фио_руководителя##</b> - ФИО непосредственного руководителя<br/><b>##фио_руководителя_мир_кода##</b> - ФИО владельца Мир-кода сотрудника<br/><b>##текущий_год##</b> - Текущий год<br/><b>##название_мероприятия##</b> - Название мероприятия<br/><b>##список_дат##</b> - Список групп для мероприятия. Даты и формат проведения<br/><b>##ориентировочная_стоимость##</b> - Ориентировочная стоимость в формате \"ХХХ руб./чел.\"<br/><b>##ссылка_на_заявку##</b> - ссылка на заявку<br/><b>##кнопка_подтверждения##</b> - формирует письмо на подтверждение заявки<br/><b>##кнопка_отклонения##</b> - формирует письмо на отклонение заявки<br/><b>##количество_дней_руководителя##</b> - показывает, через сколько дней произойдет автосогласование заявки руководителем<br/><b>##количество_дней_руководителя_мир_кода##</b> - показывает, через сколько дней произойдет автосогласование заявки владельцем Мир-кода<br/><b>##список_согласования##</b> - формирует таблицу с ифнормацией о том, кто согласовал<br/><b>##отвественный_сотрудник##</b> - формирует таблицу с ифнормацией о том, кто согласовал<br/><p>",
    page: "page_two",
    mandatory: false,
    visibility: false
},{
    name: "archive_question",
    type: "paragraph",
    value: "Вы действительно хотите перевести все существующие данные в архив?",
    page: "page_two",
    mandatory: false,
    visibility: false
});
deleteDatesArray = [];
for (i = 0; i < ArrayCount(oOptions.dates); i++) {
    deleteDatesArray.push({
        name: (StrDate(Date(oOptions.dates[i].start_date), false) + " - " + StrDate(Date(oOptions.dates[i].finish_date), false) + " " + oOptions.dates[i].event_type),
        value: String(i)
    })
}
oForm.form_fields.push({
        name: "delete_dates",
        label: "Выберите мепроприятие которое хотите удалить",
        type: "select",
        value: "0",
        entries: deleteDatesArray,
        mandatory: false,
        page: "page_two",
        mandatory: false,
        visibility: false
    });
oForm.buttons = [];
oForm.no_buttons = false;
switch(sSubmitType){
    case "step_0": {
        aMandatoryFields = ["name", "ed_org", "hour", "day", "event_type", "group_cost", "max_persons", "place", "main_collaborator_id", "auto_accept_manager_days_count", "auto_accept_mir_code_manager_days_count"];
        aVisibleFields = ["name", "ed_org", "hour", "day", "event_type", "group_cost", "max_persons", "place", "date_paragraph", "main_collaborator_id", "desc", "auto_accept_manager_days_count", "auto_accept_mir_code_manager_days_count"];
        for (i = 1; i <=5; i++) {
            aVisibleFields.push("date_" + i + "_paragraph");
            aVisibleFields.push("date_" + i + "_start");
            aVisibleFields.push("date_" + i + "_finish");
            aVisibleFields.push("date_" + i + "_hour_start");
            aVisibleFields.push("date_" + i + "_hour_finish");
            aVisibleFields.push("date_" + i + "_event_type");
        }
        for (oField in oForm.form_fields) {
            oField.type = ((ArrayOptFind(aVisibleFields, "This == oField.name") == undefined) ? "hidden" : oField.type );
            try {
                oField.validation = ((ArrayOptFind(aVisibleFields, "This == oField.name") == undefined) ? "" : oField.validation );
            }
            catch(_e){}
            oField.mandatory = (ArrayOptFind(aMandatoryFields, "This == oField.name") != undefined);
            oField.value = getFormField(oField.name, oField.value);
        }
        oForm.buttons.push({
            name: "submit",
            submit_type: "add_from_text",
            label: "Добавить из файла",
            type: "submit"
        },{
            name: "submit",
            submit_type: "add_from_list",
            label: "Добавить поиском из списка сотрудников",
            type: "submit"
        },{
            name: "submit",
            submit_type: "save",
            label: "Сохранить карточку",
            type: "submit"
        },{
            name: "cancel",
            label: "Отменить",
            type: "cancel"
        });
        break;
    }
    case "add_from_text": {
        aMandatoryFields = ["person_find"];
        aVisibleFields = ["person_find"];
        for (oField in oForm.form_fields) {
            oField.type = ((ArrayOptFind(aVisibleFields, "This == oField.name") == undefined) ? "hidden" : oField.type );
            try {
                oField.validation = ((ArrayOptFind(aVisibleFields, "This == oField.name") == undefined) ? "" : oField.validation );
            }
            catch(_e){}
            oField.mandatory = (ArrayOptFind(aMandatoryFields, "This == oField.name") != undefined);
            oField.value = getFormField(oField.name, oField.value);
        }
        oForm.buttons.push({
            name: "submit",
            submit_type: "step_0",
            label: "Отредактировать основную информацию",
            type: "submit"
        },{
            name: "submit",
            submit_type: "add_from_list",
            label: "Проверить наличие сотрудников",
            type: "submit"
        },{
            name: "submit",
            submit_type: "add_from_list",
            label: "Добавить поиском из списка сотрудников",
            type: "submit"
        },/*{
            name: "submit",
            submit_type: "save",
            label: "Сохранить карточку",
            type: "submit"
        },*/{
            name: "cancel",
            label: "Отменить",
            type: "cancel"
        })
        break;
    }
    case "add_from_list": {
        aMandatoryFields = [];
        aVisibleFields = ["select_collaborator_ids"];
        for (oField in oForm.form_fields) {
            oField.type = ((ArrayOptFind(aVisibleFields, "This == oField.name") == undefined) ? "hidden" : oField.type );
            try {
                oField.validation = ((ArrayOptFind(aVisibleFields, "This == oField.name") == undefined) ? "" : oField.validation );
            }
            catch(_e){}
            oField.mandatory = (ArrayOptFind(aMandatoryFields, "This == oField.name") != undefined);
            oField.value = getFormField(oField.name, oField.value);
        }
        oField = ArrayOptFind(oForm.form_fields, "This.name == 'select_collaborator_ids'");
        aFieldSearch = String(ArrayOptFind(oForm.form_fields, "This.name == 'person_find'").value).split("\n");
        if (ArrayCount(aFieldSearch) > 0) {
            for (sField in aFieldSearch) {
                if (Trim(sField) != "") {
                    //aCollaborators = XQuery(alert("for $elem in collaborators where $elem/is_dismiss=false() and (MatchSome($elem/fullname, ('" + ArrayMerge(aFieldSearch, "Trim(This)", "', '") + "')) or MatchSome($elem/email, ('" + ArrayMerge(aFieldSearch, "Trim(This)", "', '") + "'))) return $elem"));
                    aCollaborators = XQuery(alert("for $elem in collaborators where $elem/is_dismiss=false() and (contains($elem/fullname, " + XQueryLiteral(sField) + ") or contains($elem/email, " + XQueryLiteral(sField) + ")) return $elem"));
                    for (oCollaborator in aCollaborators) {
                        oField.value += (oField.value != "" ? ";" : "") + oCollaborator.id;
                    }
                }
            }
        }
        oForm.buttons.push({
            name: "submit",
            submit_type: "step_0",
            label: "Отредактировать основную информацию",
            type: "submit"
        },{
            name: "submit",
            submit_type: "add_from_text",
            label: "Добавить из файла",
            type: "submit"
        },{
            name: "submit",
            submit_type: "save",
            label: "Сохранить карточку",
            type: "submit"
        },{
            name: "cancel",
            label: "Отменить",
            type: "cancel"
        })
        break;
    }
    case "save":
    case "save_done": {
 
        bJustSave = (sSubmitType == "save_done");
        oOptions.id = OptInt(getFormField("event_id", 0), 0);
        aOldRequests = ArraySelectAll(XQuery("for $elem in requests where $elem/request_type_id=7528328911430055448 and $elem/object_id=" + XQueryLiteral(oOptions.id) + " and $elem/workflow_state != 'archive' return $elem"));
        aNewIDs = ArraySelect(String(getFormField("select_collaborator_ids", "")).split(";"), "This != ''");
        sPersonFullnameString = "";
        if (!bJustSave) {
            for (oOldRequest in aOldRequests) {
                if (ArrayOptFind(aNewIDs, "Int(This) == Int(oOldRequest.person_id)") == undefined) {
                    sPersonFullnameString += "<li>" + oOldRequest.person_fullname + "</li>";
                }
            }
            bJustSave = (sPersonFullnameString == "");
        }       
 
        if (bJustSave){
            if (oOptions.id > 0) {
                oDoc = tools.open_doc(oOptions.id);
            }
            else {
                oDoc = tools.new_doc_by_name("education_method");
                oDoc.BindToDb();
            }
            oDocTE = oDoc.TopElem;
            oDocTE.name = getFormField("name", "");
            oDocTE.cost = getFormField("group_cost", "");
            oDocTE.cost_type = 'group';
            oDocTE.duration = getFormField("hour", "");
            oDocTE.duration_days = getFormField("day", "");
            oDocTE.person_num = getFormField("max_persons", "");
            oDocTE.desc = getFormField("desc", "");
            oDocTE.custom_elems.ObtainChildByKey("f_vtbl_place").value = getFormField("place", "");
            oDocTE.custom_elems.ObtainChildByKey("f_vtbl_type").value = getFormField("event_type", "");
            oDocTE.custom_elems.ObtainChildByKey("f_vtbl_collaborators").value = getFormField("select_collaborator_ids", "");
            oDocTE.custom_elems.ObtainChildByKey("f_vtbl_ed_org").value = getFormField("ed_org", "");
            oDocTE.custom_elems.ObtainChildByKey("f_vtbl_main_collaborator").value = getFormField("main_collaborator_id", "");
            oDocTE.custom_elems.ObtainChildByKey("f_vtbl_auto_accept_manager_days_count").value = getFormField("auto_accept_manager_days_count", "0");
            oDocTE.custom_elems.ObtainChildByKey("f_vtbl_auto_accept_mir_code_manager_days_count").value = getFormField("auto_accept_mir_code_manager_days_count", "0");
           
            
            aDates = [];
            for (i = 1; i <= 5; i++){
                sStartDate = getFormField("date_" + i + "_start", "");
                sFinishDate = getFormField("date_" + i + "_finish");
                sStartHour = getFormField("date_" + i + "_hour_start");
                sFinishHour = getFormField("date_" + i + "_hour_finish");
                sEventType = getFormField("date_" + i + "_event_type");
                if (sStartDate != "" && sFinishDate != "" && sStartHour != "" && sFinishHour != "" && sEventType != ""){
                    aDates.push({
                        "start_date": sStartDate,
                        "finish_date": sFinishDate,
                        "start_hour": sStartHour,
                        "finish_hour": sFinishHour,
                        "event_type": sEventType
                    })
                }
            }
            oDocTE.custom_elems.ObtainChildByKey("f_vtbl_dates").value = tools.object_to_text(aDates, "json");
           
 
            oDocTE.role_id.ObtainByValue( 7410319811624320516 ) ;
            oDoc.Save();
            activeRequests = XQuery("for $elem in requests where $elem/object_id=" + oDoc.DocID + " and $elem/status_id != 'archive' return $elem");
            if (ArrayCount(activeRequests) > 0) {
                aDates = ParseJson(oDocTE.custom_elems.ObtainChildByKey("f_vtbl_dates").value);
               for (oDate in aDates){
                    dTempStartDate = Date(StrDate(Date(oDate.start_date), false) + " " + oDate.start_hour + ":00");
                    dTempFinishDate = Date(StrDate(Date(oDate.finish_date), false) + " " + oDate.finish_hour + ":00");
                    oQueryEvent = ArrayOptFirstElem(XQuery(alert("for $elem in events where $elem/type_id = 'education_method' and $elem/education_method_id = " + XQueryLiteral(oDoc.DocID) + " and $elem/start_date = " + XQueryLiteral(dTempStartDate) + " and $elem/finish_date=" + XQueryLiteral(dTempFinishDate) + " return $elem")));
                    if (oQueryEvent != undefined) {
                        oEventDoc = tools.open_doc(oQueryEvent.id);
                    }
                    else {
                        oEventDoc = OpenNewDoc( 'x-local://wtv/wtv_event.xmd' );
                        oEventDoc.BindToDb(DefaultDb);
                    }
                    oEventDocTE = oEventDoc.TopElem;
                    tools.set_event_type_id( oEventDocTE, 'education_method' );
                    oEventDocTE.education_method_id = oDoc.DocID;
                    oEventDocTE.name = oDoc.TopElem.name;
                    tools.common_filling( 'education_method', oEventDocTE, oDoc.DocID, oDoc.TopElem );
                    oEventDocTE.start_date = dTempStartDate;
                    oEventDocTE.finish_date = dTempFinishDate;
                    //oEventDocTE.date_request_rejection_over = DateOffset(oEventDocTE.start_date, (2*(0-86400)));
                    oEventDocTE.status_id = 'plan';
                    oEventDocTE.is_open = true;
                    oEventDocTE.is_public = false;
                    oEventDocTE.even_preparations.Clear();
                    if (OptInt(oDoc.TopElem.custom_elems.ObtainChildByKey("f_vtbl_main_collaborator").value, 0) > 0) {
                        oPrep = oEventDocTE.even_preparations.AddChild();
                        oPrep.person_id = Int(oDoc.TopElem.custom_elems.ObtainChildByKey("f_vtbl_main_collaborator").value);
                        oPrep.status_id = 'plan';
                        tools.common_filling( 'collaborator', oPrep, oPrep.person_id );
                    }
                    oEventDocTE.tutors.Clear();
                    if (OptInt(oDoc.TopElem.custom_elems.ObtainChildByKey("f_vtbl_main_collaborator").value, 0) > 0) {
                        oPrep = oEventDocTE.tutors.AddChild();
                        oPrep.collaborator_id = Int(oDoc.TopElem.custom_elems.ObtainChildByKey("f_vtbl_main_collaborator").value);
                        tools.common_filling( 'collaborator', oPrep, oPrep.collaborator_id );
                    }
                    oEventDocTE.event_settings.send_type = "never";
                    oEventDocTE.default_request_type_id = Int('0x64A415D467FE7413');
                    oEventDocTE.place = oDoc.TopElem.custom_elems.ObtainChildByKey("f_vtbl_place").value;
                    oEventDocTE.education_org_type = "name";
                    oEventDocTE.education_org_name = oDoc.TopElem.custom_elems.ObtainChildByKey("f_vtbl_ed_org").value;
                    oEventDocTE.event_form = "training";
                    oEventDocTE.total_cost = oEventDocTE.cost;
 
                    oEventDocTE.custom_elems.ObtainChildByKey("f_education_format").value = "Очное обучение внешнее";
                    oEventDocTE.custom_elems.ObtainChildByKey("f_education_accept_format").value = oDate.event_type;
                    //oEventDocTE.custom_elems.ObtainChildByKey("f_send_calendar_event").value = true;
                   
                    
                    
                    oEventDocTE.calc_duration_plan();
                   
                    oEventDoc.Save();
                }
 
                for (mainRequest in activeRequests) {
                    mainRequestDoc = tools.open_doc(mainRequest.id);
                    mainRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_dates").value = ArrayMerge(ParseJson(oDoc.TopElem.custom_elems.ObtainChildByKey("f_vtbl_dates").value ), "StrDate(Date(This.start_date), false) + ' - ' + StrDate(Date(This.finish_date), false) + ' с ' + This.start_hour + ' по ' + This.finish_hour + ' (' + This.event_type + ')'", "<br/>");
                    mainRequestDoc.Save();
                }
            }
 
            for (oOldRequest in aOldRequests) {
                if (ArrayOptFind(aNewIDs, "Int(This) == Int(oOldRequest.person_id)") == undefined) {
                    switch(oOldRequest.workflow_state) {
                        case "04":
                            aEvents = XQuery("for $elem in events where $elem/education_method_id = " + oOldRequest.object_id + " return $elem");
                            aSubOldRequests = ArraySelectAll(XQuery("for $elem in requests where $elem/person_id=" + oOldRequest.person_id + " and MatchSome($elem/object_id, (" + ArrayMerge(aEvents, "This.id", ",") + ")) return $elem"));
                            for (oSubOldRequest in aSubOldRequests) {
                                DeleteDoc(UrlFromDocID(oSubOldRequest.id));
                            }
                        case "03":
                            tools.create_notification("accept_budget_collaborator_delete_request", oOldRequest.person_id, '', oOldRequest.id);
                        case "02":
                            oOldRequestDocTE = tools.open_doc(oOldRequest.id).TopElem;
                            tools.create_notification("accept_budget_mircode_manager_delete_request", OptInt(oOldRequestDocTE.custom_elems.ObtainChildByKey("f_mir_code_manager_id").value, 0), '', oOldRequest.id);
                        case "01":
                        case "00":
                            tools.create_notification('accept_budget_manager_delete_request', ArrayOptFind(tools.open_doc(oOldRequest.person_id).TopElem.func_managers, "This.is_native").person_id, '', oOldRequest.id);
                        default:
                            break;
                    }
                    DeleteDoc(UrlFromDocID(oOldRequest.id));
                }
            }
 
            oForm = {
                command: "alert",
                msg: ("Данные мероприятия успешно сохранены."),
                confirm_result: {
                    command: "redirect",
                    url: ("/view_doc.html?mode=education_accept_event&object_id=" + oDoc.DocID)
                }
            }
        }
        else {
            aMandatoryFields = [];
            aVisibleFields = ["start_save_question"];
            for (oField in oForm.form_fields) {
                oField.type = ((ArrayOptFind(aVisibleFields, "This == oField.name") == undefined) ? "hidden" : oField.type );
                try {
                    oField.validation = ((ArrayOptFind(aVisibleFields, "This == oField.name") == undefined) ? "" : oField.validation );
                }
                catch(_e){}
                oField.mandatory = (ArrayOptFind(aMandatoryFields, "This == oField.name") != undefined);
                oField.value = getFormField(oField.name, oField.value);
                if (oField.name == "start_save_question") {
                    oField.value = StrReplace(String(oField.value), "##person_fullnames##", sPersonFullnameString);
                }
            }
            oForm.buttons.push({
                name: "submit",
                submit_type: "add_from_list",
                label: "Вернуться к редактированию списка участников",
                type: "submit"
            },{
                name: "submit",
                submit_type: "save_done",
                label: "Все равно сохранить данные",
                type: "submit"
            })
        }
        break;
    }
    case "start_requests_question":{
        aMandatoryFields = [];
        aVisibleFields = ["edit_mail_question", "manager_mail", "edit_mail_help"];
        for (oField in oForm.form_fields) {
            oField.type = ((ArrayOptFind(aVisibleFields, "This == oField.name") == undefined) ? "hidden" : oField.type );
            try {
                oField.validation = ((ArrayOptFind(aVisibleFields, "This == oField.name") == undefined) ? "" : oField.validation );
            }
            catch(_e){}
            oField.mandatory = (ArrayOptFind(aMandatoryFields, "This == oField.name") != undefined);
            oField.value = getFormField(oField.name, oField.value);
        }
        oForm.buttons.push({
            name: "submit",
            submit_type: "start_requests_question_2",
            label: "Сохранить изменения и перейти к письму для владельца Мир-кода",
            type: "submit"
        },{
            name: "cancel",
            label: "Отменить",
            type: "cancel"
        });
        oForm.title = "Запуск заявок на согласование потребности в обучении"
        break;
    }
    case "start_requests_question_2":{
        aMandatoryFields = [];
        aVisibleFields = ["edit_mail_question", "mir_code_manager_mail", "edit_mail_help"];
        for (oField in oForm.form_fields) {
            oField.type = ((ArrayOptFind(aVisibleFields, "This == oField.name") == undefined) ? "hidden" : oField.type );
            try {
                oField.validation = ((ArrayOptFind(aVisibleFields, "This == oField.name") == undefined) ? "" : oField.validation );
            }
            catch(_e){}
            oField.mandatory = (ArrayOptFind(aMandatoryFields, "This == oField.name") != undefined);
            oField.value = getFormField(oField.name, oField.value);
        }
        oForm.buttons.push({
            name: "submit",
            submit_type: "start_requests_question_3",
            label: "Сохранить изменения и перейти к письму для сотрудника",
            type: "submit"
        },{
            name: "cancel",
            label: "Отменить",
            type: "cancel"
        });
        oForm.title = "Запуск заявок на согласование потребности в обучении"
        break;
    }
    case "start_requests_question_3":{
        aMandatoryFields = [];
        aVisibleFields = ["edit_mail_question", "collaborator_mail", "edit_mail_help"];
        for (oField in oForm.form_fields) {
            oField.type = ((ArrayOptFind(aVisibleFields, "This == oField.name") == undefined) ? "hidden" : oField.type );
            try {
                oField.validation = ((ArrayOptFind(aVisibleFields, "This == oField.name") == undefined) ? "" : oField.validation );
            }
            catch(_e){}
            oField.mandatory = (ArrayOptFind(aMandatoryFields, "This == oField.name") != undefined);
            oField.value = getFormField(oField.name, oField.value);
        }
        oForm.buttons.push({
            name: "submit",
            submit_type: "start_requests_question_4",
            label: "Сохранить изменения и досоздать заявки",
            type: "submit"
        },{
            name: "cancel",
            label: "Отменить",
            type: "cancel"
        });
        oForm.title = "Запуск заявок на согласование потребности в обучении"
        break;
    }
    case "start_requests_question_4":{
        aMandatoryFields = [];
        aVisibleFields = ["start_requests_question"];
        for (oField in oForm.form_fields) {
            oField.type = ((ArrayOptFind(aVisibleFields, "This == oField.name") == undefined) ? "hidden" : oField.type );
            try {
                oField.validation = ((ArrayOptFind(aVisibleFields, "This == oField.name") == undefined) ? "" : oField.validation );
            }
            catch(_e){}
            oField.mandatory = (ArrayOptFind(aMandatoryFields, "This == oField.name") != undefined);
            oField.value = getFormField(oField.name, oField.value);
        }
        oForm.buttons.push({
            name: "submit",
            submit_type: "start_requests",
            label: "Запустить процесс согласования",
            type: "submit"
        },{
            name: "cancel",
            label: "Отменить",
            type: "cancel"
        });
        oForm.title = "Запуск заявок на согласование потребности в обучении"
        break;
    }
    case "start_requests": {
        alert("START-REQUEST");
        if (oOptions.id > 0) {
           
            oDoc = tools.open_doc(oOptions.id);
           
            oDoc.TopElem.custom_elems.ObtainChildByKey("close_for_edit").value = true;
           
            oDoc.TopElem.custom_elems.ObtainChildByKey("f_vtbl_manager_mail").value = getFormField('manager_mail' ,oMailTemplate.manager);
           
            oDoc.TopElem.custom_elems.ObtainChildByKey("f_vtbl_mir_code_manager_mail").value = getFormField('mir_code_manager_mail' ,oMailTemplate.mir_code_manager);
           
            oDoc.TopElem.custom_elems.ObtainChildByKey("f_vtbl_collaborator_mail").value = getFormField('collaborator_mail' ,oMailTemplate.collaborator);
           
            oDoc.Save();
           
            for (oDate in oOptions.dates){
                
                dTempStartDate = Date(StrDate(Date(oDate.start_date), false) + " " + oDate.start_hour + ":00");
 
                dTempFinishDate = Date(StrDate(Date(oDate.finish_date), false) + " " + oDate.finish_hour + ":00");
               
                oQueryEvent = ArrayOptFirstElem(XQuery(alert("for $elem in events where $elem/type_id = 'education_method' and $elem/education_method_id = " + XQueryLiteral(oOptions.id) + " and $elem/start_date = " + XQueryLiteral(dTempStartDate) + " and $elem/finish_date=" + XQueryLiteral(dTempFinishDate) + " return $elem")));
               
                if (oQueryEvent != undefined) {
                    oEventDoc = tools.open_doc(oQueryEvent.id);
                }
                else {
                    oEventDoc = OpenNewDoc( 'x-local://wtv/wtv_event.xmd' );
                    oEventDoc.BindToDb(DefaultDb);
                }
               
                oEventDocTE = oEventDoc.TopElem;
               
                tools.set_event_type_id( oEventDocTE, 'education_method' );
                oEventDocTE.education_method_id = oOptions.id;
                oEventDocTE.name = oDoc.TopElem.name;
               
                tools.common_filling( 'education_method', oEventDocTE, oOptions.id, oDoc.TopElem );
                oEventDocTE.start_date = dTempStartDate;
                oEventDocTE.finish_date = dTempFinishDate;
               
                //oEventDocTE.date_request_rejection_over = DateOffset(oEventDocTE.start_date, (2*(0-86400)));
                oEventDocTE.status_id = 'plan';
                oEventDocTE.is_open = true;
                oEventDocTE.is_public = false;
               
                oEventDocTE.even_preparations.Clear();
                if (OptInt(oDoc.TopElem.custom_elems.ObtainChildByKey("f_vtbl_main_collaborator").value, 0) > 0) {
                    oPrep = oEventDocTE.even_preparations.AddChild();
                    oPrep.person_id = Int(oDoc.TopElem.custom_elems.ObtainChildByKey("f_vtbl_main_collaborator").value);
                    oPrep.status_id = 'plan';
                    tools.common_filling( 'collaborator', oPrep, oPrep.person_id );
                }
               
                oEventDocTE.tutors.Clear();
                if (OptInt(oDoc.TopElem.custom_elems.ObtainChildByKey("f_vtbl_main_collaborator").value, 0) > 0) {
                    oPrep = oEventDocTE.tutors.AddChild();
                    oPrep.collaborator_id = Int(oDoc.TopElem.custom_elems.ObtainChildByKey("f_vtbl_main_collaborator").value);
                    tools.common_filling( 'collaborator', oPrep, oPrep.collaborator_id );
                }
               
                oEventDocTE.event_settings.send_type = "never";
               
                oEventDocTE.default_request_type_id = Int('0x64A415D467FE7413');
              
                oEventDocTE.place = oDoc.TopElem.custom_elems.ObtainChildByKey("f_vtbl_place").value;
               
                oEventDocTE.education_org_type = "name";
               
                oEventDocTE.education_org_name = oDoc.TopElem.custom_elems.ObtainChildByKey("f_vtbl_ed_org").value;
               
                oEventDocTE.event_form = "training";
               
                oEventDocTE.total_cost = oEventDocTE.cost;
               
                oEventDocTE.custom_elems.ObtainChildByKey("f_education_format").value = "Очное обучение внешнее";
                oEventDocTE.custom_elems.ObtainChildByKey("f_education_accept_format").value = oDate.event_type;
                //oEventDocTE.custom_elems.ObtainChildByKey("f_send_calendar_event").value = true;
               
                oEventDocTE.calc_duration_plan();
               
                oEventDoc.Save();
               
            }
        }
        aIDs = ArraySelect(oOptions.select_collaborator_ids.split(";"), "This != ''");
        aOldRequests = ArraySelectAll(XQuery("for $elem in requests where $elem/request_type_id=7528328911430055448 and $elem/object_id=" + XQueryLiteral(oOptions.id) + " and $elem/workflow_state != 'archive' return $elem"));
 
        for (sID in aIDs) {
           alert('Массив ID');
            alert(aIDs);
            alert('Массив ID - конец');
 
            if (ArrayOptFind(aOldRequests, "This.person_id == OptInt(sID, 0)") == undefined) {
                //oDoc = tools.open_doc(oOptions.id);
                oPerson = ArrayOptFirstElem(XQuery("for $elem in collaborators where $elem/id = " + sID + " return $elem"));
                if (oPerson != undefined) {
                    sID = oPerson.id;
               
                    oReqDoc = tools.new_doc_by_name("request");
                    oReqDocTE = oReqDoc.TopElem;
                    oReqDocTE.object_id = oOptions.id;
                    alert('Сырой sID ' + sID);
                    oReqDocTE.person_id = Int(sID);
                    alert('Person ID ' + oReqDocTE.person_id);
                   
                    
                    // oReqDocTE.person_id = Int(6088496371270897047);
                    // alert('Person ID ' + oReqDocTE.person_id);
 
                   
                    tools.common_filling( 'request_type', oReqDocTE, 7528328911430055448 );
                    tools.common_filling( 'collaborator', oReqDocTE, oReqDocTE.person_id );
                    tools.object_filling( 'education_method', oReqDocTE, oReqDocTE.object_id);
                   
                    oReqDocTE.custom_elems.ObtainChildByKey("f_vtbl_manager_mail").value = oDoc.TopElem.custom_elems.ObtainChildByKey("f_vtbl_manager_mail").value;
                   
                    oReqDocTE.custom_elems.ObtainChildByKey("f_vtbl_mir_code_manager_mail").value = oDoc.TopElem.custom_elems.ObtainChildByKey("f_vtbl_mir_code_manager_mail").value;
                   
                    oReqDocTE.custom_elems.ObtainChildByKey("f_vtbl_collaborator_mail").value = oDoc.TopElem.custom_elems.ObtainChildByKey("f_vtbl_collaborator_mail").value;
                   
                    oReqDocTE.custom_elems.ObtainChildByKey("f_vtbl_auto_accept_manager_days_count").value = oDoc.TopElem.custom_elems.ObtainChildByKey("f_vtbl_auto_accept_manager_days_count").value;
                   
                    oReqDocTE.custom_elems.ObtainChildByKey("f_vtbl_auto_accept_mir_code_manager_days_count").value = oDoc.TopElem.custom_elems.ObtainChildByKey("f_vtbl_auto_accept_mir_code_manager_days_count").value;
                   
                    oReqDocTE.custom_elems.ObtainChildByKey("f_vtbl_desc").value = oDoc.TopElem.desc;
                   
                    oReqDoc.BindToDb(DefaultDb);
               
                    alert(oReqDoc.TopElem.Xml);
                    oReqDoc.Save();
                    alert("RAMIl-SAVE");
                }
                else {
                    alert("RAMIL-ERROR: " + sID)
                }
            }
        }
        oForm = {
            command: "alert",
            msg: ("Заявки созданы."),
            confirm_result: {
                command: "reload_page"
            }
        }
        break;
    }
    case "send_remember_start": {
        aMandatoryFields = [];
        aVisibleFields = ["select_remember_option"];
        for (oField in oForm.form_fields) {
            oField.type = ((ArrayOptFind(aVisibleFields, "This == oField.name") == undefined) ? "hidden" : oField.type );
            try {
                oField.validation = ((ArrayOptFind(aVisibleFields, "This == oField.name") == undefined) ? "" : oField.validation );
            }
            catch(_e){}
            oField.mandatory = (ArrayOptFind(aMandatoryFields, "This == oField.name") != undefined);
            oField.value = getFormField(oField.name, oField.value);
        }
        oForm.buttons.push({
            name: "submit",
            submit_type: "send_remember_edit_mail",
            label: "Далее",
            type: "submit"
        },{
            name: "cancel",
            label: "Отменить",
            type: "cancel"
        });
        break;
    }
    case "send_remember_edit_mail": {
        if (getFormField("select_remember_option", "0") == "0") {
            aMandatoryFields = [];
            aVisibleFields = ["manager_mail_remember", "edit_mail_help"];
            oForm.buttons.push({
                name: "submit",
                submit_type: "send_remember_manager_start",
                label: "Сохранить и отправить напоминания руководителям",
                type: "submit"
            })
        }
        else {
            aMandatoryFields = [];
            aVisibleFields = ["mir_code_manager_mail_remember", "edit_mail_help"];
            oForm.buttons.push({
                name: "submit",
                submit_type: "send_remember_mir_code_manager_start",
                label: "Сохранить и отправить напоминания владельцам Мир-кодов",
                type: "submit"
            })
        }     
        for (oField in oForm.form_fields) {
            oField.type = ((ArrayOptFind(aVisibleFields, "This == oField.name") == undefined) ? "hidden" : oField.type );
            try {
                oField.validation = ((ArrayOptFind(aVisibleFields, "This == oField.name") == undefined) ? "" : oField.validation );
            }
            catch(_e){}
            oField.mandatory = (ArrayOptFind(aMandatoryFields, "This == oField.name") != undefined);
            oField.value = getFormField(oField.name, oField.value);
        }
        oForm.buttons.push({
            name: "cancel",
            label: "Отменить",
            type: "cancel"
        });
        break;
    }
    case "send_remember_manager_start": {
        oDoc = tools.open_doc(oOptions.id);
        oDoc.TopElem.custom_elems.ObtainChildByKey("f_vtbl_manager_mail_remember").value = getFormField('manager_mail_remember' ,oMailTemplate.manager_remember);
        oDoc.Save();
        aReqArr = XQuery("for $elem in requests where $elem/object_id=" + oOptions.id + " and $elem/workflow_state='01' return $elem");
        for (oReq in aReqArr) {
            try {
                oReqDoc = tools.open_doc(oReq.id);
                oReqDoc.TopElem.custom_elems.ObtainChildByKey("f_vtbl_manager_mail_remember").value = oDoc.TopElem.custom_elems.ObtainChildByKey("f_vtbl_manager_mail_remember").value;
                oReqDoc.Save();
                iManagerId = OptInt(oReqDoc.TopElem.custom_elems.ObtainChildByKey("f_manager_id").value, 0);
                managerDocTE = tools.open_doc(iManagerId).TopElem;
                if (managerDocTE.position_name != 'Генеральный директор' || managerDocTE.org_name != 'ВТБ Лизинг') {
                    tools.create_notification("accept_budget_manager_accept_remember", oReqDoc.TopElem.custom_elems.ObtainChildByKey("f_manager_id").value, '', oReq.id);
                }
            }
            catch(_ex){
                alert(_ex)
            }
        }
        oForm = {
                                            command: "alert",
                                            msg: ("Напоминания отправлены."),
                                            confirm_result: {
                                                           command: "reload_page"
                                            }
                              }
         break;
    }
    case "send_remember_mir_code_manager_start": {
        oDoc = tools.open_doc(oOptions.id);
        oDoc.TopElem.custom_elems.ObtainChildByKey("f_vtbl_mir_code_manager_mail_remember").value = getFormField('mir_code_manager_mail_remember' ,oMailTemplate.mir_code_manager_remember);
        oDoc.Save();
        aReqArr = XQuery("for $elem in requests where $elem/object_id=" + oOptions.id + " and $elem/workflow_state='02' return $elem");
        for (oReq in aReqArr) {
            try {
                oReqDoc = tools.open_doc(oReq.id);
                oReqDoc.TopElem.custom_elems.ObtainChildByKey("f_vtbl_mir_code_manager_mail_remember").value = oDoc.TopElem.custom_elems.ObtainChildByKey("f_vtbl_mir_code_manager_mail_remember").value;
                oReqDoc.Save();
                iManagerId = OptInt(oReqDoc.TopElem.custom_elems.ObtainChildByKey("f_manager_id").value, 0);
                managerDocTE = tools.open_doc(iManagerId).TopElem;
                if (managerDocTE.position_name != 'Генеральный директор' || managerDocTE.org_name != 'ВТБ Лизинг') {
                    tools.create_notification("accept_budget_mircode_manager_accept_remember", oReqDoc.TopElem.custom_elems.ObtainChildByKey("f_mir_code_manager_id").value, '', oReq.id);
                }
            }
            catch(_ex){
                alert(_ex)
            }
        }
        oForm = {
                                            command: "alert",
                                            msg: ("Напоминания отправлены."),
                                            confirm_result: {
                                                           command: "reload_page"
                                            }
                              }
        break;
    }
    case "archive_question": {
        aMandatoryFields = [];
        aVisibleFields = ["archive_question"];
        for (oField in oForm.form_fields) {
            oField.type = ((ArrayOptFind(aVisibleFields, "This == oField.name") == undefined) ? "hidden" : oField.type );
            try {
                oField.validation = ((ArrayOptFind(aVisibleFields, "This == oField.name") == undefined) ? "" : oField.validation );
            }
            catch(_e){}
            oField.mandatory = (ArrayOptFind(aMandatoryFields, "This == oField.name") != undefined);
            oField.value = getFormField(oField.name, oField.value);
        }
        oForm.buttons.push({
            name: "submit",
            submit_type: "archive",
            label: "Запустить процесс переноса в архив",
            type: "submit"
        },{
            name: "cancel",
            label: "Отменить",
            type: "cancel"
        });
        oForm.title = "Перенос данных в архив"
        break;
    }
    case "archive": {
        oDoc = tools.open_doc(oOptions.id);
        oDoc.TopElem.custom_elems.ObtainChildByKey("f_vtbl_collaborators").value = "";
        dateArray = [];
        archiveDates = oDoc.TopElem.custom_elems.ObtainChildByKey("f_vtbl_archive_dates").value;
        if (archiveDates != "") {
            archiveDates = ParseJson(archiveDates);
            for (elem in archiveDates) {
                dateArray.push(elem)
            }
        }
 
        oldDates =  oDoc.TopElem.custom_elems.ObtainChildByKey("f_vtbl_dates").value;
        if (oldDates != "") {
            oldDates = ParseJson(oldDates);
            for (elem in oldDates) {
                dateArray.push(elem)
           }
        }
        oDoc.TopElem.custom_elems.ObtainChildByKey("f_vtbl_archive_dates").value = tools.object_to_text(dateArray, 'json');
        oDoc.TopElem.custom_elems.ObtainChildByKey("f_vtbl_dates").value = "[]";
 
        oDoc.Save();
        aReqs = XQuery("for $elem in requests where $elem/request_type_id=7528328911430055448 and $elem/object_id=" + XQueryLiteral(oOptions.id) + " and $elem/workflow_state != 'archive' return $elem");
        for (oReq in aReqs) {
            oReqDoc = tools.open_doc(oReq.id);
            oReqDocTE = oReqDoc.TopElem;
 
            oLogChild = oReqDocTE.workflow_log_entrys.AddChild();
            oLogChild.create_date = Date();
            oLogChild.person_id = curUserID;
            oLogChild.person_fullname = curUser.fullname;
            oLogChild.begin_state = String(oReqDocTE.workflow_state);
            oLogChild.finish_state = 'archive';
            oLogChild.submited = 0;
           
            oReqDocTE.workflow_state = "archive";
            oReqDocTE.workflow_state_name = "Архив";
 
            oReqDoc.Save();
        }
        aEvents = XQuery("for $elem in events where $elem/education_method_id = " + oOptions.id + " return $elem");
        for (oEvent in aEvents) {
            oEventDoc = tools.open_doc(oEvent.id);
            if (oEventDoc.TopElem.status_id != 'close') {
                oEventDoc.TopElem.status_id = 'cancel';
                oEventDoc.Save();
            }
            oReqs = XQuery("for $elem in requests where $elem/request_type_id=7251945301995320339 and $elem/object_id=" + XQueryLiteral(oEvent.id) + " and $elem/workflow_state != 'archive' return $elem");
            for (oReq in oReqs) {
                oReqDoc = tools.open_doc(oReq.id);
                oReqDocTE = oReqDoc.TopElem;
 
                oLogChild = oReqDocTE.workflow_log_entrys.AddChild();
                oLogChild.create_date = Date();
                oLogChild.person_id = curUserID;
                oLogChild.person_fullname = curUser.fullname;
                oLogChild.begin_state = String(oReqDocTE.workflow_state);
                oLogChild.finish_state = 'archive';
                oLogChild.submited = 0;
               
                oReqDocTE.workflow_state = "archive";
                oReqDocTE.workflow_state_name = "Архив";
 
                oReqDoc.Save();
            }
        }
        oForm = {
                                            command: "alert",
                                            msg: ("Все старые данные переведены в архив"),
                                            confirm_result: {
                                                           command: "reload_page"
                                            }
                              }
        break;
       
    }
    case "delete_dates": {
        aMandatoryFields = [];
        aVisibleFields = ["delete_dates"];
        for (oField in oForm.form_fields) {
            oField.type = ((ArrayOptFind(aVisibleFields, "This == oField.name") == undefined) ? "hidden" : oField.type );
            try {
                oField.validation = ((ArrayOptFind(aVisibleFields, "This == oField.name") == undefined) ? "" : oField.validation );
            }
            catch(_e){}
            oField.mandatory = (ArrayOptFind(aMandatoryFields, "This == oField.name") != undefined);
            oField.value = getFormField(oField.name, oField.value);
        }
        oForm.buttons.push({
            name: "submit",
            submit_type: "delete_dates_action",
            label: "Удалить",
            type: "submit"
        },{
            name: "cancel",
            label: "Отменить",
            type: "cancel"
        });
        oForm.title = "Удалить мероприятие"
        oForm.msg = "Выберите дат мероприятия, которые хотите удалить. После нажатия кнопки \"Удалить\" отменить это действие будет невозможно"
        break;
    }
    case "delete_dates_action": {
        oDoc = tools.open_doc(oOptions.id);
        aDates = [];
        aDates2 = tools.read_object(oDoc.TopElem.custom_elems.ObtainChildByKey("f_vtbl_dates").value);
        for (i = 0 ; i < ArrayCount(aDates2); i++){
            if (i != Int(getFormField("delete_dates", ""))) {
                aDates.push(aDates2[i])
            }
            else {
                dTempStartDate = Date(StrDate(Date(aDates2[i].start_date), false) + " " + aDates2[i].start_hour + ":00");
                dTempFinishDate = Date(StrDate(Date(aDates2[i].finish_date), false) + " " + aDates2[i].finish_hour + ":00");
                mainRequestArray = XQuery("for $elem in requests where $elem/object_id=" + oOptions.id + " and $elem/status_id != 'archive' return $elem");
                alert("for $elem in requests where $elem/object_id=" + oOptions.id + " and $elem/status_id != 'archive' return $elem")
                for (mainRequest in mainRequestArray) {
                    mainRequestDoc = tools.open_doc(mainRequest.id);
                    mainColID = Int(mainRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_main_collaborator").value);
                    mainColl = tools.open_doc(mainColID).TopElem;
                    aPeoples = [];                  
                    aPeoples.push(OptInt(mainRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_manager_id").value, 0));
                    aPeoples.push(OptInt(mainRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_mir_code_manager_id").value,0));
                    aPeoples = ArraySelectDistinct(aPeoples, "This");
 
                    switch(mainRequestDoc.TopElem.workflow_state) {
                        case "01": {
                            if (ArrayCount(aDates2) == 1) {
                                mainRequestDoc.TopElem.status_id = 'ignore';
                                mainRequestDoc.TopElem.workflow_state = 'archive';
                                mainRequestDoc.TopElem.workflow_state_name = 'Архив';
                                mainRequestDoc.Save();
                                //alert(1)
                                tools.create_notification('accept_budget_cancel_event_from_admin', mainRequestDoc.TopElem.person_id, ("Группа по тренингу \"" + mainRequestDoc.TopElem.object_id.ForeignElem.name + "\", запланированная в даты " + StrDate(dTempStartDate, true, false) + "-" + StrDate(dTempFinishDate, true, false) + " отменена.<br/>В случае вопросов, пожалуйста, обратитесь к куратору мероприятия " + mainColl.email + "<br/>"), mainRequestDoc.TopElem.object_id);
                                //alert(2)
                                tools.create_notification('accept_budget_cancel_event_from_admin', Int(mainRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_manager_id").value), ("В данный момент у вас на согласовании находится завявка вашего сотрудника " + mainRequestDoc.TopElem.person_id.ForeignElem.fullname + " на учебное мероприятие по теме «" + mainRequestDoc.TopElem.object_id.ForeignElem.name + "», даты обучения – " + StrDate(dTempStartDate, true, false) + "-" + StrDate(dTempFinishDate, true, false) + "<br/>.Группа, запланированная в эти даты, отменена.<br/>"),  mainRequestDoc.TopElem.object_id);
                                //alert(3)
                            }
                            break;
                        }
                        case "02": {
                            if (ArrayCount(aDates2) == 1) {
                                mainRequestDoc.TopElem.status_id = 'ignore';
                                mainRequestDoc.TopElem.workflow_state = 'archive';
                                mainRequestDoc.TopElem.workflow_state_name = 'Архив';
                                mainRequestDoc.Save();
 
                                tools.create_notification('accept_budget_cancel_event_from_admin', mainRequestDoc.TopElem.person_id, ("Группа по тренингу \"" + mainRequestDoc.TopElem.object_id.ForeignElem.name + "\", запланированная в даты " + StrDate(dTempStartDate, true, false) + "-" + StrDate(dTempFinishDate, true, false) + " отменена.<br/>В случае вопросов, пожалуйста, обратитесь к куратору мероприятия " + mainColl.email + "<br/>"), mainRequestDoc.TopElem.object_id);
 
                                tools.create_notification('accept_budget_cancel_event_from_admin', Int(mainRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_manager_id").value), ("Ранее вы согласовывали вашему сотруднику " + mainRequestDoc.TopElem.person_id.ForeignElem.fullname + " учебное мероприятие по теме «" + mainRequestDoc.TopElem.object_id.ForeignElem.name + "», даты обучения – " + StrDate(dTempStartDate, true, false) + "-" + StrDate(dTempFinishDate, true, false) + "<br/>.Группа, запланированная в эти даты, отменена.<br/>"), mainRequestDoc.TopElem.object_id);
                                    
                                tools.create_notification('accept_budget_cancel_event_from_admin', Int(mainRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_mir_code_manager_id").value), ("В данный момент у вас на согласовании находится завявка вашего сотрудника " + mainRequestDoc.TopElem.person_id.ForeignElem.fullname + " на учебное мероприятие по теме «" + mainRequestDoc.TopElem.object_id.ForeignElem.name + "», даты обучения – " + StrDate(dTempStartDate, true, false) + "-" + StrDate(dTempFinishDate, true, false) + "<br/>.Группа, запланированная в эти даты, отменена.<br/>"), mainRequestDoc.TopElem.object_id);
                               
                            }
                            break;
                        }
                        case "03": {
                            if (ArrayCount(aDates2) == 1) {
                                mainRequestDoc.TopElem.status_id = 'ignore';
                                mainRequestDoc.TopElem.workflow_state = 'archive';
                                mainRequestDoc.TopElem.workflow_state_name = 'Архив';
                                mainRequestDoc.Save();
                                tools.create_notification('accept_budget_cancel_event_from_admin', mainRequestDoc.TopElem.person_id, ("Группа по тренингу \"" + mainRequestDoc.TopElem.object_id.ForeignElem.name + "\", запланированная в даты " + StrDate(dTempStartDate, true, false) + "-" + StrDate(dTempFinishDate, true, false) + " отменена.<br/>В случае вопросов, пожалуйста, обратитесь к куратору мероприятия " + mainColl.email + "<br/>"), mainRequestDoc.TopElem.object_id);
                                for (oPeopleID in aPeoples) {
                                    tools.create_notification('accept_budget_cancel_event_from_admin', oPeopleID, ("Ранее вы согласовывали вашему сотруднику " + mainRequestDoc.TopElem.person_id.ForeignElem.fullname + " учебное мероприятие по теме «" + mainRequestDoc.TopElem.object_id.ForeignElem.name + "», даты обучения – " + StrDate(dTempStartDate, true, false) + "-" + StrDate(dTempFinishDate, true, false) + "<br/>.Группа, запланированная в эти даты, отменена.<br/>"), mainRequestDoc.TopElem.object_id);
                                }
                            }
                            break;
                        }
                        case "04": {
                            if (ArrayCount(aDates2) == 1) {
                                mainRequestDoc.TopElem.status_id = 'ignore';
                                mainRequestDoc.TopElem.workflow_state = 'archive';
                                mainRequestDoc.TopElem.workflow_state_name = 'Архив';
                                mainRequestDoc.Save();
                                tools.create_notification('accept_budget_cancel_event_from_admin', mainRequestDoc.TopElem.person_id, ("Группа по тренингу \"" + mainRequestDoc.TopElem.object_id.ForeignElem.name + "\", запланированная в даты " + StrDate(dTempStartDate, true, false) + "-" + StrDate(dTempFinishDate, true, false) + " отменена.<br/>В случае вопросов, пожалуйста, обратитесь к куратору мероприятия " + mainColl.email + "<br/>"), mainRequestDoc.TopElem.object_id);
                                for (oPeopleID in aPeoples) {
                                    tools.create_notification('accept_budget_cancel_event_from_admin', oPeopleID, ("Ранее Ваш сотрудник " + mainRequestDoc.TopElem.person_id.ForeignElem.fullname + " зарегистрировался на учебное мероприятие по теме «" + mainRequestDoc.TopElem.object_id.ForeignElem.name + "», даты обучения – " + StrDate(dTempStartDate, true, false) + "-" + StrDate(dTempFinishDate, true, false) + "<br/>.Группа, запланированная в эти даты, отменена.<br/>"), mainRequestDoc.TopElem.object_id);
                                }
                            }
                            else {
                                mainRequestDoc.TopElem.status_id = 'active';
                                mainRequestDoc.TopElem.workflow_state = '03';
                                mainRequestDoc.TopElem.workflow_state_name = 'Выбор сотрудником даты мероприятия';
                                mainRequestDoc.Save();
                                tools.create_notification('accept_budget_cancel_event_from_admin', mainRequestDoc.TopElem.person_id, ("Группа по тренингу \"" + mainRequestDoc.TopElem.object_id.ForeignElem.name + "\", запланированная в даты " + StrDate(dTempStartDate, true, false) + "-" + StrDate(dTempFinishDate, true, false) + " отменена.<br/>.Предлагаем Вам записаться на другую дату, для этого пройдите по ссылке <a href=\"https://als-wt/_wt/" + mainRequestDoc.TopElem.object_id + "\">https://als-wt/_wt/" + mainRequestDoc.TopElem.object_id + "</a><br/>Если участие не актуально перейдите по ссылке https://als-wt/_wt/" + mainRequest.id + " и нажмите кнопку \"Отклонить\"<br/>"), mainRequestDoc.TopElem.object_id);
                                for (oPeopleID in aPeoples) {
                                    tools.create_notification('accept_budget_cancel_event_from_admin', oPeopleID, ("Ранее Ваш сотрудник " + mainRequestDoc.TopElem.person_id.ForeignElem.fullname + " зарегистрировался на учебное мероприятие по теме «" + mainRequestDoc.TopElem.object_id.ForeignElem.name + "», даты обучения – " + StrDate(dTempStartDate, true, false) + "-" + StrDate(dTempFinishDate, true, false) + "<br/>.Группа, запланированная в эти даты, отменена. Сотруднику направлено предложение выбрать новые даты или отказаться от обучения.<br/>"), mainRequestDoc.TopElem.object_id);
                                }
                            }
                            break;
                        }
                    }                
                    
                }              
                oQueryEvent = ArrayOptFirstElem(XQuery(alert("for $elem in events where $elem/type_id = 'education_method' and $elem/education_method_id = " + XQueryLiteral(oOptions.id) + " and $elem/start_date = " + XQueryLiteral(dTempStartDate) + " and $elem/finish_date=" + XQueryLiteral(dTempFinishDate) + " return $elem")));
                if(oQueryEvent != undefined) {
                    aRequests = ArraySelectAll(XQuery("for $elem in requests where $elem/object_id = " + oQueryEvent.id + " return $elem"));
                    for (oRequest in aRequests) {
                        DeleteDoc(UrlFromDocID(oRequest.id));
                    }
                    DeleteDoc(UrlFromDocID(oQueryEvent.id));
                }
            }
        }
       
        oDoc.TopElem.custom_elems.ObtainChildByKey("f_vtbl_dates").value = tools.object_to_text(aDates, "json");   
        oDoc.Save();
        mainRequestArray = XQuery("for $elem in requests where $elem/object_id=" + oOptions.id + " and $elem/status_id != 'archive' return $elem");
        for (mainRequest in mainRequestArray) {
            mainRequestDoc = tools.open_doc(mainRequest.id);
            mainRequestDoc.TopElem.custom_elems.ObtainChildByKey("f_dates").value = ArrayMerge(ParseJson(oDoc.TopElem.custom_elems.ObtainChildByKey("f_vtbl_dates").value ), "StrDate(Date(This.start_date), false) + ' - ' + StrDate(Date(This.finish_date), false) + ' с ' + This.start_hour + ' по ' + This.finish_hour + ' (' + This.event_type + ')'", "<br/>");
            mainRequestDoc.Save();
        }
        oForm = {
                                            command: "alert",
                                            msg: ("Мероприятие удалено"),
                                            confirm_result: {
                                                           command: "reload_page"
                                            }
                              }
        break;
    }
    default: {
        break;
    }
 
   
}
RESULT = oForm;
/*oForm = {
    command: "display_form",
    height: 320,
    title: mainTitle,
    message: "Выберите действие, которое хотите выполнить",
    form_fields: [{
        name: "step_next",
        type: "radio",
        label: "Выбор опции делегирования",
        mandatory: true,
        entries: [{
            value: "step_01",
            name: "Делегировать полностью роль руководителя (или переназначить)"
        },{
            value: "step_02",
            name: "Удалить имеющиеся делегирования"
        }]
    }],
   buttons: [{
        name: "cancel",
        label: ms_tools.get_const('c_cancel'),
        type: "cancel", css_class: "btn-cancel-custom"
    },{
        name: "submit",
        label: "Далее",
        type: "submit",
        css_class: "btn-submit-custom"
    }],
    no_buttons: false
};*/
 
