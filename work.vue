// =====================================================================
// HREDU-183. Отчёты ТЭП (Общее кол-во / План / Факт / Обязательно к прохождению) --
// удалённое действие на кнопку "Выгрузить в Excel".
//
// Выгружает ту же таблицу, что показывает выборка HREDU-183_tep_reports.js, для текущего
// режима отчёта и с теми же фильтрами. Расчёт -- копия выборки: при изменении логики в
// выборке нужно так же поменять её здесь, иначе Excel разойдётся с таблицей.
//
// Режим отчёта: result_type из URL страницы -> параметр кнопки result_type -> "total".
// Фильтры берутся из URL страницы (cur_page_url -> {{curEnv.curEnvUrl}} в LPE).
// Видимость: как в выборке ТЭП -- без ограничения по подчинённым.
//
// Колонки -- как в конфигурации виджета: статус (заливка цветом), ФИО, Должность,
// Подразделение, Дата прохождения. Готовый HTML кладётся в кэш
// tools_web.set_user_data("excel_html_" + curUserID), его отдаёт на скачивание страница
// assessment_excel_export.html.
// =====================================================================

//-------------------------------------------------------------------------
//              Область констант
//-------------------------------------------------------------------------

DEBUG = false;                          // Включает подробные логи для дебага
LOG_NAME = "remote_action";             // Имя журнала на сервере, куда будут писаться логи
CUR_OBJECT_ID = 7698682660711222927;    // ID текущего удалённого действия для подстановки в логи

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
* Необязательный параметр кнопки "result_type" -- запасной источник режима отчёта.
*/
function GetResultTypeParamSafe()
{
    var sValue;
    try { sValue = String(PARAMETERS.GetOptProperty("result_type", "")); }
    catch (_exParam) { sValue = ""; }
    if (sValue == "undefined" || sValue == "null") { sValue = ""; }
    return sValue;
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

/*
* "Старт через" (delay_days) и "Продолжительность" (days) программы.
*/
function FindProgramTiming(programNames, iProgramId)
{
    var row;
    row = ArrayOptFind(programNames, "Int(This.id) == Int(iProgramId)");
    return (row != undefined ? { delayDays: OptInt(row.delay_days, 0), days: OptInt(row.days, 0) } : { delayDays: 0, days: 0 });
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
* Цвет статуса по п.6 ТЗ: пройдено -- green; срок истёк (дней с приёма >= старт +
* продолжительность) или нет даты приёма -- red; иначе -- white.
*/
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

/*
* Строки отчёта для одного сотрудника -- по одной на каждую программу.
* @param {Object} collaborator
* @param {Object} lookups        -   Отсортированные справочники + programNames (см. LoadLookups()).
* @param {number[]} programIds
* @param {boolean} bInAudience
* @returns {Object[]}
*/
function BuildReportRows(collaborator, lookups, programIds, bInAudience)
{
    var rows, row, i, programID, bPassed, iDaysWorked, programTiming;
    rows = [];
    iDaysWorked = FindDaysWorked(lookups.hireSorted, Int(collaborator.id));
    for (i = 0; i < ArrayCount(programIds); i++)
    {
        programID = programIds[i];
        row = new Object();
        row.fullname = String(collaborator.fullname);
        row.position_name = String(collaborator.position_name);
        row.subdivision_name = String(collaborator.position_parent_name);
        row.macroregion = FindMacroregion(lookups.macroSorted, Int(collaborator.id));
        row.city = FindCity(lookups.citySorted, Int(collaborator.id));
        row.program_name = FindProgramName(lookups.programNames, programID);
        row.completion_date = FindCompletionDate(lookups.dateSorted, Int(collaborator.id), programID);
        row.in_audience = bInAudience;

        bPassed = (row.completion_date != "");
        row.is_passed = bPassed;
        programTiming = FindProgramTiming(lookups.programNames, programID);
        row.is_period_started = IsPeriodStarted(iDaysWorked, programTiming.delayDays);
        row.status_color = BuildStatusColor(bPassed, iDaysWorked, programTiming.delayDays, programTiming.days);

        rows.push(row);
    }
    return rows;
}

/*
* Режим отчёта: URL -> параметр кнопки -> "total". Неизвестный режим -- ошибка.
* @param {string} sFullUrl
* @returns {string}
*/
function GetResultType(sFullUrl)
{
    var sResultType;

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
    return sResultType;
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
        positionCommonId: OptInt(GetQueryParam(sFullUrl, "position_common_id"), 0),
        city: GetQueryParam(sFullUrl, "city")
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
        if (Int(programIds[i]) == iProgramFilter)
        {
            filteredProgramIds.push(programIds[i]);
        }
    }
    if (ArrayCount(filteredProgramIds) == 0)
    {
        throw ("Программа [" + iProgramFilter + "] не найдена среди программ выбранной матрицы");
    }
    return filteredProgramIds;
}

/*
* Названия и тайминг программ ({id, name, delay_days, days}) из задач матрицы.
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
            delay_days: (taskRowForName != undefined ? OptInt(taskRowForName.delay_days, 0) : 0),
            days: (taskRowForName != undefined ? OptInt(taskRowForName.days, 0) : 0)
        });
    }
    return programNames;
}

/*
* Отсортированные справочники по сотрудникам и даты прохождения программ.
* @param {number[]} programIds
* @param {Object[]} programNames
* @returns {Object}
*/
function LoadLookups(programIds, programNames)
{
    var lookups;

    lookups = new Object();
    lookups.mirCodeSorted = SortRowsById(GetMirCodeRows());
    lookups.statusSorted = SortRowsById(GetStatusRows());
    lookups.hireSorted = SortRowsById(GetHireDateRows());
    lookups.macroSorted = SortRowsById(GetMacroregionRows());
    lookups.citySorted = SortRowsById(GetCityRows());
    lookups.dateSorted = SortDateRowsByCollaboratorId(GetCompletionDateRows(programIds));
    lookups.programNames = programNames;
    return lookups;
}

/*
* Ручной фильтр по типовой должности.
*/
function FilterByPosition(collaboratorRows, iPositionFilter)
{
    var allowedPositionIds, filteredCollaboratorRows, i;

    if (iPositionFilter <= 0)
    {
        return collaboratorRows;
    }
    allowedPositionIds = SortIdArray(GetPositionIdsByCommonPosition(iPositionFilter));
    filteredCollaboratorRows = [];
    for (i = 0; i < ArrayCount(collaboratorRows); i++)
    {
        if (IdArrayContainsSorted(allowedPositionIds, OptInt(collaboratorRows[i].position_id, 0)))
        {
            filteredCollaboratorRows.push(collaboratorRows[i]);
        }
    }
    return filteredCollaboratorRows;
}

/*
* Ручные фильтры по макрорегиону, мир-коду и городу -- в том же порядке, что в выборке.
* @param {Object[]} collaboratorRows
* @param {Object} filters
* @param {Object} lookups
* @returns {Object[]}
*/
function FilterByAttributes(collaboratorRows, filters, lookups)
{
    var filteredCollaboratorRows, i, row;

    filteredCollaboratorRows = [];
    for (i = 0; i < ArrayCount(collaboratorRows); i++)
    {
        row = collaboratorRows[i];
        if (filters.macroregion != "" && FindMacroregion(lookups.macroSorted, Int(row.id)) != filters.macroregion) { continue; }
        if (filters.mirCode != "" && !CollaboratorHasMirCode(lookups.mirCodeSorted, Int(row.id), filters.mirCode)) { continue; }
        if (filters.city != "" && FindCity(lookups.citySorted, Int(row.id)) != filters.city) { continue; }
        filteredCollaboratorRows.push(row);
    }
    return filteredCollaboratorRows;
}

/*
* Строки (сотрудник x программа) для сотрудников из аудитории матрицы.
*/
function BuildAllReportRows(collaboratorRows, matrixRow, lookups, programIds)
{
    var reportRows, collaboratorReportRows, i, j, bInAudience;

    reportRows = [];
    for (i = 0; i < ArrayCount(collaboratorRows); i++)
    {
        bInAudience = CollaboratorMatchesMatrixAudience(collaboratorRows[i], matrixRow, lookups.mirCodeSorted, lookups.statusSorted);
        if (!bInAudience)
        {
            continue;
        }
        collaboratorReportRows = BuildReportRows(collaboratorRows[i], lookups, programIds, bInAudience);
        for (j = 0; j < ArrayCount(collaboratorReportRows); j++)
        {
            reportRows.push(collaboratorReportRows[j]);
        }
    }
    return reportRows;
}

/*
* Отбор строк по режиму: plan -- период наступил, fact -- пройдено, mandatory -- не
* пройдено, total -- все.
* @param {Object[]} reportRows
* @param {string} sResultType
* @returns {Object[]}
*/
function FilterRowsByResultType(reportRows, sResultType)
{
    var filteredResultRows, i, bKeep;

    if (sResultType != "plan" && sResultType != "fact" && sResultType != "mandatory")
    {
        return reportRows;
    }
    filteredResultRows = [];
    for (i = 0; i < ArrayCount(reportRows); i++)
    {
        if (sResultType == "plan") { bKeep = reportRows[i].is_period_started; }
        else if (sResultType == "fact") { bKeep = (reportRows[i].completion_date != ""); }
        else { bKeep = (reportRows[i].completion_date == ""); }
        if (bKeep)
        {
            filteredResultRows.push(reportRows[i]);
        }
    }
    return filteredResultRows;
}

/*
* Цвет заливки ячейки статуса -- то же значение status_color, что получает виджет.
*/
function StatusColorToHex(sStatusColor)
{
    if (sStatusColor == "green") { return "#00B050"; }
    if (sStatusColor == "red") { return "#FF0000"; }
    return "#FFFFFF";
}

/*
* Название режима для заголовка файла -- те же подписи, что в модалке "Выбор отчета".
*/
function ResultTypeTitle(sResultType)
{
    if (sResultType == "plan") { return "План"; }
    if (sResultType == "fact") { return "Факт"; }
    if (sResultType == "mandatory") { return "Обязательно к прохождению"; }
    return "Общее кол-во";
}

/*
* HTML-таблица для Excel: строка с названием режима, заголовки, строки отчёта.
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

/*
* Кладёт HTML в кэш и возвращает команду открыть страницу скачивания. Ключ кэша --
* "голая" curUserID (её читает assessment_excel_export.html), запасной -- curUserId из LPE.
* @param {string} sHtml
* @returns {Object}
*/
function SaveExportAndGetResult(sHtml)
{
    var sBareCurUserId;

    try { sBareCurUserId = String(curUserID); }
    catch (_exBareUserId) { sBareCurUserId = ""; }
    if (sBareCurUserId == "" || sBareCurUserId == "undefined" || sBareCurUserId == "null")
    {
        LogAlert(3, "SaveExportAndGetResult(). curUserID недоступна -- пробуем curUserId из LPE.");
        try { sBareCurUserId = String(curUserId); }
        catch (_exLpeUserId) { sBareCurUserId = ""; }
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
    var sFullUrl, sResultType, filters, taskRows, matrixContext, programIds, lookups;
    var collaboratorRows, reportRows;

    RESULT = new Object();

    LogAlert(2, "Run(). НАЧАЛО");

    try
    {
        sFullUrl = GetCurPageUrlSafe();
        sResultType = GetResultType(sFullUrl);
        filters = ReadFilters(sFullUrl);
        LogAlert(1, "Run(). URL=[" + sFullUrl + "]; result_type=[" + sResultType + "]");

        if (filters.matrixId == 0) { throw ("Не выбрана матрица обучения -- настройте фильтры перед выгрузкой."); }

        taskRows = GetEducationMethodTaskRows();
        matrixContext = ResolveMatrixContext(filters.matrixId, GetCompoundProgramRows(), taskRows);
        programIds = FilterProgramIds(matrixContext.programIds, filters.programId);
        lookups = LoadLookups(programIds, BuildProgramNames(taskRows, filters.matrixId, programIds));

        collaboratorRows = FilterByPosition(GetActiveCollaboratorRows(), filters.positionCommonId);
        collaboratorRows = FilterByAttributes(collaboratorRows, filters, lookups);
        reportRows = BuildAllReportRows(collaboratorRows, matrixContext.matrixRow, lookups, programIds);
        reportRows = FilterRowsByResultType(reportRows, sResultType);
        LogAlert(1, "Run(). Сотрудников после фильтров: " + ArrayCount(collaboratorRows) + "; строк отчёта: " + ArrayCount(reportRows));

        if (ArrayCount(reportRows) == 0)
        {
            RESULT = { command: "alert", view: "warning", msg: "Нет данных для выгрузки -- таблица отчёта пуста." };
            LogAlert(3, "Run(). Нет данных для выгрузки, result_type=" + sResultType);
        }
        else
        {
            RESULT = SaveExportAndGetResult(BuildExcelHtml(reportRows, sResultType));
            LogAlert(2, "Run(). Выгрузка готова: result_type=" + sResultType + ", строк " + ArrayCount(reportRows));
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
