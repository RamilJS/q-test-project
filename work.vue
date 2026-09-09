// =====================================================================
// HREDU-183. Шаг 1: модальное окно с фильтрами.
//
// Устроено по образцу рабочего "Удаленное действие кнопки" (визард создания
// заявки на подбор) -- то же самое устройство параметров/форм:
//   - PARAMETERS.GetOptProperty("form_fields") / ("form_fields_default") --
//     JSON-массивы полей формы, читаются через getParam()/getFormField().
//   - oForm.command = "display_form" -- команда показать модальное окно.
//   - Кнопки с submit_type определяют, что произойдёт при нажатии (обрабатывается
//     через switch(sSubmitType) ниже) -- ремоут-экшен вызывается заново с новым
//     form_fields при каждом нажатии кнопки формы.
//
// Параметры удалённого действия (настраиваются в LPE у кнопки, аналогично
// примеру): form_fields -- обычно пусто; form_fields_default -- обычно [].
//
// Поля фильтра (без изменений с прошлой версии):
//   matrix_id           -- foreign_elem, catalog: "cc_learning_matrice". Picker чинился
//                           через отдельную выборку uni_catalog_list_cc_learning_matrice
//                           (см. HREDU-183_uni_catalog_list_cc_learning_matrice.js) -- исправлено,
//                           работает.
//   macroregion          -- select. Список строится запросом SQL DISTINCT по custom_elem
//                           f_2ewj у активных сотрудников (GetMacroregionEntries()).
//   mir_code_id          -- foreign_elem, catalog: "cc_mir_code". foreign_elem отдаёт ID
//                           объекта каталога, а не текстовый код -- код резолвится через
//                           ResolveMirCodeText().
//   position_common_id   -- foreign_elem, catalog: "position_common".
//   program_id           -- foreign_elem, catalog: "education_method". ИЗВЕСТНОЕ
//                           ОГРАНИЧЕНИЕ: список НЕ сужается по уже выбранной матрице.
//
// =====================================================================
// ШАГ "apply" (09.09.2026) -- ПОПЫТКА №1 передать фильтры в отчёт: НЕ ПРОВЕРЕНО.
//
// Идея: по кнопке "Применить" делаем не alert(), а redirect на ту же страницу отчёта
// с фильтрами в query string. У страницы отчёта параметры выборки "Табличные данные"
// (matrix_id, macroregion, mir_code, position_common_id, program_id) должны быть
// привязаны не к {{curObject.x}}, а к подстановке, которая читает GET-параметр запроса
// (по документации в selections.md это один из трёх официальных способов передать
// параметры в выборку) -- предположительно вкладка Env или Context в редакторе
// привязки параметров. ЭТО ТЕБЕ НУЖНО ПРОВЕРИТЬ САМОМУ перед тестом всей связки:
// открой привязку параметра matrix_id у выборки "Табличных данных" и посмотри,
// есть ли на вкладках Env/Context подстановка вида "параметр запроса"/GET.
//
// ЧТО ТОЧНО НЕ ПРОВЕРЕНО и может не сработать с первого раза:
//   1. sReportUrl ниже -- ПОДСТАВЛЕН (09.09.2026, тестовый адрес времени разработки):
//      "https://als-devwt.vl.vtb/view_doc.html?mode=matrix_test". У него уже есть свой
//      query-параметр mode=matrix_test, поэтому фильтры дописываются через "&". Когда
//      появится боевой адрес -- поменять только эту строку.
//   2. Контракт oForm.command="close_form" + oForm.confirm_result={command:"redirect",url:...}
//      взят по аналогии с education_accept_edit_event (там были именно такие пары полей
//      на завершение действия), но НЕ подтверждён на этой конкретной модалке -- если после
//      "Применить" ничего не произойдёт или будет ошибка, пришли мне точный текст ошибки.
//   3. UrlEncodeSafe() ниже пытается использовать encodeURIComponent() -- это стандартная
//      функция JS, в других файлах этого проекта встречались операции, характерные для
//      обычного JS (например report.length в диагностике), так что она ВЕРОЯТНО есть, но
//      я не тестировал это в данном движке. Если macroregion (кириллица, например "Восток")
//      придёт в отчёт пустым/битым -- сообщи, будем разбираться с кодированием отдельно.
//
// Если после теста redirect либо не происходит, либо на новой странице фильтры не
// подхватываются -- НЕ пытайся чинить это в одиночку через догадки, скинь мне: (а) что
// написано в адресной строке браузера после нажатия "Применить", (б) что показывает
// Табличные данные. Тогда решим, что чинить: URL, привязку параметра или сам контракт
// confirm_result.
//
// ВРЕМЕННЫЙ ПРОВЕРОЧНЫЙ alert() (старая версия шага "apply", просто показывала, что
// пришло по каждому полю) оставлен ниже закомментированным -- если redirect совсем не
// заработает, можно быстро вернуться к нему, чтобы отдельно проверить, что сами поля
// формы (picker'ы/select) всё ещё отдают ожидаемые значения.
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

/*
 * Пытается процентно закодировать значение для query string через encodeURIComponent().
 * Если в этом скриптовом движке такой функции нет -- падает в грубый ручной запасной
 * вариант (см. предупреждение в шапке файла -- кириллица тут не гарантированно проедет,
 * это нужно проверить на реальном тесте).
 * @param {*} value
 * @returns {string}
 */
function UrlEncodeSafe(value)
{
    var sValue = String(value);
    try
    {
        return encodeURIComponent(sValue);
    }
    catch (_ex)
    {
        return sValue.replace(/ /g, "%20");
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
        catalog: "cc_mir_code",
        query_qual: ""
    },
    {
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
        iMatrixID = OptInt(getFormField("matrix_id", ""), 0);
        sMacroregion = String(getFormField("macroregion", ""));
        iMirCodeID = OptInt(getFormField("mir_code_id", ""), 0);
        iPositionCommonID = OptInt(getFormField("position_common_id", ""), 0);
        iProgramID = OptInt(getFormField("program_id", ""), 0);
        sMirCodeText = ResolveMirCodeText(iMirCodeID);

        // Адрес страницы отчёта (тестовый, время разработки -- получен от пользователя
        // 09.09.2026). ВАЖНО: у страницы уже есть свой query-параметр "mode=matrix_test",
        // поэтому фильтры добавляем через "&", а не заново через "?" -- иначе затрём его.
        // Когда появится боевой адрес, поменяй только эту строку.
        sReportUrl = "https://als-devwt.vl.vtb/view_doc.html?mode=matrix_test";

        sQueryString = "";
        sQueryString = sQueryString + "matrix_id=" + UrlEncodeSafe(iMatrixID);
        sQueryString = sQueryString + "&macroregion=" + UrlEncodeSafe(sMacroregion);
        sQueryString = sQueryString + "&mir_code=" + UrlEncodeSafe(sMirCodeText);
        sQueryString = sQueryString + "&position_common_id=" + UrlEncodeSafe(iPositionCommonID);
        sQueryString = sQueryString + "&program_id=" + UrlEncodeSafe(iProgramID);

        oForm = {
            command: "close_form",
            confirm_result: {
                command: "redirect",
                url: sReportUrl + "&" + sQueryString
            }
        };

        // ЗАПАСНОЙ ВАРИАНТ (старая проверочная версия) -- если редирект совсем не сработает,
        // раскомментируй этот блок вместо oForm выше, чтобы отдельно проверить значения полей:
        //
        // sReport = "";
        // sReport = sReport + "matrix_id = " + iMatrixID + "\r\n";
        // sReport = sReport + "macroregion = [" + sMacroregion + "]\r\n";
        // sReport = sReport + "mir_code_id = " + iMirCodeID + " (код: [" + sMirCodeText + "])\r\n";
        // sReport = sReport + "position_common_id = " + iPositionCommonID + "\r\n";
        // sReport = sReport + "program_id = " + iProgramID;
        // oForm = {
        //     command: "alert",
        //     msg: ("Выбранные фильтры (проверочный вывод):<br/><pre>" + sReport + "</pre>"),
        //     title: "Фильтры применены (пока без связи с отчётом)"
        // };

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
