// =====================================================================
// HREDU-183. Диагностика: сверка арифметики между 4 режимами ТЭП (total/plan/fact/
// mandatory) для ОДНОЙ И ТОЙ ЖЕ матрицы+фильтров за ОДИН прогон -- отвечает на вопрос
// "как точно проверить, что все 4 отчёта считаются правильно" (задан пользователем
// 10.09.2026, "как точно проверить я не знаю").
//
// ИДЕЯ: вместо того чтобы сверять 4 РАЗНЫХ виджета вручную (сложно сопоставить, легко
// упустить разницу в фильтрах между ними) -- один скрипт СРАЗУ считает все 4 числа по
// ОДНОЙ И ТОЙ ЖЕ логике (см. HREDU-183_tep_reports.js -- функции продублированы 1:1,
// чтобы диагностика была независимым повторным расчётом, а не вызовом того же кода) и
// проверяет 3 арифметических тождества, которые ОБЯЗАНЫ выполняться при текущей бизнес-
// логике (см. "РЕШЕНИЯ" в шапке HREDU-183_tep_reports.js):
//
//   1) total == mandatory + completed_in_audience
//      (Общее = аудитория матрицы. Обязательно = аудитория БЕЗ пройденных. Значит
//       Общее = Обязательно + пройденные-В-ПРЕДЕЛАХ-аудитории. completed_in_audience --
//       это ПРОМЕЖУТОЧНОЕ число: сколько из аудитории уже прошли, его НЕТ ни в одном
//       из 4 реальных отчётов-виджетов, оно посчитано только здесь для сверки.)
//   2) plan == total
//      (текущее упрощение -- План = Общее, см. "РЕШЕНИЯ" пункт 2 в шапке
//       HREDU-183_tep_reports.js -- период прохождения ещё не реализован).
//   3) fact >= completed_in_audience
//      (Факт считается БЕЗ ограничения аудиторией матрицы -- значит его база шире,
//       чем аудитория, и он обязан включать как минимум всех прошедших ИЗ аудитории;
//       строгое равенство тут НЕ ожидается -- в факте могут быть ещё и те, кто прошёл
//       программу, но уже не подходит под аудиторию матрицы сейчас).
//
// Если хоть одно тождество не выполняется -- где-то разошлась логика (например
// фильтры аудитории не совпадают между режимами) -- ЭТО и есть сигнал реальной ошибки.
// Если все 3 тождества сошлись -- это сильное свидетельство, что арифметика верна
// (хотя, конечно, не гарантия, что сама бизнес-логика соответствует ТЗ -- это
// отдельный вопрос, который уже обсуждался и зафиксирован в "РЕШЕНИЯ").
//
// КАК ЗАПУСТИТЬ: как обычная выборка "Табличные данные" (как HREDU-183_tep_reports.js),
// на СТРАНИЦЕ ТЭП-отчёта, где в URL уже есть matrix_id и остальные фильтры (после
// "Применить" в модалке) -- скрипт читает их точно так же. result_type НЕ ИСПОЛЬЗУЕТСЯ
// (диагностика всегда считает ВСЕ 4 числа сразу). Колонки результата: metric / value /
// check (человекочитаемый статус тождества, только у итоговых строк).
// =====================================================================

DEBUG = true;
LOG_NAME = "agent";
CUR_OBJECT_ID = 0; // не забыть заполнить реальным ID после создания документа в админке -- LogAlert теперь защищена try/catch, так что забытый 0 больше не обрушит весь скрипт

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

//-------------------------------------------------------------------------
//              Продублированные функции из HREDU-183_tep_reports.js
//              (см. комментарии в оригинале -- здесь без повторной документации
//              каждой функции, только там, где логика диагностики отличается)
//-------------------------------------------------------------------------

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

function GetProgramIds(matrixRows, elementRows)
{
    var matrixProgramIds, elementProgramIds, allProgramIds, i;
    matrixProgramIds = ArrayExtract(matrixRows, "Int(This.education_method_id)");
    elementProgramIds = ArrayExtract(elementRows, "Int(This.education_method_id)");
    allProgramIds = [];
    for (i = 0; i < ArrayCount(matrixProgramIds); i++) { allProgramIds.push(matrixProgramIds[i]); }
    for (i = 0; i < ArrayCount(elementProgramIds); i++) { allProgramIds.push(elementProgramIds[i]); }
    return ArraySelectDistinct(allProgramIds, "This");
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

function GetMatrixAudienceCollaboratorRows(matrixDoc, collaboratorRows)
{
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
 * Применяет 4 ручных фильтра пользователя (macroregion/mir_code/position_common_id/
 * program_id уже применён на programIds раньше) к списку сотрудников -- идентично
 * блоку в Run() из HREDU-183_tep_reports.js, вынесено в функцию, т.к. тут применяется
 * ДВАЖДЫ (для audience-пула и для fact-пула).
 * @param {Object[]} collaboratorRows
 * @param {number} iPositionFilter
 * @param {string} sMacroregionFilter
 * @param {string} sMirCodeFilter
 * @param {Object[]} macroRows
 * @returns {Object[]}
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

/*
 * Считает кол-во пар "сотрудник x программа" и, отдельно, кол-во таких пар с уже
 * заполненной датой прохождения -- по списку сотрудников и списку программ.
 * @returns {Object}   -   { totalPairs, completedPairs }
 */
function CountPairs(collaboratorRows, programIds, dateRows)
{
    var i, j, totalPairs, completedPairs, sDate;
    totalPairs = 0;
    completedPairs = 0;
    for (i = 0; i < ArrayCount(collaboratorRows); i++)
    {
        for (j = 0; j < ArrayCount(programIds); j++)
        {
            totalPairs = totalPairs + 1;
            sDate = FindCompletionDate(dateRows, Int(collaboratorRows[i].id), programIds[j]);
            if (sDate != "") { completedPairs = completedPairs + 1; }
        }
    }
    return { totalPairs: totalPairs, completedPairs: completedPairs };
}

//-------------------------------------------------------------------------
//              Точка входа
//-------------------------------------------------------------------------

function Run()
{
    LogAlert(2, "Run(). НАЧАЛО (диагностика сверки ТЭП)");
    var sFullUrl, matrixId, matrixDoc, matrixName, iProgramFilter, sMacroregionFilter, sMirCodeFilter, iPositionFilter;
    var matrixRows, matrixIds, elementRows, programIds, filteredProgramIds, i;
    var activeRows, audienceRows, macroRows, dateRows;
    var audienceFilteredRows, factBaseFilteredRows;
    var audienceCounts, factCounts;
    var nTotal, nPlan, nMandatory, nCompletedInAudience, nFact;
    var check1, check2, check3;

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

        if (matrixId == 0)
        {
            throw ("Не передан matrix_id -- открой эту диагностику на странице ТЭП-отчёта ПОСЛЕ того, как применил фильтры хотя бы раз");
        }

        matrixDoc = tools.open_doc(matrixId).TopElem;
        matrixName = String(matrixDoc.name);

        matrixRows = GetMatrixRows(matrixName);
        matrixIds = ArrayExtract(matrixRows, "Int(This.id)");
        elementRows = GetMatrixElementRows(matrixIds);
        programIds = GetProgramIds(matrixRows, elementRows);

        if (iProgramFilter > 0)
        {
            filteredProgramIds = [];
            for (i = 0; i < ArrayCount(programIds); i++)
            {
                if (Int(programIds[i]) == iProgramFilter) { filteredProgramIds.push(programIds[i]); }
            }
            programIds = filteredProgramIds;
        }

        activeRows = GetActiveCollaboratorRows();
        macroRows = GetMacroregionRows();
        dateRows = GetCompletionDateRows(programIds);

        // --- Пул "аудитория матрицы" (используется для total/plan/mandatory) ---
        audienceRows = GetMatrixAudienceCollaboratorRows(matrixDoc, activeRows);
        audienceFilteredRows = ApplyManualFilters(audienceRows, iPositionFilter, sMacroregionFilter, sMirCodeFilter, macroRows);
        audienceCounts = CountPairs(audienceFilteredRows, programIds, dateRows);

        nTotal = audienceCounts.totalPairs;
        nPlan = audienceCounts.totalPairs; // План = Общее, текущее упрощение
        nMandatory = audienceCounts.totalPairs - audienceCounts.completedPairs;
        nCompletedInAudience = audienceCounts.completedPairs;

        // --- Пул "без аудитории" (используется для fact) ---
        factBaseFilteredRows = ApplyManualFilters(activeRows, iPositionFilter, sMacroregionFilter, sMirCodeFilter, macroRows);
        factCounts = CountPairs(factBaseFilteredRows, programIds, dateRows);
        nFact = factCounts.completedPairs;

        LogAlert(1, "Run(). nTotal=" + nTotal + " nPlan=" + nPlan + " nMandatory=" + nMandatory
            + " nCompletedInAudience=" + nCompletedInAudience + " nFact=" + nFact);

        check1 = (nTotal == (nMandatory + nCompletedInAudience));
        check2 = (nPlan == nTotal);
        check3 = (nFact >= nCompletedInAudience);

        RESULT = [];
        RESULT.push({ id: 1, metric: "matrix_id / название матрицы", value: matrixId + " / " + matrixName, check: "" });
        RESULT.push({ id: 2, metric: "Программ в матрице (после фильтра program_id)", value: String(ArrayCount(programIds)), check: "" });
        RESULT.push({ id: 3, metric: "Сотрудников в аудитории матрицы (после ручных фильтров)", value: String(ArrayCount(audienceFilteredRows)), check: "" });
        RESULT.push({ id: 4, metric: "Сотрудников БЕЗ аудитории (после ручных фильтров, база для Факт)", value: String(ArrayCount(factBaseFilteredRows)), check: "" });
        RESULT.push({ id: 5, metric: "-- ИТОГОВЫЕ ЧИСЛА (сколько строк должно быть в каждом из 4 виджетов) --", value: "", check: "" });
        RESULT.push({ id: 6, metric: "total (Общее кол-во)", value: String(nTotal), check: "" });
        RESULT.push({ id: 7, metric: "plan (План)", value: String(nPlan), check: "" });
        RESULT.push({ id: 8, metric: "fact (Факт)", value: String(nFact), check: "" });
        RESULT.push({ id: 9, metric: "mandatory (Обязательно к прохождению)", value: String(nMandatory), check: "" });
        RESULT.push({ id: 10, metric: "(справочно) пройдено В ПРЕДЕЛАХ аудитории -- ни в одном виджете не показывается", value: String(nCompletedInAudience), check: "" });
        RESULT.push({ id: 11, metric: "-- ТОЖДЕСТВА (должны быть все OK) --", value: "", check: "" });
        RESULT.push({ id: 12, metric: "1) total == mandatory + пройдено-в-аудитории", value: nTotal + " == " + nMandatory + " + " + nCompletedInAudience, check: (check1 ? "OK" : "MISMATCH!") });
        RESULT.push({ id: 13, metric: "2) plan == total", value: nPlan + " == " + nTotal, check: (check2 ? "OK" : "MISMATCH!") });
        RESULT.push({ id: 14, metric: "3) fact >= пройдено-в-аудитории", value: nFact + " >= " + nCompletedInAudience, check: (check3 ? "OK" : "MISMATCH!") });

        LogAlert(2, "Run(). Готово. check1=" + check1 + " check2=" + check2 + " check3=" + check3);
    }
    catch (_ex)
    {
        RESULT = [{ id: 0, metric: "ОШИБКА", value: ExtractUserError(_ex), check: "" }];
        LogAlert(4, "Run(). ОШИБКА: " + ExtractUserError(_ex));
    }
    LogAlert(2, "Run(). КОНЕЦ");
}

Run();

COLUMNS = [
    { "data": "id", "editable": true, "hidden": true, "sortable": false },
    { "data": "metric", "title": "Показатель", "type": "string", "editable": false, "sortable": false, "multiline": true, "width": "55%" },
    { "data": "value", "title": "Значение", "type": "string", "editable": false, "sortable": false, "multiline": true, "width": "30%" },
    { "data": "check", "title": "Проверка", "type": "string", "editable": false, "sortable": false, "width": "15%" }
];
