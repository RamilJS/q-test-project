// =====================================================================
// HREDU-182. Модальное окно фильтров для страницы "Процент обученных".
//
// Поля формы: matrix_id (модульная программа, catalog "compound_program") и macroregion
// (select). Скрытые фильтры mir_code_id / position_common_id / program_id не выводятся в
// форме, а сквозно переносятся из текущего URL. По "Применить" -- redirect на ту же
// страницу (cur_page_url -> {{curEnv.curEnvUrl}}) с новыми параметрами в URL.
//
// Видимость пикера матриц:
//   - член группы УОРиАП -- все матрицы;
//   - остальные -- только активные матрицы, у которых f_mir_code подходит под мир-код
//     самого пользователя или его подчинённых по всей цепочке вниз (непосредственный
//     руководитель, is_native=true);
//   - если таких нет -- пустой список и поясняющее сообщение.
// curUserId должен быть привязан в LPE к {{curUser.id}}.
// =====================================================================

//-------------------------------------------------------------------------
//              Область констант
//-------------------------------------------------------------------------

DEBUG = false;                          // Включает подробные логи для дебага
LOG_NAME = "remote_action";             // Имя журнала на сервере, куда будут писаться логи
CUR_OBJECT_ID = 7685325909044809318;    // ID текущего удалённого действия для подстановки в логи

UORIAP_GROUP_ID = "7687978602560891542";
UORIAP_GROUP_XQUERY_TYPE = "groups";

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

function GetParam(sName, sDefault)
{
    var sValue = PARAMETERS.GetOptProperty(sName);
    if (sDefault != undefined && (sValue == undefined || sValue == ""))
    {
        sValue = sDefault;
    }
    return sValue;
}

function GetFormField(sName, sDefault)
{
    var sValue = ArrayOptFind(aFormFields, ("This.name == " + XQueryLiteral(sName)));
    sValue = (sValue != undefined ? sValue.value : sValue);
    if (sDefault != undefined && (sValue == undefined || sValue == ""))
    {
        sValue = sDefault;
    }
    return sValue;
}

function GetFormFieldDefault(sName, sDefault)
{
    var sValue = ArrayOptFind(aFormFieldsDef, ("This.name == " + XQueryLiteral(sName)));
    sValue = (sValue != undefined ? sValue.value : sValue);
    if (sDefault != undefined && (sValue == undefined || sValue == ""))
    {
        sValue = sDefault;
    }
    return sValue;
}

/*
* ID документа cc_mir_code -> текстовый код (то, что лежит в f_mir_codes у сотрудников).
*/
function ResolveMirCodeText(iMirCodeID)
{
    if (OptInt(iMirCodeID, 0) <= 0)
    {
        return "";
    }
    try
    {
        return String(tools.open_doc(Int(iMirCodeID)).TopElem.name);
    }
    catch (_ex)
    {
        LogAlert(3, "ResolveMirCodeText(). Не удалось открыть мир-код id=" + iMirCodeID + ": " + ExtractUserError(_ex));
        return "";
    }
}

/*
* URL текущей страницы: в удалённом действии Request.Url ненадёжен, основной источник --
* параметр cur_page_url.
*/
function GetCurPageUrlSafe()
{
    var sUrl;

    sUrl = GetParam("cur_page_url", "");
    if (sUrl != "")
    {
        return sUrl;
    }

    LogAlert(3, "GetCurPageUrlSafe(). Параметр cur_page_url пуст -- пробуем Request.Url");
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
* "0" в picker-поле платформа воспринимает как ID документа -- превращаем в "".
*/
function SanitizeIdFieldValue(sValue)
{
    if (sValue == "0" || sValue == undefined)
    {
        return "";
    }
    return sValue;
}

/*
* Убирает из URL указанный GET-параметр вместе со значением.
*/
function RemoveQueryParam(sUrl, sParamName)
{
    var sAmpMarker, sQMarkMarker, iMarkerPos, iValueStart, iAmpPos, iUrlLen, sBefore, sAfter;

    iUrlLen = StrLen(sUrl);

    sAmpMarker = "&" + sParamName + "=";
    iMarkerPos = StrOptSubStrPos(sUrl, sAmpMarker, false);
    if (iMarkerPos != undefined)
    {
        iValueStart = iMarkerPos + StrLen(sAmpMarker);
        iAmpPos = StrOptSubStrPos(sUrl, "&", false, iValueStart);
        sBefore = StrRangePos(sUrl, 0, iMarkerPos);
        sAfter = (iAmpPos != undefined ? StrRangePos(sUrl, iAmpPos, iUrlLen) : "");
        return sBefore + sAfter;
    }

    sQMarkMarker = "?" + sParamName + "=";
    iMarkerPos = StrOptSubStrPos(sUrl, sQMarkMarker, false);
    if (iMarkerPos != undefined)
    {
        iValueStart = iMarkerPos + StrLen(sQMarkMarker);
        iAmpPos = StrOptSubStrPos(sUrl, "&", false, iValueStart);
        sBefore = StrRangePos(sUrl, 0, iMarkerPos + 1);
        sAfter = (iAmpPos != undefined ? StrRangePos(sUrl, iAmpPos + 1, iUrlLen) : "");
        return sBefore + sAfter;
    }

    return sUrl;
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
        LogAlert(1, "GetCurUserIdSafe(). curUserId: [" + sRaw + "]");
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
    var groupRows, iGroupId, groupDoc, collabList, foundRow, sRawUserId;

    sRawUserId = String(iCurUserId);
    if (sRawUserId == "" || sRawUserId == "0" || sRawUserId == "undefined" || sRawUserId == "null")
    {
        LogAlert(3, "IsUorApMember(). iCurUserId пустой/нулевой -- считаем, что не УОРиАП.");
        return false;
    }

    try
    {
        groupRows = ArraySelectAll(XQuery(
            "for $elem in " + UORIAP_GROUP_XQUERY_TYPE + " where $elem/id = '" + UORIAP_GROUP_ID + "' return $elem"
        ));
        if (ArrayCount(groupRows) == 0)
        {
            throw ("Группа с id=" + UORIAP_GROUP_ID + " не найдена через тип '" + UORIAP_GROUP_XQUERY_TYPE + "'");
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
    LogAlert(1, "GetManagerIdRows(). Строк: " + ArrayCount(rows));
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
        if (midVal == iTarget)
        {
            startIdx = mid;
            hi = mid - 1;
        }
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

/*
* Мир-коды пользователя и всех его подчинённых, без дублей.
*/
function GetRelevantMirCodes(iCurUserId, subordinateIds)
{
    var mirCodeRows, sortedMirCodeRows, relevantIds, i, row, codes, j, allCodes;

    mirCodeRows = GetMirCodeRows();
    sortedMirCodeRows = SortRowsById(mirCodeRows);

    relevantIds = [Int(iCurUserId)];
    for (i = 0; i < ArrayCount(subordinateIds); i++) { relevantIds.push(Int(subordinateIds[i])); }

    allCodes = [];
    for (i = 0; i < ArrayCount(relevantIds); i++)
    {
        row = BinarySearchById(sortedMirCodeRows, relevantIds[i]);
        if (row == undefined) { continue; }
        codes = ExtractMirCodes(row.mir_codes);
        for (j = 0; j < ArrayCount(codes); j++) { allCodes.push(codes[j]); }
    }

    return ArraySelectDistinct(allCodes, "This");
}

function IsActiveText(sValue)
{
    try { return tools_web.is_true(sValue); }
    catch (_ex) { return (String(sValue) == "true" || String(sValue) == "1"); }
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

function GetCompoundProgramRowsForFilter()
{
    var sqlText;
    sqlText = "";
    sqlText = sqlText + "select cs.id,\r\n";
    sqlText = sqlText + "       c.data.value('(*/name)[1]', 'varchar(max)') as name,\r\n";
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_matrix_active'']/value)[1]', 'varchar(max)') as f_matrix_active,\r\n";
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_mir_code'']/value)[1]', 'varchar(max)') as f_mir_code\r\n";
    sqlText = sqlText + "from compound_programs cs\r\n";
    sqlText = sqlText + "inner join compound_program c on c.id = cs.id";
    return ArraySelectAll(XQuery("sql:" + sqlText));
}

/*
* ID активных модульных программ, у которых f_mir_code подходит хотя бы под один из
* переданных мир-кодов. Программа без f_mir_code не подходит ни под один.
*/
function GetMatrixIdsByMirCodes(aTargetMirCodes)
{
    var allRows, i, j, row, matrixIdSet;

    allRows = GetCompoundProgramRowsForFilter();

    matrixIdSet = [];
    for (i = 0; i < ArrayCount(allRows); i++)
    {
        row = allRows[i];
        if (!IsActiveText(row.f_matrix_active)) { continue; }
        if (row.f_mir_code == undefined || String(row.f_mir_code) == "") { continue; }
        for (j = 0; j < ArrayCount(aTargetMirCodes); j++)
        {
            if (MatchAnySemicolonPattern(row.f_mir_code, aTargetMirCodes[j], true))
            {
                matrixIdSet.push(Int(row.id));
                break;
            }
        }
    }

    matrixIdSet = ArraySelectDistinct(matrixIdSet, "This");
    LogAlert(1, "GetMatrixIdsByMirCodes(). Всего программ: " + ArrayCount(allRows) + "; подходящих активных: " + ArrayCount(matrixIdSet));
    return matrixIdSet;
}

/*
* query_qual для пикера матриц по правилам видимости (см. шапку файла). Если подходящих
* матриц нет -- пишет пояснение в oForm.message.
* @param {Object} oForm
* @returns {string}
*/
function GetMatrixQueryQual(oForm)
{
    var iCurUserId, managerRows, sortedByManagerRows, subordinateIds, relevantMirCodes, matrixIds;

    iCurUserId = GetCurUserIdSafe();
    if (IsUorApMember(iCurUserId))
    {
        LogAlert(1, "GetMatrixQueryQual(). userId=" + iCurUserId + " -- УОРиАП, пикер без ограничений.");
        return "";
    }

    managerRows = GetManagerIdRows();
    sortedByManagerRows = SortRowsByManagerId(managerRows);
    subordinateIds = GetManagerHierarchySubordinateIds(sortedByManagerRows, iCurUserId);
    relevantMirCodes = GetRelevantMirCodes(iCurUserId, subordinateIds);
    matrixIds = GetMatrixIdsByMirCodes(relevantMirCodes);
    LogAlert(1, "GetMatrixQueryQual(). userId=" + iCurUserId + "; подчинённых: " + ArrayCount(subordinateIds)
        + "; мир-коды: [" + ArrayMerge(relevantMirCodes, "This", ", ") + "]; матриц: [" + ArrayMerge(matrixIds, "This", ", ") + "]");

    if (ArrayCount(matrixIds) == 0)
    {
        oForm.message = "У вас и ваших подчинённых нет ни одной подходящей активной учебной программы -- список матриц обучения пуст.";
        // id=0 не существует -- пикер покажет пустой список.
        return "MatchSome($elem/id,(0))";
    }

    // ID матриц -- числами, без кавычек (в кавычках пикер падает).
    return "MatchSome($elem/id,(" + ArrayMerge(matrixIds, "This", ",") + "))";
}

/*
* Значения фильтров из текущего URL страницы.
* @param {string} sPageUrl
* @returns {Object}
*/
function ReadDefaultsFromUrl(sPageUrl)
{
    var sMacroregion;

    sMacroregion = GetQueryParam(sPageUrl, "macroregion");
    // Значение select должно совпадать с одним из entries -- иначе "Все макрорегионы".
    if (sMacroregion != "Сибирь" && sMacroregion != "Юг" && sMacroregion != "Москва" && sMacroregion != "СЗФО")
    {
        sMacroregion = "";
    }

    return {
        matrixId: SanitizeIdFieldValue(GetQueryParam(sPageUrl, "matrix_id")),
        macroregion: sMacroregion,
        mirCodeId: SanitizeIdFieldValue(GetQueryParam(sPageUrl, "mir_code_id")),
        positionCommonId: SanitizeIdFieldValue(GetQueryParam(sPageUrl, "position_common_id")),
        programId: SanitizeIdFieldValue(GetQueryParam(sPageUrl, "program_id"))
    };
}

/*
* Поля формы фильтров.
* @param {Object} oDefaults
* @param {string} sMatrixQueryQual
* @returns {Object[]}
*/
function BuildFormFields(oDefaults, sMatrixQueryQual)
{
    return [
        {
            name: "matrix_id",
            label: "Матрица обучения",
            title: "Выберите матрицу обучения",
            type: "foreign_elem",
            value: oDefaults.matrixId,
            mandatory: true,
            multiple: false,
            catalog: "compound_program",
            query_qual: sMatrixQueryQual
        },
        {
            // Значения должны буквально совпадать с f_2ewj у сотрудников.
            name: "macroregion",
            label: "Макрорегион",
            type: "select",
            value: oDefaults.macroregion,
            entries: [
                { name: "Все макрорегионы", value: "" },
                { name: "Сибирь", value: "Сибирь" },
                { name: "Юг", value: "Юг" },
                { name: "Москва", value: "Москва" },
                { name: "СЗФО", value: "СЗФО" }
            ],
            mandatory: false
        }
    ];
}

/*
* Форма фильтров (первый шаг).
* @param {Object} oDefaults
* @returns {Object}
*/
function BuildFilterForm(oDefaults)
{
    var oForm, oField;

    oForm = new Object();
    oForm.command = "display_form";
    oForm.height = 320;
    oForm.title = "Фильтры отчёта (Процент обученных)";
    oForm.message = null;

    oForm.form_fields = BuildFormFields(oDefaults, GetMatrixQueryQual(oForm));
    for (oField in oForm.form_fields)
    {
        oField.value = GetFormField(oField.name, oField.value);
    }

    oForm.no_buttons = false;
    oForm.buttons = [
        { name: "submit", submit_type: "apply", label: "Применить", type: "submit" },
        { name: "cancel", label: "Отмена", type: "cancel" }
    ];
    return oForm;
}

/*
* "Применить": redirect на ту же страницу с новыми фильтрами. Скрытых полей нет в форме --
* для них фолбэк на значения из текущего URL, иначе они сбрасывались бы.
* @param {string} sPageUrl
* @param {Object} oDefaults
* @returns {Object}
*/
function BuildApplyResult(sPageUrl, oDefaults)
{
    var iMirCodeID, sCleanBaseUrl, oQueryParams, sSeparator, sFullUrl;

    iMirCodeID = OptInt(GetFormField("mir_code_id", oDefaults.mirCodeId), 0);
    oQueryParams = {
        matrix_id: String(OptInt(GetFormField("matrix_id", ""), 0)),
        macroregion: String(GetFormField("macroregion", "")),
        mir_code: ResolveMirCodeText(iMirCodeID),
        mir_code_id: String(iMirCodeID),
        position_common_id: String(OptInt(GetFormField("position_common_id", oDefaults.positionCommonId), 0)),
        program_id: String(OptInt(GetFormField("program_id", oDefaults.programId), 0))
    };

    sCleanBaseUrl = sPageUrl;
    sCleanBaseUrl = RemoveQueryParam(sCleanBaseUrl, "matrix_id");
    sCleanBaseUrl = RemoveQueryParam(sCleanBaseUrl, "macroregion");
    sCleanBaseUrl = RemoveQueryParam(sCleanBaseUrl, "mir_code_id");
    sCleanBaseUrl = RemoveQueryParam(sCleanBaseUrl, "mir_code");
    sCleanBaseUrl = RemoveQueryParam(sCleanBaseUrl, "position_common_id");
    sCleanBaseUrl = RemoveQueryParam(sCleanBaseUrl, "program_id");

    sSeparator = (StrOptSubStrPos(sCleanBaseUrl, "?", false) != undefined ? "&" : "?");
    sFullUrl = sCleanBaseUrl + sSeparator + UrlEncodeQuery(oQueryParams);
    LogAlert(1, "BuildApplyResult(). Redirect: " + sFullUrl);

    return {
        command: "close_form",
        confirm_result: {
            command: "redirect",
            url: sFullUrl
        }
    };
}

/*
* Точка входа.
* @returns {void}
*/
function Run()
{
    var sSubmitType, sPageUrl, oDefaults;

    LogAlert(2, "Run(). НАЧАЛО");

    try
    {
        aFormFields = ParseJson(GetParam("form_fields", "[]"));
        aFormFieldsDef = ParseJson(GetParam("form_fields_default", "[]"));
        sSubmitType = GetFormField("__submit_type__", GetFormFieldDefault("__submit_type__", "step_0"));

        sPageUrl = GetCurPageUrlSafe();
        oDefaults = ReadDefaultsFromUrl(sPageUrl);
        LogAlert(1, "Run(). sSubmitType=[" + sSubmitType + "]; URL=[" + sPageUrl + "]; matrix_id=[" + oDefaults.matrixId
            + "] macroregion=[" + oDefaults.macroregion + "] mir_code_id=[" + oDefaults.mirCodeId
            + "] position_common_id=[" + oDefaults.positionCommonId + "] program_id=[" + oDefaults.programId + "]");

        if (sSubmitType == "apply")
        {
            RESULT = BuildApplyResult(sPageUrl, oDefaults);
        }
        else
        {
            RESULT = BuildFilterForm(oDefaults);
        }
    }
    catch (_exMain)
    {
        LogAlert(4, "Run(). КРИТИЧЕСКАЯ ОШИБКА: " + ExtractUserError(_exMain));
        RESULT = {
            command: "alert",
            msg: ("Ошибка в модалке фильтров (HREDU_182_filtry_percent.js):<br/><pre>" + ExtractUserError(_exMain) + "</pre>"),
            title: "ОШИБКА"
        };
    }

    LogAlert(2, "Run(). КОНЕЦ");
}

//-------------------------------------------------------------------------
//              Область основного кода
//-------------------------------------------------------------------------

Run();
