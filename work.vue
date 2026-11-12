// =====================================================================
// HREDU-183 / HREDU-182. Инфо-блок фильтров страницы ТЭП (matrix_param_tap) -- выборка
// для виджета "Табличные данные".
//
// Пять строк { name, value, id }: Макрорегион, Город, Матрица обучения, Учебная
// программа, Тип отчёта. Нет значения -- "-", тип отчёта по умолчанию "Общее кол-во".
//
// URL страницы берётся из параметра cur_page_url -> {{curEnv.curEnvUrl}} в LPE (после
// обновления платформы Request.Url в выборке недоступен).
// =====================================================================

//-------------------------------------------------------------------------
//              Область констант
//-------------------------------------------------------------------------

DEBUG = false;                          // Включает подробные логи для дебага
LOG_NAME = "agent";                     // Имя журнала на сервере, куда будут писаться логи
CUR_OBJECT_ID = 7686800706407988539;    // ID выборки matrix_param_tap

//-------------------------------------------------------------------------
//              Область функций
//-------------------------------------------------------------------------

/*
* Выводит сообщение в логи. Сбой логирования не должен ронять основной код.
* @param {number} typeLog   -   Уровень логов. 1 - [DEBUG], 2 - [INFO], 3 - [WARN], 4 - [ERROR].
* @param {string} message   -   Сообщение для логов.
* @returns {void}
*/
function LogAlert(typeLog, message)
{
    try
    {
        tools.call_code_library_method("vtbl_log_lib", "LogAlert", [LOG_NAME, typeLog, CUR_OBJECT_ID, message, DEBUG]);
    }
    catch (_exLog)
    {
    }
}

/*
* Полный URL текущей страницы. После обновления платформы Request в контексте выборки
* недоступен -- основной источник параметр cur_page_url (привязан в LPE к
* {{curEnv.curEnvUrl}}), Request.Url оставлен запасным вариантом (старая версия платформы).
* @returns {string}
*/
function GetRequestUrlSafe()
{
    var sUrl;
    sUrl = "";
    try { sUrl = String(cur_page_url); } catch (_exParam) { sUrl = ""; }
    if (sUrl != "" && sUrl != "undefined" && sUrl != "null")
    {
        LogAlert(2, "GetRequestUrlSafe(). Источник: cur_page_url");
        return sUrl;
    }
    try
    {
        sUrl = String(Request.Url);
        LogAlert(2, "GetRequestUrlSafe(). Источник: Request.Url");
        return sUrl;
    }
    catch (_exReq)
    {
        LogAlert(3, "GetRequestUrlSafe(). Нет ни cur_page_url, ни Request.Url");
        return "";
    }
}

/*
* Значение GET-параметра из URL (без regex и строковых методов JS).
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
        if (iParamPos == undefined) { return ""; }
        iValueStart = iParamPos + StrLen(sQMarkMarker);
    }
    iAmpPos = StrOptSubStrPos(sUrl, "&", false, iValueStart);
    iValueEnd = (iAmpPos != undefined ? iAmpPos : iUrlLen);
    sRawValue = StrRangePos(sUrl, iValueStart, iValueEnd);
    try { return UrlDecode(sRawValue); }
    catch (_exDecode) { return sRawValue; }
}

/*
* Название документа по ID (матрица, учебная программа); "" -- если ID не передан или
* документ не открылся.
* @param {number} iDocID
* @returns {string}
*/
function GetDocNameSafe(iDocID)
{
    if (OptInt(iDocID, 0) <= 0)
    {
        return "";
    }
    try
    {
        return String(tools.open_doc(Int(iDocID)).TopElem.name);
    }
    catch (_ex)
    {
        LogAlert(3, "GetDocNameSafe(). Не удалось открыть документ id=" + iDocID + ": " + ExtractUserError(_ex));
        return "";
    }
}

/*
* Подпись режима отчёта -- как в select "Режим отчёта" фильтров. Пусто/неизвестно --
* "Общее кол-во" (режим по умолчанию в выборке ТЭП).
* @param {string} sResultType
* @returns {string}
*/
function ResolveResultTypeLabel(sResultType)
{
    switch (sResultType)
    {
        case "total": { return "Общее кол-во"; }
        case "plan": { return "План"; }
        case "fact": { return "Факт"; }
        case "mandatory": { return "Обязательно к прохождению"; }
        default: { return "Общее кол-во"; }
    }
}

/*
* Добавляет строку инфо-блока в RESULT; пустое значение -- "-".
* @param {string} sName
* @param {string} sValue
* @returns {void}
*/
function PushInfoRow(sName, sValue)
{
    RESULT.push({
        name: sName,
        value: (sValue != "" ? sValue : "-"),
        id: ArrayCount(RESULT)
    });
}

/*
* Точка входа.
* @returns {void}
*/
function Run()
{
    var sFullUrl, sMacroregion, sCity, iMatrixID, sMatrixName, iProgramID, sProgramName;
    var sResultType, sResultTypeLabel;

    ERROR = 0;
    MESSAGE = "";
    RESULT = [];

    LogAlert(2, "Run(). НАЧАЛО");

    try
    {
        sFullUrl = GetRequestUrlSafe();
        LogAlert(1, "Run(). URL = [" + sFullUrl + "]");

        sMacroregion = GetQueryParam(sFullUrl, "macroregion");
        sCity = GetQueryParam(sFullUrl, "city");
        iMatrixID = OptInt(GetQueryParam(sFullUrl, "matrix_id"), 0);
        sMatrixName = GetDocNameSafe(iMatrixID);
        iProgramID = OptInt(GetQueryParam(sFullUrl, "program_id"), 0);
        sProgramName = GetDocNameSafe(iProgramID);
        sResultType = GetQueryParam(sFullUrl, "result_type");
        sResultTypeLabel = ResolveResultTypeLabel(sResultType);
        LogAlert(1, "Run(). macroregion=[" + sMacroregion + "] city=[" + sCity + "] matrix_id=" + iMatrixID
            + " matrixName=[" + sMatrixName + "] program_id=" + iProgramID + " programName=[" + sProgramName + "]"
            + " result_type=[" + sResultType + "]");

        PushInfoRow("Макрорегион", sMacroregion);
        PushInfoRow("Город", sCity);
        PushInfoRow("Матрица обучения", sMatrixName);
        PushInfoRow("Учебная программа", sProgramName);
        PushInfoRow("Тип отчёта", sResultTypeLabel);

        LogAlert(2, "Run(). Готово. Строк: " + ArrayCount(RESULT));
    }
    catch (_ex)
    {
        ERROR = 1;
        MESSAGE = ExtractUserError(_ex);
        LogAlert(4, "Run(). КРИТИЧЕСКАЯ ОШИБКА: " + MESSAGE);
    }

    LogAlert(2, "Run(). КОНЕЦ");
}

//-------------------------------------------------------------------------
//              Область основного кода
//-------------------------------------------------------------------------

Run();
