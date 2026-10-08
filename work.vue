// =====================================================================
// HREDU-239 (бывшая HREDU-181). "Восток_полный_список" -- удалённое действие на кнопку
// "Выгрузить отчёт". Строит Excel-выгрузку (через HTML-таблицу с Office-namespace --
// тот же проверенный приём, что в примерах пользователя vtbl_full_education_report.js и
// assessment_report.js): одна строка на сотрудника (ФИО, Должность, Подразделение,
// Макрорегион), затем ПО ОДНОЙ КОЛОНКЕ НА КАЖДУЮ учебную программу выбранной матрицы --
// дата прохождения или пусто. Число колонок переменное (зависит от матрицы) -- в HTML
// это не проблема (в отличие от виджета "Табличные данные" с фиксированной
// Конфигурацией), поэтому отдельная выборка под виджет здесь не нужна.
//
// СХЕМА СТРАНИЦЫ (со слов пользователя, 07.10.2026):
//   1. Параметры "Матрица" и "Макрорегион" -- отображаются на странице (значения
//      приходят через URL, см. ниже).
//   2. Кнопка "Настроить фильтры" -- открывает модалку. ПЕРЕИСПОЛЬЗУЕМ БЕЗ ИЗМЕНЕНИЙ
//      HREDU_182_filtry_percent.js -- она уже спроектирована именно под этот случай
//      (см. её собственную шапку: "...и, если понадобится, для 'Восток полный список'").
//      Та модалка пишет выбранные matrix_id/macroregion в URL текущей страницы (redirect)
//      -- ТОЧНО тот же механизм, что уже используют страницы ТЭП/"Процент обученных".
//   3. Кнопка "Выгрузить отчёт" -- запускает ЭТОТ файл.
//
// ИСТОЧНИК ДАННЫХ -- та же модель HREDU-182/183/237: compound_program ("Модульные
// программы"), вложенные задачи (programs/program) с type == "education_method" --
// это и есть колонки отчёта; аудитория -- custom_elems самого compound_program,
// проверяется один раз на пару (сотрудник x матрица). ВСЕ функции гейта видимости,
// аудитории и подбора матрицы/программ ниже -- БУКВАЛЬНАЯ КОПИЯ уже подтверждённого
// реальными тестами кода из HREDU-182_procent_obuchennyh.js и HREDU_182_filtry_percent.js
// -- история находок и фиксов там же, здесь не повторяем.
//
// ВИДИМОСТЬ: строго по подчинённости (ТЗ HREDU-239) -- сотрудник УОРиАП видит всех,
// остальные -- только себя и подчинённых по всей цепочке вниз (GetManagerHierarchySubordinateIds).
//
// НЕ ПРОВЕРЕНО РЕАЛЬНЫМ ТЕСТОМ, ТРЕБУЕТ ПОДТВЕРЖДЕНИЯ ДО ЗАПУСКА В ПРОД:
//   1. curUserId должен быть привязан как LPE-параметр в редакторе страниц ИМЕННО к этому
//      удалённому действию (кнопка "Выгрузить отчёт") -- так же, как это уже сделано для
//      HREDU_182_filtry_percent.js и выборки отчёта. Без этой привязки GetCurUserIdSafe()
//      вернёт 0 и отчёт по fail-safe будет пустым.
//   2. РЕШЕНО (08.10.2026): EXPORT_PAGE_URL = "assessment_excel_export.html" -- переиспользуем
//      СУЩЕСТВУЮЩУЮ универсальную страницу-компаньон (та же, что уже работает для отчёта
//      по тестированию), никакой новой страницы заводить не нужно -- она ничего не знает
//      про конкретный отчёт, просто отдаёт то, что лежит в кэше под "excel_html_" + curUserID.
//   3. CUR_OBJECT_ID ниже -- ЗАГЛУШКА (0). Нужен реальный уникальный id этого агента для
//      логов (см. правила логирования) -- подставить настоящий id после создания агента
//      на платформе.
//   4. Порядок колонок-программ = порядок их появления в GetEducationMethodTaskRows()
//      (то есть порядок задач programs/program внутри XML самой матрицы). Если нужен
//      другой порядок (например, алфавитный) -- сообщить, это однострочная правка.
//   5. Каталог "compound_program" для пикера матрицы в HREDU_182_filtry_percent.js помечен
//      там как НЕ подтверждённый реальным тестом -- тот же риск относится и сюда
//      (GetCompoundProgramRows/GetEducationMethodTaskRows читают через SQL compound_programs/
//      compound_program, а не через пикер, так что риск только в скрипте-источнике, если
//      имя таблицы отличается -- стоит проверить тем же первым реальным тестом).
//   6. Ключ кэша tools_web.set_user_data() собран из "голой" curUserID (с большой D) --
//      ТОЧНО как в обоих примерах пользователя (vtbl_full_education_report.js,
//      assessment_report.js) -- тот же ключ, который читает assessment_excel_export.html.
//      Если эта переменная недоступна без LPE-привязки в этом удалённом действии -- откат
//      на iCurUserId (см. комментарий прямо над вызовом set_user_data() в Run()).
// =====================================================================

//-------------------------------------------------------------------------
//              Область констант
//-------------------------------------------------------------------------

DEBUG = false;
LOG_NAME = "agent";
CUR_OBJECT_ID = 0; // ЗАГЛУШКА -- подставить реальный id агента, см. п.3 выше.

// Переиспользуем ТВОЮ уже рабочую страницу-компаньон -- она универсальна (читает
// tools_web.get_user_data("excel_html_" + curUserID) и отдаёт на скачивание, без привязки
// к конкретному отчёту), поэтому отдельная страница под "Восток_полный_список" не нужна.
EXPORT_PAGE_URL = "assessment_excel_export.html";

//-------------------------------------------------------------------------
//              Область функций
//-------------------------------------------------------------------------

/*
* Выводит сообщение в логи. Обёрнута в try/catch -- сбой логирования не должен ронять
* основной код.
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
        // сбой логирования не должен ронять основной код
    }
}

/*
* Достаёт полный URL текущей страницы. В контексте УДАЛЁННОГО ДЕЙСТВИЯ Request.Url НЕ
* отражает видимый адрес страницы -- нужен параметр "cur_page_url" (LPE-подстановка
* {{curEnv.curEnvUrl}}), см. HREDU_182_filtry_percent.js/HREDU-183_filtry_modal_shag1.js.
* @returns {string}
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
    try { return String(Request.Url); }
    catch (_ex) { return ""; }
}

/*
* Вырезает значение GET-параметра из полного URL строки -- без regex, без .indexOf/.substring
* (не существуют в этом движке), через штатный строковый API платформы.
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
        if (iParamPos == undefined) { return ""; }
        iValueStart = iParamPos + StrLen(sQMarkMarker);
    }
    iAmpPos = StrOptSubStrPos(sUrl, "&", false, iValueStart);
    iValueEnd = (iAmpPos != undefined ? iAmpPos : iUrlLen);
    sRawValue = StrRangePos(sUrl, iValueStart, iValueEnd);
    try { return UrlDecode(sRawValue); }
    catch (_exDecode) { return sRawValue; }
}

// --- Гейт видимости по подчинённости (БУКВАЛЬНАЯ КОПИЯ из HREDU-182_procent_obuchennyh.js) ---

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

// --- Подбор матрицы/программ (БУКВАЛЬНАЯ КОПИЯ из HREDU-182_procent_obuchennyh.js) ---

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
* Задачи-колонки ОДНОЙ выбранной матрицы, в порядке их появления в GetEducationMethodTaskRows()
* (см. п.4 в шапке файла -- порядок = порядок задач в самой матрице).
* @param {Object[]} taskRows
* @param {number} matrixId
* @returns {Object[]}   -   Массив {id, name}, без дублей по education_method_id.
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

// --- Сотрудники/макрорегион/мир-код/статус (БУКВАЛЬНАЯ КОПИЯ из HREDU-182_procent_obuchennyh.js) ---

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

// --- Даты прохождения (БУКВАЛЬНАЯ КОПИЯ из HREDU-182_procent_obuchennyh.js) ---

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

// --- Сборка HTML-таблицы "Excel" (тот же приём, что в vtbl_full_education_report.js) ---

/*
* Строит HTML-таблицу с Office XML namespace -- Excel открывает такой HTML как книгу.
* Колонки: ФИО, Должность, Подразделение, Макрорегион + по одной на каждую программу
* матрицы (programColumns), в их порядке.
* @param {string} sMatrixName
* @param {Object[]} programColumns   -   GetOrderedProgramColumns(): {id, name}.
* @param {Object[]} employeeRows     -   {fullname, position_name, position_parent_name, macroregion, id}.
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

//-------------------------------------------------------------------------
//              Точка входа
//-------------------------------------------------------------------------

function Run()
{
    var sPageUrl, iMatrixId, sMacroregionFilter, allProgramRows, taskRows, matrixRow, programColumns;
    var iCurUserId, managerRows, sortedByManagerRows, subordinateIds, sortedSubordinateIds;
    var activeRows, subFilteredRows, k;
    var macroRows, macroSorted, mirCodeRows, mirCodeSorted, statusRows, statusSorted;
    var dateRows, dateSorted;
    var employeeRows, i, bInAudience, sMacroregionForRow;
    var sHtml, sExportUrl, sBareCurUserId;

    ERROR = 0;
    MESSAGE = "";
    RESULT = new Object();

    try
    {
        LogAlert(2, "Run(). НАЧАЛО (Восток_полный_список)");

        iCurUserId = GetCurUserIdSafe();
        sortedSubordinateIds = undefined; // undefined = без ограничения (УОРиАП)
        if (!IsUorApMember(iCurUserId))
        {
            managerRows = GetManagerIdRows();
            sortedByManagerRows = SortRowsByManagerId(managerRows);
            subordinateIds = GetManagerHierarchySubordinateIds(sortedByManagerRows, iCurUserId);
            if (ArrayCount(subordinateIds) == 0)
            {
                throw "У вас нет ни одного подчинённого -- отчёт недоступен.";
            }
            sortedSubordinateIds = SortIdArray(subordinateIds);
        }

        sPageUrl = GetCurPageUrlSafe();
        iMatrixId = OptInt(GetQueryParam(sPageUrl, "matrix_id"), 0);
        sMacroregionFilter = GetQueryParam(sPageUrl, "macroregion");

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
        mirCodeRows = GetMirCodeRows();
        mirCodeSorted = SortRowsById(mirCodeRows);
        statusRows = GetStatusRows();
        statusSorted = SortRowsById(statusRows);

        dateRows = GetCompletionDateRows(ArrayExtract(programColumns, "Int(id)"));
        dateSorted = SortDateRowsByCollaboratorId(dateRows);

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

        if (ArrayCount(employeeRows) == 0)
        {
            RESULT = {
                command: "alert",
                view: "warning",
                msg: "Данных не найдено -- попробуйте изменить параметры фильтра (матрица/макрорегион) или проверьте, что у вас/ваших подчинённых есть сотрудники в аудитории этой матрицы."
            };
            LogAlert(2, "Run(). КОНЕЦ (пусто)");
            return;
        }

        employeeRows = ArraySort(employeeRows, "This.fullname", "+");

        sHtml = BuildExcelHtml(String(matrixRow.name), programColumns, employeeRows, dateSorted);

        // ИСПРАВЛЕНО (08.10.2026, по факту 404 на странице-компаньоне) -- ключ кэша теперь
        // собирается из "голой" curUserID (с большой D), ТОЧНО как в обоих примерах
        // пользователя (vtbl_full_education_report.js и assessment_report.js) -- именно эту
        // переменную, скорее всего, читает существующая страница-экспорта. iCurUserId
        // (LPE-привязанная curUserId) используется ТОЛЬКО для гейта видимости выше -- это
        // отдельная переменная, подтверждённая рабочей для иерархии руководителей, но не
        // для этого кэша. Если sBareCurUserId ниже пуст (переменная недоступна без LPE-
        // привязки в этом удалённом действии) -- откат на iCurUserId, чтобы кэш хотя бы
        // не остался без ключа вообще; но тогда страница-экспорт, скорее всего, всё равно
        // не найдёт данные, пока не будет ясности, какую переменную она реально читает.
        try { sBareCurUserId = String(curUserID); }
        catch (_exBareUserId) { sBareCurUserId = ""; }
        if (sBareCurUserId == "" || sBareCurUserId == "undefined" || sBareCurUserId == "null")
        {
            sBareCurUserId = String(iCurUserId);
        }
        tools_web.set_user_data("excel_html_" + sBareCurUserId, ({ "html": sHtml }), 3600);

        sExportUrl = EXPORT_PAGE_URL;
        RESULT = {
            command: "alert",
            msg: "Файл готов в выгрузке",
            confirm_result: {
                command: "new_window",
                url: sExportUrl
            }
        };

        LogAlert(2, "Run(). КОНЕЦ (сотрудников в выгрузке: " + ArrayCount(employeeRows) + ", колонок-программ: " + ArrayCount(programColumns) + ")");
    }
    catch (_ex)
    {
        ERROR = 1;
        MESSAGE = String(_ex);
        RESULT = { command: "alert", view: "warning", msg: MESSAGE };
        LogAlert(4, "Run(). ОШИБКА: " + MESSAGE);
    }
}

Run();
