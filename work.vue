// =====================================================================
// HREDU-239. "Полный отчёт по матрицам" -- удалённое действие на кнопку "Выгрузить отчёт".
//
// Строит Excel (HTML-таблица с Office-namespace): одна строка на сотрудника (ФИО,
// Должность, Подразделение, Макрорегион) и по одной колонке на каждую учебную программу
// выбранной матрицы -- дата прохождения или пусто. Порядок колонок -- порядок задач в
// матрице.
//
// Фильтры matrix_id и macroregion берутся из URL страницы (cur_page_url ->
// {{curEnv.curEnvUrl}}). Видимость: член УОРиАП видит всех, остальные -- только
// подчинённых по всей цепочке вниз (curUserId -> {{curUser.id}} в LPE).
//
// Готовый HTML кладётся в кэш tools_web.set_user_data("excel_html_" + curUserID), его
// отдаёт на скачивание страница assessment_excel_export.html.
// =====================================================================

//-------------------------------------------------------------------------
//              Область констант
//-------------------------------------------------------------------------

DEBUG = false;                          // Включает подробные логи для дебага
LOG_NAME = "remote_action";             // Имя журнала на сервере, куда будут писаться логи
CUR_OBJECT_ID = 7681618713416661052;    // ID текущего удалённого действия для подстановки в логи

EXPORT_PAGE_URL = "assessment_excel_export.html";

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
* URL текущей страницы: в удалённом действии Request.Url ненадёжен, основной источник --
* параметр cur_page_url.
*/
function GetCurPageUrlSafe()
{
    var sUrl;
    try { sUrl = String(PARAMETERS.GetOptProperty("cur_page_url", "")); }
    catch (_exParam) { sUrl = ""; }
    if (sUrl != "")
    {
        return sUrl;
    }
    LogAlert(3, "GetCurPageUrlSafe(). Параметр cur_page_url пуст -- пробуем Request.Url");
    try { return String(Request.Url); }
    catch (_ex) { return ""; }
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
* ID текущего пользователя из LPE-параметра curUserId; 0, если параметр не привязан.
*/
function GetCurUserIdSafe()
{
    var sRaw;
    try
    {
        sRaw = String(curUserId);
        if (sRaw != "" && sRaw != "0" && sRaw != "undefined" && sRaw != "null")
        {
            return curUserId;
        }
    }
    catch (_exDirect)
    {
        LogAlert(3, "GetCurUserIdSafe(). curUserId недоступна (параметр не привязан в LPE?): " + ExtractUserError(_exDirect));
    }
    LogAlert(3, "GetCurUserIdSafe(). curUserId не дала валидного значения -- возвращаем 0.");
    return 0;
}

/*
* Входит ли пользователь в группу УОРиАП. Сравнение ID -- строками (19-значные ID).
*/
function IsUorApMember(iCurUserId)
{
    var groupRows, iGroupId, groupDoc, collabList, foundRow, sRawUserId, sGroupId, sGroupXQueryType;

    sGroupId = "7687978602560891542";
    sGroupXQueryType = "groups";

    sRawUserId = String(iCurUserId);
    if (sRawUserId == "" || sRawUserId == "0" || sRawUserId == "undefined" || sRawUserId == "null")
    {
        return false;
    }

    try
    {
        groupRows = ArraySelectAll(XQuery(
            "for $elem in " + sGroupXQueryType + " where $elem/id = '" + sGroupId + "' return $elem"
        ));
        if (ArrayCount(groupRows) == 0)
        {
            throw ("Группа с id=" + sGroupId + " не найдена через тип '" + sGroupXQueryType + "'");
        }
        iGroupId = Int(groupRows[0].id);
        groupDoc = tools.open_doc(iGroupId).TopElem;
        collabList = groupDoc.collaborators.collaborator;
        foundRow = ArrayOptFind(collabList, "String(This.collaborator_id) == String(iCurUserId)");
        LogAlert(1, "IsUorApMember(). userId=" + sRawUserId + "; совпадение=" + (foundRow != undefined));
        return (foundRow != undefined);
    }
    catch (_exDoc)
    {
        LogAlert(4, "IsUorApMember(). ОШИБКА: " + ExtractUserError(_exDoc) + " -- считаем, что не УОРиАП.");
        return false;
    }
}

/*
* {id, manager_id} всех действующих сотрудников, manager_id -- непосредственный
* руководитель (func_manager с is_native=true).
*/
function GetManagerIdRows()
{
    var sqlText, rows;
    sqlText = "";
    sqlText = sqlText + "select cs.id,\r\n";
    sqlText = sqlText + "       c.data.value('(*/func_managers/func_manager[is_native=true()]/person_id)[1]', 'varchar(max)') as manager_id\r\n";
    sqlText = sqlText + "from collaborators cs\r\n";
    sqlText = sqlText + "inner join collaborator c on c.id = cs.id\r\n";
    sqlText = sqlText + "where cs.is_dismiss != 1";
    rows = ArraySelectAll(XQuery("sql:" + sqlText));
    return rows;
}

function SortRowsByManagerId(managerRows)
{
    return ArraySort(managerRows, "OptInt(This.manager_id, 0)", "+");
}

/*
* Индекс первой строки с данным manager_id (бинарный поиск), -1 если нет.
*/
function FindManagerRangeStart(sortedByManagerRows, iManagerId)
{
    var lo, hi, mid, midVal, iTarget, startIdx;
    iTarget = Int(iManagerId);
    lo = 0;
    hi = ArrayCount(sortedByManagerRows) - 1;
    startIdx = -1;
    while (lo <= hi)
    {
        mid = Int((lo + hi) / 2);
        midVal = OptInt(sortedByManagerRows[mid].manager_id, 0);
        if (midVal == iTarget) { startIdx = mid; hi = mid - 1; }
        else if (midVal < iTarget) { lo = mid + 1; }
        else { hi = mid - 1; }
    }
    return startIdx;
}

/*
* Прямые подчинённые руководителя (один уровень). Самого руководителя исключаем --
* у вершины иерархии is_native-запись указывает на него самого.
*/
function CollectDirectSubordinateIds(sortedByManagerRows, iManagerId)
{
    var startIdx, i, n, result, iTarget, iRowId;
    result = [];
    iTarget = Int(iManagerId);
    startIdx = FindManagerRangeStart(sortedByManagerRows, iTarget);
    if (startIdx == -1) { return result; }
    n = ArrayCount(sortedByManagerRows);
    for (i = startIdx; i < n && OptInt(sortedByManagerRows[i].manager_id, 0) == iTarget; i++)
    {
        iRowId = Int(sortedByManagerRows[i].id);
        if (iRowId != iTarget) { result.push(iRowId); }
    }
    return result;
}

/*
* Все подчинённые по цепочке вниз -- BFS без рекурсии, с защитой от циклов.
*/
function GetManagerHierarchySubordinateIds(sortedByManagerRows, iCurUserId)
{
    var visitedIds, visitedSorted, frontier, nextFrontier, allSubordinates;
    var i, j, directIds, iManagerId, iSubId;

    allSubordinates = [];
    visitedIds = [Int(iCurUserId)];
    frontier = [Int(iCurUserId)];

    while (ArrayCount(frontier) > 0)
    {
        visitedSorted = SortIdArray(visitedIds);
        nextFrontier = [];
        for (i = 0; i < ArrayCount(frontier); i++)
        {
            iManagerId = frontier[i];
            directIds = CollectDirectSubordinateIds(sortedByManagerRows, iManagerId);
            for (j = 0; j < ArrayCount(directIds); j++)
            {
                iSubId = directIds[j];
                if (!IdArrayContainsSorted(visitedSorted, iSubId))
                {
                    allSubordinates.push(iSubId);
                    visitedIds.push(iSubId);
                    nextFrontier.push(iSubId);
                }
            }
        }
        frontier = nextFrontier;
    }
    return allSubordinates;
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

function IsActiveText(sValue)
{
    try { return tools_web.is_true(sValue); }
    catch (_ex) { return (String(sValue) == "true" || String(sValue) == "1"); }
}

/*
* Все модульные программы (compound_program) с полями аудитории -- одним SQL.
*/
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
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_collaborator_statuses_exclude'']/value)[1]', 'varchar(max)') as f_collaborator_statuses_exclude\r\n";
    sqlText = sqlText + "from compound_programs cs\r\n";
    sqlText = sqlText + "inner join compound_program c on c.id = cs.id";
    return ArraySelectAll(XQuery("sql:" + sqlText));
}

/*
* Задачи типа "Учебная программа" (education_method) всех модульных программ -- одним SQL.
*/
function GetEducationMethodTaskRows()
{
    var sqlText;
    sqlText = "";
    sqlText = sqlText + "select cs.id as matrix_id,\r\n";
    sqlText = sqlText + "       t.p.value('(education_method_id)[1]', 'bigint') as education_method_id,\r\n";
    sqlText = sqlText + "       t.p.value('(type)[1]', 'varchar(50)') as ptype,\r\n";
    sqlText = sqlText + "       t.p.value('(name)[1]', 'varchar(max)') as pname\r\n";
    sqlText = sqlText + "from compound_programs cs\r\n";
    sqlText = sqlText + "inner join compound_program c on c.id = cs.id\r\n";
    sqlText = sqlText + "cross apply c.data.nodes('/*/programs/program') as t(p)\r\n";
    sqlText = sqlText + "where t.p.value('(type)[1]', 'varchar(50)') = 'education_method'";
    return ArraySelectAll(XQuery("sql:" + sqlText));
}

/*
* Колонки-программы выбранной матрицы {id, name} в порядке задач, без дублей.
*/
function GetOrderedProgramColumns(taskRows, matrixId)
{
    var columns, seenIds, i, iProgramId;
    columns = [];
    seenIds = [];
    for (i = 0; i < ArrayCount(taskRows); i++)
    {
        if (Int(taskRows[i].matrix_id) != Int(matrixId)) { continue; }
        iProgramId = OptInt(taskRows[i].education_method_id, 0);
        if (iProgramId <= 0) { continue; }
        if (ArrayOptFind(seenIds, "Int(This) == Int(iProgramId)") != undefined) { continue; }
        seenIds.push(iProgramId);
        columns.push({ id: iProgramId, name: String(taskRows[i].pname) });
    }
    return columns;
}

/*
* Строка матрицы по id; ошибка, если не найдена или деактивирована.
*/
function ResolveMatrixRow(matrixId, allProgramRows)
{
    var matrixRow;
    matrixRow = ArrayOptFind(allProgramRows, "Int(This.id) == Int(matrixId)");
    if (matrixRow == undefined)
    {
        throw ("Не найдено модульной программы (compound_program) с id=" + matrixId);
    }
    if (!IsActiveText(matrixRow.f_matrix_active))
    {
        throw ("Модульная программа [" + matrixRow.name + "] (id=" + matrixId + ") деактивирована (f_matrix_active) -- отчёт недоступен для деактивированных программ.");
    }
    return matrixRow;
}

function GetActiveCollaboratorRows()
{
    return ArraySelectAll(XQuery("for $elem in collaborators where $elem/is_dismiss=false() return $elem"));
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

function FindMacroregion(sortedMacroRows, collaboratorID)
{
    var macroRow;
    macroRow = BinarySearchById(sortedMacroRows, collaboratorID);
    return (macroRow != undefined && macroRow.macroregion != undefined ? String(macroRow.macroregion) : "");
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

/*
* Замена .split() (не работает на платформе).
*/
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
        if (iPos == undefined) { parts.push(StrRangePos(sText, iStart, iLen)); break; }
        parts.push(StrRangePos(sText, iStart, iPos));
        iStart = iPos + iDelimLen;
    }
    return parts;
}

/*
* f_mir_codes сотрудника ("КОД#...|КОД#...") -> массив кодов.
*/
function ExtractMirCodes(rawValue)
{
    var parts, fields, codes, i;
    codes = [];
    parts = ArrayDirect(ArraySelect(SplitByDelimiter(String(rawValue), "|"), "This != ''"));
    for (i = 0; i < ArrayCount(parts); i++)
    {
        fields = ArrayDirect(ArraySelect(SplitByDelimiter(String(parts[i]), "#"), "This != ''"));
        if (ArrayCount(fields) > 0) { codes.push(String(fields[0])); }
    }
    return codes;
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

function SplitByStar(sPattern)
{
    var parts, iLen, iStart, iPos;
    parts = [];
    iLen = StrLen(sPattern);
    iStart = 0;
    while (true)
    {
        iPos = StrOptSubStrPos(sPattern, "*", true, iStart);
        if (iPos == undefined) { parts.push(StrRangePos(sPattern, iStart, iLen)); break; }
        parts.push(StrRangePos(sPattern, iStart, iPos));
        iStart = iPos + 1;
    }
    return parts;
}

/*
* Матчинг wildcard-паттерна ("*текст*") без regex. 3-й параметр StrOptSubStrPos:
* true -- игнорировать регистр.
*/
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

/*
* true, если текст подходит хотя бы под один паттерн из ";"-списка.
*/
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

/*
* Положительная ось аудитории: пустой список паттернов -- без ограничения.
*/
function AxisMatches(sPatternsList, sSingleValue)
{
    if (sPatternsList == undefined || String(sPatternsList) == "") { return true; }
    return MatchAnySemicolonPattern(sPatternsList, sSingleValue, true);
}

/*
* Ось-исключение: пустой список паттернов -- никого не исключаем.
*/
function AxisExcludeMatches(sPatternsList, sSingleValue)
{
    if (sPatternsList == undefined || String(sPatternsList) == "") { return false; }
    return MatchAnySemicolonPattern(sPatternsList, sSingleValue, true);
}

/*
* Ось мир-кода: у сотрудника может быть несколько кодов, достаточно совпадения одного.
*/
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

/*
* Входит ли сотрудник в аудиторию матрицы: сначала исключения (хватает одного
* совпадения), затем "И" между заполненными осями.
*/
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

/*
* Минимальная дата прохождения по каждой паре (сотрудник, программа).
*/
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

function SortDateRowsByCollaboratorId(dateRows)
{
    return ArraySort(dateRows, "Int(This.collaborator_id)", "+");
}

/*
* Дата прохождения сотрудником программы: бинарный поиск первой строки сотрудника,
* затем короткий проход по его строкам. "" -- не проходил.
*/
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

/*
* HTML-таблица для Excel: шапка с названием матрицы, заголовки колонок, строки сотрудников.
* @param {string} sMatrixName
* @param {Object[]} programColumns   -   {id, name}.
* @param {Object[]} employeeRows     -   {id, fullname, position_name, position_parent_name, macroregion}.
* @param {Object[]} sortedDateRows
* @returns {string}
*/
function BuildExcelHtml(sMatrixName, programColumns, employeeRows, sortedDateRows)
{
    var html, i, j, emp, sDate;

    html = new Binary();
    html.AppendStr("<html xmlns:o=\"urn:schemas-microsoft-com:office:office\" xmlns:x=\"urn:schemas-microsoft-com:office:excel\" xmlns=\"http://www.w3.org/TR/REC-html40\"><head><meta charset=\"utf-8\"></head><body>");
    html.AppendStr("<table border=\"1\" cellpadding=\"2\" cellspacing=\"0\">");

    html.AppendStr("<tr><td colspan=\"4\"></td>");
    for (j = 0; j < ArrayCount(programColumns); j++)
    {
        html.AppendStr("<td bgcolor=\"#98ddfc\"><b>" + HtmlEncode(sMatrixName) + "</b></td>");
    }
    html.AppendStr("</tr>");

    html.AppendStr("<tr>");
    html.AppendStr("<td bgcolor=\"#FCE5CD\"><b>ФИО</b></td>");
    html.AppendStr("<td bgcolor=\"#FCE5CD\"><b>Должность</b></td>");
    html.AppendStr("<td bgcolor=\"#FCE5CD\"><b>Подразделение</b></td>");
    html.AppendStr("<td bgcolor=\"#FCE5CD\"><b>Макрорегион</b></td>");
    for (j = 0; j < ArrayCount(programColumns); j++)
    {
        html.AppendStr("<td bgcolor=\"#FCE5CD\"><b>" + HtmlEncode(programColumns[j].name) + "</b></td>");
    }
    html.AppendStr("</tr>");

    for (i = 0; i < ArrayCount(employeeRows); i++)
    {
        emp = employeeRows[i];
        html.AppendStr("<tr>");
        html.AppendStr("<td>" + HtmlEncode(emp.fullname) + "</td>");
        html.AppendStr("<td>" + HtmlEncode(emp.position_name) + "</td>");
        html.AppendStr("<td>" + HtmlEncode(emp.position_parent_name) + "</td>");
        html.AppendStr("<td>" + HtmlEncode(emp.macroregion) + "</td>");
        for (j = 0; j < ArrayCount(programColumns); j++)
        {
            sDate = FindCompletionDate(sortedDateRows, Int(emp.id), programColumns[j].id);
            html.AppendStr("<td align=\"center\">" + HtmlEncode(sDate) + "</td>");
        }
        html.AppendStr("</tr>");
    }

    html.AppendStr("</table></body></html>");
    return html.GetStr();
}

/*
* Матрица по id и её колонки-программы; ошибка, если матрица не выбрана или у неё нет
* учебных программ.
* @param {number} iMatrixId
* @returns {Object}   -   {matrixRow, programColumns}.
*/
function ResolveMatrixContext(iMatrixId)
{
    var allProgramRows, taskRows, matrixRow, programColumns;

    if (iMatrixId == 0)
    {
        throw "Не выбрана матрица обучения -- настройте фильтры перед выгрузкой.";
    }

    allProgramRows = GetCompoundProgramRows();
    taskRows = GetEducationMethodTaskRows();

    matrixRow = ResolveMatrixRow(iMatrixId, allProgramRows);
    programColumns = GetOrderedProgramColumns(taskRows, iMatrixId);
    if (ArrayCount(programColumns) == 0)
    {
        throw ("У модульной программы [" + matrixRow.name + "] не найдено ни одной задачи с типом \"Учебная программа\" (education_method)");
    }
    return { matrixRow: matrixRow, programColumns: programColumns };
}

/*
* Отсортированные ID подчинённых пользователя (вся цепочка вниз) или undefined для
* члена УОРиАП (без ограничения). Нет подчинённых -- ошибка.
* @param {number} iCurUserId
* @returns {number[]}
*/
function GetSortedSubordinateIds(iCurUserId)
{
    var managerRows, sortedByManagerRows, subordinateIds;

    if (IsUorApMember(iCurUserId))
    {
        LogAlert(1, "GetSortedSubordinateIds(). userId=" + iCurUserId + " -- УОРиАП, без ограничения.");
        return undefined;
    }

    managerRows = GetManagerIdRows();
    sortedByManagerRows = SortRowsByManagerId(managerRows);
    subordinateIds = GetManagerHierarchySubordinateIds(sortedByManagerRows, iCurUserId);
    LogAlert(1, "GetSortedSubordinateIds(). userId=" + iCurUserId + "; подчинённых: " + ArrayCount(subordinateIds));
    if (ArrayCount(subordinateIds) == 0)
    {
        throw "У вас нет ни одного подчинённого -- отчёт недоступен.";
    }
    return SortIdArray(subordinateIds);
}

/*
* Оставляет в пуле только подчинённых; sortedSubordinateIds == undefined -- без изменений.
* @param {Object[]} activeRows
* @param {number[]} sortedSubordinateIds
* @returns {Object[]}
*/
function FilterBySubordinates(activeRows, sortedSubordinateIds)
{
    var subFilteredRows, k;

    if (sortedSubordinateIds == undefined)
    {
        return activeRows;
    }
    subFilteredRows = [];
    for (k = 0; k < ArrayCount(activeRows); k++)
    {
        if (IdArrayContainsSorted(sortedSubordinateIds, Int(activeRows[k].id))) { subFilteredRows.push(activeRows[k]); }
    }
    return subFilteredRows;
}

/*
* Строки сотрудников для Excel: аудитория матрицы + фильтр по макрорегиону.
* @param {Object[]} activeRows
* @param {Object} matrixRow
* @param {string} sMacroregionFilter
* @returns {Object[]}
*/
function CollectEmployeeRows(activeRows, matrixRow, sMacroregionFilter)
{
    var macroSorted, mirCodeSorted, statusSorted, employeeRows, i, bInAudience, sMacroregionForRow;

    macroSorted = SortRowsById(GetMacroregionRows());
    mirCodeSorted = SortRowsById(GetMirCodeRows());
    statusSorted = SortRowsById(GetStatusRows());

    employeeRows = [];
    for (i = 0; i < ArrayCount(activeRows); i++)
    {
        bInAudience = CollaboratorMatchesMatrixAudience(activeRows[i], matrixRow, mirCodeSorted, statusSorted);
        if (!bInAudience) { continue; }

        sMacroregionForRow = FindMacroregion(macroSorted, Int(activeRows[i].id));
        if (sMacroregionFilter != "" && sMacroregionForRow != sMacroregionFilter) { continue; }

        employeeRows.push({
            id: Int(activeRows[i].id),
            fullname: String(activeRows[i].fullname),
            position_name: String(activeRows[i].position_name),
            position_parent_name: String(activeRows[i].position_parent_name),
            macroregion: sMacroregionForRow
        });
    }
    LogAlert(1, "CollectEmployeeRows(). Сотрудников в пуле: " + ArrayCount(activeRows) + "; в выгрузке: " + ArrayCount(employeeRows));
    return employeeRows;
}

/*
* Кладёт HTML в кэш и возвращает команду открыть страницу скачивания. Ключ кэша --
* "голая" curUserID (её читает assessment_excel_export.html), запасной -- iCurUserId.
* @param {string} sHtml
* @param {number} iCurUserId
* @returns {Object}
*/
function SaveExportAndGetResult(sHtml, iCurUserId)
{
    var sBareCurUserId;

    try { sBareCurUserId = String(curUserID); }
    catch (_exBareUserId) { sBareCurUserId = ""; }
    if (sBareCurUserId == "" || sBareCurUserId == "undefined" || sBareCurUserId == "null")
    {
        LogAlert(3, "SaveExportAndGetResult(). curUserID недоступна -- ключ кэша по curUserId=" + iCurUserId);
        sBareCurUserId = String(iCurUserId);
    }
    tools_web.set_user_data("excel_html_" + sBareCurUserId, ({ "html": sHtml }), 3600);

    return {
        command: "alert",
        msg: "Файл готов в выгрузке",
        confirm_result: {
            command: "new_window",
            url: EXPORT_PAGE_URL
        }
    };
}

/*
* Точка входа.
* @returns {void}
*/
function Run()
{
    var sPageUrl, iMatrixId, sMacroregionFilter, matrixContext;
    var iCurUserId, sortedSubordinateIds, activeRows, dateSorted, employeeRows, sHtml;

    ERROR = 0;
    MESSAGE = "";
    RESULT = new Object();

    LogAlert(2, "Run(). НАЧАЛО");

    try
    {
        iCurUserId = GetCurUserIdSafe();
        sortedSubordinateIds = GetSortedSubordinateIds(iCurUserId);

        sPageUrl = GetCurPageUrlSafe();
        iMatrixId = OptInt(GetQueryParam(sPageUrl, "matrix_id"), 0);
        sMacroregionFilter = GetQueryParam(sPageUrl, "macroregion");
        LogAlert(1, "Run(). URL=[" + sPageUrl + "]; matrix_id=" + iMatrixId + "; macroregion=[" + sMacroregionFilter + "]");

        matrixContext = ResolveMatrixContext(iMatrixId);
        activeRows = FilterBySubordinates(GetActiveCollaboratorRows(), sortedSubordinateIds);
        dateSorted = SortDateRowsByCollaboratorId(GetCompletionDateRows(ArrayExtract(matrixContext.programColumns, "Int(id)")));
        employeeRows = CollectEmployeeRows(activeRows, matrixContext.matrixRow, sMacroregionFilter);

        if (ArrayCount(employeeRows) == 0)
        {
            RESULT = { command: "alert", view: "warning", msg: "Данных не найдено -- попробуйте изменить параметры фильтра (матрица/макрорегион) или проверьте, что у вас/ваших подчинённых есть сотрудники в аудитории этой матрицы." };
            LogAlert(3, "Run(). Данных не найдено -- файл не сформирован.");
        }
        else
        {
            employeeRows = ArraySort(employeeRows, "This.fullname", "+");
            sHtml = BuildExcelHtml(String(matrixContext.matrixRow.name), matrixContext.programColumns, employeeRows, dateSorted);
            RESULT = SaveExportAndGetResult(sHtml, iCurUserId);
            LogAlert(2, "Run(). Выгрузка готова: сотрудников " + ArrayCount(employeeRows) + ", колонок-программ " + ArrayCount(matrixContext.programColumns));
        }
    }
    catch (_ex)
    {
        ERROR = 1;
        MESSAGE = String(_ex);
        RESULT = { command: "alert", view: "warning", msg: MESSAGE };
        LogAlert(4, "Run(). КРИТИЧЕСКАЯ ОШИБКА: " + MESSAGE);
    }

    LogAlert(2, "Run(). КОНЕЦ");
}

//-------------------------------------------------------------------------
//              Область основного кода
//-------------------------------------------------------------------------

Run();
