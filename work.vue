

DEBUG = true;              // На проде поставить false после тестирования
LOG_NAME = "agent";        // TODO: заполнить после создания документа в админке
CUR_OBJECT_ID = 0;         // TODO: заполнить ID документа выборки после её создания в админке (LogAlert защищена try/catch -- забытый 0 не обрушит Run())

// TODO (14.09.2026, ОБЯЗАТЕЛЬНО ЗАПОЛНИТЬ ПЕРЕД ИСПОЛЬЗОВАНИЕМ КЛИКАБЕЛЬНОСТИ): реальный
// адрес страницы ТЭП-отчётов (та же страница, где сейчас 4 виджета ТЭП + кнопка
// "Настроить фильтры" на HREDU-183_filtry_modal_shag1.js). В тестах использовался
// "mode=matrix_test" -- если это НЕ финальная production-страница, замени на неё.
TEP_REPORT_PAGE_URL = "/view_doc.html?mode=matrix_report";

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

function GetRequestUrlSafe()
{
    try { return String(Request.Url); }
    catch (_ex) { return ""; }
}

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

function GetMatrixRows(matrixName)
{
    return ArraySelectAll(XQuery("for $elem in cc_learning_matrices where $elem/name = " + XQueryLiteral(matrixName) + " return $elem"));
}

function GetMatrixElementRows(matrixIds)
{
    return ArraySelectAll(XQuery("for $elem in cc_learning_matrice_elements where MatchSome($elem/cc_learning_matrice_id, (" + ArrayMerge(matrixIds, "This", ",") + ")) and $elem/is_active=true() return $elem"));
}

/*
 * ИСПРАВЛЕНО (15.09.2026, аварийное -- "Int(), Unknown source, line 113"): реальный тест
 * пользователя ("Матрица тест 2 программы") показал матрицу, у которой ВООБЩЕ НЕТ
 * собственного education_method_id (в XML поля нет вовсе -- обе программы заданы только
 * через её 2 элемента cc_learning_matrice_element с РАЗНЫМИ education_method_id). Это
 * ЗАКОННЫЙ, ожидаемый кейс (см. переписку 15.09.2026 про cc_learning_matrice_element) --
 * матрица может быть просто "контейнером", а сами программы -- только на элементах.
 * Старый код брал "Int(This.education_method_id)" БЕЗ защиты -- Int(undefined) падает с
 * "Unknown source". Фикс: OptInt(This.education_method_id, 0) вместо Int(...), плюс
 * отбрасываем нули при сборке allProgramIds (0 -- это "поле не заполнено", а не реальный
 * id программы, попадание 0 в programIds сломало бы SQL "in (...)" в GetCompletionDateRows()).
 */
function GetProgramIds(matrixRows, elementRows)
{
    var matrixProgramIds, elementProgramIds, allProgramIds, i;
    matrixProgramIds = ArrayExtract(matrixRows, "OptInt(This.education_method_id, 0)");
    elementProgramIds = ArrayExtract(elementRows, "OptInt(This.education_method_id, 0)");
    allProgramIds = [];
    for (i = 0; i < ArrayCount(matrixProgramIds); i++) { if (Int(matrixProgramIds[i]) > 0) { allProgramIds.push(matrixProgramIds[i]); } }
    for (i = 0; i < ArrayCount(elementProgramIds); i++) { if (Int(elementProgramIds[i]) > 0) { allProgramIds.push(elementProgramIds[i]); } }
    return ArraySelectDistinct(allProgramIds, "This");
}

function ResolveProgramIds(matrixId, matrixName)
{
    var matrixRows, matrixIds, elementRows, programIds;
    matrixRows = GetMatrixRows(matrixName);
    matrixIds = ArrayExtract(matrixRows, "Int(This.id)");
    if (ArrayCount(matrixIds) == 0)
    {
        throw ("Не найдено ни одной записи cc_learning_matrice с названием [" + matrixName + "]");
    }
    elementRows = GetMatrixElementRows(matrixIds);
    programIds = GetProgramIds(matrixRows, elementRows);
    if (ArrayCount(programIds) == 0)
    {
        throw ("У матрицы [" + matrixName + "] не найдено ни одной активной программы");
    }
    return programIds;
}

function GetActiveCollaboratorRows()
{
    return ArraySelectAll(XQuery("for $elem in collaborators where $elem/is_dismiss=false() return $elem"));
}

function GetPositionIdsByCommonPosition(iCommonPositionFilter)
{
    var positionRows, positionIds, i;
    positionRows = ArraySelectAll(XQuery("for $elem in positions where $elem/position_common_id = " + iCommonPositionFilter + " return $elem"));
    positionIds = [];
    for (i = 0; i < ArrayCount(positionRows); i++) { positionIds.push(Int(positionRows[i].id)); }
    return positionIds;
}

function IdArrayContains(idArray, value)
{
    var i;
    for (i = 0; i < ArrayCount(idArray); i++)
    {
        if (Int(idArray[i]) == Int(value)) { return true; }
    }
    return false;
}

function GetMacroregionRows()
{
    var sqlText;
    sqlText = "";
    sqlText = sqlText + "select cs.id,\r\n";
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_2ewj'']/value)[1]', 'varchar(max)') as macroregion\r\n";
    sqlText = sqlText + "from collaborators cs\r\n";
    sqlText = sqlText + "inner join collaborator c on c.id = cs.id\r\n";
    sqlText = sqlText + "where cs.is_dismiss != 1";
    return ArraySelectAll(XQuery("sql:" + sqlText));
}

/*
 * НОВОЕ (14.09.2026): город -- custom_elem "sity" (имя поля подтверждено пользователем
 * реальным XML документа collaborator). Та же схема, что GetMacroregionRows()/
 * GetMirCodeRows() -- один SQL на всех активных сотрудников сразу.
 * @returns {Object[]}   -   Массив {id, sity}.
 */
function GetCityRows()
{
    LogAlert(1, "GetCityRows(). НАЧАЛО");
    var sqlText, rows;
    sqlText = "";
    sqlText = sqlText + "select cs.id,\r\n";
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''sity'']/value)[1]', 'varchar(max)') as sity\r\n";
    sqlText = sqlText + "from collaborators cs\r\n";
    sqlText = sqlText + "inner join collaborator c on c.id = cs.id\r\n";
    sqlText = sqlText + "where cs.is_dismiss != 1";
    rows = ArraySelectAll(XQuery("sql:" + sqlText));
    LogAlert(1, "GetCityRows(). Строк: " + ArrayCount(rows));
    LogAlert(1, "GetCityRows(). КОНЕЦ");
    return rows;
}

/*
 * Находит город конкретного сотрудника; "(без города)" если поле пустое/не найдено
 * (см. ДОПУЩЕНИЕ №2 в шапке файла -- чтобы не терять данные молча).
 * @param {Object[]} cityRows
 * @param {number} collaboratorID
 * @returns {string}
 */
function FindCity(cityRows, collaboratorID)
{
    var cityRow, sCity;
    cityRow = ArrayOptFind(cityRows, "Int(This.id) == Int(collaboratorID)");
    sCity = (cityRow != undefined && cityRow.sity != undefined ? String(cityRow.sity) : "");
    return (sCity != "" ? sCity : "(без города)");
}

function GetMirCodeRows()
{
    var sqlText;
    sqlText = "";
    sqlText = sqlText + "select cs.id,\r\n";
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_mir_codes'']/value)[1]', 'varchar(max)') as mir_codes\r\n";
    sqlText = sqlText + "from collaborators cs\r\n";
    sqlText = sqlText + "inner join collaborator c on c.id = cs.id\r\n";
    sqlText = sqlText + "where cs.is_dismiss != 1";
    return ArraySelectAll(XQuery("sql:" + sqlText));
}

function ExtractMirCodes(rawValue)
{
    var parts, fields, codes, i;
    codes = [];
    parts = ArrayDirect(ArraySelect(String(rawValue).split("|"), "This != ''"));
    for (i = 0; i < ArrayCount(parts); i++)
    {
        fields = ArrayDirect(ArraySelect(String(parts[i]).split("#"), "This != ''"));
        if (ArrayCount(fields) > 0) { codes.push(String(fields[0])); }
    }
    return codes;
}

function CollaboratorHasMirCode(mirCodeRows, collaboratorID, mirCodeFilter)
{
    var row, codes;
    row = ArrayOptFind(mirCodeRows, "Int(This.id) == Int(collaboratorID)");
    if (row == undefined) { return false; }
    codes = ExtractMirCodes(row.mir_codes);
    return (ArrayOptFind(codes, "String(This) == String(mirCodeFilter)") != undefined);
}

function ResolveMirCodeText(iMirCodeID)
{
    if (OptInt(iMirCodeID, 0) <= 0) { return ""; }
    try { return String(tools.open_doc(Int(iMirCodeID)).TopElem.name); }
    catch (_ex) { return ""; }
}

/*
 * НОВОЕ (16.09.2026): резолвит id учебной программы (education_method) в текст --
 * та же схема, что ResolveMirCodeText() (каталог "education_method" -- см.
 * catalog: "education_method" у поля program_id в HREDU-183_filtry_modal_shag1.js).
 * @param {number} iProgramId
 * @returns {string}
 */
function ResolveProgramText(iProgramId)
{
    if (OptInt(iProgramId, 0) <= 0) { return "(без программы)"; }
    try { return String(tools.open_doc(Int(iProgramId)).TopElem.name); }
    catch (_ex) { return "id=" + iProgramId; }
}

/*
 * Резолвит id программы в текст ЧЕРЕЗ УЖЕ ГОТОВЫЙ КЭШ (см. Run() -- programNames
 * строится ОДИН РАЗ на все уникальные programIds, а не по разу на каждого сотрудника x
 * программу -- иначе tools.open_doc() дёргался бы много тысяч раз).
 * @param {Object[]} programNames   -   Массив {id, name}.
 * @param {number} iProgramId
 * @returns {string}
 */
function FindProgramName(programNames, iProgramId)
{
    var row;
    row = ArrayOptFind(programNames, "Int(This.id) == Int(iProgramId)");
    return (row != undefined ? String(row.name) : "id=" + iProgramId);
}

function GetMatrixAudienceCollaboratorRows(matrixDoc, collaboratorRows)
{
    LogAlert(1, "GetMatrixAudienceCollaboratorRows(). НАЧАЛО");
    var iAudiencePositionCommonId, iAudienceMirCodeId, sAudienceMirCodeText;
    var allowedPositionIds, mirCodeRows, filteredRows, i;

    iAudiencePositionCommonId = OptInt(matrixDoc.position_common_id, 0);
    iAudienceMirCodeId = OptInt(matrixDoc.mir_code_id, 0);
    sAudienceMirCodeText = ResolveMirCodeText(iAudienceMirCodeId);

    filteredRows = collaboratorRows;

    if (iAudiencePositionCommonId > 0)
    {
        allowedPositionIds = GetPositionIdsByCommonPosition(iAudiencePositionCommonId);
        collaboratorRows = filteredRows;
        filteredRows = [];
        for (i = 0; i < ArrayCount(collaboratorRows); i++)
        {
            if (IdArrayContains(allowedPositionIds, OptInt(collaboratorRows[i].position_id, 0))) { filteredRows.push(collaboratorRows[i]); }
        }
    }

    if (sAudienceMirCodeText != "")
    {
        mirCodeRows = GetMirCodeRows();
        collaboratorRows = filteredRows;
        filteredRows = [];
        for (i = 0; i < ArrayCount(collaboratorRows); i++)
        {
            if (CollaboratorHasMirCode(mirCodeRows, Int(collaboratorRows[i].id), sAudienceMirCodeText)) { filteredRows.push(collaboratorRows[i]); }
        }
    }

    LogAlert(1, "GetMatrixAudienceCollaboratorRows(). КОНЕЦ. Итого в аудитории: " + ArrayCount(filteredRows));
    return filteredRows;
}

/*
 * Применяет 4 ручных фильтра пользователя -- идентично HREDU-183_tep_reports.js.
 */
function ApplyManualFilters(collaboratorRows, iPositionFilter, sMacroregionFilter, sMirCodeFilter, macroRows)
{
    var allowedPositionIds, mirCodeRows, filteredRows, i, macroRow;

    filteredRows = collaboratorRows;

    if (iPositionFilter > 0)
    {
        allowedPositionIds = GetPositionIdsByCommonPosition(iPositionFilter);
        collaboratorRows = filteredRows;
        filteredRows = [];
        for (i = 0; i < ArrayCount(collaboratorRows); i++)
        {
            if (IdArrayContains(allowedPositionIds, OptInt(collaboratorRows[i].position_id, 0))) { filteredRows.push(collaboratorRows[i]); }
        }
    }

    if (sMacroregionFilter != "")
    {
        collaboratorRows = filteredRows;
        filteredRows = [];
        for (i = 0; i < ArrayCount(collaboratorRows); i++)
        {
            macroRow = ArrayOptFind(macroRows, "Int(This.id) == Int(collaboratorRows[i].id)");
            if (macroRow != undefined && String(macroRow.macroregion) == sMacroregionFilter) { filteredRows.push(collaboratorRows[i]); }
        }
    }

    if (sMirCodeFilter != "")
    {
        mirCodeRows = GetMirCodeRows();
        collaboratorRows = filteredRows;
        filteredRows = [];
        for (i = 0; i < ArrayCount(collaboratorRows); i++)
        {
            if (CollaboratorHasMirCode(mirCodeRows, Int(collaboratorRows[i].id), sMirCodeFilter)) { filteredRows.push(collaboratorRows[i]); }
        }
    }

    return filteredRows;
}

function GetCompletionDateRows(programIds)
{
    var sqlText;
    sqlText = "";
    sqlText = sqlText + "select ec.collaborator_id, e.education_method_id, min(ec.start_date) as first_date\r\n";
    sqlText = sqlText + "from event_collaborators ec\r\n";
    sqlText = sqlText + "join events e on e.id = ec.event_id\r\n";
    sqlText = sqlText + "where e.education_method_id in (" + ArrayMerge(programIds, "This", ",") + ")\r\n";
    sqlText = sqlText + "group by ec.collaborator_id, e.education_method_id";
    return ArraySelectAll(XQuery("sql:" + sqlText));
}

function FindCompletionDate(dateRows, collaboratorID, programID)
{
    var dateRow;
    dateRow = ArrayOptFind(dateRows, "Int(This.collaborator_id) == Int(collaboratorID) && Int(This.education_method_id) == Int(programID)");
    return (dateRow != undefined ? StrDate(Date(dateRow.first_date), false) : "");
}

/*
 * ИЗМЕНЕНО (16.09.2026): накопитель теперь по ПАРЕ (город, программа), а не только по
 * городу -- см. "ГРУППИРОВКА ПО (ГОРОД, ПРОГРАММА)" в шапке файла. Ищем циклом (как и
 * раньше) -- ArraySelect по строке-выражению не годится для ИЗМЕНЯЕМОГО накопителя.
 * @param {Object[]} acc
 * @param {string} sCity
 * @param {number} iProgramId
 * @param {string} sProgramName
 * @returns {Object}
 */
function GetOrCreateCityProgramAcc(acc, sCity, iProgramId, sProgramName)
{
    var i;
    for (i = 0; i < ArrayCount(acc); i++)
    {
        if (acc[i].city == sCity && Int(acc[i].programId) == Int(iProgramId)) { return acc[i]; }
    }
    var newAcc;
    newAcc = { city: sCity, programId: Int(iProgramId), programName: sProgramName, total: 0, mandatory: 0, fact: 0 };
    acc.push(newAcc);
    return newAcc;
}

/*
 * Округление факт/план в проценты (до целого, обычное арифметическое округление).
 * "-" если план = 0 (см. ДОПУЩЕНИЕ -- в реальных данных пока не встречалось, но
 * возможно в теории, если у города вся аудитория уже "выпала" -- на деле план всегда
 * = общее в этой версии, так что план=0 означает и общее=0, т.е. города вообще нет
 * в аудитории -- такая строка сюда не попадёт, см. ДОПУЩЕНИЕ №1).
 * @param {number} nFact
 * @param {number} nPlan
 * @returns {string}
 */
function FormatPercent(nFact, nPlan)
{
    if (nPlan <= 0) { return "-"; }
    return String(Int((nFact / nPlan) * 100 + 0.5)) + "%";
}

/*
 * ИСПРАВЛЕНИЕ (14.09.2026, БАГ С "&macroregion="): реальный тест показал, что итоговая
 * ссылка искажается ПРИ ОТОБРАЖЕНИИ виджетом "Табличные данные" -- "&macroregion="
 * превращалось в "%C2%AForegion=" (т.е. "&macr" пропадало, вместо него -- символ "¯",
 * U+00AF). Причина: "macr" -- это ИМЕННО ТАКОЕ имя у "легаси" HTML-сущности безточки
 * с запятой (как &amp, &lt, &nbsp) -- она означает символ "¯" (macron) и НЕ требует ";"
 * на конце. Судя по всему, виджет вставляет значение поля "link"/*_link ПРЯМО в атрибут
 * href как HTML-текст, без экранирования "&" в "&amp;" -- поэтому браузер видит в
 * "&macroregion=" сначала "&macr" (валидная сущность!) и стирает её, заменяя на "¯",
 * а не сам символ "&". Никакого отношения к UrlEncodeQuery()/percent-encoding это не
 * имеет -- проблема на уровне HTML, а не URL. Фикс: экранируем "&" САМИ в "&amp;" перед
 * тем, как класть готовую ссылку в поле RESULT -- тогда браузер сначала раскодирует
 * "&amp;" обратно в "&", и только ПОСЛЕ этого получившийся URL uже не содержит "&macr"
 * как отдельную подстроку для сущности. Без regex -- см. HtmlEscapeAmp() ниже, тот же
 * строковый API (StrOptSubStrPos/StrRangePos/StrLen), что и в GetQueryParam().
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

/*
 * Строит ссылку на страницу ТЭП-отчётов (HREDU-183_tep_reports.js) с нужным
 * набором параметров -- ровно те же параметры, что читает сама ТЭП-выборка
 * (см. GetQueryParam(...) в HREDU-183_tep_reports.js): matrix_id, macroregion,
 * mir_code, position_common_id, program_id, result_type + НОВЫЙ параметр city
 * (см. HREDU-183_tep_reports.js -- добавлен туда для этого дрилл-дауна).
 *
 * sCity = "" (пустая строка) -> ссылка ведёт на ВЕСЬ матрикс без фильтра по городу
 * (используется для строки "Общий итог").
 *
 * ИЗМЕНЕНО (16.09.2026): program_id теперь берётся ИЗ КОНКРЕТНОЙ СТРОКИ (её программа
 * матрицы), а не из ручного фильтра пользователя -- раз строка теперь и так соответствует
 * ровно одной программе (см. "ГРУППИРОВКА ПО (ГОРОД, ПРОГРАММА)"), логично, чтобы клик по
 * ней вёл в ТЭП-отчёт, УЖЕ отфильтрованный именно по этой программе. iProgramId=0 ->
 * без фильтра по программе (используется для строки "Общий итог").
 *
 * @param {number} iMatrixId
 * @param {string} sMacroregionFilter
 * @param {string} sMirCodeFilter
 * @param {number} iPositionFilter
 * @param {number} iProgramId
 * @param {string} sResultType   -   "total"|"plan"|"fact"|"mandatory"
 * @param {string} sCity
 * @returns {string}
 */
function BuildTepLink(iMatrixId, sMacroregionFilter, sMirCodeFilter, iPositionFilter, iProgramId, sResultType, sCity)
{
    var oQueryParams, sQueryString, sSeparator;
    oQueryParams = {
        matrix_id: String(iMatrixId),
        macroregion: sMacroregionFilter,
        mir_code: sMirCodeFilter,
        position_common_id: String(iPositionFilter),
        program_id: String(iProgramId),
        result_type: sResultType,
        city: sCity
    };
    sQueryString = UrlEncodeQuery(oQueryParams);
    sSeparator = (StrOptSubStrPos(TEP_REPORT_PAGE_URL, "?", false) != undefined ? "&" : "?");
    // HtmlEscapeAmp() -- см. комментарий над ней: "&" экранируем в "&amp;", потому что
    // виджет вставляет это значение прямо в HTML (href) без собственного экранирования.
    return HtmlEscapeAmp(TEP_REPORT_PAGE_URL + sSeparator + sQueryString);
}

//-------------------------------------------------------------------------
//              Точка входа
//-------------------------------------------------------------------------

function Run()
{
    LogAlert(2, "Run(). НАЧАЛО (Процент обученных)");
    var sFullUrl, matrixId, matrixDoc, matrixName, iProgramFilter, sMacroregionFilter, sMirCodeFilter, iPositionFilter;
    var programIds, filteredProgramIds, programNames, i, j;
    var activeRows, audienceRows, audienceFilteredRows, factBaseFilteredRows;
    var macroRows, mirCodeRows, cityRows, dateRows;
    var acc, cityProgramAcc, sCity, sProgramName, sDate, row;
    var totalAcc, resultRows, id;

    RESULT = [];

    try
    {
        sFullUrl = GetRequestUrlSafe();
        LogAlert(1, "Run(). Request.Url = [" + sFullUrl + "]");

        matrixId = OptInt(GetQueryParam(sFullUrl, "matrix_id"), 0);
        iProgramFilter = OptInt(GetQueryParam(sFullUrl, "program_id"), 0);
        sMacroregionFilter = GetQueryParam(sFullUrl, "macroregion");
        sMirCodeFilter = GetQueryParam(sFullUrl, "mir_code");
        iPositionFilter = OptInt(GetQueryParam(sFullUrl, "position_common_id"), 0);

        LogAlert(1, "Run(). matrixId=" + matrixId + " programFilter=" + iProgramFilter
            + " macroregionFilter=[" + sMacroregionFilter + "] mirCodeFilter=[" + sMirCodeFilter
            + "] positionCommonIdFilter=" + iPositionFilter);

        if (matrixId == 0)
        {
            throw ("Не передан matrix_id -- выбранная пользователем матрица обучения");
        }

        matrixDoc = tools.open_doc(matrixId).TopElem;
        matrixName = String(matrixDoc.name);

        programIds = ResolveProgramIds(matrixId, matrixName);

        if (iProgramFilter > 0)
        {
            filteredProgramIds = [];
            for (i = 0; i < ArrayCount(programIds); i++)
            {
                if (Int(programIds[i]) == iProgramFilter) { filteredProgramIds.push(programIds[i]); }
            }
            programIds = filteredProgramIds;
            if (ArrayCount(programIds) == 0)
            {
                throw ("Программа [" + iProgramFilter + "] не найдена среди программ выбранной матрицы");
            }
        }

        activeRows = GetActiveCollaboratorRows();
        macroRows = GetMacroregionRows();
        cityRows = GetCityRows();
        dateRows = GetCompletionDateRows(programIds);

        // НОВОЕ (16.09.2026): имена программ резолвим ОДИН РАЗ на все уникальные
        // programIds (не по разу на каждого сотрудника x программу -- иначе
        // tools.open_doc() дёргался бы многие тысячи раз, см. FindProgramName()).
        programNames = [];
        for (j = 0; j < ArrayCount(programIds); j++)
        {
            programNames.push({ id: Int(programIds[j]), name: ResolveProgramText(programIds[j]) });
        }

        // --- Пул "аудитория матрицы" (для total/plan/mandatory) ---
        audienceRows = GetMatrixAudienceCollaboratorRows(matrixDoc, activeRows);
        audienceFilteredRows = ApplyManualFilters(audienceRows, iPositionFilter, sMacroregionFilter, sMirCodeFilter, macroRows);
        LogAlert(1, "Run(). Сотрудников в аудитории (после ручных фильтров): " + ArrayCount(audienceFilteredRows));

        // --- Пул "без аудитории" (для fact) ---
        factBaseFilteredRows = ApplyManualFilters(activeRows, iPositionFilter, sMacroregionFilter, sMirCodeFilter, macroRows);
        LogAlert(1, "Run(). Сотрудников БЕЗ аудитории (после ручных фильтров, база для Факт): " + ArrayCount(factBaseFilteredRows));

        acc = [];

        // total/mandatory -- по каждому (сотрудник аудитории x программа), сгруппировано
        // по ПАРЕ (город, программа) -- см. "ГРУППИРОВКА ПО (ГОРОД, ПРОГРАММА)" в шапке.
        for (i = 0; i < ArrayCount(audienceFilteredRows); i++)
        {
            sCity = FindCity(cityRows, Int(audienceFilteredRows[i].id));
            for (j = 0; j < ArrayCount(programIds); j++)
            {
                sProgramName = FindProgramName(programNames, programIds[j]);
                cityProgramAcc = GetOrCreateCityProgramAcc(acc, sCity, programIds[j], sProgramName);
                sDate = FindCompletionDate(dateRows, Int(audienceFilteredRows[i].id), programIds[j]);
                cityProgramAcc.total = cityProgramAcc.total + 1;
                if (sDate == "") { cityProgramAcc.mandatory = cityProgramAcc.mandatory + 1; }
            }
        }

        // fact -- по каждому (сотрудник БЕЗ аудитории x программа), только ПРОЙДЕННЫЕ.
        // ДОПУЩЕНИЕ №1 (см. шапку файла, изменено 16.09.2026): накопитель (город,
        // программа) создаём здесь ТОЛЬКО когда реально есть завершение (sDate != "") --
        // иначе при большой факт-базе (тысячи активных сотрудников x несколько программ)
        // получился бы взрыв пустых строк "город x программа x 0 x 0", которых никто не
        // хочет видеть. Пара (город, программа), где есть только факт без аудитории,
        // всё равно получает свою строку (Общее=0), просто не для КАЖДОЙ комбинации, а
        // только там, где реально кто-то прошёл.
        for (i = 0; i < ArrayCount(factBaseFilteredRows); i++)
        {
            sCity = FindCity(cityRows, Int(factBaseFilteredRows[i].id));
            for (j = 0; j < ArrayCount(programIds); j++)
            {
                sDate = FindCompletionDate(dateRows, Int(factBaseFilteredRows[i].id), programIds[j]);
                if (sDate != "")
                {
                    sProgramName = FindProgramName(programNames, programIds[j]);
                    cityProgramAcc = GetOrCreateCityProgramAcc(acc, sCity, programIds[j], sProgramName);
                    cityProgramAcc.fact = cityProgramAcc.fact + 1;
                }
            }
        }

        // Сортировка -- сначала по городу, потом по названию программы -- ОБЫЧНЫМ ЦИКЛОМ
        // (пузырьком), а не через возможную функцию-хелпер вроде ArraySort(): такая
        // функция НИ РАЗУ не встречалась и не подтверждалась в этом тикете (в отличие от
        // ArraySelectDistinct/ArrayExtract/ArrayMerge и т.д.), а гадать с непроверенными
        // функциями платформы уже дорого обходилось (regex, function-as-value,
        // .indexOf/.substring -- см. историю тикета) -- поэтому используем только то,
        // что 100% работает: простые циклы и операторы сравнения.
        var iOuter, iInner, tmpAcc;
        for (iOuter = 0; iOuter < ArrayCount(acc) - 1; iOuter++)
        {
            for (iInner = 0; iInner < ArrayCount(acc) - 1 - iOuter; iInner++)
            {
                if (acc[iInner].city > acc[iInner + 1].city
                    || (acc[iInner].city == acc[iInner + 1].city && acc[iInner].programName > acc[iInner + 1].programName))
                {
                    tmpAcc = acc[iInner];
                    acc[iInner] = acc[iInner + 1];
                    acc[iInner + 1] = tmpAcc;
                }
            }
        }

        resultRows = [];
        id = 0;
        totalAcc = { total: 0, mandatory: 0, fact: 0 };
        for (i = 0; i < ArrayCount(acc); i++)
        {
            id = id + 1;
            row = acc[i];
            resultRows.push({
                id: id,
                city: row.city,
                program: row.programName,
                total: row.total,
                plan: row.total, // План = Общее, см. "РЕШЕНИЯ" в шапке
                fact: row.fact,
                percent: FormatPercent(row.fact, row.total),
                mandatory: row.mandatory,
                // ПОДТВЕРЖДЕНО (14.09.2026, реальный тест пользователя): виджет "Табличные
                // данные" различает клик ТОЛЬКО по строке целиком -- один "link" на всю
                // строку. ИЗМЕНЕНО (16.09.2026): program_id в ссылке теперь берётся из
                // КОНКРЕТНОЙ строки (row.programId), а не из общего фильтра -- см.
                // BuildTepLink(). Режим по-прежнему фиксирован на "total"; план/факт/
                // обязательно пользователь смотрит либо прямо в этой таблице, либо
                // переключает "Режим отчёта" вручную на целевой странице.
                link: BuildTepLink(matrixId, sMacroregionFilter, sMirCodeFilter, iPositionFilter, row.programId, "total", row.city)
            });
            totalAcc.total = totalAcc.total + row.total;
            totalAcc.mandatory = totalAcc.mandatory + row.mandatory;
            totalAcc.fact = totalAcc.fact + row.fact;
        }

        id = id + 1;
        // sCity = "" и iProgramId = 0 для "Общий итог" -- ссылка ведёт на ВЕСЬ матрикс
        // (без фильтра по городу и без фильтра по программе).
        resultRows.push({
            id: id,
            city: "Общий итог",
            program: "-",
            total: totalAcc.total,
            plan: totalAcc.total,
            fact: totalAcc.fact,
            percent: FormatPercent(totalAcc.fact, totalAcc.total),
            mandatory: totalAcc.mandatory,
            link: BuildTepLink(matrixId, sMacroregionFilter, sMirCodeFilter, iPositionFilter, 0, "total", "")
        });

        RESULT = resultRows;
        LogAlert(2, "Run(). Готово. Строк (город x программа): " + (ArrayCount(resultRows) - 1) + " + итоговая строка");
    }
    catch (_ex)
    {
        RESULT = [];
        ERROR = 1;
        MESSAGE = ExtractUserError(_ex);
        LogAlert(4, "Run(). ОШИБКА: " + MESSAGE);
    }
    LogAlert(2, "Run(). КОНЕЦ");
}

Run();

// ЗАКРЫТО (14.09.2026, КЛИКАБЕЛЬНОСТЬ): реальный тест пользователя подтвердил, что
// виджет "Табличные данные" различает клик ТОЛЬКО по строке целиком -- отдельной ссылки
// "на конкретную ячейку/число" у него нет (независимо от того, по какой колонке
// кликнули, срабатывает один и тот же "link" всей строки). Поэтому четыре поля
// total_link/plan_link/fact_link/mandatory_link и соответствующие им закомментированные
// варианты колонок -- УБРАНЫ как мёртвый код (см. историю тикета -- раньше они были
// здесь как непроверенная гипотеза). РЕШЕНИЕ: один клик по строке города -> ТЭП-отчёт в
// режиме "total"; план/факт/обязательно пользователь смотрит НЕ переходом по клику, а
// либо прямо в этой таблице (числа уже видны), либо переключает "Режим отчёта" вручную
// в фильтрах на целевой странице (см. HREDU-183_filtry_modal_shag1.js).
// ДОБАВЛЕНО (16.09.2026, по запросу пользователя): колонка "Учебная программа" сразу
// после "Город" -- строка теперь соответствует паре (город, программа), а не только
// городу, см. "ГРУППИРОВКА ПО (ГОРОД, ПРОГРАММА)" в шапке файла.
COLUMNS = [
    { "data": "id", "editable": true, "hidden": true, "sortable": false },
    { "data": "link", "hidden": true, "editable": false, "sortable": false }, // проверенный row-level link
    { "data": "city", "title": "Город", "type": "string", "editable": false, "sortable": true },
    { "data": "program", "title": "Учебная программа", "type": "string", "editable": false, "sortable": true },
    { "data": "total", "title": "Общее кол-во сотрудников", "type": "integer", "editable": false, "sortable": true },
    { "data": "plan", "title": "План", "type": "integer", "editable": false, "sortable": true },
    { "data": "fact", "title": "Факт", "type": "integer", "editable": false, "sortable": true },
    { "data": "percent", "title": "Процент", "type": "string", "editable": false, "sortable": false },
    { "data": "mandatory", "title": "Обязательно к прохождению", "type": "integer", "editable": false, "sortable": true }
];
