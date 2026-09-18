
// HREDU-183 / HREDU-182. Инфо-блок "Макрорегион / Матрица обучения" -- выборка для
// Табличных данных.
//
// НАЗНАЧЕНИЕ (17.09.2026, по прямой просьбе пользователя; РАСШИРЕНО 18.09.2026 -- добавлены
// город и учебная программа, тоже по прямой просьбе пользователя): на страницах отчётов
// ("ТЭП" -- HREDU-183_tep_reports.js, и "Процент обученных" -- HREDU-182_procent_obuchennyh.js)
// нужен ещё один, отдельный от таблицы с данными, блок -- просто список ("нередактируемый"),
// который показывает пользователю, какие сейчас выбраны фильтры (макрорегион, город,
// матрица обучения, учебная программа), а не данные отчёта. Пример желаемого вида на
// странице:
//   Макрорегион: Москва
//   Город: Воронеж
//   Матрица обучения: Матрица ССП Тестовая
//   Учебная программа: Основы делового общения
// Выше этого блока -- кнопка "Настроить фильтры" (открывает модалку, HREDU-183_filtry_modal_shag1.js
// или HREDU_182_filtry_percent.js), ниже -- сама таблица отчёта.
//
// КАК УСТРОЕНО: это ОТДЕЛЬНАЯ выборка (не часть HREDU-183_tep_reports.js/
// HREDU-182_procent_obuchennyh.js) -- вешается на СВОЙ виджет "Табличные данные" рядом с
// основной таблицей на каждой из страниц. Она читает Request.Url ТЕКУЩЕЙ страницы (тот же
// приём, что и в остальных выборках этого тикета -- см. GetRequestUrlSafe()/GetQueryParam()
// ниже, идентичны версиям в HREDU-183_tep_reports.js/HREDU-182_procent_obuchennyh.js) и
// достаёт из URL четыре параметра:
//   macroregion -- ТЕКСТОВОЕ значение (с 16.09.2026 это обычный текстовый фильтр, без
//                  привязки к справочнику -- см. HREDU-183_filtry_modal_shag1.js/
//                  HREDU_182_filtry_percent.js) -- берётся из URL "как есть", резолвить
//                  в отдельный документ не нужно.
//   city        -- ТЕКСТОВОЕ значение (та же природа, что и macroregion, см. HREDU-183_filtry_modal_shag1.js) --
//                  берётся из URL "как есть", без похода в базу.
//   matrix_id   -- ID документа cc_learning_matrice -- резолвится в НАЗВАНИЕ матрицы через
//                  tools.open_doc() (тот же приём, что для матрицы в HREDU-183_tep_reports.js:
//                  matrixDoc = tools.open_doc(matrixId).TopElem; matrixName = String(matrixDoc.name);).
//   program_id  -- ID документа education_method (это и есть "Учебная программа" -- см.
//                  GetProgramTitles() в HREDU-183_tep_reports.js) -- резолвится в название
//                  программы тем же приёмом через tools.open_doc().
//
// ОДНА И ТА ЖЕ выборка подходит для ОБЕИХ страниц без изменений -- оба модальных окна
// фильтров кладут значения в URL под одними и теми же именами параметров (macroregion,
// city, matrix_id, program_id), см. HREDU-183_filtry_modal_shag1.js и HREDU_182_filtry_percent.js.
// ЗАМЕЧАНИЕ: на странице "Процент обученных" фильтра program_id в модалке нет (там нет
// выбора конкретной программы -- отчёт сразу показывает все программы матрицы построчно),
// поэтому там "Учебная программа" в этом блоке всегда будет "-" -- это ожидаемо, не баг.
//
// ФОРМАТ RESULT: по образцу выборки-примера, которую прислал пользователь (education_accept_event_card,
// result_type="fields") -- плоский массив строк { name, value, id }, где name -- подпись
// строки, value -- значение. Ровно ЧЕТЫРЕ строки, порядок фиксирован:
//   [0] { name: "Макрорегион",        value: <текст макрорегиона или "-">        , id: 0 }
//   [1] { name: "Город",              value: <текст города или "-">              , id: 1 }
//   [2] { name: "Матрица обучения",   value: <название матрицы или "-">          , id: 2 }
//   [3] { name: "Учебная программа",  value: <название программы или "-">        , id: 3 }
// Если параметра нет в URL (фильтр ещё не выбран/страница открыта "с нуля") -- value = "-",
// а не пустая строка, чтобы в блоке не было визуально "провисающей" пустоты после
// двоеточия.
//
// В ЭТОЙ выборке НЕТ параметра result_type и НЕТ switch по нему (в отличие от примера
// пользователя) -- она всегда отдаёт один и тот же список из 2 строк, других режимов не
// предусмотрено.

DEBUG = true;              // На проде поставить false после тестирования
LOG_NAME = "agent";        // TODO: уточнить после создания документа в админке
CUR_OBJECT_ID = 0;         // TODO: заполнить ID документа выборки после её создания в админке

//-------------------------------------------------------------------------
//              Область функций
//-------------------------------------------------------------------------

function LogAlert(typeLog, message)
{
    // См. подробное объяснение этой защиты в HREDU-183_tep_reports.js -- логирование не
    // должно ронять основной код, если LOG_NAME/CUR_OBJECT_ID ещё не настроены в админке.
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
 * Достаёт полный URL текущей страницы. В контексте ВЫБОРКИ (не удалённого действия)
 * Request.Url надёжен -- см. HREDU-183_diagnostic_get_params.js, идентична версии в
 * HREDU-183_tep_reports.js/HREDU-182_procent_obuchennyh.js.
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
 * (.indexOf/.substring в этом движке нет), через штатный строковый API платформы.
 * Идентична версии в HREDU-183_tep_reports.js/HREDU-182_procent_obuchennyh.js/
 * HREDU-181_vostok_polny_spisok_draft.js.
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
 * Резолвит ID документа cc_learning_matrice в его название (это и есть "Матрица
 * обучения" на странице). Обёрнут в try/catch по образцу ResolveMirCodeText() в
 * HREDU-183_filtry_modal_shag1.js -- если матрицы с таким ID уже нет (удалена) или
 * matrix_id вообще не передан/некорректен, тихо возвращает "" (а не роняет всю выборку).
 * @param {number} iMatrixID
 * @returns {string}   -   Название матрицы или "" если не найдена/не выбрана.
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
        return "";
    }
}

/*
 * Резолвит ID документа education_method в его название (это и есть "Учебная программа"
 * на странице) -- тот же приём, что GetMatrixNameSafe() выше и GetProgramTitles() в
 * HREDU-183_tep_reports.js. Обёрнут в try/catch -- если программы с таким ID уже нет или
 * program_id не передан/некорректен, тихо возвращает "" (а не роняет всю выборку).
 * ДОБАВЛЕНО (18.09.2026, по прямой просьбе пользователя).
 * @param {number} iProgramID
 * @returns {string}   -   Название программы или "" если не найдена/не выбрана.
 */
function GetProgramNameSafe(iProgramID)
{
    if (OptInt(iProgramID, 0) <= 0)
    {
        return "";
    }
    try
    {
        return String(tools.open_doc(Int(iProgramID)).TopElem.name);
    }
    catch (_ex)
    {
        return "";
    }
}

/*
 * Точка входа. Собирает инфо-блок из 4 строк (макрорегион, город, матрица обучения,
 * учебная программа) по текущим значениям фильтров из URL страницы.
 * @returns {void}
 */
function Run()
{
    var sFullUrl, sMacroregion, sCity, iMatrixID, sMatrixName, iProgramID, sProgramName;

    ERROR = 0;
    MESSAGE = "";
    RESULT = [];

    try
    {
        LogAlert(2, "Run(). НАЧАЛО");

        sFullUrl = GetRequestUrlSafe();
        LogAlert(1, "Run(). Request.Url = [" + sFullUrl + "]");

        sMacroregion = GetQueryParam(sFullUrl, "macroregion");
        sCity = GetQueryParam(sFullUrl, "city");
        iMatrixID = OptInt(GetQueryParam(sFullUrl, "matrix_id"), 0);
        sMatrixName = GetMatrixNameSafe(iMatrixID);
        iProgramID = OptInt(GetQueryParam(sFullUrl, "program_id"), 0);
        sProgramName = GetProgramNameSafe(iProgramID);
        LogAlert(1, "Run(). macroregion=[" + sMacroregion + "] city=[" + sCity + "] matrix_id=" + iMatrixID
            + " matrixName=[" + sMatrixName + "] program_id=" + iProgramID + " programName=[" + sProgramName + "]");

        RESULT.push({
            name: "Макрорегион",
            value: (sMacroregion != "" ? sMacroregion : "-"),
            id: ArrayCount(RESULT)
        });
        RESULT.push({
            name: "Город",
            value: (sCity != "" ? sCity : "-"),
            id: ArrayCount(RESULT)
        });
        RESULT.push({
            name: "Матрица обучения",
            value: (sMatrixName != "" ? sMatrixName : "-"),
            id: ArrayCount(RESULT)
        });
        RESULT.push({
            name: "Учебная программа",
            value: (sProgramName != "" ? sProgramName : "-"),
            id: ArrayCount(RESULT)
        });

        LogAlert(2, "Run(). Готово. Строк: " + ArrayCount(RESULT));
    }
    catch (_ex)
    {
        ERROR = 1;
        MESSAGE = ExtractUserError(_ex);
        LogAlert(4, "Run(). ОШИБКА: " + MESSAGE);
    }
    LogAlert(2, "Run(). КОНЕЦ");
}

//-------------------------------------------------------------------------
//              Область основного кода
//-------------------------------------------------------------------------

Run();
