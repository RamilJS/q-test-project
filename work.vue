// =====================================================================
// Matrix param percent 
// HREDU-182 / HREDU-183. Инфо-блок "Макрорегион / Матрица обучения"
// (matrix_param_percent) -- выборка для виджета на страницах "Процент обученных" и ТЭП.
//
// Показывает текущие фильтры страницы двумя строками { name, value, id }:
//   Макрорегион       -- значение macroregion из URL "как есть";
//   Матрица обучения  -- название документа по matrix_id из URL.
// Нет значения -- "-".
//
// URL страницы берётся из параметра cur_page_url -> {{curEnv.curEnvUrl}} в LPE (после
// обновления платформы Request.Url в выборке недоступен).
// =====================================================================

//-------------------------------------------------------------------------
//              Область констант
//-------------------------------------------------------------------------

DEBUG = false;                          // Включает подробные логи для дебага
LOG_NAME = "agent";                     // Имя журнала на сервере, куда будут писаться логи
CUR_OBJECT_ID = 7686520945469837239;    // ID выборки matrix_param_percent

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
* Название матрицы по ID; "" -- если ID не передан или документ не открылся.
* @param {number} iMatrixID
* @returns {string}
*/
function GetMatrixNameSafe(iMatrixID)
{
    if (OptInt(iMatrixID, 0) <= 0)
    {
        return "";
    }
    try
    {
        return String(tools.open_doc(Int(iMatrixID)).TopElem.name);
    }
    catch (_ex)
    {
        LogAlert(3, "GetMatrixNameSafe(). Не удалось открыть матрицу id=" + iMatrixID + ": " + ExtractUserError(_ex));
        return "";
    }
}

/*
* Точка входа.
* @returns {void}
*/
function Run()
{
    var sFullUrl, sMacroregion, iMatrixID, sMatrixName;

    ERROR = 0;
    MESSAGE = "";
    RESULT = [];

    LogAlert(2, "Run(). НАЧАЛО");

    try
    {
        sFullUrl = GetRequestUrlSafe();
        LogAlert(1, "Run(). URL = [" + sFullUrl + "]");

        sMacroregion = GetQueryParam(sFullUrl, "macroregion");
        iMatrixID = OptInt(GetQueryParam(sFullUrl, "matrix_id"), 0);
        sMatrixName = GetMatrixNameSafe(iMatrixID);
        LogAlert(1, "Run(). macroregion=[" + sMacroregion + "] matrix_id=" + iMatrixID + " matrixName=[" + sMatrixName + "]");

        RESULT.push({
            name: "Макрорегион",
            value: (sMacroregion != "" ? sMacroregion : "-"),
            id: ArrayCount(RESULT)
        });
        RESULT.push({
            name: "Матрица обучения",
            value: (sMatrixName != "" ? sMatrixName : "-"),
            id: ArrayCount(RESULT)
        });

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
