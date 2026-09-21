sLogName = 'HREDU_183_breadcrumbs';
EnableLog(sLogName, true);
function alert(sInputObj)
{
    LogEvent(sLogName, sInputObj);
    return sInputObj;
}

// =====================================================================
// HREDU-182/183. Breadcrumbs -- ТОЛЬКО для страницы отчётов ТЭП
// (https://als-devwt.vl.vtb/view_doc.html?mode=matrix_report).
//
// ПЕРЕДЕЛАНО ЗАНОВО (21.09.2026, по прямому указанию пользователя после того, как
// прошлая версия вообще ничего не выводила ни на одной странице):
//   1. На странице "Процент обученных" (mode=matrix_educated_percent) крошек вообще
//      НЕ ДОЛЖНО БЫТЬ -- эта выборка на неё НЕ ставится, виджет туда не добавляем.
//   2. На странице ТЭП (mode=matrix_report) -- ДВА уровня крошек:
//        "Отчет процент обученных" (кликабельная, ведёт на страницу "Процент
//        обученных") / "Отчет План" (или "Отчет Факт"/"Отчет Общее кол-во"/"Отчет
//        Обязательно к прохождению" -- по текущему result_type в URL этой самой
//        страницы, см. HREDU-183_tep_reports.js).
//
// ЧТО ИЗМЕНИЛОСЬ В САМОМ КОДЕ (причина, почему раньше НИЧЕГО не выводилось): прошлая
// версия ОБЯЗАТЕЛЬНО читала внешний служебный параметр result_type (LPE-привязка
// виджета, паттерн "fields"/"comment" из присланного пользователем примера) и строила
// крошки ТОЛЬКО внутри case "fields". Если у ЭТОГО конкретного виджета result_type НЕ
// привязан в админке вообще (похоже, так и было) -- обращение к необъявленной
// глобальной переменной result_type кидает исключение ДО входа в switch, ловится общим
// catch, и RESULT остаётся пустым -- то есть крошка не показывается вообще, без единой
// видимой ошибки. ТЕПЕРЬ эта зависимость от внешнего result_type ПОЛНОСТЬЮ УБРАНА --
// скрипт строит крошки БЕЗУСЛОВНО, читая всё, что ему нужно, САМ из Request.Url. Больше
// не нужно ничего настраивать в LPE-параметрах виджета (никаких "fields"/"comment") --
// просто прикрепить эту выборку к виджету хлебных крошек на странице ТЭП, и она сама
// разберётся.
//
// ДОМЕННЫЙ result_type (total|plan|fact|mandatory) читается ИЗ URL страницы ТЭП --
// GetRequestUrlSafe()/GetQueryParam() СКОПИРОВАНЫ ДОСЛОВНО из HREDU-183_tep_reports.js/
// HREDU-182_procent_obuchennyh.js (подтверждены рабочими именно в контексте "выборки").
// Если параметра нет в URL -- дефолт "total", ТО ЖЕ поведение, что и в самом отчёте ТЭП.
// =====================================================================

DEBUG = true;              // На проде поставить false после тестирования
LOG_NAME = "agent";        // TODO: заполнить после создания документа выборки в админке
CUR_OBJECT_ID = 0;         // TODO: заполнить ID документа выборки после её создания в админке (LogAlert защищена try/catch -- забытый 0 не обрушит код)

// Относительная ссылка на страницу "Процент обученных" -- ТОЧНО такой же приём (без
// домена), как TEP_REPORT_PAGE_URL в HREDU-182_procent_obuchennyh.js/BuildTepLink().
PERCENT_PAGE_URL = "/view_doc.html?mode=matrix_educated_percent";

//-------------------------------------------------------------------------
//              Область функций
//-------------------------------------------------------------------------

function LogAlert(typeLog, message)
{
    try
    {
        tools.call_code_library_method("vtbl_log_lib", "LogAlert", [LOG_NAME, typeLog, CUR_OBJECT_ID, message, DEBUG]);
    }
    catch (_exLog)
    {
        // ничего -- сбой логирования не должен ронять основной код
    }
}

/*
 * Достаёт полный URL текущей страницы. СКОПИРОВАНО ДОСЛОВНО из HREDU-183_tep_reports.js/
 * HREDU-182_procent_obuchennyh.js -- подтверждена рабочей именно в контексте "выборки".
 * @returns {string}
 */
function GetRequestUrlSafe()
{
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
 * Вырезает значение GET-параметра из полного URL строки -- без regex, без методов строк
 * (их нет в этом движке), через штатный API платформы (StrOptSubStrPos/StrRangePos/
 * StrLen), декодирование через UrlDecode(). СКОПИРОВАНО ДОСЛОВНО из
 * HREDU-183_tep_reports.js/HREDU-182_procent_obuchennyh.js.
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
 * Маппинг доменного result_type страницы ТЭП в текст крошки -- та же семантика
 * "Тип отчёта", что и ResolveResultTypeLabel() в HREDU-183_182_info_makroregion_matrica.js.
 * @param {string} sResultType   -   "total"|"plan"|"fact"|"mandatory".
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

//-------------------------------------------------------------------------
//              Точка входа
//-------------------------------------------------------------------------

RESULT = [];

try
{
    var sFullUrl, sReportResultType, sResultTypeLabel;

    sFullUrl = GetRequestUrlSafe();
    sReportResultType = GetQueryParam(sFullUrl, "result_type");
    if (sReportResultType == "")
    {
        sReportResultType = "total";
    }
    sResultTypeLabel = ResolveResultTypeLabel(sReportResultType);

    LogAlert(1, "Breadcrumbs (ТЭП). Request.Url=[" + sFullUrl + "] result_type=[" + sReportResultType + "] -> label=[" + sResultTypeLabel + "]");

    // Уровень 1 -- кликабельная крошка на страницу "Процент обученных".
    // ДОБАВЛЕНО (21.09.2026, реальный тест): битая ссылка вида
    // "https://.../%D0%9E%D1%82%D1%87%D0%B5%D1%82..." (домен + urlencoded "name")
    // показала, что виджет крошек строит href НЕ из "value" -- похоже, ему нужно
    // ОТДЕЛЬНОЕ поле (по аналогии с "Табличными данными", где строка кликается по
    // отдельному полю "link", а не по "value"/"name"). Пока не подтверждено ТОЧНОЕ имя
    // поля, кладём ссылку СРАЗУ в оба вероятных поля ("value" и "link") -- лишнее поле
    // в объекте RESULT ничего не ломает (уже проверено на других выборках этого
    // тикета), а виджет возьмёт то, которое реально привязано в LPE.
    RESULT.push({
        "name": "Отчет процент обученных",
        "value": PERCENT_PAGE_URL,
        "link": PERCENT_PAGE_URL,
        "id": ArrayCount(RESULT)
    });

    // Уровень 2 -- текущий тип отчёта ТЭП, некликабельный (текущая страница).
    RESULT.push({
        "name": "Отчет " + sResultTypeLabel,
        "value": "",
        "link": "",
        "id": ArrayCount(RESULT)
    });
}
catch (_ex)
{
    // Сбой хлебных крошек не должен ронять всю страницу отчёта -- просто не покажем крошку.
    RESULT = [];
    try
    {
        LogAlert(4, "Breadcrumbs (ТЭП). ОШИБКА: " + ExtractUserError(_ex));
    }
    catch (_exLog2)
    {
        // не роняем код из-за сбоя логирования самой ошибки
    }
}
