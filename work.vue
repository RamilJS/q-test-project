// =====================================================================
// HREDU-183. Шаг 1: модальное окно с фильтрами.
//
// ИСПРАВЛЕНИЕ (09.09.2026, аварийное): предыдущая версия ЛОМАЛА весь файл целиком --
// даже стартовый display_form не открывался. Главный подозреваемый: UrlEncodeSafe()
// использовала regex-литерал (/ /g) в запасной ветке -- скриптовый движок WebTutor,
// судя по всему, НЕ поддерживает синтаксис регулярных выражений, и это ломает разбор
// (компиляцию) всего файла целиком, а не только ветку "apply", где эта функция
// реально вызывается. Regex убран (используется split/join без regex).
//
// ЗАЩИТА ОТ ПОВТОРЕНИЯ ТАКОГО ЖЕ СБОЯ: весь основной код теперь обёрнут в один
// try/catch -- если где-то ещё есть невидимая проблема, вместо "ничего не происходит"
// ты увидишь alert() с точным текстом ошибки (через ExtractUserError). Плюс по твоей
// просьбе добавлены чек-пойнты DebugAlert() на каждом шаге -- если DEBUG = true, они
// покажут alert() на каждой стадии, чтобы точно видеть, до какого места код доходит.
// Когда всё заработает и надоест -- поставь DEBUG = false, чек-пойнты замолчат, но
// try/catch-защита останется (она не зависит от DEBUG).
//
// Устроено по образцу рабочего "Удаленное действие кнопки" (визард создания
// заявки на подбор):
//   - PARAMETERS.GetOptProperty("form_fields") / ("form_fields_default") --
//     JSON-массивы полей формы, читаются через getParam()/getFormField().
//   - oForm.command = "display_form" -- команда показать модальное окно.
//   - Кнопки с submit_type определяют, что произойдёт при нажатии (обрабатывается
//     через switch(sSubmitType) ниже).
//
// Параметры удалённого действия (настраиваются в LPE у кнопки): form_fields --
// обычно пусто; form_fields_default -- обычно [].
//
// Поля фильтра:
//   matrix_id           -- foreign_elem, catalog: "cc_learning_matrice".
//   macroregion          -- select, список из GetMacroregionEntries() (SQL DISTINCT).
//   mir_code_id          -- foreign_elem, catalog: "cc_mir_code" -- резолвится в текст
//                           через ResolveMirCodeText().
//   position_common_id   -- foreign_elem, catalog: "position_common".
//   program_id           -- foreign_elem, catalog: "education_method".
//
// ШАГ "apply" (09.09.2026) -- redirect на страницу отчёта с фильтрами в query string.
// Тестовый адрес: https://als-devwt.vl.vtb/view_doc.html?mode=matrix_test (у него уже
// есть свой параметр mode=matrix_test, поэтому фильтры дописываются через "&"). НА
// ПРОДЕ будет другой адрес (сказал пользователь 09.09.2026): https://als-wt/view_doc.html?mode=matrix
// -- поменять тогда только sReportUrl ниже.
//
// ПОДТВЕРЖДЕНО (09.09.2026, реальный тест): контракт oForm.command="close_form" +
// oForm.confirm_result={command:"redirect",url:...} работает, редирект происходит.
// Табличные данные читают GET-параметры не через подстановку в UI (в Env/Context её
// нет), а сама выборка читает их из Request.Url и парсит вручную -- см.
// HREDU-183_diagnostic_get_params.js.
//
// ИСПРАВЛЕНО (09.09.2026, кодирование): раньше кодировали через encodeURIComponent()
// (стандартный JS -- UTF-8: кириллица уезжала как %D0%A3%D0%A4%D0%9E). Но родная
// функция платформы для декодирования на стороне выборки -- UrlDecode() -- судя по
// примеру в документации (%E0%EF%F0%EE%EB -> "апрол"), ждёт ОДНОБАЙТНУЮ кодировку,
// не UTF-8: декодирование UTF-8-строки через неё дало бы кракозябру. Поэтому теперь
// кодируем ТОЖЕ родной функцией платформы -- UrlEncodeQuery(obj) -- она сама собирает
// "имя1=значение1&имя2=значение2&..." из объекта, в той же схеме, что понимает
// UrlDecode() на другом конце.
// =====================================================================

DEBUG = true;

/*
 * Чек-пойнт для отладки -- alert() с номером шага, только если DEBUG = true.
 * Обёрнут в try/catch, чтобы сама отладочная печать не могла обрушить скрипт,
 * если в каком-то контексте alert()/LogAlert недоступны.
 * @param {string} sStep
 */
function DebugAlert(sStep)
{
    if (!DEBUG)
    {
        return;
    }
    try
    {
        alert("[DEBUG] " + sStep);
    }
    catch (_exDebug)
    {
        // ничего -- отладочная печать не должна ронять основной код
    }
}

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
 * текстовый код (то, что реально лежит в custom_elem f_mir_codes у сотрудников).
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

DebugAlert("0. Файл начал выполняться");

try
{
    DebugAlert("1. Читаем form_fields/form_fields_default");
    aFormFields = ParseJson(getParam("form_fields", "[]"));
    aFormFieldsDef = ParseJson(getParam("form_fields_default", "[]"));
    sSubmitType = getFormField("__submit_type__", getFormFieldDefault("__submit_type__", "step_0"));
    DebugAlert("2. sSubmitType = [" + sSubmitType + "]");

    oForm = new Object();
    oForm.command = "display_form";
    oForm.height = 320;
    oForm.title = "Фильтры отчёта (Восток)";
    oForm.message = null;

    DebugAlert("3. Строим GetMacroregionEntries()");
    aMacroregionEntries = GetMacroregionEntries();
    DebugAlert("4. GetMacroregionEntries() построен, пунктов: " + ArrayCount(aMacroregionEntries));

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
            entries: aMacroregionEntries,
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
    DebugAlert("5. oForm.form_fields собран, полей: " + ArrayCount(oForm.form_fields));

    for (oField in oForm.form_fields)
    {
        oField.value = getFormField(oField.name, oField.value);
    }
    DebugAlert("6. Значения полей проставлены из aFormFields");

    oForm.buttons = [];
    oForm.no_buttons = false;

    switch (sSubmitType)
    {
        case "apply":
        {
            DebugAlert("7a. Ветка apply -- читаем значения полей");
            iMatrixID = OptInt(getFormField("matrix_id", ""), 0);
            sMacroregion = String(getFormField("macroregion", ""));
            iMirCodeID = OptInt(getFormField("mir_code_id", ""), 0);
            iPositionCommonID = OptInt(getFormField("position_common_id", ""), 0);
            iProgramID = OptInt(getFormField("program_id", ""), 0);
            DebugAlert("7b. matrix_id=" + iMatrixID + " macroregion=[" + sMacroregion + "] mir_code_id=" + iMirCodeID + " position_common_id=" + iPositionCommonID + " program_id=" + iProgramID);

            sMirCodeText = ResolveMirCodeText(iMirCodeID);
            DebugAlert("7c. mir_code резолвлен в текст: [" + sMirCodeText + "]");

            // Адрес страницы отчёта (тестовый, время разработки). У страницы уже есть
            // свой query-параметр "mode=matrix_test", поэтому фильтры добавляем через
            // "&". Когда появится боевой адрес -- поменять только эту строку.
            sReportUrl = "https://als-devwt.vl.vtb/view_doc.html?mode=matrix_test";

            // Родная функция платформы -- сама собирает "имя1=значение1&имя2=значение2&..."
            // и кодирует значения в той же схеме, что понимает UrlDecode() на стороне выборки.
            oQueryParams = {
                matrix_id: String(iMatrixID),
                macroregion: sMacroregion,
                mir_code: sMirCodeText,
                position_common_id: String(iPositionCommonID),
                program_id: String(iProgramID)
            };
            sQueryString = UrlEncodeQuery(oQueryParams);

            sFullUrl = sReportUrl + "&" + sQueryString;
            DebugAlert("7d. Итоговый URL redirect: " + sFullUrl);

            oForm = {
                command: "close_form",
                confirm_result: {
                    command: "redirect",
                    url: sFullUrl
                }
            };

            // ЗАПАСНОЙ ВАРИАНТ (проверочный alert вместо редиректа) -- если после починки
            // регэкспа модалка открывается, но именно redirect не срабатывает -- раскомментируй
            // этот блок вместо oForm выше, чтобы отдельно проверить, что сами значения полей
            // (picker'ы/select) верны, независимо от механизма передачи в отчёт:
            //
            // oForm = {
            //     command: "alert",
            //     msg: ("Выбранные фильтры (проверочный вывод):<br/><pre>" + sFullUrl + "</pre>"),
            //     title: "Фильтры применены (пока без связи с отчётом)"
            // };

            DebugAlert("7e. Ветка apply завершена, RESULT будет = close_form/redirect");
            break;
        }
        case "step_0":
        default:
        {
            DebugAlert("8. Ветка step_0/default -- добавляем кнопки Применить/Отмена");
            oForm.buttons.push(
                { name: "submit", submit_type: "apply", label: "Применить", type: "submit" },
                { name: "cancel", label: "Отмена", type: "cancel" }
            );
            break;
        }
    }

    RESULT = oForm;
    DebugAlert("9. RESULT успешно собран, sSubmitType был [" + sSubmitType + "]");
}
catch (_exMain)
{
    // ГЛАВНАЯ ЗАЩИТА (09.09.2026): если где-то в коде выше вылетит ЛЮБАЯ ошибка --
    // вместо "ничего не происходит"/пустого падения покажем alert с точным текстом.
    RESULT = {
        command: "alert",
        msg: ("Ошибка в модалке фильтров (HREDU-183_filtry_modal_shag1.js):<br/><pre>" + ExtractUserError(_exMain) + "</pre>"),
        title: "ОШИБКА"
    };
}
