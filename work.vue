// =====================================================================
// HREDU-182. "Процент обученных" -- удалённое действие на кнопку "Выгрузить в Excel".
//
// Выгружает ту же таблицу, что показывает выборка HREDU-182_procent_obuchennyh.js: строки
// (город x учебная программа), колонки Общее / План / Факт / Процент / Обязательно и
// итоговая строка "Общий итог", в том же порядке. Расчёт -- копия выборки: при изменении
// логики в выборке нужно так же поменять её здесь, иначе Excel разойдётся с таблицей.
//
// Фильтры берутся из URL страницы (cur_page_url -> {{curEnv.curEnvUrl}} в LPE).
// Видимость -- только сужение пула, как в выборке: член УОРиАП видит всех, остальные --
// подчинённых по всей цепочке вниз (curUserId -> {{curUser.id}} в LPE). Нет подчинённых --
// сообщение "нет данных" (на странице в этом случае тоже пустая таблица).
//
// Готовый HTML кладётся в кэш tools_web.set_user_data("excel_html_" + curUserID), его
// отдаёт на скачивание страница assessment_excel_export.html.
// =====================================================================

//-------------------------------------------------------------------------
//              Область констант
//-------------------------------------------------------------------------

DEBUG = false;                          // Включает подробные логи для дебага
LOG_NAME = "remote_action";             // Имя журнала на сервере, куда будут писаться логи
CUR_OBJECT_ID = 7698662221284771962;    // ID текущего удалённого действия для подстановки в логи

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
    if (sUrl != "" && sUrl != "undefined" && sUrl != "null")
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
        LogAlert(3, "GetCurUserIdSafe(). curUserId недоступна (не привязана в LPE к кнопке?): " + ExtractUserError(_exDirect));
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
    var sqlText;
    sqlText = "";
    sqlText = sqlText + "select cs.id,\r\n";
    sqlText = sqlText + "       c.data.value('(*/func_managers/func_manager[is_native=true()]/person_id)[1]', 'varchar(max)') as manager_id\r\n";
    sqlText = sqlText + "from collaborators cs\r\n";
    sqlText = sqlText + "inner join collaborator c on c.id = cs.id\r\n";
    sqlText = sqlText + "where cs.is_dismiss != 1";
    return ArraySelectAll(XQuery("sql:" + sqlText));
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
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_subdivision_child'']/value)[1]', 'varchar(max)') as f_subdivision_child,\r\n";
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
    sqlText = sqlText + "       t.p.value('(object_id)[1]', 'bigint') as object_id,\r\n";
    sqlText = sqlText + "       t.p.value('(education_method_id)[1]', 'bigint') as education_method_id,\r\n";
    sqlText = sqlText + "       t.p.value('(type)[1]', 'varchar(50)') as ptype,\r\n";
    sqlText = sqlText + "       t.p.value('(delay_days)[1]', 'int') as delay_days,\r\n";
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

/*
* Матрица по id и её учебные программы; ошибка, если матрица не найдена, деактивирована
* или в ней нет учебных программ.
*/
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
        throw ("У модульной программы [" + matrixRow.name + "] не найдено ни одной задачи с типом \"Учебная программа\" (education_method)");
    }
    return { matrixRow: matrixRow, programIds: programIds };
}

function FindProgramName(programNames, iProgramId)
{
    var row;
    row = ArrayOptFind(programNames, "Int(This.id) == Int(iProgramId)");
    return (row != undefined ? String(row.name) : "id=" + iProgramId);
}

/*
* "Старт через" (delay_days) программы.
*/
function FindProgramDelayDays(programNames, iProgramId)
{
    var row;
    row = ArrayOptFind(programNames, "Int(This.id) == Int(iProgramId)");
    return (row != undefined ? OptInt(row.delay_days, 0) : 0);
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

/*
* Дней с даты приёма до сегодня -- считается в SQL (DATEDIFF).
*/
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

/*
* Дней с даты приёма; -1, если даты приёма нет.
*/
function FindDaysWorked(sortedHireRows, collaboratorID)
{
    var row;
    row = BinarySearchById(sortedHireRows, collaboratorID);
    return (row != undefined ? OptInt(row.days_worked, -1) : -1);
}

/*
* ТЗ п.3: период прохождения наступил, если дней с приёма >= "Старт через". Нет даты
* приёма -- считаем наступившим.
*/
function IsPeriodStarted(iDaysWorked, iDelayDays)
{
    if (iDaysWorked < 0)
    {
        return true;
    }
    return (iDaysWorked >= OptInt(iDelayDays, 0));
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

function CollaboratorHasMirCode(sortedMirCodeRows, collaboratorID, mirCodeFilter)
{
    var row, codes;
    row = BinarySearchById(sortedMirCodeRows, collaboratorID);
    if (row == undefined) { return false; }
    codes = ExtractMirCodes(row.mir_codes);
    return (ArrayOptFind(codes, "String(This) == String(mirCodeFilter)") != undefined);
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
* Ручные фильтры страницы: типовая должность, макрорегион, мир-код.
* @param {Object[]} collaboratorRows
* @param {Object} filters   -   см. ReadFilters().
* @param {Object} lookups   -   см. LoadLookups().
* @returns {Object[]}
*/
function ApplyManualFilters(collaboratorRows, filters, lookups)
{
    var allowedPositionIds, filteredRows, i, macroRow, iPositionFilter, sMacroregionFilter, sMirCodeFilter;

    iPositionFilter = filters.positionCommonId;
    sMacroregionFilter = filters.macroregion;
    sMirCodeFilter = filters.mirCode;
 
    filteredRows = collaboratorRows;
 
    if (iPositionFilter > 0)
    {
        allowedPositionIds = SortIdArray(GetPositionIdsByCommonPosition(iPositionFilter));
        collaboratorRows = filteredRows;
        filteredRows = [];
        for (i = 0; i < ArrayCount(collaboratorRows); i++)
        {
            if (IdArrayContainsSorted(allowedPositionIds, OptInt(collaboratorRows[i].position_id, 0))) { filteredRows.push(collaboratorRows[i]); }
        }
    }
 
    if (sMacroregionFilter != "")
    {
        collaboratorRows = filteredRows;
        filteredRows = [];
        for (i = 0; i < ArrayCount(collaboratorRows); i++)
        {
            macroRow = BinarySearchById(lookups.macroSorted, Int(collaboratorRows[i].id));
            if (macroRow != undefined && String(macroRow.macroregion) == sMacroregionFilter) { filteredRows.push(collaboratorRows[i]); }
        }
    }
 
    if (sMirCodeFilter != "")
    {
        collaboratorRows = filteredRows;
        filteredRows = [];
        for (i = 0; i < ArrayCount(collaboratorRows); i++)
        {
            if (CollaboratorHasMirCode(lookups.mirCodeSorted, Int(collaboratorRows[i].id), sMirCodeFilter)) { filteredRows.push(collaboratorRows[i]); }
        }
    }
 
    return filteredRows;
}

/*
* Накопитель для пары (город, программа) -- находит существующий или создаёт новый.
*/
function GetOrCreateCityProgramAcc(acc, sCity, iProgramId, sProgramName, sMacroregion)
{
    var i, newAcc;
    for (i = 0; i < ArrayCount(acc); i++)
    {
        if (acc[i].city == sCity && Int(acc[i].programId) == Int(iProgramId)) { return acc[i]; }
    }
    newAcc = { city: sCity, programId: Int(iProgramId), programName: sProgramName, macroregion: sMacroregion, total: 0, plan: 0, mandatory: 0, fact: 0 };
    acc.push(newAcc);
    return newAcc;
}

/*
* факт/план в % с округлением; "-" при плане 0. Сначала умножение, потом деление --
* деление целых в движке теряет дробную часть.
*/
function FormatPercent(nFact, nPlan)
{
    var iFact, iPlan, iRounded;
    if (nPlan <= 0) { return "-"; }
    iFact = Int(nFact);
    iPlan = Int(nPlan);
    iRounded = Int((iFact * 100 + Int(iPlan / 2)) / iPlan);
    return String(iRounded) + "%";
}

/*
* HTML-таблица для Excel: видимые колонки виджета в том же порядке, итоговая строка жирным.
* @param {Object[]} resultRows   -   Строки в формате RESULT выборки (включая "Общий итог").
* @returns {string}
*/
function BuildExcelHtml(resultRows)
{
    var html, i, row, sTdStyle;
 
    html = new Binary();
    html.AppendStr("<html xmlns:o=\"urn:schemas-microsoft-com:office:office\" xmlns:x=\"urn:schemas-microsoft-com:office:excel\" xmlns=\"http://www.w3.org/TR/REC-html40\"><head><meta charset=\"utf-8\"></head><body>");
    html.AppendStr("<table border=\"1\" cellpadding=\"2\" cellspacing=\"0\">");
 
    html.AppendStr("<tr>");
    html.AppendStr("<td bgcolor=\"#FCE5CD\"><b>Город</b></td>");
    html.AppendStr("<td bgcolor=\"#FCE5CD\"><b>Учебная программа</b></td>");
    html.AppendStr("<td bgcolor=\"#FCE5CD\"><b>Общее кол-во сотрудников</b></td>");
    html.AppendStr("<td bgcolor=\"#FCE5CD\"><b>План</b></td>");
    html.AppendStr("<td bgcolor=\"#FCE5CD\"><b>Факт</b></td>");
    html.AppendStr("<td bgcolor=\"#FCE5CD\"><b>Процент</b></td>");
    html.AppendStr("<td bgcolor=\"#FCE5CD\"><b>Обязательно к прохождению</b></td>");
    html.AppendStr("</tr>");
 
    for (i = 0; i < ArrayCount(resultRows); i++)
    {
        row = resultRows[i];
        sTdStyle = (i == ArrayCount(resultRows) - 1 ? " style=\"font-weight:bold\"" : "");
        html.AppendStr("<tr>");
        html.AppendStr("<td" + sTdStyle + ">" + HtmlEncode(String(row.city)) + "</td>");
        html.AppendStr("<td" + sTdStyle + ">" + HtmlEncode(String(row.program)) + "</td>");
        html.AppendStr("<td" + sTdStyle + " align=\"center\">" + String(row.total) + "</td>");
        html.AppendStr("<td" + sTdStyle + " align=\"center\">" + String(row.plan) + "</td>");
        html.AppendStr("<td" + sTdStyle + " align=\"center\">" + String(row.fact) + "</td>");
        html.AppendStr("<td" + sTdStyle + " align=\"center\">" + HtmlEncode(String(row.percent)) + "</td>");
        html.AppendStr("<td" + sTdStyle + " align=\"center\">" + String(row.mandatory) + "</td>");
        html.AppendStr("</tr>");
    }
 
    html.AppendStr("</table></body></html>");
    return html.GetStr();
}

/*
* Отсортированные ID подчинённых пользователя (вся цепочка вниз); undefined для члена
* УОРиАП (без сужения); пустой массив -- подчинённых нет.
* @param {number} iCurUserId
* @returns {number[]}
*/
function GetSortedSubordinateIds(iCurUserId)
{
    var managerRows, sortedByManagerRows, subordinateIds;

    if (IsUorApMember(iCurUserId))
    {
        LogAlert(1, "GetSortedSubordinateIds(). userId=" + iCurUserId + " -- УОРиАП, без сужения.");
        return undefined;
    }

    managerRows = GetManagerIdRows();
    sortedByManagerRows = SortRowsByManagerId(managerRows);
    subordinateIds = GetManagerHierarchySubordinateIds(sortedByManagerRows, iCurUserId);
    LogAlert(1, "GetSortedSubordinateIds(). userId=" + iCurUserId + "; подчинённых: " + ArrayCount(subordinateIds));
    if (ArrayCount(subordinateIds) == 0)
    {
        return [];
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
* Фильтры страницы из URL.
* @param {string} sFullUrl
* @returns {Object}
*/
function ReadFilters(sFullUrl)
{
    return {
        matrixId: OptInt(GetQueryParam(sFullUrl, "matrix_id"), 0),
        programId: OptInt(GetQueryParam(sFullUrl, "program_id"), 0),
        macroregion: GetQueryParam(sFullUrl, "macroregion"),
        mirCode: GetQueryParam(sFullUrl, "mir_code"),
        positionCommonId: OptInt(GetQueryParam(sFullUrl, "position_common_id"), 0)
    };
}

/*
* Оставляет одну программу, если задан фильтр program_id; нет такой в матрице -- ошибка.
* @param {number[]} programIds
* @param {number} iProgramFilter
* @returns {number[]}
*/
function FilterProgramIds(programIds, iProgramFilter)
{
    var filteredProgramIds, i;

    if (iProgramFilter <= 0)
    {
        return programIds;
    }
    filteredProgramIds = [];
    for (i = 0; i < ArrayCount(programIds); i++)
    {
        if (Int(programIds[i]) == iProgramFilter) { filteredProgramIds.push(programIds[i]); }
    }
    if (ArrayCount(filteredProgramIds) == 0)
    {
        throw ("Программа [" + iProgramFilter + "] не найдена среди программ выбранной матрицы");
    }
    return filteredProgramIds;
}

/*
* Названия программ и "Старт через" ({id, name, delay_days}) из задач матрицы.
*/
function BuildProgramNames(taskRows, matrixId, programIds)
{
    var programNames, j, taskRowForName;

    programNames = [];
    for (j = 0; j < ArrayCount(programIds); j++)
    {
        taskRowForName = ArrayOptFind(taskRows, "Int(This.matrix_id) == Int(matrixId) && Int(This.education_method_id) == Int(programIds[j])");
        programNames.push({
            id: Int(programIds[j]),
            name: (taskRowForName != undefined && taskRowForName.pname != undefined ? String(taskRowForName.pname) : "id=" + programIds[j]),
            delay_days: (taskRowForName != undefined ? OptInt(taskRowForName.delay_days, 0) : 0)
        });
    }
    return programNames;
}

/*
* Отсортированные справочники по сотрудникам, даты прохождения и названия программ.
* @param {number[]} programIds
* @param {Object[]} programNames
* @returns {Object}
*/
function LoadLookups(programIds, programNames)
{
    var lookups;

    lookups = new Object();
    lookups.macroSorted = SortRowsById(GetMacroregionRows());
    lookups.citySorted = SortRowsById(GetCityRows());
    lookups.dateSorted = SortDateRowsByCollaboratorId(GetCompletionDateRows(programIds));
    lookups.mirCodeSorted = SortRowsById(GetMirCodeRows());
    lookups.statusSorted = SortRowsById(GetStatusRows());
    lookups.hireSorted = SortRowsById(GetHireDateRows());
    lookups.programNames = programNames;
    return lookups;
}

/*
* Общее / План / Обязательно по парам (город, программа) -- только сотрудники из
* аудитории матрицы. План -- период прохождения наступил (IsPeriodStarted()).
*/
function AccumulateTotals(acc, collaboratorRows, matrixRow, lookups, programIds)
{
    var i, j, bInAudience, sCity, sMacroregionForRow, iDaysWorked, sProgramName, cityProgramAcc, sDate;

    for (i = 0; i < ArrayCount(collaboratorRows); i++)
    {
        bInAudience = CollaboratorMatchesMatrixAudience(collaboratorRows[i], matrixRow, lookups.mirCodeSorted, lookups.statusSorted);
        if (!bInAudience) { continue; }
        sCity = FindCity(lookups.citySorted, Int(collaboratorRows[i].id));
        sMacroregionForRow = FindMacroregion(lookups.macroSorted, Int(collaboratorRows[i].id));
        iDaysWorked = FindDaysWorked(lookups.hireSorted, Int(collaboratorRows[i].id));
        for (j = 0; j < ArrayCount(programIds); j++)
        {
            sProgramName = FindProgramName(lookups.programNames, programIds[j]);
            cityProgramAcc = GetOrCreateCityProgramAcc(acc, sCity, programIds[j], sProgramName, sMacroregionForRow);
            sDate = FindCompletionDate(lookups.dateSorted, Int(collaboratorRows[i].id), programIds[j]);
            cityProgramAcc.total = cityProgramAcc.total + 1;
            if (IsPeriodStarted(iDaysWorked, FindProgramDelayDays(lookups.programNames, programIds[j]))) { cityProgramAcc.plan = cityProgramAcc.plan + 1; }
            if (sDate == "") { cityProgramAcc.mandatory = cityProgramAcc.mandatory + 1; }
        }
    }
}

/*
* Факт по парам (город, программа) -- прошедшие сотрудники из аудитории матрицы.
*/
function AccumulateFacts(acc, collaboratorRows, matrixRow, lookups, programIds)
{
    var i, j, bInAudience, sCity, sMacroregionForRow, sProgramName, cityProgramAcc, sDate;

    for (i = 0; i < ArrayCount(collaboratorRows); i++)
    {
        bInAudience = CollaboratorMatchesMatrixAudience(collaboratorRows[i], matrixRow, lookups.mirCodeSorted, lookups.statusSorted);
        if (!bInAudience) { continue; }
        sCity = FindCity(lookups.citySorted, Int(collaboratorRows[i].id));
        sMacroregionForRow = FindMacroregion(lookups.macroSorted, Int(collaboratorRows[i].id));
        for (j = 0; j < ArrayCount(programIds); j++)
        {
            sDate = FindCompletionDate(lookups.dateSorted, Int(collaboratorRows[i].id), programIds[j]);
            if (sDate != "")
            {
                sProgramName = FindProgramName(lookups.programNames, programIds[j]);
                cityProgramAcc = GetOrCreateCityProgramAcc(acc, sCity, programIds[j], sProgramName, sMacroregionForRow);
                cityProgramAcc.fact = cityProgramAcc.fact + 1;
            }
        }
    }
}

/*
* Сортировка строк как в выборке: по программе, затем по городу (пузырьком, на месте).
*/
function SortAccByProgramAndCity(acc)
{
    var iOuter, iInner, tmpAcc;

    for (iOuter = 0; iOuter < ArrayCount(acc) - 1; iOuter++)
    {
        for (iInner = 0; iInner < ArrayCount(acc) - 1 - iOuter; iInner++)
        {
            if (acc[iInner].programName > acc[iInner + 1].programName
                || (acc[iInner].programName == acc[iInner + 1].programName && acc[iInner].city > acc[iInner + 1].city))
            {
                tmpAcc = acc[iInner];
                acc[iInner] = acc[iInner + 1];
                acc[iInner + 1] = tmpAcc;
            }
        }
    }
}

/*
* Строки таблицы в формате выборки + итоговая строка "Общий итог".
* @param {Object[]} acc
* @returns {Object[]}
*/
function BuildResultRows(acc)
{
    var resultRows, id, totalAcc, i, row;

    resultRows = [];
    id = 0;
    totalAcc = { total: 0, plan: 0, mandatory: 0, fact: 0 };
    for (i = 0; i < ArrayCount(acc); i++)
    {
        id = id + 1;
        row = acc[i];
        resultRows.push({
            id: id,
            city: row.city,
            program: row.programName,
            total: row.total,
            plan: row.plan,
            fact: row.fact,
            percent: FormatPercent(row.fact, row.plan),
            mandatory: row.mandatory
        });
        totalAcc.total = totalAcc.total + row.total;
        totalAcc.plan = totalAcc.plan + row.plan;
        totalAcc.mandatory = totalAcc.mandatory + row.mandatory;
        totalAcc.fact = totalAcc.fact + row.fact;
    }

    id = id + 1;
    resultRows.push({
        id: id,
        city: "Общий итог",
        program: "-",
        total: totalAcc.total,
        plan: totalAcc.plan,
        fact: totalAcc.fact,
        percent: FormatPercent(totalAcc.fact, totalAcc.plan),
        mandatory: totalAcc.mandatory
    });
    return resultRows;
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
        msg: "Файл готов к выгрузке",
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
    var iCurUserId, sortedSubordinateIds, sFullUrl, filters, taskRows, matrixContext, programIds;
    var lookups, manualFilteredRows, acc, resultRows;

    RESULT = new Object();

    LogAlert(2, "Run(). НАЧАЛО");

    try
    {
        iCurUserId = GetCurUserIdSafe();
        sortedSubordinateIds = GetSortedSubordinateIds(iCurUserId);
        if (sortedSubordinateIds != undefined && ArrayCount(sortedSubordinateIds) == 0)
        {
            // Та же ситуация, в которой выборка отдаёт пустую таблицу.
            RESULT = { command: "alert", view: "warning", msg: "Нет данных для выгрузки -- таблица отчёта пуста." };
            LogAlert(3, "Run(). Не УОРиАП и нет подчинённых (таблица на странице тоже пуста), userId=" + iCurUserId);
        }
        else
        {
            sFullUrl = GetCurPageUrlSafe();
            filters = ReadFilters(sFullUrl);
            LogAlert(1, "Run(). URL=[" + sFullUrl + "]");
            if (filters.matrixId == 0) { throw ("Не выбрана матрица обучения -- настройте фильтры перед выгрузкой."); }

            taskRows = GetEducationMethodTaskRows();
            matrixContext = ResolveMatrixContext(filters.matrixId, GetCompoundProgramRows(), taskRows);
            programIds = FilterProgramIds(matrixContext.programIds, filters.programId);
            lookups = LoadLookups(programIds, BuildProgramNames(taskRows, filters.matrixId, programIds));
            manualFilteredRows = ApplyManualFilters(FilterBySubordinates(GetActiveCollaboratorRows(), sortedSubordinateIds), filters, lookups);

            acc = [];
            AccumulateTotals(acc, manualFilteredRows, matrixContext.matrixRow, lookups, programIds);
            AccumulateFacts(acc, manualFilteredRows, matrixContext.matrixRow, lookups, programIds);
            SortAccByProgramAndCity(acc);
            resultRows = BuildResultRows(acc);

            RESULT = SaveExportAndGetResult(BuildExcelHtml(resultRows), iCurUserId);
            LogAlert(2, "Run(). Выгрузка готова: строк (город x программа) " + (ArrayCount(resultRows) - 1) + " + итоговая; сотрудников в пуле " + ArrayCount(manualFilteredRows));
        }
    }
    catch (_ex)
    {
        RESULT = { command: "alert", view: "warning", msg: ExtractUserError(_ex) };
        LogAlert(4, "Run(). КРИТИЧЕСКАЯ ОШИБКА: " + ExtractUserError(_ex));
    }

    LogAlert(2, "Run(). КОНЕЦ");
}

//-------------------------------------------------------------------------
//              Область основного кода
//-------------------------------------------------------------------------

Run();
