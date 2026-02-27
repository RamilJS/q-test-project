

aFormFields = ParseJson( getParam("form_fields", "[]") );
sSubmitType = getFormField("__submit_type__", "");

// Данные текущего пользователя (тим-лид)
var oOptions = new Object();
oOptions.id = curUserID;
oOptions.name = "Тим-лид";

iTeamLeadID = curUserID;
teamLeadDoc = tools.open_doc(iTeamLeadID);
teamLeadDocTE = teamLeadDoc.TopElem;
teamLeadName = teamLeadDocTE.fullname;
teamLeadPosition = teamLeadDocTE.position_name;
teamLeadPositionParentName = teamLeadDocTE.position_parent_name;

// ---- Логика показа поля yearResult ----
var finalDateStr = getFormField("finalDate", "");
var currentYear = Year(Date());
var showYearResult = false;

if (finalDateStr != undefined && finalDateStr != null && finalDateStr != "") {
    var finalYearStr = String(finalDateStr).substr(0,4);
    var finalYear = Int(finalYearStr);
    // показываем только если год больше текущего (дата в будущем)
    if (finalYear > currentYear) {
        showYearResult = true;
    }
}

// ---- Построение массива полей формы ----
var formFields = [
    {
        name: "teamLeadName",
        label: "Тим-лид:",
        type: "string",
        value: teamLeadName,
        mandatory: false,
        multiple: false,
        visibility: true,
        page: "page_one"
    },
    {
        name: "teamLeadPosition",
        label: "Должность Тим-лида",
        type: "string",
        value: "(" + teamLeadPosition + ", " + teamLeadPositionParentName + ")",
        mandatory: false,
        multiple: false,
        visibility: true,
        page: "page_one"
    },
    {
        name: "subprojectName",
        label: "Формулировка/Название подпроекта:",
        type: "text",
        value: '',
        mandatory: true,
        multiple: false,
        visibility: true,
        page: "page_one"
    },
    {
        name: "clientName",
        label: "Заказчик",
        title: "Выбрать заказчика",
        type: "foreign_elem",
        value: '',
        mandatory: true,
        multiple: false,
        catalog: "collaborator",
        query_qual: "",
        page: "page_one"
    },
    {
        name: "sponsorName",
        label: "Спонсоры",
        title: "Выбрать спонсоров",
        type: "foreign_elem",
        value: '',
        mandatory: true,
        multiple: true,
        catalog: "collaborator",
        query_qual: "",
        page: "page_one"
    },
    {
        name: "strategicInitiative",
        label: "Стратегическая инициатива",
        type: "foreign_elem",
        value: '',
        mandatory: true,
        multiple: false,
        catalog: "task",
        query_qual: "$elem/task_type_id=6946456410735586707",
        visibility: true,
        page: "page_one"
    },
    {
        name: "project",
        label: "Проект",
        type: "foreign_elem",
        value: '',
        mandatory: true,
        multiple: false,
        catalog: "task",
        query_qual: ("$elem/task_type_id=6946456476236259360 and $elem/parent_task_id={{param}}"),
        visibility: [
            { parent: "strategicInitiative", value: "notnull" }
        ],
        page: "page_one"
    },
    {
        name: "subprojectLevel",
        label: "Уровень подпроекта",
        type: "select",
        value: '',
        validation: "noneempty",
        error_message: "Выберите уровень подпроекта",
        entries: [
            { name: "Пусто", value: "Пусто" },
            { name: "Операционное улучшение", value: "Операционное улучшение" },
            { name: "Страт инициатива", value: "Страт инициатива" },
            { name: "Регуляторный / групповой проект", value: "Регуляторный / групповой проект" },
            { name: "Организационные изменения", value: "Организационные изменения" },
            { name: "Back-log", value: "Back-log" }
        ],
        mandatory: true,
        page: "page_one"
    },
    {
        name: "finalDate",
        label: "Финальная дата завершения",
        type: "date",
        value: '',
        mandatory: true,
        multiple: false,
        query_qual: "",
        page: "page_one"
    },
    {
        name: "result",
        label: "Ожидаемый результат на финальную дату завершения (критерии успешности)",
        type: "text",
        richtext: true,
        value: '',
        mandatory: false,
        page: "page_one"
    }
];

// Добавляем yearResult только если дата в будущем
if (showYearResult) {
    formFields.push({
        name: "yearResult",
        label: "Ожидаемый результат на конец текущего года (критерии успешности)",
        type: "text",
        richtext: true,
        value: getFormField("yearResult", ""),
        mandatory: true,  // обязательно, т.к. поле видно
        page: "page_one"
    });
}

// Остальные поля (добавляются всегда)
formFields.push(
    {
        name: "checkboxCHR",
        label: "CHR на доработку",
        type: "bool",
        value: "",
        mandatory: false,
        page: "page_one"
    },
    {
        name: "numberCHR",
        label: "Номера CHR (через точку с запятой)",
        type: "string",
        richtext: true,
        value: getFormField("numberCHR",""),
        mandatory: true,  // обязательно, но проверка будет условной при сохранении
        visibility: [
            { parent: "checkboxCHR", value: true }
        ],
        page: "page_one"
    },
    {
        name: "subprojectValue",
        label: "Вес подпроекта (не менее 5%)",
        type: "integer",
        column: 1,
        value: 5,
        mandatory: true,
        page: "page_one"
    }
);

// Собираем объект формы
oForm = {
    command: "display_form",
    height: 320,
    title: 'Формирование подпроекта',
    message: null,
    form_fields: formFields,
    pages: [{ name: "page_one", buttons: [] }],
    is_one_page: true,
    buttons: [
        { name: "cancel", label: "Отменить", type: "cancel" },
        { name: "submit", submit_type: "save", label: "Создать подпроект", type: "submit" }
    ],
    no_buttons: false
};

// Обработка сохранения
switch(sSubmitType){
    case "save": {
        var aErrorArray = [];

        // ---- Дополнительная проверка на отставание даты ----
        var finalDateVal = getFormField("finalDate", "");
        if (finalDateVal) {
            var finalYear = Int(String(finalDateVal).substr(0,4));
            if (finalYear < currentYear) {
                aErrorArray.push("Финальная дата завершения указана некорректно (год не может быть меньше текущего).");
            }
        }

        // ---- Базовые проверки обязательных полей ----
        if (getFormField("teamLeadName", "") == "")
            aErrorArray.push("Заполните ФИО Тим-лида.");
        if (getFormField("subprojectName", "") == "")
            aErrorArray.push("Заполните формулировку/название подпроекта.");
        if (getFormField("clientName", "") == "")
            aErrorArray.push("Выберите заказчика.");
        if (getFormField("sponsorName", "") == "")
            aErrorArray.push("Выберите спонсоров.");
        if (getFormField("strategicInitiative", "") == "")
            aErrorArray.push("Выберите стратегическую инициативу.");
        if (getFormField("project", "") == "")
            aErrorArray.push("Выберите проект.");
        if (getFormField("subprojectLevel", "") == "")
            aErrorArray.push("Выберите уровень подпроекта.");
        if (getFormField("finalDate", "") == "")
            aErrorArray.push("Выберите финальную дату завершения.");
        if (getFormField("result", "") == "" && getFormField("yearResult", "") == "")
            aErrorArray.push("Заполните ожидаемый результат (хотя бы одно из полей).");
        if (getFormField("subprojectValue", "") == "")
            aErrorArray.push("Укажите вес подпроекта.");

        // ---- Условная проверка numberCHR (видно только при отмеченном чекбоксе) ----
        var isCHRChecked = getFormField("checkboxCHR", "0");
        if ( (isCHRChecked == "1" || isCHRChecked == "true" || isCHRChecked == true) && getFormField("numberCHR", "") == "" ) {
            aErrorArray.push("Заполните номера CHR.");
        }

        // ---- Условная проверка yearResult (если поле было показано, т.е. дата в будущем) ----
        if (showYearResult && getFormField("yearResult", "") == "") {
            aErrorArray.push("Заполните ожидаемый результат на конец текущего года.");
        }

        if (ArrayCount(aErrorArray) > 0) {
            RESULT = {
                command: "alert",
                msg: ("Пожалуйста, устраните ошибки:<ul><li>" + ArrayMerge(aErrorArray, "This", "</li><li>") + "</li></ul>"),
                view: "warning",
                title: "Ошибки заполнения"
            };
            break;
        }

        // ---- Создание заявки ----
        var oDoc = tools.open_doc(oOptions.id);
        var oReqDoc = tools.new_doc_by_name("request");
        var oReqDocTE = oReqDoc.TopElem;
        oReqDocTE.person_id = oOptions.id;
        oReqDocTE.request_type_id = 6930230432270138388;

        tools.common_filling('request_type', oReqDocTE, oReqDocTE.request_type_id);
        tools.common_filling('collaborator', oReqDocTE, oReqDocTE.person_id);
        var oRequestTypeDocTE = tools.open_doc(oReqDocTE.request_type_id).TopElem;
        var iWorkflowID = oRequestTypeDocTE.workflow_id;

        // Заполнение пользовательских полей
        oReqDocTE.custom_elems.ObtainChildByKey("f_initiative_id").value = getFormField("strategicInitiative", "");
        oReqDocTE.custom_elems.ObtainChildByKey("f_project_id").value = getFormField("project", "");
        oReqDocTE.custom_elems.ObtainChildByKey("f_final_date").value = getFormField("finalDate", "");
        oReqDocTE.custom_elems.ObtainChildByKey("f_final_result").value = getFormField("result", "");
        oReqDocTE.custom_elems.ObtainChildByKey("f_year_result").value = getFormField("yearResult", "");
        oReqDocTE.custom_elems.ObtainChildByKey("f_percent").value = getFormField("subprojectValue", "");
        oReqDocTE.custom_elems.ObtainChildByKey("f_comment").value = getFormField("subprojectName", "");
        oReqDocTE.custom_elems.ObtainChildByKey("f_level_type").value = getFormField("subprojectLevel", "");
        oReqDocTE.custom_elems.ObtainChildByKey("f_is_chr").value = getFormField("checkboxCHR", "");
        oReqDocTE.custom_elems.ObtainChildByKey("f_chrs").value = getFormField("numberCHR", "");

        // Заказчик
        var client = getFormField("clientName", "");
        if (client) {
            var clientDoc = tools.open_doc(client);
            oReqDocTE.object_id = clientDoc.DocID;
            oReqDocTE.object_name = clientDoc.TopElem.fullname;
            oReqDocTE.object_type = "id";
        }

        // Спонсоры
        var sponsorStr = getFormField("sponsorName", "");
        if (sponsorStr != "") {
            var sponsorIds = String(sponsorStr).split(";");
            for (var i = 0; i < ArrayCount(sponsorIds); i++) {

            }
        }

        oReqDoc.BindToDb(DefaultDb);
        oReqDoc.Save();

        var sNewReqID = oReqDoc.DocID;
        RESULT = {
            command: "alert",
            msg: "Заявка создана",
            confirm_result: {
                command: "redirect",
                url: ("view_doc.html?mode=vtbl_purpose_project_page&object_id=" + sNewReqID)
            }
        };
        break;
    }

    default:
        RESULT = oForm;
        break;
}
