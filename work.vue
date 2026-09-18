// HREDU-183 / HREDU-182. Инфо-блок "Макрорегион / Матрица обучения" -- выборка для
// Табличных данных.
//
// НАЗНАЧЕНИЕ (17.09.2026, по прямой просьбе пользователя; РАСШИРЕНО 18.09.2026 -- добавлены
// город, учебная программа и тип отчёта, тоже по прямой просьбе пользователя): на
// страницах отчётов ("ТЭП" -- HREDU-183_tep_reports.js, и "Процент обученных" --
// HREDU-182_procent_obuchennyh.js) нужен ещё один, отдельный от таблицы с данными, блок --
// просто список ("нередактируемый"), который показывает пользователю, какие сейчас
// выбраны фильтры (макрорегион, город, матрица обучения, учебная программа, тип отчёта),
// а не данные отчёта. Пример желаемого вида на странице:
//   Макрорегион: Москва
//   Город: Воронеж
//   Матрица обучения: Матрица ССП Тестовая
//   Учебная программа: Основы делового общения
//   Тип отчёта: Факт
// Выше этого блока -- кнопка "Настроить фильтры" (открывает модалку, HREDU-183_filtry_modal_shag1.js
// или HREDU_182_filtry_percent.js), ниже -- сама таблица отчёта.
//
// ВАЖНОЕ ЗАМЕЧАНИЕ (18.09.2026, по вопросу пользователя про macroregion на странице ТЭП):
// "Макрорегион" в этом блоке -- это ЗНАЧЕНИЕ ФИЛЬТРА из URL (параметр macroregion), а НЕ то
// же самое, что колонка "Макрорегион" в самой таблице отчёта ТЭП. В таблице ТЭП колонка
// "Макрорегион" -- это АТРИБУТ КАЖДОГО СОТРУДНИКА (row.macroregion = FindMacroregion(macroRows,
// id) в HREDU-183_tep_reports.js) -- у каждой строки-сотрудника СВОЙ макрорегион, взятый из
// его личных данных, и он показывается ПРАВИЛЬНО независимо от того, передан ли где-то в URL
// единый фильтр "macroregion". Фильтр же macroregion -- это ДОПОЛНИТЕЛЬНОЕ ручное сужение
// списка (см. sMacroregionFilter в HREDU-183_tep_reports.js: если он пустой -- фильтрации по
// макрорегиону просто нет, показываются сотрудники ЛЮБОГО макрорегиона, каждый со своим
// правильным значением в колонке). Раз на странице ТЭП поле macroregion сейчас СКРЫТО в
// модалке (см. правку от 17.09.2026 в HREDU-183_filtry_modal_shag1.js -- видимым остался
// только "Режим отчёта"), единственный способ попасть на страницу ТЭП с macroregion в URL --
// прийти по прямой ссылке, где он уже прописан (например, drill-down из "Процент обученных"
// сейчас не передаёт macroregion в ссылку на ТЭП -- см. BuildTepLink() в
// HREDU-182_procent_obuchennyh.js). Поэтому "-" в этом блоке для "Макрорегион" на странице
// ТЭП -- ОЖИДАЕМОЕ поведение (фильтр правда не выбран), а не ошибка выборки: таблица при
// этом всё равно корректно показывает макрорегион К А Ж Д О Г О сотрудника в своей колонке,
// т.к. эта колонка не зависит от фильтра. Если нужно, чтобы блок тоже показывал macroregion в
// такой ситуации -- варианты: (а) добавить обратно поле "macroregion" в модалку ТЭП
// (см. закомментированный блок в HREDU-183_filtry_modal_shag1.js), либо (б) прокинуть
// macroregion в drill-down ссылку из "Процент обученных" в BuildTepLink(), если он там
// известен на момент клика -- скажи, какой вариант нужен, сделаю.
//
// КАК УСТРОЕНО: это ОТДЕЛЬНАЯ выборка (не часть HREDU-183_tep_reports.js/
// HREDU-182_procent_obuchennyh.js) -- вешается на СВОЙ виджет "Табличные данные" рядом с
// основной таблицей на каждой из страниц. Она читает Request.Url ТЕКУЩЕЙ страницы (тот же
// приём, что и в остальных выборках этого тикета -- см. GetRequestUrlSafe()/GetQueryParam()
// ниже, идентичны версиям в HREDU-183_tep_reports.js/HREDU-182_procent_obuchennyh.js) и
// достаёт из URL пять параметров:
//   macroregion -- ТЕКСТОВОЕ значение (с 16.09.2026 это обычный текстовый фильтр, без
//                  привязки к справочнику -- см. HREDU-183_filtry_modal_shag1.js/
//                  HREDU_182_filtry_percent.js) -- берётся из URL "как есть", резолвить
//                  в отдельный документ не нужно. См. ВАЖНОЕ ЗАМЕЧАНИЕ выше про смысл поля.
//   city        -- ТЕКСТОВОЕ значение (та же природа, что и macroregion, см. HREDU-183_filtry_modal_shag1.js) --
//                  берётся из URL "как есть", без похода в базу.
//   matrix_id   -- ID документа cc_learning_matrice -- резолвится в НАЗВАНИЕ матрицы через
//                  tools.open_doc() (тот же приём, что для матрицы в HREDU-183_tep_reports.js:
//                  matrixDoc = tools.open_doc(matrixId).TopElem; matrixName = String(matrixDoc.name);).
//   program_id  -- ID документа education_method (это и есть "Учебная программа" -- см.
//                  GetProgramTitles() в HREDU-183_tep_reports.js) -- резолвится в название
//                  программы тем же приёмом через tools.open_doc().
//   result_type -- ДОБАВЛЕНО (18.09.2026, по прямой просьбе пользователя): код режима отчёта
//                  ТЭП (total/plan/fact/mandatory, см. поле "result_type" в
//                  HREDU-183_filtry_modal_shag1.js) -- резолвится в ЧИТАЕМУЮ русскую подпись
//                  через ResolveResultTypeLabel() ниже (те же подписи, что в select-е
//                  модалки: "Общее кол-во"/"План"/"Факт"/"Обязательно к прохождению"). Если
//                  параметра нет в URL -- по умолчанию считаем "total" (см. тот же дефолт в
//                  HREDU-183_tep_reports.js/HREDU-183_filtry_modal_shag1.js), а не "-": на
//                  странице ТЭП отчёт ВСЕГДА показывается в каком-то режиме, даже если
//                  явно не выбран.
//
// ОДНА И ТА ЖЕ выборка подходит для ОБЕИХ страниц без изменений -- оба модальных окна
// фильтров кладут значения в URL под одними и теми же именами параметров (macroregion,
// city, matrix_id, program_id, result_type), см. HREDU-183_filtry_modal_shag1.js и
// HREDU_182_filtry_percent.js. ЗАМЕЧАНИЯ:
//   - на странице "Процент обученных" фильтра program_id в модалке нет (там нет выбора
//     конкретной программы -- отчёт сразу показывает все программы матрицы построчно),
//     поэтому там "Учебная программа" в этом блоке всегда будет "-" -- это ожидаемо, не баг.
//   - на странице "Процент обученных" параметра result_type тоже нет вообще (см. шапку
//     HREDU_182_filtry_percent.js -- она его не читает и не пишет), поэтому там "Тип
//     отчёта" всегда будет отображать дефолт "Общее кол-во" -- само значение там не имеет
//     смысла (отчёт "Процент обученных" не разбит на режимы), это ожидаемо.
//
// ФОРМАТ RESULT: по образцу выборки-примера, которую прислал пользователь (education_accept_event_card,
// result_type="fields") -- плоский массив строк { name, value, id }, где name -- подпись
// строки, value -- значение. Ровно ПЯТЬ строк, порядок фиксирован:
//   [0] { name: "Макрорегион",        value: <текст макрорегиона или "-">                  , id: 0 }
//   [1] { name: "Город",              value: <текст города или "-">                        , id: 1 }
//   [2] { name: "Матрица обучения",   value: <название матрицы или "-">                    , id: 2 }
//   [3] { name: "Учебная программа",  value: <название программы или "-">                  , id: 3 }
//   [4] { name: "Тип отчёта",         value: <русская подпись режима, по умолчанию "Общее кол-во"> , id: 4 }
// Если параметра нет в URL (фильтр ещё не выбран/страница открыта "с нуля") -- value = "-",
// а не пустая строка (КРОМЕ "Тип отчёта" -- там дефолт "Общее кол-во", см. выше), чтобы в
// блоке не было визуально "провисающей" пустоты после двоеточия.
//
// В ЭТОЙ выборке НЕТ параметра result_type у СЕБЯ САМОЙ (нет switch, который бы переключал,
// ЧТО отдавать, в отличие от примера пользователя, где result_type определял набор полей) --
// result_type здесь используется только КАК ОДНО ИЗ ЗНАЧЕНИЙ для отображения (строка [4]),
// а не как переключатель поведения выборки. Она всегда отдаёт один и тот же список из 5
// строк, других режимов работы не предусмотрено.

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
 * Резолвит код result_type (total/plan/fact/mandatory) в ту же русскую подпись, что
 * используется в select-е "Режим отчёта" в HREDU-183_filtry_modal_shag1.js (entries: name/
 * value) -- чтобы в инфо-блоке был читаемый текст, а не техническое имя параметра. Пустая
 * строка/неизвестное значение -- считаем дефолтом "total" (та же логика по умолчанию, что
 * в HREDU-183_tep_reports.js: если result_type не передан, отчёт всё равно показывается,
 * просто в режиме "Общее кол-во"). ДОБАВЛЕНО (18.09.2026, по прямой просьбе пользователя).
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
        default: { return "Общее кол-во"; } // дефолт -- см. комментарий выше
    }
}

/*
 * Точка входа. Собирает инфо-блок из 5 строк (макрорегион, город, матрица обучения,
 * учебная программа, тип отчёта) по текущим значениям фильтров из URL страницы.
 * @returns {void}
 */
function Run()
{
    var sFullUrl, sMacroregion, sCity, iMatrixID, sMatrixName, iProgramID, sProgramName;
    var sResultType, sResultTypeLabel;

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
        sResultType = GetQueryParam(sFullUrl, "result_type");
        sResultTypeLabel = ResolveResultTypeLabel(sResultType);
        LogAlert(1, "Run(). macroregion=[" + sMacroregion + "] city=[" + sCity + "] matrix_id=" + iMatrixID
            + " matrixName=[" + sMatrixName + "] program_id=" + iProgramID + " programName=[" + sProgramName + "]"
            + " result_type=[" + sResultType + "] resultTypeLabel=[" + sResultTypeLabel + "]");

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
        RESULT.push({
            name: "Тип отчёта",
            value: sResultTypeLabel,
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
