sLogName = 'HREDU_183_excel_export_tep';
EnableLog(sLogName, true);
function alert(sInputObj) {
    LogEvent(sLogName, sInputObj);
    return sInputObj;
}

// =====================================================================
// HREDU-183. Отчёты ТЭП (Общее кол-во / План / Факт / Обязательно к прохождению) --
// удалённое действие на кнопку "Выгрузить в Excel".
//
// ЗАДАЧА (20.10.2026, со слов пользователя): выгрузить в Excel РОВНО ту таблицу, которую
// пользователь сейчас видит в виджете "Табличные данные" на странице ТЭП (выборка
// HREDU-183_tep_reports.js) -- для ТЕКУЩЕГО открытого режима (total/plan/fact/mandatory),
// с теми же фильтрами, теми же строками в том же порядке.
//
// РЕЖИМ ОТЧЁТА: выборка берёт result_type из URL страницы (его туда пишут модалка
// "Выбор отчета" HREDU-183_filtry_modal_shag1.js и кнопки режимов). Здесь -- то же самое,
// но из cur_page_url (Request.Url в удалённом действии ненадёжен). Порядок поиска:
//   1. result_type из URL страницы (основной путь);
//   2. необязательный параметр кнопки "result_type" (PARAMETERS) -- запасной вариант,
//      на случай, если на какой-то странице режим задан не через URL, а фиксированно;
//   3. "total" -- тот же дефолт, что в выборке.
//
// ОТКУДА ЛОГИКА: ВСЕ функции расчёта -- БУКВАЛЬНАЯ КОПИЯ из HREDU-183_tep_reports.js
// (версия от 19.10.2026, План по ТЗ п.3 -- IsPeriodStarted()). Блок отбора строк в Run()
// -- тот же код, что в Run() выборки, только результат идёт в HTML-таблицу Excel, а не в
// RESULT виджета. История находок/фиксов -- в выборке, здесь не повторяется. ЕСЛИ
// МЕНЯЕТСЯ ЛОГИКА В ВЫБОРКЕ -- ТАК ЖЕ ПОМЕНЯТЬ ЗДЕСЬ, иначе Excel разойдётся с таблицей.
//
// ВИДИМОСТЬ: в выборке ТЭП НЕТ ни проверки доступа, ни сужения пула по подчинённым --
// поэтому здесь их тоже НЕТ (иначе Excel разошёлся бы с таблицей).
//
// КОЛОНКИ -- как в Конфигурации виджета (20.10.2026, прислал пользователь):
//   status_color (view: color), fullname, position_name, subdivision_name, completion_date.
// В Excel status_color -- ячейка, залитая тем же цветом (green/red/white), без текста.
// Учебная программа / город / макрорегион в виджете НЕ показываются -- в Excel тоже нет.
//
// ВЫГРУЗКА В EXCEL -- тот же приём, что в HREDU-182_excel_export_percent.js и HREDU-239
// (подтверждён рабочим): HTML-таблица -> tools_web.set_user_data("excel_html_" +
// curUserID) -> new_window на assessment_excel_export.html.
//
// НАСТРОЙКА КНОПКИ В LPE:
//   - cur_page_url -> {{curEnv.curEnvUrl}}  -- обязательно (фильтры + режим отчёта).
//   - curUserId    -> {{curUser.id}}        -- не обязательно, только запасной ключ кэша
//                                              (основной -- "голая" curUserID, как в
//                                              уже работающей выгрузке "Процента обученных").
// =====================================================================

EXPORT_PAGE_URL = "assessment_excel_export.html";

//-------------------------------------------------------------------------
//              URL / параметры
//-------------------------------------------------------------------------

function GetCurPageUrlSafe()
{
    var sUrl;
    try { sUrl = String(PARAMETERS.GetOptProperty("cur_page_url", "")); }
    catch (_exParam) { sUrl = ""; }
    if (sUrl != "" && sUrl != "undefined" && sUrl != "null")
    {
        return sUrl;
    }
    try { return String(Request.Url); }
    catch (_ex) { return ""; }
}

/*
* Необязательный параметр кнопки "result_type" -- запасной источник режима отчёта.
* @returns {string}
*/
function GetResultTypeParamSafe()
{
    var sValue;
    try { sValue = String(PARAMETERS.GetOptProperty("result_type", "")); }
    catch (_exParam) { sValue = ""; }
    if (sValue == "undefined" || sValue == "null") { sValue = ""; }
    return sValue;
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

//-------------------------------------------------------------------------
//              Матрица / программы (копия из выборки)
//-------------------------------------------------------------------------

function IsActiveText(sValue)
{
    try { return tools_web.is_true(sValue); }
    catch (_ex) { return (String(sValue) == "true" || String(sValue) == "1"); }
}

function GetCompoundProgramRows()
{
    var sqlText;
    sqlText = "";
    sqlText = sqlText + "select cs.id,\r\n";
    sqlText = sqlText + "       c.data.value('(*/name)[1]', 'varchar(max)') as name,\r\n";
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_matrix_active'']/value)[1]', 'varchar(max)') as f_matrix_active,\r\n";
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_position_names'']/value)[1]', 'varchar(max)') as f_position_names,\r\n";
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_position_names_exclude'']/value)[1]', 'varchar(max)') as f_position_names_exclude,\r\n";
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_mir_code'']/value)[1]', 'varchar(max)') as f_mir_code,\r\n";
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_mir_code_exclude'']/value)[1]', 'varchar(max)') as f_mir_code_exclude,\r\n";
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_org_names'']/value)[1]', 'varchar(max)') as f_org_names,\r\n";
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_org_names_exclude'']/value)[1]', 'varchar(max)') as f_org_names_exclude,\r\n";
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_subdivision_names'']/value)[1]', 'varchar(max)') as f_subdivision_names,\r\n";
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_subdivision_names_exclude'']/value)[1]', 'varchar(max)') as f_subdivision_names_exclude,\r\n";
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_subdivision_child'']/value)[1]', 'varchar(max)') as f_subdivision_child,\r\n";
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_collaborator_statuses_exclude'']/value)[1]', 'varchar(max)') as f_collaborator_statuses_exclude\r\n";
    sqlText = sqlText + "from compound_programs cs\r\n";
    sqlText = sqlText + "inner join compound_program c on c.id = cs.id";
    return ArraySelectAll(XQuery("sql:" + sqlText));
}

function GetEducationMethodTaskRows()
{
    var sqlText;
    sqlText = "";
    sqlText = sqlText + "select cs.id as matrix_id,\r\n";
    sqlText = sqlText + "       t.p.value('(object_id)[1]', 'bigint') as object_id,\r\n";
    sqlText = sqlText + "       t.p.value('(education_method_id)[1]', 'bigint') as education_method_id,\r\n";
    sqlText = sqlText + "       t.p.value('(type)[1]', 'varchar(50)') as ptype,\r\n";
    sqlText = sqlText + "       t.p.value('(delay_days)[1]', 'int') as delay_days,\r\n";
    sqlText = sqlText + "       t.p.value('(days)[1]', 'int') as days,\r\n";
    sqlText = sqlText + "       t.p.value('(name)[1]', 'varchar(max)') as pname\r\n";
    sqlText = sqlText + "from compound_programs cs\r\n";
    sqlText = sqlText + "inner join compound_program c on c.id = cs.id\r\n";
    sqlText = sqlText + "cross apply c.data.nodes('/*/programs/program') as t(p)\r\n";
    sqlText = sqlText + "where t.p.value('(type)[1]', 'varchar(50)') = 'education_method'";
    return ArraySelectAll(XQuery("sql:" + sqlText));
}

function GetProgramIds(taskRows, matrixId)
{
    var ids, i;
    ids = [];
    for (i = 0; i < ArrayCount(taskRows); i++)
    {
        if (Int(taskRows[i].matrix_id) == Int(matrixId) && OptInt(taskRows[i].education_method_id, 0) > 0)
        {
            ids.push(Int(taskRows[i].education_method_id));
        }
    }
    return ArraySelectDistinct(ids, "This");
}

function ResolveMatrixContext(matrixId, allProgramRows, taskRows)
{
    var matrixRow, programIds;

    matrixRow = ArrayOptFind(allProgramRows, "Int(This.id) == Int(matrixId)");
    if (matrixRow == undefined)
    {
        throw ("Не найдено модульной программы (compound_program) с id=" + matrixId);
    }
    if (!IsActiveText(matrixRow.f_matrix_active))
    {
        throw ("Модульная программа [" + matrixRow.name + "] (id=" + matrixId + ") деактивирована (f_matrix_active) -- отчёт недоступен для деактивированных программ.");
    }

    programIds = GetProgramIds(taskRows, matrixId);
    if (ArrayCount(programIds) == 0)
    {
        throw ("У модульной программы [" + matrixRow.name + "] не найдено ни одной задачи с типом [Учебная программа] (education_method)");
    }

    return { matrixRow: matrixRow, programIds: programIds };
}

function FindProgramName(programNames, iProgramId)
{
    var row;
    row = ArrayOptFind(programNames, "Int(This.id) == Int(iProgramId)");
    return (row != undefined ? String(row.name) : "id=" + iProgramId);
}

function FindProgramTiming(programNames, iProgramId)
{
    var row;
    row = ArrayOptFind(programNames, "Int(This.id) == Int(iProgramId)");
    return (row != undefined ? { delayDays: OptInt(row.delay_days, 0), days: OptInt(row.days, 0) } : { delayDays: 0, days: 0 });
}

//-------------------------------------------------------------------------
//              Сотрудники и их атрибуты (копия из выборки)
//-------------------------------------------------------------------------

function GetActiveCollaboratorRows()
{
    return ArraySelectAll(XQuery("for $elem in collaborators where $elem/is_dismiss=false() return $elem"));
}

function GetPositionIdsByCommonPosition(iCommonPositionFilter)
{
    var positionRows, positionIds, i;
    positionRows = ArraySelectAll(XQuery("for $elem in positions where $elem/position_common_id = " + iCommonPositionFilter + " return $elem"));
    positionIds = [];
    for (i = 0; i < ArrayCount(positionRows); i++)
    {
        positionIds.push(Int(positionRows[i].id));
    }
    return positionIds;
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

function GetCityRows()
{
    var sqlText;
    sqlText = "";
    sqlText = sqlText + "select cs.id,\r\n";
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''sity'']/value)[1]', 'varchar(max)') as sity\r\n";
    sqlText = sqlText + "from collaborators cs\r\n";
    sqlText = sqlText + "inner join collaborator c on c.id = cs.id\r\n";
    sqlText = sqlText + "where cs.is_dismiss != 1";
    return ArraySelectAll(XQuery("sql:" + sqlText));
}

function GetHireDateRows()
{
    var sqlText;
    sqlText = "";
    sqlText = sqlText + "select cs.id,\r\n";
    sqlText = sqlText + "       DATEDIFF(day, c.data.value('(*/hire_date)[1]', 'datetime'), GETDATE()) as days_worked\r\n";
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

function GetStatusRows()
{
    var sqlText;
    sqlText = "";
    sqlText = sqlText + "select cs.id,\r\n";
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''CurrentState'']/value)[1]', 'varchar(max)') as status\r\n";
    sqlText = sqlText + "from collaborators cs\r\n";
    sqlText = sqlText + "inner join collaborator c on c.id = cs.id\r\n";
    sqlText = sqlText + "where cs.is_dismiss != 1";
    return ArraySelectAll(XQuery("sql:" + sqlText));
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

//-------------------------------------------------------------------------
//              Сортировка / бинарный поиск (копия из выборки)
//-------------------------------------------------------------------------

function SortRowsById(rows)
{
    return ArraySort(rows, "Int(This.id)", "+");
}

function BinarySearchById(sortedRows, targetId)
{
    var lo, hi, mid, midId, iTarget;
    iTarget = Int(targetId);
    lo = 0;
    hi = ArrayCount(sortedRows) - 1;
    while (lo <= hi)
    {
        mid = Int((lo + hi) / 2);
        midId = Int(sortedRows[mid].id);
        if (midId == iTarget) { return sortedRows[mid]; }
        else if (midId < iTarget) { lo = mid + 1; }
        else { hi = mid - 1; }
    }
    return undefined;
}

function SortIdArray(idArray)
{
    return ArraySort(idArray, "Int(This)", "+");
}

function IdArrayContainsSorted(sortedIdArray, value)
{
    var lo, hi, mid, midVal, iTarget;
    iTarget = Int(value);
    lo = 0;
    hi = ArrayCount(sortedIdArray) - 1;
    while (lo <= hi)
    {
        mid = Int((lo + hi) / 2);
        midVal = Int(sortedIdArray[mid]);
        if (midVal == iTarget) { return true; }
        else if (midVal < iTarget) { lo = mid + 1; }
        else { hi = mid - 1; }
    }
    return false;
}

function SortDateRowsByCollaboratorId(dateRows)
{
    return ArraySort(dateRows, "Int(This.collaborator_id)", "+");
}

function FindCity(sortedCityRows, collaboratorID)
{
    var cityRow, sCity;
    cityRow = BinarySearchById(sortedCityRows, collaboratorID);
    sCity = (cityRow != undefined && cityRow.sity != undefined ? String(cityRow.sity) : "");
    return (sCity != "" ? sCity : "(без города)");
}

function FindMacroregion(sortedMacroRows, collaboratorID)
{
    var macroRow;
    macroRow = BinarySearchById(sortedMacroRows, collaboratorID);
    return (macroRow != undefined && macroRow.macroregion != undefined ? String(macroRow.macroregion) : "");
}

function FindDaysWorked(sortedHireRows, collaboratorID)
{
    var row;
    row = BinarySearchById(sortedHireRows, collaboratorID);
    return (row != undefined ? OptInt(row.days_worked, -1) : -1);
}

/*
* ТЗ п.3 -- ИДЕНТИЧНА выборке.
*/
function IsPeriodStarted(iDaysWorked, iDelayDays)
{
    if (iDaysWorked < 0)
    {
        return true;
    }
    return (iDaysWorked >= OptInt(iDelayDays, 0));
}

function FindCompletionDate(sortedDateRows, collaboratorID, programID)
{
    var lo, hi, mid, midId, iTarget, iProgram, startIdx, i, n;
    iTarget = Int(collaboratorID);
    iProgram = Int(programID);
    lo = 0;
    hi = ArrayCount(sortedDateRows) - 1;
    startIdx = -1;
    while (lo <= hi)
    {
        mid = Int((lo + hi) / 2);
        midId = Int(sortedDateRows[mid].collaborator_id);
        if (midId == iTarget) { startIdx = mid; hi = mid - 1; }
        else if (midId < iTarget) { lo = mid + 1; }
        else { hi = mid - 1; }
    }
    if (startIdx == -1) { return ""; }
    n = ArrayCount(sortedDateRows);
    for (i = startIdx; i < n && Int(sortedDateRows[i].collaborator_id) == iTarget; i++)
    {
        if (Int(sortedDateRows[i].education_method_id) == iProgram)
        {
            return StrDate(Date(sortedDateRows[i].first_date), false);
        }
    }
    return "";
}

//-------------------------------------------------------------------------
//              Аудитория матрицы (копия из выборки)
//-------------------------------------------------------------------------

function SplitByDelimiter(sText, sDelim)
{
    var parts, iLen, iDelimLen, iStart, iPos;
    parts = [];
    if (sText == undefined || sText == "") { return parts; }
    iLen = StrLen(sText);
    iDelimLen = StrLen(sDelim);
    iStart = 0;
    while (true)
    {
        iPos = StrOptSubStrPos(sText, sDelim, false, iStart);
        if (iPos == undefined)
        {
            parts.push(StrRangePos(sText, iStart, iLen));
            break;
        }
        parts.push(StrRangePos(sText, iStart, iPos));
        iStart = iPos + iDelimLen;
    }
    return parts;
}

function ExtractMirCodes(rawValue)
{
    var parts, fields, codes, i;
    codes = [];
    parts = ArrayDirect(ArraySelect(SplitByDelimiter(String(rawValue), "|"), "This != ''"));
    for (i = 0; i < ArrayCount(parts); i++)
    {
        fields = ArrayDirect(ArraySelect(SplitByDelimiter(String(parts[i]), "#"), "This != ''"));
        if (ArrayCount(fields) > 0)
        {
            codes.push(String(fields[0]));
        }
    }
    return codes;
}

function CollaboratorHasMirCode(sortedMirCodeRows, collaboratorID, mirCodeFilter)
{
    var row, codes;
    row = BinarySearchById(sortedMirCodeRows, collaboratorID);
    if (row == undefined)
    {
        return false;
    }
    codes = ExtractMirCodes(row.mir_codes);
    return (ArrayOptFind(codes, "String(This) == String(mirCodeFilter)") != undefined);
}

function SplitByStar(sPattern)
{
    var parts, iLen, iStart, iPos;
    parts = [];
    iLen = StrLen(sPattern);
    iStart = 0;
    while (true)
    {
        iPos = StrOptSubStrPos(sPattern, "*", true, iStart);
        if (iPos == undefined)
        {
            parts.push(StrRangePos(sPattern, iStart, iLen));
            break;
        }
        parts.push(StrRangePos(sPattern, iStart, iPos));
        iStart = iPos + 1;
    }
    return parts;
}

function WildcardMatch(sPattern, sText, bIgnoreCase)
{
    var parts, i, sSeg, iTextLen, iSearchPos, iFoundPos, bLeadingStar, bTrailingStar;
    if (sPattern == "") { return false; }
    bLeadingStar = (StrRangePos(sPattern, 0, 1) == "*");
    bTrailingStar = (StrRangePos(sPattern, StrLen(sPattern) - 1, StrLen(sPattern)) == "*");
    parts = SplitByStar(sPattern);
    iTextLen = StrLen(sText);
    iSearchPos = 0;
    for (i = 0; i < ArrayCount(parts); i++)
    {
        sSeg = parts[i];
        if (sSeg == "") { continue; }
        iFoundPos = StrOptSubStrPos(sText, sSeg, bIgnoreCase, iSearchPos);
        if (iFoundPos == undefined) { return false; }
        if (i == 0 && !bLeadingStar && iFoundPos != 0) { return false; }
        iSearchPos = iFoundPos + StrLen(sSeg);
    }
    if (!bTrailingStar && iSearchPos != iTextLen) { return false; }
    return true;
}

function MatchAnySemicolonPattern(sPatternsList, sText, bIgnoreCase)
{
    var patterns, i;
    if (sPatternsList == undefined || sPatternsList == "") { return false; }
    patterns = ArraySelect(SplitByDelimiter(String(sPatternsList), ";"), "This != ''");
    for (i = 0; i < ArrayCount(patterns); i++)
    {
        if (WildcardMatch(patterns[i], sText, bIgnoreCase)) { return true; }
    }
    return false;
}

function AxisMatches(sPatternsList, sSingleValue)
{
    if (sPatternsList == undefined || String(sPatternsList) == "") { return true; }
    return MatchAnySemicolonPattern(sPatternsList, sSingleValue, true);
}

function AxisExcludeMatches(sPatternsList, sSingleValue)
{
    if (sPatternsList == undefined || String(sPatternsList) == "") { return false; }
    return MatchAnySemicolonPattern(sPatternsList, sSingleValue, true);
}

function MirCodeAxisMatches(sPatternsList, employeeCodes)
{
    var i;
    if (sPatternsList == undefined || String(sPatternsList) == "") { return true; }
    for (i = 0; i < ArrayCount(employeeCodes); i++)
    {
        if (MatchAnySemicolonPattern(sPatternsList, employeeCodes[i], true)) { return true; }
    }
    return false;
}

function MirCodeAxisExcludeMatches(sPatternsList, employeeCodes)
{
    var i;
    if (sPatternsList == undefined || String(sPatternsList) == "") { return false; }
    for (i = 0; i < ArrayCount(employeeCodes); i++)
    {
        if (MatchAnySemicolonPattern(sPatternsList, employeeCodes[i], true)) { return true; }
    }
    return false;
}

function CollaboratorMatchesMatrixAudience(collaboratorRow, matrixRow, sortedMirCodeRows, sortedStatusRows)
{
    var sPosition, sOrg, sSubdivision, sStatus, employeeCodes, mirCodeRow, statusRow;

    sPosition = String(collaboratorRow.position_name);
    sOrg = String(collaboratorRow.org_name);
    sSubdivision = String(collaboratorRow.position_parent_name);

    mirCodeRow = BinarySearchById(sortedMirCodeRows, Int(collaboratorRow.id));
    employeeCodes = (mirCodeRow != undefined ? ExtractMirCodes(mirCodeRow.mir_codes) : []);

    statusRow = BinarySearchById(sortedStatusRows, Int(collaboratorRow.id));
    sStatus = (statusRow != undefined && statusRow.status != undefined ? String(statusRow.status) : "");

    if (AxisExcludeMatches(matrixRow.f_position_names_exclude, sPosition)) { return false; }
    if (AxisExcludeMatches(matrixRow.f_org_names_exclude, sOrg)) { return false; }
    if (AxisExcludeMatches(matrixRow.f_subdivision_names_exclude, sSubdivision)) { return false; }
    if (MirCodeAxisExcludeMatches(matrixRow.f_mir_code_exclude, employeeCodes)) { return false; }
    if (AxisExcludeMatches(matrixRow.f_collaborator_statuses_exclude, sStatus)) { return false; }

    if (!AxisMatches(matrixRow.f_position_names, sPosition)) { return false; }
    if (!AxisMatches(matrixRow.f_org_names, sOrg)) { return false; }
    if (!AxisMatches(matrixRow.f_subdivision_names, sSubdivision)) { return false; }
    if (!MirCodeAxisMatches(matrixRow.f_mir_code, employeeCodes)) { return false; }

    return true;
}

//-------------------------------------------------------------------------
//              Строки отчёта и цвет статуса (копия из выборки)
//-------------------------------------------------------------------------

function BuildStatusColor(bPassed, iDaysWorked, iDelayDays, iDurationDays)
{
    if (bPassed)
    {
        return "green";
    }
    if (iDaysWorked < 0)
    {
        return "red";
    }
    if (iDaysWorked >= (OptInt(iDelayDays, 0) + OptInt(iDurationDays, 0)))
    {
        return "red";
    }
    return "white";
}

function BuildReportRows(collaborator, macroSorted, citySorted, dateSorted, hireSorted, programNames, programIds, bInAudience)
{
    var rows, row, i, programID, bPassed, iDaysWorked, programTiming;
    rows = [];
    iDaysWorked = FindDaysWorked(hireSorted, Int(collaborator.id));
    for (i = 0; i < ArrayCount(programIds); i++)
    {
        programID = programIds[i];
        row = new Object();
        row.fullname = String(collaborator.fullname);
        row.position_name = String(collaborator.position_name);
        row.subdivision_name = String(collaborator.position_parent_name);
        row.macroregion = FindMacroregion(macroSorted, Int(collaborator.id));
        row.city = FindCity(citySorted, Int(collaborator.id));
        row.program_name = FindProgramName(programNames, programID);
        row.completion_date = FindCompletionDate(dateSorted, Int(collaborator.id), programID);
        row.in_audience = bInAudience;

        bPassed = (row.completion_date != "");
        row.is_passed = bPassed;
        programTiming = FindProgramTiming(programNames, programID);
        row.is_period_started = IsPeriodStarted(iDaysWorked, programTiming.delayDays);
        row.status_color = BuildStatusColor(bPassed, iDaysWorked, programTiming.delayDays, programTiming.days);

        rows.push(row);
    }
    return rows;
}

//-------------------------------------------------------------------------
//              HTML для Excel
//-------------------------------------------------------------------------

/*
* Цвет заливки ячейки статуса -- то же значение status_color, что получает виджет.
* @param {string} sStatusColor   -   "green" | "red" | "white".
* @returns {string}
*/
function StatusColorToHex(sStatusColor)
{
    if (sStatusColor == "green") { return "#00B050"; }
    if (sStatusColor == "red") { return "#FF0000"; }
    return "#FFFFFF";
}

/*
* Название режима для заголовка файла -- те же подписи, что в модалке "Выбор отчета".
* @param {string} sResultType
* @returns {string}
*/
function ResultTypeTitle(sResultType)
{
    if (sResultType == "plan") { return "План"; }
    if (sResultType == "fact") { return "Факт"; }
    if (sResultType == "mandatory") { return "Обязательно к прохождению"; }
    return "Общее кол-во";
}

/*
* Колонки -- как в Конфигурации виджета: status_color, fullname, position_name,
* subdivision_name, completion_date. Первая строка -- название режима отчёта.
* @param {Object[]} reportRows
* @param {string} sResultType
* @returns {string}
*/
function BuildExcelHtml(reportRows, sResultType)
{
    var html, i, row;

    html = new Binary();
    html.AppendStr("<html xmlns:o=\"urn:schemas-microsoft-com:office:office\" xmlns:x=\"urn:schemas-microsoft-com:office:excel\" xmlns=\"http://www.w3.org/TR/REC-html40\"><head><meta charset=\"utf-8\"></head><body>");
    html.AppendStr("<table border=\"1\" cellpadding=\"2\" cellspacing=\"0\">");

    html.AppendStr("<tr><td colspan=\"5\" bgcolor=\"#98ddfc\"><b>" + HtmlEncode(ResultTypeTitle(sResultType)) + "</b></td></tr>");

    html.AppendStr("<tr>");
    html.AppendStr("<td bgcolor=\"#FCE5CD\"><b>Статус</b></td>");
    html.AppendStr("<td bgcolor=\"#FCE5CD\"><b>ФИО</b></td>");
    html.AppendStr("<td bgcolor=\"#FCE5CD\"><b>Должность</b></td>");
    html.AppendStr("<td bgcolor=\"#FCE5CD\"><b>Подразделение</b></td>");
    html.AppendStr("<td bgcolor=\"#FCE5CD\"><b>Дата прохождения</b></td>");
    html.AppendStr("</tr>");

    for (i = 0; i < ArrayCount(reportRows); i++)
    {
        row = reportRows[i];
        html.AppendStr("<tr>");
        html.AppendStr("<td bgcolor=\"" + StatusColorToHex(String(row.status_color)) + "\"></td>");
        html.AppendStr("<td>" + HtmlEncode(String(row.fullname)) + "</td>");
        html.AppendStr("<td>" + HtmlEncode(String(row.position_name)) + "</td>");
        html.AppendStr("<td>" + HtmlEncode(String(row.subdivision_name)) + "</td>");
        html.AppendStr("<td align=\"center\">" + HtmlEncode(String(row.completion_date)) + "</td>");
        html.AppendStr("</tr>");
    }

    html.AppendStr("</table></body></html>");
    return html.GetStr();
}

//-------------------------------------------------------------------------
//              Точка входа
//-------------------------------------------------------------------------

function Run()
{
    var sResultType, sFullUrl, matrixId;
    var iProgramFilter, sMacroregionFilter, sMirCodeFilter, iPositionFilter, sCityFilter;
    var matrixContext, matrixRow, allProgramRows, taskRows;
    var programIds, programNames, collaboratorRows, macroRows, mirCodeRows, cityRows, dateRows, statusRows, hireRows;
    var macroSorted, citySorted, dateSorted, mirCodeSorted, statusSorted, hireSorted;
    var collaboratorReportRows, i, j, taskRowForName, bInAudience;
    var filteredProgramIds, filteredCollaboratorRows, allowedPositionIds, filteredResultRows;
    var reportRows, sHtml, sBareCurUserId;

    RESULT = new Object();

    try
    {
        alert("Run(). НАЧАЛО (Excel, ТЭП)");

        sFullUrl = GetCurPageUrlSafe();
        alert("Run(). URL страницы: [" + sFullUrl + "]");

        // Режим отчёта -- см. шапку файла (URL -> параметр кнопки -> "total").
        sResultType = GetQueryParam(sFullUrl, "result_type");
        if (sResultType == "")
        {
            sResultType = GetResultTypeParamSafe();
        }
        if (sResultType == "")
        {
            sResultType = "total";
        }
        if (sResultType != "total" && sResultType != "plan" && sResultType != "fact" && sResultType != "mandatory")
        {
            throw ("Неизвестный режим отчёта result_type=[" + sResultType + "] -- ожидается одно из: total, plan, fact, mandatory");
        }
        alert("Run(). sResultType=[" + sResultType + "]");

        matrixId = OptInt(GetQueryParam(sFullUrl, "matrix_id"), 0);
        iProgramFilter = OptInt(GetQueryParam(sFullUrl, "program_id"), 0);
        sMacroregionFilter = GetQueryParam(sFullUrl, "macroregion");
        sMirCodeFilter = GetQueryParam(sFullUrl, "mir_code");
        iPositionFilter = OptInt(GetQueryParam(sFullUrl, "position_common_id"), 0);
        sCityFilter = GetQueryParam(sFullUrl, "city");

        if (matrixId == 0)
        {
            throw ("Не выбрана матрица обучения -- настройте фильтры перед выгрузкой.");
        }

        allProgramRows = GetCompoundProgramRows();
        taskRows = GetEducationMethodTaskRows();

        matrixContext = ResolveMatrixContext(matrixId, allProgramRows, taskRows);
        matrixRow = matrixContext.matrixRow;
        programIds = matrixContext.programIds;

        if (iProgramFilter > 0)
        {
            filteredProgramIds = [];
            for (i = 0; i < ArrayCount(programIds); i++)
            {
                if (Int(programIds[i]) == iProgramFilter)
                {
                    filteredProgramIds.push(programIds[i]);
                }
            }
            programIds = filteredProgramIds;
            if (ArrayCount(programIds) == 0)
            {
                throw ("Программа [" + iProgramFilter + "] не найдена среди программ выбранной матрицы");
            }
        }

        programNames = [];
        for (j = 0; j < ArrayCount(programIds); j++)
        {
            taskRowForName = ArrayOptFind(taskRows, "Int(This.matrix_id) == Int(matrixId) && Int(This.education_method_id) == Int(programIds[j])");
            programNames.push({
                id: Int(programIds[j]),
                name: (taskRowForName != undefined && taskRowForName.pname != undefined ? String(taskRowForName.pname) : "id=" + programIds[j]),
                delay_days: (taskRowForName != undefined ? OptInt(taskRowForName.delay_days, 0) : 0),
                days: (taskRowForName != undefined ? OptInt(taskRowForName.days, 0) : 0)
            });
        }

        collaboratorRows = GetActiveCollaboratorRows();

        mirCodeRows = GetMirCodeRows();
        mirCodeSorted = SortRowsById(mirCodeRows);

        statusRows = GetStatusRows();
        statusSorted = SortRowsById(statusRows);

        hireRows = GetHireDateRows();
        hireSorted = SortRowsById(hireRows);

        // Ручные фильтры -- в том же порядке, что в выборке.
        if (iPositionFilter > 0)
        {
            allowedPositionIds = SortIdArray(GetPositionIdsByCommonPosition(iPositionFilter));
            filteredCollaboratorRows = [];
            for (i = 0; i < ArrayCount(collaboratorRows); i++)
            {
                if (IdArrayContainsSorted(allowedPositionIds, OptInt(collaboratorRows[i].position_id, 0)))
                {
                    filteredCollaboratorRows.push(collaboratorRows[i]);
                }
            }
            collaboratorRows = filteredCollaboratorRows;
        }

        macroRows = GetMacroregionRows();
        macroSorted = SortRowsById(macroRows);
        if (sMacroregionFilter != "")
        {
            filteredCollaboratorRows = [];
            for (i = 0; i < ArrayCount(collaboratorRows); i++)
            {
                if (FindMacroregion(macroSorted, Int(collaboratorRows[i].id)) == sMacroregionFilter)
                {
                    filteredCollaboratorRows.push(collaboratorRows[i]);
                }
            }
            collaboratorRows = filteredCollaboratorRows;
        }

        if (sMirCodeFilter != "")
        {
            filteredCollaboratorRows = [];
            for (i = 0; i < ArrayCount(collaboratorRows); i++)
            {
                if (CollaboratorHasMirCode(mirCodeSorted, Int(collaboratorRows[i].id), sMirCodeFilter))
                {
                    filteredCollaboratorRows.push(collaboratorRows[i]);
                }
            }
            collaboratorRows = filteredCollaboratorRows;
        }

        cityRows = GetCityRows();
        citySorted = SortRowsById(cityRows);

        if (sCityFilter != "")
        {
            filteredCollaboratorRows = [];
            for (i = 0; i < ArrayCount(collaboratorRows); i++)
            {
                if (FindCity(citySorted, Int(collaboratorRows[i].id)) == sCityFilter)
                {
                    filteredCollaboratorRows.push(collaboratorRows[i]);
                }
            }
            collaboratorRows = filteredCollaboratorRows;
        }

        dateRows = GetCompletionDateRows(programIds);
        dateSorted = SortDateRowsByCollaboratorId(dateRows);

        // Строки (сотрудник x программа) по аудитории -- как в выборке.
        reportRows = [];
        for (i = 0; i < ArrayCount(collaboratorRows); i++)
        {
            bInAudience = CollaboratorMatchesMatrixAudience(collaboratorRows[i], matrixRow, mirCodeSorted, statusSorted);
            if (!bInAudience)
            {
                continue;
            }
            collaboratorReportRows = BuildReportRows(collaboratorRows[i], macroSorted, citySorted, dateSorted, hireSorted, programNames, programIds, bInAudience);
            for (j = 0; j < ArrayCount(collaboratorReportRows); j++)
            {
                reportRows.push(collaboratorReportRows[j]);
            }
        }

        // Разбиение по режиму -- как в выборке.
        if (sResultType == "plan")
        {
            filteredResultRows = [];
            for (i = 0; i < ArrayCount(reportRows); i++)
            {
                if (reportRows[i].is_period_started)
                {
                    filteredResultRows.push(reportRows[i]);
                }
            }
            reportRows = filteredResultRows;
        }
        else if (sResultType == "fact")
        {
            filteredResultRows = [];
            for (i = 0; i < ArrayCount(reportRows); i++)
            {
                if (reportRows[i].completion_date != "")
                {
                    filteredResultRows.push(reportRows[i]);
                }
            }
            reportRows = filteredResultRows;
        }
        else if (sResultType == "mandatory")
        {
            filteredResultRows = [];
            for (i = 0; i < ArrayCount(reportRows); i++)
            {
                if (reportRows[i].completion_date == "")
                {
                    filteredResultRows.push(reportRows[i]);
                }
            }
            reportRows = filteredResultRows;
        }

        if (ArrayCount(reportRows) == 0)
        {
            RESULT = { command: "alert", view: "warning", msg: "Нет данных для выгрузки -- таблица отчёта пуста." };
            alert("Run(). КОНЕЦ -- пусто, result_type=" + sResultType);
            return;
        }

        sHtml = BuildExcelHtml(reportRows, sResultType);

        // Ключ кэша -- "голая" curUserID, как в работающей выгрузке "Процента обученных".
        // Фолбэк -- curUserId из LPE (если привязан).
        try { sBareCurUserId = String(curUserID); }
        catch (_exBareUserId) { sBareCurUserId = ""; }
        if (sBareCurUserId == "" || sBareCurUserId == "undefined" || sBareCurUserId == "null")
        {
            try { sBareCurUserId = String(curUserId); }
            catch (_exLpeUserId) { sBareCurUserId = ""; }
        }
        tools_web.set_user_data("excel_html_" + sBareCurUserId, ({ "html": sHtml }), 3600);

        RESULT = {
            command: "alert",
            msg: "Файл готов к выгрузке",
            confirm_result: {
                command: "new_window",
                url: EXPORT_PAGE_URL
            }
        };

        alert("Run(). КОНЕЦ (успех), result_type=" + sResultType + ", строк: " + ArrayCount(reportRows));
    }
    catch (_ex)
    {
        RESULT = { command: "alert", view: "warning", msg: ExtractUserError(_ex) };
        alert("Run(). ОШИБКА: " + ExtractUserError(_ex));
    }
}

Run();
