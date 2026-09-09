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
// Поля фильтра (текущее состояние на 08.09.2026, после нескольких раундов отладки):
//   matrix_id           -- select (НЕ foreign_elem). Изначально был foreign_elem с
//                           catalog: "cc_learning_matrice" -- имя catalog ПОДТВЕРЖДЕНО
//                           диагностикой как верное (см. HREDU-183_diagnostic_doc_type_names.js:
//                           .Name реального документа матрицы буквально "cc_learning_matrice"),
//                           но picker (dlg_select) падал с "invalid XAML format", а прямое
//                           чтение коллекции cc_learning_matrices в GetMatrixEntries() -- с
//                           "недостаточно прав" (при том, что та же коллекция без проблем
//                           читается в HREDU-181 в контексте "выборки", а не "удалённого
//                           действия" по кнопке). Похоже на проблему прав доступа к этому
//                           справочнику именно в контексте выполнения от имени текущего
//                           пользователя -- НЕ БАГ В ЭТОМ ФАЙЛЕ, нужно проверить у админа
//                           WebTutor права на cc_learning_matrice/cc_learning_matrices для
//                           тестовой роли. GetMatrixEntries() обёрнута в try/catch, чтобы
//                           сбой на этом поле не ронял всю модалку целиком.
//   macroregion          -- select. Список значений строится НЕ вручную, а запросом
//                           SQL DISTINCT по custom_elem f_2ewj у активных сотрудников
//                           (GetMacroregionEntries() ниже, тоже в try/catch) -- чтобы не
//                           хардкодить список макрорегионов и не рассинхронизироваться с
//                           реальными данными.
//   mir_code_id          -- foreign_elem, catalog: "cc_mir_code" (singular имя типа
//                           документа -- подтверждено, поле реально работает). ВАЖНО:
//                           foreign_elem возвращает ID объекта каталога, а не сам код
//                           (например "LASK") -- код для фильтрации на сотрудниках хранится
//                           как текст в custom_elem f_mir_codes. Поэтому на шаге "apply" ID
//                           резолвится в код через tools.open_doc(id).TopElem.name (см.
//                           ResolveMirCodeText()) -- по аналогии с getMirCodeObject() в
//                           education_accept_event_card.
//   position_common_id   -- foreign_elem, catalog: "position_common". Имя ПОДТВЕРЖДЕНО
//                           диагностикой (открыли документ position_common_id=
//                           7679114597049004988 из HREDU-181 и прочитали .Name).
//   program_id           -- foreign_elem, каталог education_method (singular, подтверждено
//                           рабочим примером пользователя). ИЗВЕСТНОЕ ОГРАНИЧЕНИЕ шага 1:
//                           список программ НЕ фильтруется по уже выбранной матрице (это
//                           была бы "каскадная" зависимость одного поля от другого) -- в
//                           примерах, что я видел, такого механизма нет (query_qual
//                           статический, не видит значения других полей формы). Пока это
//                           просто полный список всех программ. Можно будет сузить позже,
//                           когда решим вопрос связи с отчётом.
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
    entries = [{ name: "Все", value: "" }];
    try
    {
        sqlText = "";
        sqlText = sqlText + "select distinct c.data.value('(*/custom_elems/custom_elem[name=''f_2ewj'']/value)[1]', 'varchar(max)') as macroregion\r\n";
        sqlText = sqlText + "from collaborators cs\r\n";
        sqlText = sqlText + "inner join collaborator c on c.id = cs.id\r\n";
        sqlText = sqlText + "where cs.is_dismiss != 1";
        rows = ArraySelectAll(XQuery("sql:" + sqlText));
        for (i = 0; i < ArrayCount(rows); i++)
        {
            sVal = String(rows[i].macroregion);
            if (sVal != "")
            {
                entries.push({ name: sVal, value: sVal });
            }
        }
    }
    catch (_ex)
    {
        // ЗАЩИТА (08.09.2026): та же причина, что у GetMatrixEntries() -- см. комментарий там.
        entries = [{ name: "-- ошибка загрузки списка: " + ExtractUserError(_ex) + " --", value: "" }];
    }
    return entries;
}

/*
 * ОБХОДНОЙ ПУТЬ (08.09.2026): у picker'а (dlg_select) для каталога cc_learning_matrice
 * платформа выдаёт ошибку "invalid XAML format" при клике -- имя catalog проверено и
 * верное (см. диагностику .Name выше), похоже на проблему конфигурации самого экрана
 * выбора для этого справочника на стороне платформы/админки, не на баг в этом файле.
 * Не дожидаясь починки, строим список матриц сами через select (как и для
 * макрорегиона) -- матриц обычно немного, выпадающий список подходит не хуже picker'а.
 * @returns {Object[]}   -   Массив {name, value} (value -- id матрицы как строка).
 */
function GetMatrixEntries()
{
    var rows, entries, i;
    entries = [{ name: "-- выберите матрицу --", value: "" }];
    try
    {
        rows = ArraySelectAll(XQuery("for $elem in cc_learning_matrices order by $elem/name return $elem"));
        for (i = 0; i < ArrayCount(rows); i++)
        {
            entries.push({ name: String(rows[i].name), value: String(Int(rows[i].id)) });
        }
    }
    catch (_ex)
    {
        // ЗАЩИТА (08.09.2026): раньше падение здесь (например "недостаточно прав" на
        // cc_learning_matrices при выполнении от имени текущего пользователя, в отличие от
        // "выборок", где эта же коллекция читается без проблем) валило ВСЮ модалку целиком,
        // так как GetMatrixEntries() вызывалась без try/catch при построении формы. Теперь
        // при сбое поле просто остаётся с одним пунктом-заглушкой и текстом ошибки внутри
        // него -- остальные поля формы всё равно открываются и работают.
        entries = [{ name: "-- ошибка загрузки списка: " + ExtractUserError(_ex) + " --", value: "" }];
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
        // ИСПРАВЛЕНО (09.09.2026): вернули foreign_elem -- причина ошибки picker'а
        // ("invalid XAML format") найдена и устранена: не хватало зарегистрированной
        // выборки-каталога uni_catalog_list_cc_learning_matrice для этого справочника
        // (см. HREDU-183_uni_catalog_list_cc_learning_matrice.js). Обходной путь через
        // select (GetMatrixEntries()) больше не нужен, но саму функцию в файле оставил --
        // вдруг снова понадобится как запасной вариант.
        name: "matrix_id",
        label: "Матрица обучения *",
        title: "Выберите матрицу обучения",
        type: "foreign_elem",
        value: "",
        mandatory: true,
        multiple: false,
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
        // ИСПРАВЛЕНО (08.09.2026): реальное системное имя узнали точно через диагностику
        // (открыли документ типовой должности id=7679114597049004988 -- тот самый, что уже
        // использовался в фильтре HREDU-181 -- и прочитали его .Name). Оказалось
        // "position_common" (порядок слов обратный тому, что я угадывал -- "common_position").
        name: "position_common_id",
        label: "Типовая должность",
        title: "Выберите типовую должность",
        type: "foreign_elem",
        value: "",
        mandatory: false,
        multiple: false,
        catalog: "position_common",
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
