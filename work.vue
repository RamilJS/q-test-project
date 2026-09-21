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

DEBUG = true;              // На проде поставить false после тестирования
LOG_NAME = "agent";        // TODO: заполнить после создания документа выборки в админке
CUR_OBJECT_ID = 0;         

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

/*
 * ДОБАВЛЕНО (21.09.2026, реальный тест -- битая ссылка на саму себя): та же самая
 * "&macr"-ловушка, что уже была найдена и исправлена в BuildTepLink()
 * (HREDU-182_procent_obuchennyh.js, см. подробное объяснение там). Виджет вставляет
 * значение поля ссылки ПРЯМО в HTML-атрибут href БЕЗ экранирования "&" в "&amp;" --
 * поэтому "&macroregion=Москва" браузер читает как легаси-HTML-сущность "&macr"
 * (символ "¯", без точки с запятой на конце, как у "&amp"/"&lt"/"&nbsp") + "oregion=...".
 * Реальный симптом СОВПАДАЕТ один в один: "matrix_id=...%C2%AForegion=..." --
 * %C2%AF это UTF-8 для U+00AF (macron), "&macr" пропало, "oregion=" осталось. Фикс --
 * ТОТ ЖЕ САМЫЙ: экранируем "&" в "&amp;" САМИ, ДО того как класть ссылку в RESULT --
 * тогда браузер сначала раскодирует "&amp;" обратно в "&", и итоговый URL уже не
 * содержит "&macr" как отдельную подстроку. Без regex -- тот же строковый API
 * (StrOptSubStrPos/StrRangePos/StrLen), что и в GetQueryParam().
 * @param {string} sUrl
 * @returns {string}
 */
function HtmlEscapeAmp(sUrl)
{
    var sResult, iPos, iUrlLen, iSearchStart;
    sResult = "";
    iSearchStart = 0;
    iUrlLen = StrLen(sUrl);
    while (true)
    {
        iPos = StrOptSubStrPos(sUrl, "&", false, iSearchStart);
        if (iPos == undefined)
        {
            sResult = sResult + StrRangePos(sUrl, iSearchStart, iUrlLen);
            break;
        }
        sResult = sResult + StrRangePos(sUrl, iSearchStart, iPos) + "&amp;";
        iSearchStart = iPos + 1;
    }
    return sResult;
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
        "value": HtmlEscapeAmp(PERCENT_PAGE_URL),
        "link": HtmlEscapeAmp(PERCENT_PAGE_URL),
        "id": ArrayCount(RESULT)
    });

    // Уровень 2 -- текущий тип отчёта ТЭП.
    // ИСПРАВЛЕНО (21.09.2026, реальный тест): пустая строка "" в value/link вела на
    // ГОЛЫЙ ДОМЕН (https://als-devwt.vl.vtb, без пути) -- то есть виджет крошек, похоже,
    // ВСЕГДА оборачивает крошку в кликабельную ссылку, даже когда link/value пустой (то
    // есть просто убрать/закомментировать эти поля здесь не поможет -- по этой же
    // причине виджет вообще был кликабелен и раньше). Пользователь подтвердил, что
    // приемлемы ОБА варианта -- некликабельно ИЛИ ссылка на ТЕКУЩУЮ страницу -- поэтому
    // САМЫЙ НАДЁЖНЫЙ вариант (не зависящий от того, как именно виджет обрабатывает
    // пустое значение) -- это ссылка САМА НА СЕБЯ: sFullUrl (уже вычислен выше через
    // GetRequestUrlSafe()) -- реальный текущий адрес страницы СО ВСЕМИ текущими
    // фильтрами. Клик по этой крошке просто перезагружает ту же страницу с теми же
    // параметрами -- для пользователя неотличимо от "некликабельно".
    RESULT.push({
        "name": "Отчет " + sResultTypeLabel,
        "value": HtmlEscapeAmp(sFullUrl),
        "link": HtmlEscapeAmp(sFullUrl),
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
