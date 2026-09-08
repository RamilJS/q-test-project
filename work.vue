// =====================================================================
// HREDU-183. Шаг 1: модальное окно с фильтрами (пока САМО ПО СЕБЕ, без связи
// с "Табличными данными" -- по указанию пользователя от 08.09.2026: сначала
// делаем и проверяем фильтры отдельно, потом решаем, как подключить их к отчёту).
//
// Устроено по образцу рабочего "Удаленное действие кнопки" (визард создания
// заявки на подбор) -- то же самое устройство параметров/форм:
//   - PARAMETERS.GetOptProperty("form_fields") / ("form_fields_default") --
//     JSON-массивы полей формы, читаются через getParam()/getFormField().
//   - oForm.command = "display_form" -- команда показать модальное окно.
//   - Кнопки с submit_type определяют, что произойдёт при нажатии (обрабатывается
//     через switch(sSubmitType) ниже) -- ремоут-экшен вызывается заново с новым
//     form_fields при каждом нажатии кнопки формы.
// В отличие от примера-визарда, здесь ОДИН шаг (все фильтры на одном экране),
// без сохранения документа -- просто собираем значения фильтров.
//
// Параметры удалённого действия (настраиваются в LPE у кнопки, аналогично
// примеру): form_fields -- обычно пусто; form_fields_default -- обычно [].
//
// Поля фильтра:
//   matrix_id           -- foreign_elem, каталог cc_learning_matrices. Обязательное:
//                           без выбранной матрицы отчёт по HREDU-181/183 не строится.
//   macroregion          -- select. Список значений строится НЕ вручную, а запросом
//                           SQL DISTINCT по custom_elem f_2ewj у активных сотрудников
//                           (GetMacroregionEntries() ниже) -- чтобы не хардкодить
//                           список макрорегионов и не рассинхронизироваться с
//                           реальными данными.
//   mir_code_id          -- foreign_elem, каталог cc_mir_codes. ВАЖНО: foreign_elem
//                           возвращает ID объекта каталога, а не сам код (например
//                           "LASK") -- код для фильтрации на сотрудниках хранится как
//                           текст в custom_elem f_mir_codes. Поэтому на шаге "apply"
//                           ID резолвится в код через tools.open_doc(id).TopElem.name
//                           (см. ResolveMirCodeText()) -- по аналогии с
//                           getMirCodeObject() в education_accept_event_card.
//   position_common_id   -- foreign_elem, каталог -- НЕ ПОДТВЕРЖДЕНО, пользователь
//                           настраивал ссылку на объект "типовая должность" сам в LPE,
//                           точное системное имя каталога мне не известно. Ниже стоит
//                           "common_position" как ПРЕДПОЛОЖЕНИЕ (по аналогии с полем
//                           positions.position_common_id, см. диагностику в HREDU-181) --
//                           если каталог не грузится в выпадающем списке при тесте,
//                           нужно заменить на реальное имя (можно посмотреть в LPE,
//                           там, где ты сам настраивал этот фильтр: Показать в XML
//                           у самого поля/справочника).
//   program_id           -- foreign_elem, каталог education_method. ИЗВЕСТНОЕ
//                           ОГРАНИЧЕНИЕ шага 1: список программ НЕ фильтруется по
//                           уже выбранной матрице (это была бы "каскадная" зависимость
//                           одного поля от другого) -- в примерах, что я видел, такого
//                           механизма нет (query_qual статический, не видит значения
//                           других полей формы). Пока это просто полный список всех
//                           программ. Можно будет сузить позже, когда решим вопрос
//                           связи с отчётом.
//
// По кнопке "Применить" (submit_type = "apply") ЭТОТ ШАГ НЕ СВЯЗЫВАЕТ фильтры ни
// с какой выборкой -- просто показывает alert() с тем, что реально было выбрано
// (специально, чтобы проверить: возвращает ли каждый picker то, что ожидается --
// ID для foreign_elem, текст для select). Это временная проверочная логика,
// уберём/заменим, когда решим, как передавать значения в "Табличные данные".
// =====================================================================

function getParam(sName, sDefault) {
    var sValue = PARAMETERS.GetOptProperty(sName);
    if (sDefault != undefined && (sValue == undefined || sValue == "")) {
        sValue = sDefault;
    }
    return sValue;
}

function getFormField(sName, sDefault) {
    var sValue = ArrayOptFind(aFormFields, ("This.name == " + XQueryLiteral(sName)));
    sValue = (sValue != undefined ? sValue.value : sValue);
    if (sDefault != undefined && (sValue == undefined || sValue == "")) {
        sValue = sDefault;
    }
    return sValue;
}

function getFormFieldDefault(sName, sDefault) {
    var sValue = ArrayOptFind(aFormFieldsDef, ("This.name == " + XQueryLiteral(sName)));
    sValue = (sValue != undefined ? sValue.value : sValue);
    if (sDefault != undefined && (sValue == undefined || sValue == "")) {
        sValue = sDefault;
    }
    return sValue;
}

/*
 * Строит список entries для select-поля "Макрорегион" -- DISTINCT по реальным
 * значениям custom_elem f_2ewj у активных сотрудников (не хардкод).
 * @returns {Object[]}   -   Массив {name, value}, первый пункт -- "Все".
 */
function GetMacroregionEntries()
{
    var sqlText, rows, entries, i, sVal;
    sqlText = "";
    sqlText = sqlText + "select distinct c.data.value('(*/custom_elems/custom_elem[name=''f_2ewj'']/value)[1]', 'varchar(max)') as macroregion\r\n";
    sqlText = sqlText + "from collaborators cs\r\n";
    sqlText = sqlText + "inner join collaborator c on c.id = cs.id\r\n";
    sqlText = sqlText + "where cs.is_dismiss != 1";
    rows = ArraySelectAll(XQuery("sql:" + sqlText));
    entries = [{ name: "Все", value: "" }];
    for (i = 0; i < ArrayCount(rows); i++)
    {
        sVal = String(rows[i].macroregion);
        if (sVal != "")
        {
            entries.push({ name: sVal, value: sVal });
        }
    }
    return entries;
}

/*
 * Резолвит ID объекта cc_mir_codes (то, что реально возвращает foreign_elem) в его
 * текстовый код (то, что реально лежит в custom_elem f_mir_codes у сотрудников) --
 * по аналогии с getMirCodeObject() в education_accept_event_card.
 * @param {number} iMirCodeID   -   ID документа cc_mir_codes.
 * @returns {string}            -   Код (например "LASK") или "" если не найден/не выбран.
 */
function ResolveMirCodeText(iMirCodeID)
{
    if (OptInt(iMirCodeID, 0) <= 0)
    {
        return "";
    }
    try
    {
        return String(tools.open_doc(Int(iMirCodeID)).TopElem.name);
    }
    catch (_ex)
    {
        return "";
    }
}

aFormFields = ParseJson(getParam("form_fields", "[]"));
aFormFieldsDef = ParseJson(getParam("form_fields_default", "[]"));
sSubmitType = getFormField("__submit_type__", getFormFieldDefault("__submit_type__", "step_0"));

oForm = new Object();
oForm.command = "display_form";
oForm.height = 320;
oForm.title = "Фильтры отчёта (Восток)";
oForm.message = null;
oForm.form_fields = [
    {
        name: "matrix_id",
        label: "Матрица обучения *",
        title: "Выберите матрицу обучения",
        type: "foreign_elem",
        value: "",
        mandatory: true,
        multiple: false,
        // ИСПРАВЛЕНО (08.09.2026): у foreign_elem catalog -- это ИМЯ ТИПА ДОКУМЕНТА
        // (singular, как в tools.open_doc_by_name/new_doc_by_name), а НЕ название
        // XQuery-коллекции во множественном числе. cc_learning_matrices (плюрал,
        // имя коллекции) вызывал ошибку рендера dlg_select.xaml -- заменено на
        // предположительный singular по аналогии с cc_learning_matrice_elements
        // (плюрал коллекции элементов) -> singular "cc_learning_matrice_element".
        // ПРОВЕРИТЬ при тесте.
        catalog: "cc_learning_matrice",
        query_qual: ""
    },
    {
        name: "macroregion",
        label: "Макрорегион",
        type: "select",
        value: "",
        entries: GetMacroregionEntries(),
        mandatory: false,
        visibility: false
    },
    {
        name: "mir_code_id",
        label: "Мир-код",
        title: "Выберите мир-код",
        type: "foreign_elem",
        value: "",
        mandatory: false,
        multiple: false,
        // ИСПРАВЛЕНО (08.09.2026): та же причина, что и у matrix_id -- catalog должен
        // быть singular именем типа документа, не именем коллекции cc_mir_codes.
        // ПРОВЕРИТЬ при тесте.
        catalog: "cc_mir_code",
        query_qual: ""
    },
    {
        // TODO: имя каталога "common_position" НЕ ПОДТВЕРЖДЕНО -- это тоже, скорее
        // всего, должно быть singular имя типа документа (см. ИСПРАВЛЕНО у matrix_id/
        // mir_code_id выше), но правильное имя мне неоткуда взять самому. У тебя уже
        // ЕСТЬ рабочий пример: поле "типовая должность", которое ты сам настраивал в
        // LPE для фильтра в HREDU-181. Открой его настройки там и пришли мне точное
        // значение catalog оттуда -- скорее всего ошибка на этом поле останется, пока
        // не заменим на реальное имя.
        name: "position_common_id",
        label: "Типовая должность",
        title: "Выберите типовую должность",
        type: "foreign_elem",
        value: "",
        mandatory: false,
        multiple: false,
        catalog: "common_position",
        query_qual: ""
    },
    {
        // ИЗВЕСТНОЕ ОГРАНИЧЕНИЕ: список НЕ сужается по уже выбранной матрице -- см. шапку файла.
        name: "program_id",
        label: "Учебная программа",
        title: "Выберите учебную программу",
        type: "foreign_elem",
        value: "",
        mandatory: false,
        multiple: false,
        catalog: "education_method",
        query_qual: ""
    }
];

for (oField in oForm.form_fields)
{
    oField.value = getFormField(oField.name, oField.value);
}

oForm.buttons = [];
oForm.no_buttons = false;

switch (sSubmitType)
{
    case "apply":
    {
        // ВРЕМЕННО: просто показываем, что реально пришло по каждому полю -- чтобы
        // проверить, что picker'ы отдают ожидаемые значения (ID для foreign_elem,
        // текст для select), прежде чем решать, как передать это в отчёт.
        sMatrixID = OptInt(getFormField("matrix_id", ""), 0);
        sMacroregion = String(getFormField("macroregion", ""));
        sMirCodeID = OptInt(getFormField("mir_code_id", ""), 0);
        sPositionCommonID = OptInt(getFormField("position_common_id", ""), 0);
        sProgramID = OptInt(getFormField("program_id", ""), 0);

        sReport = "";
        sReport = sReport + "matrix_id = " + sMatrixID + "\r\n";
        sReport = sReport + "macroregion = [" + sMacroregion + "]\r\n";
        sReport = sReport + "mir_code_id = " + sMirCodeID + " (код: [" + ResolveMirCodeText(sMirCodeID) + "])\r\n";
        sReport = sReport + "position_common_id = " + sPositionCommonID + "\r\n";
        sReport = sReport + "program_id = " + sProgramID;

        oForm = {
            command: "alert",
            msg: ("Выбранные фильтры (проверочный вывод):<br/><pre>" + sReport + "</pre>"),
            title: "Фильтры применены (пока без связи с отчётом)"
        };
        break;
    }
    case "step_0":
    default:
    {
        oForm.buttons.push(
            { name: "submit", submit_type: "apply", label: "Применить", type: "submit" },
            { name: "cancel", label: "Отмена", type: "cancel" }
        );
        break;
    }
}

RESULT = oForm;
