EnableLog('matrix_filters_btn_plan', true);
function alert(_string) {
    LogEvent('matrix_filters_btn_plan', _string);
    return _string;
}

/*
 * Чек-пойнт для отладки -- alert() с номером шага, только если DEBUG = true.
 * @param {string} sStep
 */
DEBUG = true;
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

/*
 * Резолвит ID объекта cc_mir_codes в его текстовый код -- см. подробный комментарий в
 * HREDU-183_filtry_modal_shag1.js (не менялось, скопировано без изменений).
 * @param {number} iMirCodeID
 * @returns {string}
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
 * Достаёт полный URL текущей страницы -- см. подробный комментарий в
 * HREDU-183_filtry_modal_shag1.js (не менялось, скопировано без изменений). Способ 1:
 * параметр cur_page_url ({{curEnv.curEnvUrl}}), способ 2 (запасной): Request.Url.
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
 * Вырезает значение GET-параметра из URL -- см. HREDU-183_filtry_modal_shag1.js
 * (не менялось, скопировано без изменений).
 * @param {string} sUrl
 * @param {string} sParamName
 * @returns {string}
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
 * Превращает "0" (признак "ничего не выбрано" для picker-полей) обратно в "" -- см.
 * HREDU-183_filtry_modal_shag1.js (не менялось, скопировано без изменений).
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
 * Убирает из URL старое значение GET-параметра -- см. HREDU-183_filtry_modal_shag1.js
 * (не менялось, скопировано без изменений).
 * @param {string} sUrl
 * @param {string} sParamName
 * @returns {string}
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

DebugAlert("0. Файл начал выполняться (кнопка 'План', result_type зашит как 'plan')");

try
{
    // ДОБАВЛЕНО (02.10.2026): в отличие от HREDU-183_filtry_modal_shag1.js здесь НЕТ формы и
    // НЕТ выбора пользователя -- result_type ЗАШИТ прямо в код, под эту конкретную кнопку.
    sResultType = "plan";

    DebugAlert("1. Читаем текущий URL страницы (cur_page_url, затем Request.Url как запасной план)");
    sModalPageUrl = GetCurPageUrlSafe();
    DebugAlert("1b. Итоговый URL, который используем: [" + sModalPageUrl + "]");

    // Остальные фильтры (matrix_id/macroregion/mir_code_id/position_common_id/program_id/
    // city) -- те же самые СКВОЗНЫЕ параметры, что в HREDU-183_filtry_modal_shag1.js (там
    // поля формы для них уже скрыты -- см. комментарий в том файле -- значит и там, и
    // здесь их значение берётся НАПРЯМУЮ из текущего URL, без формы).
    sDefaultMatrixID = SanitizeIdFieldValue(GetQueryParam(sModalPageUrl, "matrix_id"));
    sDefaultMacroregion = GetQueryParam(sModalPageUrl, "macroregion");
    sDefaultMirCodeID = SanitizeIdFieldValue(GetQueryParam(sModalPageUrl, "mir_code_id"));
    sDefaultPositionCommonID = SanitizeIdFieldValue(GetQueryParam(sModalPageUrl, "position_common_id"));
    sDefaultProgramID = SanitizeIdFieldValue(GetQueryParam(sModalPageUrl, "program_id"));
    sDefaultCity = GetQueryParam(sModalPageUrl, "city");

    DebugAlert("1c. Сквозные значения из URL: matrix_id=[" + sDefaultMatrixID + "] macroregion=[" + sDefaultMacroregion
        + "] mir_code_id=[" + sDefaultMirCodeID + "] position_common_id=[" + sDefaultPositionCommonID
        + "] program_id=[" + sDefaultProgramID + "] city=[" + sDefaultCity + "]");

    iMatrixID = OptInt(sDefaultMatrixID, 0);
    sMacroregion = String(sDefaultMacroregion);
    iMirCodeID = OptInt(sDefaultMirCodeID, 0);
    iPositionCommonID = OptInt(sDefaultPositionCommonID, 0);
    iProgramID = OptInt(sDefaultProgramID, 0);
    sCity = String(sDefaultCity);

    sMirCodeText = ResolveMirCodeText(iMirCodeID);
    DebugAlert("2. mir_code резолвлен в текст: [" + sMirCodeText + "]");

    // Стираем старые значения всех 7 параметров со страницы и дописываем новые (та же
    // логика переносимости, что в HREDU-183_filtry_modal_shag1.js) -- result_type среди
    // них ВСЕГДА result_type=plan для этой кнопки.
    sCleanBaseUrl = sModalPageUrl;
    sCleanBaseUrl = RemoveQueryParam(sCleanBaseUrl, "matrix_id");
    sCleanBaseUrl = RemoveQueryParam(sCleanBaseUrl, "macroregion");
    sCleanBaseUrl = RemoveQueryParam(sCleanBaseUrl, "mir_code_id");
    sCleanBaseUrl = RemoveQueryParam(sCleanBaseUrl, "mir_code");
    sCleanBaseUrl = RemoveQueryParam(sCleanBaseUrl, "position_common_id");
    sCleanBaseUrl = RemoveQueryParam(sCleanBaseUrl, "program_id");
    sCleanBaseUrl = RemoveQueryParam(sCleanBaseUrl, "result_type");
    sCleanBaseUrl = RemoveQueryParam(sCleanBaseUrl, "city");
    DebugAlert("3. Текущая страница без старых фильтров: [" + sCleanBaseUrl + "]");

    oQueryParams = {
        matrix_id: String(iMatrixID),
        macroregion: sMacroregion,
        mir_code: sMirCodeText,
        mir_code_id: String(iMirCodeID),
        position_common_id: String(iPositionCommonID),
        program_id: String(iProgramID),
        result_type: sResultType,
        city: sCity
    };
    sQueryString = UrlEncodeQuery(oQueryParams);

    sSeparator = (StrOptSubStrPos(sCleanBaseUrl, "?", false) != undefined ? "&" : "?");
    sFullUrl = sCleanBaseUrl + sSeparator + sQueryString;
    DebugAlert("4. Итоговый URL redirect: " + sFullUrl);

    // ПРОВЕРИТЬ РЕАЛЬНЫМ ТЕСТОМ (см. флаг в шапке файла) -- простой redirect без
    // close_form/confirm_result, т.к. кнопка не открывает модалку.
    RESULT = {
        command: "redirect",
        url: sFullUrl
    };

    DebugAlert("5. RESULT собран (redirect на result_type=plan)");
}
catch (_exMain)
{
    RESULT = {
        command: "alert",
        msg: ("Ошибка в кнопке 'План' (HREDU-183_set_mode_plan.js):<br/><pre>" + ExtractUserError(_exMain) + "</pre>"),
        title: "ОШИБКА"
    };
}
