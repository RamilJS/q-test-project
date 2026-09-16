// =====================================================================
// HREDU-182. Модальное окно с фильтрами -- ТОЛЬКО для страницы "Процент обученных".
//
// РАЗДЕЛЕНО (14.09.2026, по прямому указанию пользователя): раньше одна и та же
// модалка (HREDU-183_filtry_modal_shag1.js) висела и на страницах ТЭП, и на странице
// "Процент обученных" -- но у ТЭП есть поле "Режим отчёта" (result_type), которое
// "Процент обученных" вообще не использует (HREDU-182_procent_obuchennyh.js его не
// читает). Чтобы не тащить лишнее поле на страницу, где оно не нужно, и не путать
// пользователя -- РАЗДЕЛИЛИ на два независимых удалённых действия:
//   - HREDU-183_filtry_modal_shag1.js -- ОСТАЁТСЯ БЕЗ ИЗМЕНЕНИЙ, только для 4 страниц
//     ТЭП-отчётов (там же живёт поле "Режим отчёта").
//   - HREDU_182_filtry_percent.js (этот файл) -- НОВЫЙ, для страницы "Процент
//     обученных" и, если понадобится, для "Восток полный список". Это ПОЛНАЯ КОПИЯ
//     HREDU-183_filtry_modal_shag1.js, из которой убрано ВСЁ, что относится к
//     "Режим отчёта"/result_type (поле формы, чтение из URL, запись в URL,
//     RemoveQueryParam) -- остальная логика (5 фильтров, восстановление значений,
//     переносимость на любую страницу через cur_page_url/{{curEnv.curEnvUrl}},
//     санитизация "0" для picker-полей, кодирование через UrlEncodeQuery/UrlDecode)
//     -- идентична, без изменений. См. HREDU-183_filtry_modal_shag1.js -- там вся
//     история находок и фиксов (regex ломает весь файл, function-as-value ломает весь
//     файл, .indexOf/.substring не существуют, Request.Url не работает в удалённом
//     действии, "0" в picker-поле роняет платформу) описана подробно, здесь не
//     повторяем.
//
// Поля фильтра:
//   matrix_id           -- foreign_elem, catalog: "cc_learning_matrice".
//   macroregion          -- ИЗМЕНЕНО (16.09.2026): было select, теперь обычное текстовое
//                           поле (значение вводится вручную "как есть").
//   mir_code_id          -- foreign_elem, catalog: "cc_mir_code" -- резолвится в текст
//                           через ResolveMirCodeText().
//   position_common_id   -- foreign_elem, catalog: "position_common".
//   program_id           -- foreign_elem, catalog: "education_method".
//   (У этой страницы фильтра по городу нет -- "Процент обученных" сама строит по одной
//   строке НА КАЖДЫЙ город, а не фильтрует их; город как фильтр нужен только на страницах
//   ТЭП, см. HREDU-183_filtry_modal_shag1.js.)


DEBUG = true;

/*
 * Чек-пойнт для отладки -- alert() с номером шага, только если DEBUG = true.
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

// УБРАНО (16.09.2026): GetMacroregionEntries() строила SQL DISTINCT список значений для
// select-поля "Макрорегион" -- больше не нужна, т.к. поле стало обычным текстовым вводом
// (см. изменение поля macroregion в oForm.form_fields ниже, та же правка, что и в
// HREDU-183_filtry_modal_shag1.js).

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

/*
 * Достаёт полный URL текущей страницы. В контексте УДАЛЁННОГО ДЕЙСТВИЯ Request.Url НЕ
 * отражает видимый адрес страницы -- нужен параметр "cur_page_url", привязанный в LPE
 * к подстановке {{curEnv.curEnvUrl}} (см. подробности в HREDU-183_filtry_modal_shag1.js).
 * @returns {string}
 */
function GetCurPageUrlSafe()
{
    var sUrl;

    sUrl = getParam("cur_page_url", "");
    if (sUrl != "")
    {
        return sUrl;
    }

    try
    {
        return String(Request.Url);
    }
    catch (_ex)
    {
        return "";
    }
}

/*
 * Вырезает значение GET-параметра из полного URL строки -- без regex, без методов
 * строк (.indexOf/.substring здесь не существуют), через штатный строковый API
 * платформы.
 * @param {string} sUrl         -   Полный URL (например Request.Url).
 * @param {string} sParamName   -   Имя параметра, например "matrix_id".
 * @returns {string}             -   Значение параметра или "" если не найден.
 */
function GetQueryParam(sUrl, sParamName)
{
    var sAmpMarker, sQMarkMarker, iParamPos, iValueStart, iAmpPos, iValueEnd, sRawValue, iUrlLen;

    iUrlLen = StrLen(sUrl);

    sAmpMarker = "&" + sParamName + "=";
    iParamPos = StrOptSubStrPos(sUrl, sAmpMarker, false);
    if (iParamPos != undefined)
    {
        iValueStart = iParamPos + StrLen(sAmpMarker);
    }
    else
    {
        sQMarkMarker = "?" + sParamName + "=";
        iParamPos = StrOptSubStrPos(sUrl, sQMarkMarker, false);
        if (iParamPos == undefined)
        {
            return "";
        }
        iValueStart = iParamPos + StrLen(sQMarkMarker);
    }

    iAmpPos = StrOptSubStrPos(sUrl, "&", false, iValueStart);
    iValueEnd = (iAmpPos != undefined ? iAmpPos : iUrlLen);

    sRawValue = StrRangePos(sUrl, iValueStart, iValueEnd);

    try
    {
        return UrlDecode(sRawValue);
    }
    catch (_exDecode)
    {
        return sRawValue;
    }
}

/*
 * "0" из URL для picker-полей (foreign_elem) означает "не выбрано", НЕ реальный ID --
 * платформа иначе пытается открыть несуществующий документ №0 (см. подробности в
 * HREDU-183_filtry_modal_shag1.js).
 * @param {string} sValue
 * @returns {string}
 */
function SanitizeIdFieldValue(sValue)
{
    if (sValue == "0" || sValue == undefined)
    {
        return "";
    }
    return sValue;
}

/*
 * Убирает из URL старое значение указанного GET-параметра, не трогая остальную часть
 * адреса -- нужно для переносимости (redirect на ТУ ЖЕ страницу, откуда открыли).
 * @param {string} sUrl
 * @param {string} sParamName
 * @returns {string}   -   URL без этого параметра (если параметра не было -- вернёт как есть).
 */
function RemoveQueryParam(sUrl, sParamName)
{
    var sAmpMarker, sQMarkMarker, iMarkerPos, iValueStart, iAmpPos, iUrlLen, sBefore, sAfter;

    iUrlLen = StrLen(sUrl);

    sAmpMarker = "&" + sParamName + "=";
    iMarkerPos = StrOptSubStrPos(sUrl, sAmpMarker, false);
    if (iMarkerPos != undefined)
    {
        iValueStart = iMarkerPos + StrLen(sAmpMarker);
        iAmpPos = StrOptSubStrPos(sUrl, "&", false, iValueStart);
        sBefore = StrRangePos(sUrl, 0, iMarkerPos);
        sAfter = (iAmpPos != undefined ? StrRangePos(sUrl, iAmpPos, iUrlLen) : "");
        return sBefore + sAfter;
    }

    sQMarkMarker = "?" + sParamName + "=";
    iMarkerPos = StrOptSubStrPos(sUrl, sQMarkMarker, false);
    if (iMarkerPos != undefined)
    {
        iValueStart = iMarkerPos + StrLen(sQMarkMarker);
        iAmpPos = StrOptSubStrPos(sUrl, "&", false, iValueStart);
        sBefore = StrRangePos(sUrl, 0, iMarkerPos + 1); // включая сам "?"
        sAfter = (iAmpPos != undefined ? StrRangePos(sUrl, iAmpPos + 1, iUrlLen) : "");
        return sBefore + sAfter;
    }

    return sUrl;
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
    oForm.title = "Фильтры отчёта (Процент обученных)";
    oForm.message = null;

    DebugAlert("3b. Читаем текущий URL страницы для восстановления фильтров (сначала параметр cur_page_url, потом Request.Url)");
    sModalPageUrl = GetCurPageUrlSafe();
    DebugAlert("3b2. Итоговый URL, который используем: [" + sModalPageUrl + "]");
    sDefaultMatrixID = SanitizeIdFieldValue(GetQueryParam(sModalPageUrl, "matrix_id"));
    sDefaultMacroregion = GetQueryParam(sModalPageUrl, "macroregion");
    sDefaultMirCodeID = SanitizeIdFieldValue(GetQueryParam(sModalPageUrl, "mir_code_id"));
    sDefaultPositionCommonID = SanitizeIdFieldValue(GetQueryParam(sModalPageUrl, "position_common_id"));
    sDefaultProgramID = SanitizeIdFieldValue(GetQueryParam(sModalPageUrl, "program_id"));
    DebugAlert("3c. Значения по умолчанию из URL: matrix_id=[" + sDefaultMatrixID + "] macroregion=[" + sDefaultMacroregion
        + "] mir_code_id=[" + sDefaultMirCodeID + "] position_common_id=[" + sDefaultPositionCommonID
        + "] program_id=[" + sDefaultProgramID + "]");

    oForm.form_fields = [
        {
            name: "matrix_id",
            label: "Матрица обучения *",
            title: "Выберите матрицу обучения",
            type: "foreign_elem",
            value: sDefaultMatrixID,
            mandatory: true,
            multiple: false,
            catalog: "cc_learning_matrice",
            query_qual: ""
        },
        {
            // ИЗМЕНЕНО (16.09.2026, по прямой просьбе пользователя): было select со
            // списком значений из GetMacroregionEntries() (SQL DISTINCT, убрана) -- стало
            // обычное текстовое поле, значение вводится вручную и сравнивается "как есть".
            // Убрано и visibility: false -- поле было скрыто, для текстового поля ввода
            // это не нужно. См. тот же комментарий про непроверенность типа "string" в
            // HREDU-183_filtry_modal_shag1.js -- относится и сюда.
            name: "macroregion",
            label: "Макрорегион",
            type: "string",
            value: sDefaultMacroregion,
            mandatory: false
        },
        {
            name: "mir_code_id",
            label: "Мир-код",
            title: "Выберите мир-код",
            type: "foreign_elem",
            value: sDefaultMirCodeID,
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
            value: sDefaultPositionCommonID,
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
            value: sDefaultProgramID,
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

            // Переносимость: redirect на ТУ ЖЕ страницу, откуда открыли модалку.
            sCleanBaseUrl = sModalPageUrl;
            sCleanBaseUrl = RemoveQueryParam(sCleanBaseUrl, "matrix_id");
            sCleanBaseUrl = RemoveQueryParam(sCleanBaseUrl, "macroregion");
            sCleanBaseUrl = RemoveQueryParam(sCleanBaseUrl, "mir_code_id");
            sCleanBaseUrl = RemoveQueryParam(sCleanBaseUrl, "mir_code");
            sCleanBaseUrl = RemoveQueryParam(sCleanBaseUrl, "position_common_id");
            sCleanBaseUrl = RemoveQueryParam(sCleanBaseUrl, "program_id");
            DebugAlert("7c2. Текущая страница без старых фильтров: [" + sCleanBaseUrl + "]");

            oQueryParams = {
                matrix_id: String(iMatrixID),
                macroregion: sMacroregion,
                mir_code: sMirCodeText,
                mir_code_id: String(iMirCodeID),
                position_common_id: String(iPositionCommonID),
                program_id: String(iProgramID)
            };
            sQueryString = UrlEncodeQuery(oQueryParams);

            sSeparator = (StrOptSubStrPos(sCleanBaseUrl, "?", false) != undefined ? "&" : "?");
            sFullUrl = sCleanBaseUrl + sSeparator + sQueryString;
            DebugAlert("7d. Итоговый URL redirect: " + sFullUrl);

            oForm = {
                command: "close_form",
                confirm_result: {
                    command: "redirect",
                    url: sFullUrl
                }
            };

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
    RESULT = {
        command: "alert",
        msg: ("Ошибка в модалке фильтров (HREDU_182_filtry_percent.js):<br/><pre>" + ExtractUserError(_exMain) + "</pre>"),
        title: "ОШИБКА"
    };
}
      
