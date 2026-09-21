sLogName = 'HREDU_182_183_breadcrumbs';
EnableLog(sLogName, true);
function alert(sInputObj)
{
    LogEvent(sLogName, sInputObj);
    return sInputObj;
}

// =====================================================================
// HREDU-182/183. ОДНА универсальная выборка хлебных крошек НА ОБЕ страницы отчётов --
// "Процент обученных" (HREDU-182_procent_obuchennyh.js) и ТЭП (HREDU-183_tep_reports.js).
//
// ИЗМЕНЕНО (21.09.2026, по прямому замечанию пользователя): раньше было 2 отдельных
// файла (по одному на страницу) -- пользователь явно попросил ОДНУ выборку, ставящуюся
// на оба виджета, по образцу присланного примера ("Заявка на подбор №...", один файл на
// разные "типы" через switch(result_type)). Объединено сюда, разделение теперь идёт
// ВНУТРИ одного Run()-блока по URL текущей страницы, а не по разным файлам.
//
// ФОРМАТ КРОШКИ: "Отчет Процент обученных" (страница "Процент обученных", ВСЕГДА
// статично -- у неё нет режимов result_type в URL, см. HREDU-182_procent_obuchennyh.js)
// либо "Отчет План" / "Отчет Факт" / "Отчет Общее кол-во" / "Отчет Обязательно к
// прохождению" (страница ТЭП, ДИНАМИЧЕСКИ, по текущему result_type в её URL, см.
// HREDU-183_tep_reports.js).
//
// КАК ОПРЕДЕЛЯЕМ, НА КАКОЙ МЫ СТРАНИЦЕ (одна выборка -- значит нужно самим понять,
// откуда её вызвали): по параметру "mode" в Request.Url -- ТОЧНЫЕ адреса подтверждены
// пользователем 21.09.2026:
//   "Процент обученных"  -- https://als-devwt.vl.vtb/view_doc.html?mode=matrix_educated_percent
//   ТЭП (План/Факт/...)  -- https://als-devwt.vl.vtb/view_doc.html?mode=matrix_report
// ИСПРАВЛЕНО (21.09.2026): раньше проверялось ТОЛЬКО "есть ли mode=matrix_report" --
// если нет, по умолчанию считалось, что это страница "Процент обученных". Реальный тест
// показал, что крошка на "Процент обученных" не работала -- ПРИЧИНА могла быть либо в
// том, что виджет на этой странице ещё не настроен в админке (result_type[внешний] не
// привязан к "fields"), либо в том, что на проде URL отличался от того, что
// предполагалось изначально (просто "не matrix_report"). Теперь ОБЕ страницы
// определяются ЯВНО, по СВОИМ ТОЧНЫМ значениям "mode=" -- никаких "по умолчанию"
// веток: если mode не совпал НИ С ОДНИМ из двух известных значений, крошка НЕ строится
// вообще (пустой RESULT, а не ошибочный текст) -- так сразу видно в логе (см.
// LogAlert() ниже с самим Request.Url), если появится третья страница или mode
// поменяется снова, вместо того чтобы молча показывать неверную крошку.
//
// ВНИМАНИЕ, ДВА РАЗНЫХ "result_type" В ЭТОМ ФАЙЛЕ (легко перепутать):
//   1. ВНЕШНИЙ result_type (переменная sResType ниже) -- служебный LPE-параметр самого
//      ВИДЖЕТА хлебных крошек, платформенный паттерн "fields"/"comment" (см. пример
//      "Заявка на подбор №...", присланный пользователем 21.09.2026, и
//      HREDU-183_182_info_makroregion_matrica.js, где тот же паттерн уже встречался).
//      Переключает, строить ли крошку вообще.
//   2. ДОМЕННЫЙ result_type (переменная sReportResultType ниже) -- значение ИЗ URL
//      САМОЙ СТРАНИЦЫ ТЭП (total|plan|fact|mandatory, см. Run() в
//      HREDU-183_tep_reports.js). Читается ИСКЛЮЧИТЕЛЬНО из Request.Url -- та же пара
//      GetRequestUrlSafe()/GetQueryParam(), СКОПИРОВАННАЯ ДОСЛОВНО из
//      HREDU-183_tep_reports.js/HREDU-182_procent_obuchennyh.js (подтверждена рабочей
//      именно в контексте "выборки"). Если в URL параметра нет -- по умолчанию "total",
//      ТО ЖЕ поведение, что и в самом отчёте ТЭП. Используется ТОЛЬКО когда мы уже
//      определили, что находимся на странице ТЭП (см. выше) -- на странице "Процент
//      обученных" эта переменная не читается вообще.
//
// Матрица (matrix_id) здесь НЕ резолвится и tools.open_doc() не вызывается ни для одной
// из страниц -- подтверждено пользователем, крошка ТЭП показывает только тип отчёта, без
// названия матрицы -- поэтому выборка лёгкая и быстрая, без единого SQL/XQuery-запроса.
// =====================================================================

DEBUG = true;              // На проде поставить false после тестирования
LOG_NAME = "agent";        // TODO: заполнить после создания документа выборки в админке
CUR_OBJECT_ID = 0;         // TODO: заполнить ID документа выборки после её создания в админке (LogAlert защищена try/catch -- забытый 0 не обрушит код)

// ТОЧНЫЕ значения "mode" для каждой из двух страниц -- подтверждены пользователем
// 21.09.2026 реальными production-адресами (см. "КАК ОПРЕДЕЛЯЕМ, НА КАКОЙ МЫ СТРАНИЦЕ"
// в шапке файла).
PERCENT_PAGE_MODE = "mode=matrix_educated_percent";
TEP_PAGE_MODE = "mode=matrix_report";

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
 * HREDU-182_procent_obuchennyh.js -- подтверждена рабочей именно в контексте "выборки"
 * (Request.Url ненадёжен в удалённых действиях, но здесь работает).
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
 * "Тип отчёта", что и ResolveResultTypeLabel() в HREDU-183_182_info_makroregion_matrica.js
 * (специально продублирована здесь, а не вызвана оттуда -- каждая выборка в этом проекте
 * самодостаточна и не зависит от других файлов, см. установленную конвенцию тикета).
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
    var sResType, sFullUrl, bIsTepPage, bIsPercentPage, sReportResultType, sResultTypeLabel;

    // ВНЕШНИЙ result_type -- см. "ВНИМАНИЕ, ДВА РАЗНЫХ result_type" в шапке файла (п.1).
    sResType = result_type;
    LogAlert(1, "Breadcrumbs (общая). result_type[внешний, виджет]=[" + sResType + "]");

    switch (sResType)
    {
        case "fields":
        {
            sFullUrl = GetRequestUrlSafe();

            // Определяем страницу ЯВНО по ОБОИМ известным значениям "mode" -- см.
            // "ИСПРАВЛЕНО (21.09.2026)" в шапке файла: никаких "по умолчанию" веток.
            bIsTepPage = (StrOptSubStrPos(sFullUrl, TEP_PAGE_MODE, false) != undefined);
            bIsPercentPage = (StrOptSubStrPos(sFullUrl, PERCENT_PAGE_MODE, false) != undefined);
            LogAlert(1, "Breadcrumbs (общая). Request.Url=[" + sFullUrl + "] bIsTepPage=" + bIsTepPage + " bIsPercentPage=" + bIsPercentPage);

            if (bIsTepPage)
            {
                // ДОМЕННЫЙ result_type -- см. "ВНИМАНИЕ, ДВА РАЗНЫХ result_type" (п.2).
                sReportResultType = GetQueryParam(sFullUrl, "result_type");
                if (sReportResultType == "")
                {
                    sReportResultType = "total";
                }
                sResultTypeLabel = ResolveResultTypeLabel(sReportResultType);
                LogAlert(1, "Breadcrumbs (ТЭП). result_type[доменный, из URL]=[" + sReportResultType + "] -> label=[" + sResultTypeLabel + "]");

                RESULT.push({
                    "name": "Отчет " + sResultTypeLabel,
                    "value": "",
                    "id": ArrayCount(RESULT)
                });
            }
            else if (bIsPercentPage)
            {
                // Страница "Процент обученных" -- крошка ВСЕГДА статична, см. шапку файла.
                RESULT.push({
                    "name": "Отчет Процент обученных",
                    "value": "",
                    "id": ArrayCount(RESULT)
                });
            }
            else
            {
                // НИ ОДИН из двух известных mode не совпал -- см. "ИСПРАВЛЕНО
                // (21.09.2026)" в шапке файла: намеренно НЕ строим крошку "на всякий
                // случай", чтобы не показать неверный текст молча -- лучше пустая
                // крошка + строка в логе с реальным Request.Url для диагностики.
                LogAlert(3, "Breadcrumbs (общая). Ни один известный mode не совпал -- крошка не построена. Request.Url=[" + sFullUrl + "]");
            }
            break;
        }
        default:
        {
            break;
        }
    }
}
catch (_ex)
{
    // Сбой хлебных крошек не должен ронять всю страницу отчёта -- просто не покажем крошку.
    RESULT = [];
    try
    {
        LogAlert(4, "Breadcrumbs (общая). ОШИБКА: " + ExtractUserError(_ex));
    }
    catch (_exLog2)
    {
        // не роняем код из-за сбоя логирования самой ошибки
    }
}
