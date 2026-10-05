
sLogName = 'HREDU_182_7685313676595870594';
EnableLog(sLogName, true);
function alert(sInputObj) {
    LogEvent(sLogName, sInputObj);
    return sInputObj;
};

// =====================================================================
// HREDU-182. "Процент обученных" -- выборка для Табличных данных.
//
// Источник данных -- модульные программы WebSoft (compound_program): matrix_id из URL --
// id документа compound_program. Внутри -- задачи (programs/program), берём только
// type == "education_method". Аудитория (должность/мир-код/оргструктура/подразделение +
// статус-исключения, "*"-wildcard, ";"-списки) -- custom_elems самого compound_program,
// одна на всю программу (см. CollaboratorMatchesMatrixAudience()/WildcardMatch()).
//
// Строка отчёта = пара (город, программа). Показатели:
//   Общее (total)       -- аудитория программы + ручные фильтры, по городу.
//   План (plan)          -- = Общее (период прохождения не реализован).
//   Факт (fact)           -- прошедшие программу И входящие в её аудиторию, по городу.
//   Обязательно (mandatory) -- аудитория МИНУС прошедшие.
//   Процент (percent)     -- факт/план, округление до целого, "-" если план = 0.
// Строка попадает в отчёт, если total > 0 или fact > 0. Сотрудники без города -- группа
// "(без города)". Сортировка -- по названию программы, затем по городу внутри неё;
// "Общий итог" всегда последней строкой. Фильтры (matrix_id/macroregion/mir_code/
// position_common_id/program_id) читаются из URL.
//
// Видимость: сотрудник УОРиАП -- видит весь отчёт без ограничений; руководитель (есть
// хотя бы 1 подчинённый по всей цепочке вниз, по func_manager/is_native=true) -- видит
// отчёт, ограниченный своими подчинёнными; остальные -- RESULT = [].
//
// RESULT -- массив строк напрямую, без обёртки.
// =====================================================================

DEBUG = false;
LOG_NAME = "agent";
CUR_OBJECT_ID = 7685313676595870594;

TEP_REPORT_PAGE_URL = "/view_doc.html?mode=matrix_report";

// Группа УОРиАП -- поиск по id (не по code). Значения продублированы локальными var
// внутри IsUorApMember() (на этой платформе обращение к этим глобалам изнутри функции
// в этом документе давало ошибку "not defined") -- оставлены здесь для документации.
UORIAP_GROUP_ID = "7687978602560891542";
UORIAP_GROUP_XQUERY_TYPE = "groups";

//-------------------------------------------------------------------------
//              Область функций
//-------------------------------------------------------------------------

/*
* Достаёт ID текущего пользователя портала из LPE-параметра curUserId ({{curUser.id}}
* привязан в редакторе LPE к этому виджету). Fail-safe: 0, если параметр не привязан.
* @returns {number}
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
    }
    return 0;
}

/*
* Проверяет, входит ли сотрудник с id=iCurUserId в группу УОРиАП (группа ищется по id).
* Fail-safe: любая ошибка/невалидный id -> false (доступа нет).
* @param {string} iCurUserId
* @returns {boolean}
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
        return (foundRow != undefined);
    }
    catch (_exDoc)
    {
        return false;
    }
}

/*
* Возвращает {id, manager_id} для ВСЕХ активных сотрудников одним SQL -- manager_id -- id
* непосредственного руководителя (func_manager с is_native=true, типизированное булево
* поле -- сравнение через true()/false(), не строкой/числом).
* @returns {Object[]}
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

/*
* Сортирует строки GetManagerIdRows() по manager_id -- для FindManagerRangeStart()/
* CollectDirectSubordinateIds() (бинарный поиск диапазона). Строки без manager_id
* получают ключ сортировки 0 (реальный id руководителя всегда > 0).
* @param {Object[]} managerRows
* @returns {Object[]}
*/
function SortRowsByManagerId(managerRows)
{
    return ArraySort(managerRows, "OptInt(This.manager_id, 0)", "+");
}

/*
* Бинарный поиск ПЕРВОЙ строки с данным manager_id в массиве, отсортированном через
* SortRowsByManagerId().
* @param {Object[]} sortedByManagerRows
* @param {number} iManagerId
* @returns {number}   -   Индекс первой строки с этим manager_id, или -1 если ни одной.
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
* Возвращает id ВСЕХ прямых подчинённых (один уровень вниз) руководителя iManagerId.
* Самая вершина иерархии указывает сама на себя (person_id == свой id) -- строки, где id
* сотрудника совпадает с iManagerId, исключаются из результата.
* @param {Object[]} sortedByManagerRows
* @param {number} iManagerId
* @returns {number[]}
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
* Находит id всей цепочки подчинённых вниз (BFS, без рекурсии) для сотрудника
* iCurUserId. Защита от циклов -- visited-список, пересортировываемый один раз на
* уровень BFS (не на каждого сотрудника).
* @param {Object[]} sortedByManagerRows
* @param {number} iCurUserId
* @returns {number[]}   -   id всех подчинённых по всей цепочке вниз (без самого iCurUserId).
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

/*
* Достаёт полный URL текущей страницы. В контексте выборки Request.Url надёжен.
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
* Вырезает значение GET-параметра из полного URL строки (без regex/.indexOf -- не
* поддерживаются этим движком -- через StrOptSubStrPos/StrRangePos/StrLen).
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
* Безопасно интерпретирует текстовое "булево" значение custom_elem (f_matrix_active и
* т.п.) -- те же значения, что пишет/читает редактор матриц через tools_web.is_true().
* @param {string} sValue
* @returns {boolean}
*/
function IsActiveText(sValue)
{
    try { return tools_web.is_true(sValue); }
    catch (_ex) { return (String(sValue) == "true" || String(sValue) == "1"); }
}

/*
* Массовое чтение модульных программ (compound_program) -- id, название и ВСЕ custom_elems
* аудитории -- одним SQL по всем документам сразу.
* @returns {Object[]}   -   {id, name, f_matrix_active, f_position_names, f_position_names_exclude,
*                            f_mir_code, f_mir_code_exclude, f_org_names, f_org_names_exclude,
*                            f_subdivision_names, f_subdivision_names_exclude, f_subdivision_child,
*                            f_collaborator_statuses_exclude}.
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
* Массовое чтение задач типа "Учебная программа" (education_method) из ВСЕХ модульных
* программ одним SQL через XML .nodes(). Эл. курсы (type=course) отфильтрованы в SQL.
* @returns {Object[]}   -   {matrix_id, object_id, education_method_id, ptype, delay_days, pname}.
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

/*
* Собирает programIds из taskRows для ОДНОЙ конкретной матрицы (matrixId).
* @param {Object[]} taskRows
* @param {number} matrixId
* @returns {number[]}
*/
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
* Ищет матрицу по id бинарным поиском в bulk-SQL результате (GetCompoundProgramRows()).
* Различает "нет вообще" (matrixRow == undefined) от "деактивирована" (f_matrix_active).
* @param {number} matrixId
* @param {Object[]} allProgramRows   -   Результат GetCompoundProgramRows().
* @param {Object[]} taskRows         -   Результат GetEducationMethodTaskRows() (по ВСЕМ матрицам).
* @returns {Object}   -   { matrixRow: Object, programIds: number[] }.
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

/*
* Читает всех действующих сотрудников (базовый пул, до аудитории и до ручных фильтров).
* @returns {Object[]}
*/
function GetActiveCollaboratorRows()
{
    return ArraySelectAll(XQuery("for $elem in collaborators where $elem/is_dismiss=false() return $elem"));
}

/*
* Находит id документов коллекции "positions", у которых position_common_id совпадает
* с переданным ID.
* @param {number} iCommonPositionFilter
* @returns {number[]}
*/
function GetPositionIdsByCommonPosition(iCommonPositionFilter)
{
    var positionRows, positionIds, i;
    positionRows = ArraySelectAll(XQuery("for $elem in positions where $elem/position_common_id = " + iCommonPositionFilter + " return $elem"));
    positionIds = [];
    for (i = 0; i < ArrayCount(positionRows); i++) { positionIds.push(Int(positionRows[i].id)); }
    return positionIds;
}

/*
* Проверяет вхождение числа в массив чисел линейным циклом.
* @param {number[]} idArray
* @param {number} value
* @returns {boolean}
*/
function IdArrayContains(idArray, value)
{
    var i;
    for (i = 0; i < ArrayCount(idArray); i++)
    {
        if (Int(idArray[i]) == Int(value)) { return true; }
    }
    return false;
}

/*
* Достаёт макрорегион (custom_elem f_2ewj) по всем действующим сотрудникам одним SQL.
* @returns {Object[]}
*/
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
* Город -- custom_elem "sity" (так называется реальное поле в системе, это не опечатка
* в этом файле). Нужен для drill-down по городу из "Процент обученных".
* @returns {Object[]}   -   Массив {id, sity}.
*/
function GetCityRows()
{
    var sqlText, rows;
    sqlText = "";
    sqlText = sqlText + "select cs.id,\r\n";
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''sity'']/value)[1]', 'varchar(max)') as sity\r\n";
    sqlText = sqlText + "from collaborators cs\r\n";
    sqlText = sqlText + "inner join collaborator c on c.id = cs.id\r\n";
    sqlText = sqlText + "where cs.is_dismiss != 1";
    rows = ArraySelectAll(XQuery("sql:" + sqlText));
    return rows;
}

/*
* Сортирует строки с полем "id" по возрастанию -- для бинарного поиска (BinarySearchById()).
* @param {Object[]} rows   -   Строки с полем "id" (macroRows/cityRows/mirCodeRows/statusRows).
* @returns {Object[]}
*/
function SortRowsById(rows)
{
    return ArraySort(rows, "Int(This.id)", "+");
}

/*
* Бинарный поиск строки с полем "id" == targetId в массиве, отсортированном через
* SortRowsById().
* @param {Object[]} sortedRows
* @param {number} targetId
* @returns {Object}   -   Найденная строка или undefined.
*/
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

/*
* Сортирует массив чисел по возрастанию -- для бинарного поиска (IdArrayContainsSorted()).
* @param {number[]} idArray
* @returns {number[]}
*/
function SortIdArray(idArray)
{
    return ArraySort(idArray, "Int(This)", "+");
}

/*
* Бинарный поиск значения в массиве чисел, отсортированном через SortIdArray().
* @param {number[]} sortedIdArray
* @param {number} value
* @returns {boolean}
*/
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

/*
* Находит город конкретного сотрудника; "(без города)" если поле пустое/не найдено.
* @param {Object[]} sortedCityRows
* @param {number} collaboratorID
* @returns {string}
*/
function FindCity(sortedCityRows, collaboratorID)
{
    var cityRow, sCity;
    cityRow = BinarySearchById(sortedCityRows, collaboratorID);
    sCity = (cityRow != undefined && cityRow.sity != undefined ? String(cityRow.sity) : "");
    return (sCity != "" ? sCity : "(без города)");
}

/*
* Ищет макрорегион конкретного сотрудника.
* @param {Object[]} sortedMacroRows
* @param {number} collaboratorID
* @returns {string}
*/
function FindMacroregion(sortedMacroRows, collaboratorID)
{
    var macroRow;
    macroRow = BinarySearchById(sortedMacroRows, collaboratorID);
    return (macroRow != undefined && macroRow.macroregion != undefined ? String(macroRow.macroregion) : "");
}

/*
* Сортирует dateRows по collaborator_id (ключевое поле здесь -- collaborator_id, не id).
* @param {Object[]} dateRows
* @returns {Object[]}
*/
function SortDateRowsByCollaboratorId(dateRows)
{
    return ArraySort(dateRows, "Int(This.collaborator_id)", "+");
}

/*
* Достаёт сырое значение мир-кодов (custom_elem f_mir_codes) по всем действующим
* сотрудникам одним SQL-запросом.
* @returns {Object[]}
*/
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
* Разбивает sText по строке-разделителю sDelim (без regex/.split() -- не поддерживаются
* этим движком -- через StrOptSubStrPos/StrRangePos).
* @param {string} sText
* @param {string} sDelim
* @returns {string[]}
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

/*
* Проверяет, есть ли у сотрудника указанный мир-код среди любых его мир-кодов.
* @param {Object[]} sortedMirCodeRows
* @param {number} collaboratorID
* @param {string} mirCodeFilter
* @returns {boolean}
*/
function CollaboratorHasMirCode(sortedMirCodeRows, collaboratorID, mirCodeFilter)
{
    var row, codes;
    row = BinarySearchById(sortedMirCodeRows, collaboratorID);
    if (row == undefined) { return false; }
    codes = ExtractMirCodes(row.mir_codes);
    return (ArrayOptFind(codes, "String(This) == String(mirCodeFilter)") != undefined);
}

/*
* Статус сотрудника -- custom_elem "CurrentState" на карточке collaborator. Нужен для
* f_collaborator_statuses_exclude у модульной программы.
* @returns {Object[]}   -   Массив {id, status}.
*/
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
* Разбивает паттерн вида "* текст *" по звёздочкам (без regex).
*/
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
* Матчит ОДИН паттерн вида "* текст *" против ОДНОГО текста.
* @param {string} sPattern
* @param {string} sText
* @param {boolean} bIgnoreCase   -   true -- игнорировать регистр (3-й параметр
*                                    StrOptSubStrPos работает наоборот от названия).
* @returns {boolean}
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
* Проверяет текст против списка паттернов, разделённых ";". true, если текст подходит
* хотя бы под один паттерн из списка.
* @param {string} sPatternsList
* @param {string} sText
* @param {boolean} bIgnoreCase
* @returns {boolean}
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
* Положительная ось аудитории (f_position_names/f_org_names/f_subdivision_names).
* Пустой список паттернов = ось не задана = без ограничения по этой оси.
* @param {string} sPatternsList
* @param {string} sSingleValue
* @returns {boolean}
*/
function AxisMatches(sPatternsList, sSingleValue)
{
    if (sPatternsList == undefined || String(sPatternsList) == "") { return true; }
    return MatchAnySemicolonPattern(sPatternsList, sSingleValue, true);
}

/*
* Ось-исключение (f_position_names_exclude и т.п.) -- пустой список = никого не исключаем.
* @param {string} sPatternsList
* @param {string} sSingleValue
* @returns {boolean}
*/
function AxisExcludeMatches(sPatternsList, sSingleValue)
{
    if (sPatternsList == undefined || String(sPatternsList) == "") { return false; }
    return MatchAnySemicolonPattern(sPatternsList, sSingleValue, true);
}

/*
* Ось мир-кода -- у сотрудника может быть несколько кодов -- совпадение, если хотя бы
* один код сотрудника подходит хотя бы под один паттерн программы.
* @param {string} sPatternsList
* @param {string[]} employeeCodes
* @returns {boolean}
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
* Проверяет, входит ли сотрудник в аудиторию модульной программы (одна аудитория на всю
* матрицу, проверяется один раз на пару сотрудник x матрица). "И" между заполненными
* осями, "ИЛИ" между паттернами внутри одной оси.
* @param {Object} collaboratorRow      -   Строка из GetActiveCollaboratorRows().
* @param {Object} matrixRow            -   Строка из GetCompoundProgramRows() -- аудитория ЭТОЙ матрицы.
* @param {Object[]} sortedMirCodeRows  -   GetMirCodeRows() + SortRowsById().
* @param {Object[]} sortedStatusRows   -   GetStatusRows() + SortRowsById().
* @returns {boolean}
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
* Применяет 4 ручных фильтра пользователя: типовая должность, макрорегион, мир-код.
* Все три поиска по вспомогательным массивам -- через заранее отсортированные массивы и
* бинарный поиск.
* @param {Object[]} collaboratorRows
* @param {number} iPositionFilter
* @param {string} sMacroregionFilter
* @param {string} sMirCodeFilter
* @param {Object[]} sortedMacroRows
* @param {Object[]} sortedMirCodeRows
* @returns {Object[]}
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

/*
* Находит минимальную дату прохождения по каждому сотруднику и программе.
* @param {number[]} programIds
* @returns {Object[]}
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

/*
* Ищет дату прохождения конкретного сотрудника по конкретной программе (бинарный поиск
* первой строки сотрудника + короткий линейный проход только по его строкам).
* @param {Object[]} sortedDateRows
* @param {number} collaboratorID
* @param {number} programID
* @returns {string}
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
        if (midId == iTarget)
        {
            startIdx = mid;
            hi = mid - 1;
        }
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
* Находит/создаёт накопитель для пары (город, программа).
* @param {Object[]} acc
* @param {string} sCity
* @param {number} iProgramId
* @param {string} sProgramName
* @param {string} sMacroregion   -   Макрорегион города -- записывается только при создании строки.
* @returns {Object}
*/
function GetOrCreateCityProgramAcc(acc, sCity, iProgramId, sProgramName, sMacroregion)
{
    var i;
    for (i = 0; i < ArrayCount(acc); i++)
    {
        if (acc[i].city == sCity && Int(acc[i].programId) == Int(iProgramId)) { return acc[i]; }
    }
    var newAcc;
    newAcc = { city: sCity, programId: Int(iProgramId), programName: sProgramName, macroregion: sMacroregion, total: 0, mandatory: 0, fact: 0 };
    acc.push(newAcc);
    return newAcc;
}

/*
* Округление факт/план в проценты (целочисленная арифметика -- сначала умножаем факт на
* 100, потом делим на план, иначе целочисленное деление теряет дробную часть раньше
* времени). "-" если план = 0.
* @param {number} nFact
* @param {number} nPlan
* @returns {string}
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
* Экранирует "&" в "&amp;" перед тем, как класть готовую ссылку в поле RESULT -- виджет
* вставляет значение поля "link" прямо в HTML (href) без собственного экранирования, и
* "&macroregion=" иначе читается браузером как сущность "&macr" + "egion=".
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
* Строит ссылку на страницу ТЭП-отчётов с нужным набором параметров: matrix_id,
* macroregion, mir_code, position_common_id, program_id, result_type, city.
* sCity = "" -> ссылка без фильтра по городу (для строки "Общий итог").
* iProgramId = 0 -> без фильтра по программе (для строки "Общий итог").
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
    return HtmlEscapeAmp(TEP_REPORT_PAGE_URL + sSeparator + sQueryString);
}

//-------------------------------------------------------------------------
//              Точка входа
//-------------------------------------------------------------------------

function Run()
{
    alert("Run(). НАЧАЛО (Процент обученных)");
    var sFullUrl, matrixId, iProgramFilter, sMacroregionFilter, sMirCodeFilter, iPositionFilter;
    var matrixContext, matrixRow, allProgramRows, taskRows, bInAudience;
    var programIds, filteredProgramIds, programNames, i, j;
    var activeRows, manualFilteredRows;
    var macroRows, mirCodeRows, cityRows, dateRows, statusRows;
    var macroSorted, citySorted, dateSorted;
    var mirCodeSorted, statusSorted;
    var acc, cityProgramAcc, sCity, sProgramName, sDate, row;
    var totalAcc, resultRows, id;
    var sMacroregionForRow, sLinkMacroregion;
    var iCurUserId;
    var managerRows, sortedByManagerRows, subordinateIds, sortedSubordinateIds;
    var iActiveCountBeforeHierarchy, subFilteredRows, k;

    RESULT = [];

    try
    {
        // Гейт видимости по роли -- выполняется первым, до разбора URL и SQL по самому
        // отчёту.
        iCurUserId = GetCurUserIdSafe();
        sortedSubordinateIds = undefined; // undefined = ограничения по подчинённым НЕТ (УОРиАП видит всех)
        if (!IsUorApMember(iCurUserId))
        {
            // Не УОРиАП -- проверяем, руководитель ли этот пользователь (есть ли у него
            // хоть один подчинённый по всей цепочке вниз).
            managerRows = GetManagerIdRows();
            sortedByManagerRows = SortRowsByManagerId(managerRows);
            subordinateIds = GetManagerHierarchySubordinateIds(sortedByManagerRows, iCurUserId);

            if (ArrayCount(subordinateIds) == 0)
            {
                RESULT = [];
                return;
            }

            sortedSubordinateIds = SortIdArray(subordinateIds);
        }

        sFullUrl = GetRequestUrlSafe();

        matrixId = OptInt(GetQueryParam(sFullUrl, "matrix_id"), 0);
        iProgramFilter = OptInt(GetQueryParam(sFullUrl, "program_id"), 0);
        sMacroregionFilter = GetQueryParam(sFullUrl, "macroregion");
        sMirCodeFilter = GetQueryParam(sFullUrl, "mir_code");
        iPositionFilter = OptInt(GetQueryParam(sFullUrl, "position_common_id"), 0);

        if (matrixId == 0)
        {
            throw ("Не передан matrix_id -- выбранная пользователем матрица обучения");
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

        // Если пользователь прошёл гейт как руководитель (sortedSubordinateIds !=
        // undefined) -- ограничиваем пул сотрудников только его подчинёнными. Если
        // пользователь -- УОРиАП, пул не трогаем.
        if (sortedSubordinateIds != undefined)
        {
            iActiveCountBeforeHierarchy = ArrayCount(activeRows);
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

        programNames = [];
        var taskRowForName;
        for (j = 0; j < ArrayCount(programIds); j++)
        {
            taskRowForName = ArrayOptFind(taskRows, "Int(This.matrix_id) == Int(matrixId) && Int(This.education_method_id) == Int(programIds[j])");
            programNames.push({
                id: Int(programIds[j]),
                name: (taskRowForName != undefined && taskRowForName.pname != undefined ? String(taskRowForName.pname) : "id=" + programIds[j])
            });
        }

        // Пул для total/mandatory и для fact совпадает (оба -- активные + ручные
        // фильтры), считаем один раз.
        manualFilteredRows = ApplyManualFilters(activeRows, iPositionFilter, sMacroregionFilter, sMirCodeFilter, macroSorted, mirCodeSorted);

        acc = [];

        // total/mandatory -- по каждому (сотрудник x программа), сгруппировано по паре
        // (город, программа), только если сотрудник входит в аудиторию матрицы.
        for (i = 0; i < ArrayCount(manualFilteredRows); i++)
        {
            bInAudience = CollaboratorMatchesMatrixAudience(manualFilteredRows[i], matrixRow, mirCodeSorted, statusSorted);
            if (!bInAudience) { continue; }
            sCity = FindCity(citySorted, Int(manualFilteredRows[i].id));
            sMacroregionForRow = FindMacroregion(macroSorted, Int(manualFilteredRows[i].id));
            for (j = 0; j < ArrayCount(programIds); j++)
            {
                sProgramName = FindProgramName(programNames, programIds[j]);
                cityProgramAcc = GetOrCreateCityProgramAcc(acc, sCity, programIds[j], sProgramName, sMacroregionForRow);
                sDate = FindCompletionDate(dateSorted, Int(manualFilteredRows[i].id), programIds[j]);
                cityProgramAcc.total = cityProgramAcc.total + 1;
                if (sDate == "") { cityProgramAcc.mandatory = cityProgramAcc.mandatory + 1; }
            }
        }

        // fact -- по каждому (сотрудник x программа), только прошедшие, тоже с проверкой
        // аудитории. Накопитель создаём только когда реально есть завершение (sDate != "").
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

        // Сортировка -- сначала по названию программы, потом по городу внутри неё
        // (пузырьком -- только проверенные конструкции, без непроверенной функции сортировки).
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
                plan: row.total, // План = Общее
                fact: row.fact,
                percent: FormatPercent(row.fact, row.total),
                mandatory: row.mandatory,
                // Виджет "Табличные данные" различает клик только по строке целиком --
                // один "link" на всю строку, режим фиксирован на "total".
                link: BuildTepLink(matrixId, (row.macroregion != "" ? row.macroregion : sMacroregionFilter), sMirCodeFilter, iPositionFilter, row.programId, "total", row.city)
            });
            totalAcc.total = totalAcc.total + row.total;
            totalAcc.mandatory = totalAcc.mandatory + row.mandatory;
            totalAcc.fact = totalAcc.fact + row.fact;
        }

        id = id + 1;
        // "Общий итог" -- sCity = "" и iProgramId = 0, ссылка ведёт на весь матрикс.
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
    }
    catch (_ex)
    {
        RESULT = [];
        ERROR = 1;
        MESSAGE = ExtractUserError(_ex);
        alert("Run(). ОШИБКА: " + MESSAGE);
    }
    alert("Run(). КОНЕЦ");
}

Run();

// Виджет "Табличные данные" различает клик только по строке целиком -- отдельной
// ссылки "на конкретную ячейку/число" у него нет, поэтому отдельных полей
// total_link/plan_link/fact_link/mandatory_link здесь нет: один клик по строке города ->
// ТЭП-отчёт в режиме "total"; план/факт/обязательно видны прямо в этой таблице или через
// переключение "Режим отчёта" на целевой странице.
COLUMNS = [
    { "data": "id", "editable": true, "hidden": true, "sortable": false },
    { "data": "link", "hidden": true, "editable": false, "sortable": false },
    { "data": "city", "title": "Город", "type": "string", "editable": false, "sortable": true },
    { "data": "program", "title": "Учебная программа", "type": "string", "editable": false, "sortable": true },
    { "data": "total", "title": "Общее кол-во сотрудников", "type": "integer", "editable": false, "sortable": true },
    { "data": "plan", "title": "План", "type": "integer", "editable": false, "sortable": true },
    { "data": "fact", "title": "Факт", "type": "integer", "editable": false, "sortable": true },
    { "data": "percent", "title": "Процент", "type": "string", "editable": false, "sortable": false },
    { "data": "mandatory", "title": "Обязательно к прохождению", "type": "integer", "editable": false, "sortable": true }
];
