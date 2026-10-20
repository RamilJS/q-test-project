sLogName = 'HREDU_182_excel_export_percent';
EnableLog(sLogName, true);
function alert(sInputObj) {
    LogEvent(sLogName, sInputObj);
    return sInputObj;
}

// =====================================================================
// HREDU-182. "Процент обученных" -- удалённое действие на кнопку "Выгрузить в Excel".
//
// ЗАДАЧА (20.10.2026, со слов пользователя): пользователь должен получить в Excel РОВНО
// ту таблицу, которую видит в виджете "Табличные данные" на странице "Процент обученных"
// (выборка HREDU-182_procent_obuchennyh.js) -- те же строки (город x учебная программа),
// те же колонки, те же числа, та же итоговая строка "Общий итог", тот же порядок строк.
// Никаких расхождений.
//
// ОТКУДА ЛОГИКА: ВСЕ функции расчёта ниже -- БУКВАЛЬНАЯ КОПИЯ из
// HREDU-182_procent_obuchennyh.js (версия от 19.10.2026, План по ТЗ п.3 -- аудитория +
// наступивший период, IsPeriodStarted()). Блок сборки таблицы в Run() -- тот же код, что
// в Run() выборки, только результат кладётся не в RESULT виджета, а в HTML-таблицу Excel.
// История находок/фиксов по каждой функции -- в выборке, здесь не повторяется. ЕСЛИ
// МЕНЯЕТСЯ ЛОГИКА В ВЫБОРКЕ -- ТАК ЖЕ ПОМЕНЯТЬ ЗДЕСЬ, иначе Excel разойдётся с таблицей.
//
// ВИДИМОСТЬ (по прямому указанию пользователя, 20.10.2026): проверка ДОСТУПА (закрыть
// отчёт) сюда НЕ переносится -- она уже есть на фильтрах и в выборке. Переносится ТОЛЬКО
// сужение пула сотрудников, чтобы данные совпадали с таблицей:
//   - член УОРиАП -- пул не сужается (все сотрудники);
//   - иначе -- пул = вся цепочка подчинённых вниз (тот же BFS, что в выборке).
// Если подчинённых нет -- выборка отдаёт пустую таблицу (RESULT = []); здесь в этом
// случае показываем сообщение "нет данных" вместо пустого файла -- это то же самое
// состояние, а не отдельная проверка доступа.
//
// ВЫГРУЗКА В EXCEL -- тот же приём, что в рабочем удалённом действии HREDU-239
// ("Восток_полный_список"): HTML-таблица с Office-namespace -> tools_web.set_user_data()
// под ключом "excel_html_" + curUserID -> new_window на универсальную страницу-компаньон
// assessment_excel_export.html (она отдаёт кэш на скачивание).
//
// НАСТРОЙКА КНОПКИ В LPE (обязательно, иначе данные не совпадут с таблицей):
//   - cur_page_url -> {{curEnv.curEnvUrl}}  -- фильтры (matrix_id, macroregion, mir_code,
//                                              position_common_id, program_id) берутся из
//                                              URL страницы; Request.Url в удалённом
//                                              действии ненадёжен.
//   - curUserId    -> {{curUser.id}}        -- нужен для сужения пула по подчинённым.
//
// НЕ ПРОВЕРЕНО РЕАЛЬНЫМ ТЕСТОМ (20.10.2026): весь файл -- первый запуск. Каждая функция
// по отдельности уже работает в выборке/HREDU-239, новое здесь только склейка + HTML.
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
//              Сужение пула по подчинённым (копия из выборки, БЕЗ закрытия доступа)
//-------------------------------------------------------------------------

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
        alert("GetCurUserIdSafe(). curUserId недоступна (не привязана в LPE к кнопке?) -- ошибка: " + ExtractUserError(_exDirect));
    }
    return 0;
}

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
        return (foundRow != undefined);
    }
    catch (_exDoc)
    {
        alert("IsUorApMember(). ОШИБКА: " + ExtractUserError(_exDoc) + " -- считаем, что не УОРиАП.");
        return false;
    }
}

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

//-------------------------------------------------------------------------
//              Сортировка / бинарный поиск
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

function FindProgramDelayDays(programNames, iProgramId)
{
    var row;
    row = ArrayOptFind(programNames, "Int(This.id) == Int(iProgramId)");
    return (row != undefined ? OptInt(row.delay_days, 0) : 0);
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
* ТЗ п.3: период прохождения наступил (дней с даты приёма >= delay_days). ИДЕНТИЧНА
* выборке -- нет hire_date (-1) -> считаем наступившим.
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

function CollaboratorHasMirCode(sortedMirCodeRows, collaboratorID, mirCodeFilter)
{
    var row, codes;
    row = BinarySearchById(sortedMirCodeRows, collaboratorID);
    if (row == undefined) { return false; }
    codes = ExtractMirCodes(row.mir_codes);
    return (ArrayOptFind(codes, "String(This) == String(mirCodeFilter)") != undefined);
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

/*
* Ручные фильтры страницы -- копия ApplyManualFilters() из выборки.
*/
function ApplyManualFilters(collaboratorRows, iPositionFilter, sMacroregionFilter, sMirCodeFilter, sortedMacroRows, sortedMirCodeRows)
{
    var allowedPositionIds, filteredRows, i, macroRow;

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
            macroRow = BinarySearchById(sortedMacroRows, Int(collaboratorRows[i].id));
            if (macroRow != undefined && String(macroRow.macroregion) == sMacroregionFilter) { filteredRows.push(collaboratorRows[i]); }
        }
    }

    if (sMirCodeFilter != "")
    {
        collaboratorRows = filteredRows;
        filteredRows = [];
        for (i = 0; i < ArrayCount(collaboratorRows); i++)
        {
            if (CollaboratorHasMirCode(sortedMirCodeRows, Int(collaboratorRows[i].id), sMirCodeFilter)) { filteredRows.push(collaboratorRows[i]); }
        }
    }

    return filteredRows;
}

//-------------------------------------------------------------------------
//              Агрегация (копия из выборки)
//-------------------------------------------------------------------------

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
* факт/план в % -- сначала умножение, потом деление (целочисленное деление движка).
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

//-------------------------------------------------------------------------
//              HTML для Excel (тот же приём, что в HREDU-239)
//-------------------------------------------------------------------------

/*
* Колонки -- те же и в том же порядке, что видимые колонки виджета (COLUMNS в выборке):
* Город, Учебная программа, Общее кол-во сотрудников, План, Факт, Процент, Обязательно к
* прохождению. Скрытые id/link не выгружаются.
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
        // Последняя строка -- "Общий итог", выделяем жирным.
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

//-------------------------------------------------------------------------
//              Точка входа
//-------------------------------------------------------------------------

function Run()
{
    var sFullUrl, matrixId, iProgramFilter, sMacroregionFilter, sMirCodeFilter, iPositionFilter;
    var matrixContext, matrixRow, allProgramRows, taskRows, bInAudience;
    var programIds, filteredProgramIds, programNames, i, j;
    var activeRows, manualFilteredRows;
    var macroRows, mirCodeRows, cityRows, dateRows, statusRows, hireRows;
    var macroSorted, citySorted, dateSorted, mirCodeSorted, statusSorted, hireSorted;
    var iDaysWorked, acc, cityProgramAcc, sCity, sProgramName, sDate, row;
    var totalAcc, resultRows, id, sMacroregionForRow;
    var iCurUserId, managerRows, sortedByManagerRows, subordinateIds, sortedSubordinateIds;
    var subFilteredRows, k, taskRowForName, iOuter, iInner, tmpAcc;
    var sHtml, sBareCurUserId;

    RESULT = new Object();

    try
    {
        alert("Run(). НАЧАЛО (Excel, Процент обученных)");

        // --- Сужение пула (без закрытия доступа) -- см. шапку файла ---
        iCurUserId = GetCurUserIdSafe();
        sortedSubordinateIds = undefined; // undefined = без сужения (УОРиАП)
        if (!IsUorApMember(iCurUserId))
        {
            managerRows = GetManagerIdRows();
            sortedByManagerRows = SortRowsByManagerId(managerRows);
            subordinateIds = GetManagerHierarchySubordinateIds(sortedByManagerRows, iCurUserId);
            if (ArrayCount(subordinateIds) == 0)
            {
                // Та же ситуация, в которой выборка отдаёт пустую таблицу.
                RESULT = { command: "alert", view: "warning", msg: "Нет данных для выгрузки -- таблица отчёта пуста." };
                alert("Run(). КОНЕЦ -- не УОРиАП и нет подчинённых (таблица на странице тоже пуста), userId=" + iCurUserId);
                return;
            }
            sortedSubordinateIds = SortIdArray(subordinateIds);
        }

        // --- Фильтры из URL страницы (те же параметры, что читает выборка) ---
        sFullUrl = GetCurPageUrlSafe();
        alert("Run(). URL страницы: [" + sFullUrl + "]");

        matrixId = OptInt(GetQueryParam(sFullUrl, "matrix_id"), 0);
        iProgramFilter = OptInt(GetQueryParam(sFullUrl, "program_id"), 0);
        sMacroregionFilter = GetQueryParam(sFullUrl, "macroregion");
        sMirCodeFilter = GetQueryParam(sFullUrl, "mir_code");
        iPositionFilter = OptInt(GetQueryParam(sFullUrl, "position_common_id"), 0);

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
                if (Int(programIds[i]) == iProgramFilter) { filteredProgramIds.push(programIds[i]); }
            }
            programIds = filteredProgramIds;
            if (ArrayCount(programIds) == 0)
            {
                throw ("Программа [" + iProgramFilter + "] не найдена среди программ выбранной матрицы");
            }
        }

        activeRows = GetActiveCollaboratorRows();

        if (sortedSubordinateIds != undefined)
        {
            subFilteredRows = [];
            for (k = 0; k < ArrayCount(activeRows); k++)
            {
                if (IdArrayContainsSorted(sortedSubordinateIds, Int(activeRows[k].id))) { subFilteredRows.push(activeRows[k]); }
            }
            activeRows = subFilteredRows;
        }

        macroRows = GetMacroregionRows();
        macroSorted = SortRowsById(macroRows);
        cityRows = GetCityRows();
        citySorted = SortRowsById(cityRows);
        dateRows = GetCompletionDateRows(programIds);
        dateSorted = SortDateRowsByCollaboratorId(dateRows);
        mirCodeRows = GetMirCodeRows();
        mirCodeSorted = SortRowsById(mirCodeRows);
        statusRows = GetStatusRows();
        statusSorted = SortRowsById(statusRows);
        hireRows = GetHireDateRows();
        hireSorted = SortRowsById(hireRows);

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

        manualFilteredRows = ApplyManualFilters(activeRows, iPositionFilter, sMacroregionFilter, sMirCodeFilter, macroSorted, mirCodeSorted);

        acc = [];

        // total/plan/mandatory -- как в выборке.
        for (i = 0; i < ArrayCount(manualFilteredRows); i++)
        {
            bInAudience = CollaboratorMatchesMatrixAudience(manualFilteredRows[i], matrixRow, mirCodeSorted, statusSorted);
            if (!bInAudience) { continue; }
            sCity = FindCity(citySorted, Int(manualFilteredRows[i].id));
            sMacroregionForRow = FindMacroregion(macroSorted, Int(manualFilteredRows[i].id));
            iDaysWorked = FindDaysWorked(hireSorted, Int(manualFilteredRows[i].id));
            for (j = 0; j < ArrayCount(programIds); j++)
            {
                sProgramName = FindProgramName(programNames, programIds[j]);
                cityProgramAcc = GetOrCreateCityProgramAcc(acc, sCity, programIds[j], sProgramName, sMacroregionForRow);
                sDate = FindCompletionDate(dateSorted, Int(manualFilteredRows[i].id), programIds[j]);
                cityProgramAcc.total = cityProgramAcc.total + 1;
                if (IsPeriodStarted(iDaysWorked, FindProgramDelayDays(programNames, programIds[j]))) { cityProgramAcc.plan = cityProgramAcc.plan + 1; }
                if (sDate == "") { cityProgramAcc.mandatory = cityProgramAcc.mandatory + 1; }
            }
        }

        // fact -- как в выборке.
        for (i = 0; i < ArrayCount(manualFilteredRows); i++)
        {
            bInAudience = CollaboratorMatchesMatrixAudience(manualFilteredRows[i], matrixRow, mirCodeSorted, statusSorted);
            if (!bInAudience) { continue; }
            sCity = FindCity(citySorted, Int(manualFilteredRows[i].id));
            sMacroregionForRow = FindMacroregion(macroSorted, Int(manualFilteredRows[i].id));
            for (j = 0; j < ArrayCount(programIds); j++)
            {
                sDate = FindCompletionDate(dateSorted, Int(manualFilteredRows[i].id), programIds[j]);
                if (sDate != "")
                {
                    sProgramName = FindProgramName(programNames, programIds[j]);
                    cityProgramAcc = GetOrCreateCityProgramAcc(acc, sCity, programIds[j], sProgramName, sMacroregionForRow);
                    cityProgramAcc.fact = cityProgramAcc.fact + 1;
                }
            }
        }

        // Сортировка -- как в выборке: программа, затем город (пузырьком).
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

        sHtml = BuildExcelHtml(resultRows);

        // Ключ кэша -- "голая" curUserID (с большой D), как в HREDU-239 и примерах --
        // именно его читает assessment_excel_export.html. Фолбэк на curUserId из LPE.
        try { sBareCurUserId = String(curUserID); }
        catch (_exBareUserId) { sBareCurUserId = ""; }
        if (sBareCurUserId == "" || sBareCurUserId == "undefined" || sBareCurUserId == "null")
        {
            sBareCurUserId = String(iCurUserId);
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

        alert("Run(). КОНЕЦ (успех), строк (город x программа): " + (ArrayCount(resultRows) - 1) + " + итоговая");
    }
    catch (_ex)
    {
        RESULT = { command: "alert", view: "warning", msg: ExtractUserError(_ex) };
        alert("Run(). ОШИБКА: " + ExtractUserError(_ex));
    }
}

Run();
