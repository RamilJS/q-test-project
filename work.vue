// HREDU-183. ТЭП_общее_кол-во / ТЭП_план / ТЭП_факт / ТЭП_обязательно -- выборка для
// Табличных данных. Один файл, 4 режима через параметр result_type: total | plan |
// fact | mandatory.
//   total        -- все сотрудники, подходящие под аудиторию матрицы (должность/
//                    мир-код/орг-структура/подразделение + статус-исключения).
//   plan         -- то же самое, что total (период прохождения не реализован).
//   fact         -- сотрудники из аудитории матрицы, прошедшие тренинг (НЕ зависимо
//                    от срока прохождения).
//   mandatory    -- аудитория матрицы МИНУС те, кто прошёл (пустая completion_date).
// matrix_id в URL -- id документа "Модульная программа" (compound_program). Внутри
// него -- вложенная коллекция programs/program (задачи), учитываются только задачи с
// type == "education_method". Аудитория (custom_elems) -- ОДНА НА ВСЮ МОДУЛЬНУЮ
// ПРОГРАММУ, проверяется один раз на пару (сотрудник x матрица) -- см.
// CollaboratorMatchesMatrixAudience().
// Фильтры (matrix_id, macroregion, mir_code, position_common_id, program_id, city)
// читаются из Request.Url (надёжно в контексте ВЫБОРКИ).
// RESULT -- простой массив строк, без обёртки.
//
// Цветовая дифференциация (status_color, см. BuildStatusColor()) -- настройка виджета
// "Табличные данные": первой колонкой {"name": "status_color", "width": "5%", "view":
// "color"}. Имя поля в Конфигурации должно БУКВАЛЬНО совпадать с именем поля в RESULT
// ("status_color") -- несовпадение не даёт ошибки, просто тихо не работает.

//-------------------------------------------------------------------------
//              Область констант
//-------------------------------------------------------------------------

DEBUG = false;                               // Включает подробные логи для дебага
LOG_NAME = "agent";                          // Имя журнала на сервере, куда будут писаться логи
CUR_OBJECT_ID = 7683878110140100214;         // ID текущего агента для подстановки в логи

//-------------------------------------------------------------------------
//              Область функций
//-------------------------------------------------------------------------

/*
* Выводит сообщение в логи. Обёрнута в try/catch -- сбой логирования не должен ронять
* основной код.
* @param {string} typeLog   -   Уровень логов. 1 - [DEBUG], 2 - [INFO], 3 - [WARN], 4 - [ERROR].
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
* Достаёт полный URL текущей страницы. В контексте ВЫБОРКИ Request.Url надёжен.
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
* Вырезает значение GET-параметра из полного URL строки -- без regex, без строковых
* методов (не поддерживаются этим движком), через StrOptSubStrPos()/StrRangePos()/
* StrLen(), декодирование через UrlDecode().
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
* Безопасно интерпретирует текстовое "булево" значение custom_elem (f_matrix_active и т.п.).
* @param {string} sValue
* @returns {boolean}
*/
function IsActiveText(sValue)
{
    try { return tools_web.is_true(sValue); }
    catch (_ex) { return (String(sValue) == "true" || String(sValue) == "1"); }
}

/*
* Массовое чтение самих модульных программ (compound_program) -- id, название и ВСЕ
* custom_elems аудитории -- одним SQL-запросом по всем документам сразу.
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
* программ одним SQL-запросом через XML .nodes().
* @returns {Object[]}   -   {matrix_id, object_id, education_method_id, ptype, delay_days, days, pname}.
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

/*
* Собирает programIds ОДНОЙ конкретной матрицы (matrixId) из taskRows (задачи ВСЕХ
* матриц, уже отфильтрованные по type=education_method в самом SQL).
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
* Читает всех действующих сотрудников (базовый пул, ДО аудитории матрицы и ДО ручных
* фильтров).
* @returns {Object[]}
*/
function GetActiveCollaboratorRows()
{
    return ArraySelectAll(XQuery("for $elem in collaborators where $elem/is_dismiss=false() return $elem"));
}

/*
* Находит ID документов коллекции "positions", у которых position_common_id совпадает
* с переданным ID.
* @param {number} iCommonPositionFilter
* @returns {number[]}
*/
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

/*
* Достаёт макрорегион (custom_elem f_2ewj) по всем действующим сотрудникам одним SQL-запросом.
* @returns {Object[]}
*/
function GetMacroregionRows()
{
    var sqlText, macroRows;
    sqlText = "";
    sqlText = sqlText + "select cs.id,\r\n";
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_2ewj'']/value)[1]', 'varchar(max)') as macroregion\r\n";
    sqlText = sqlText + "from collaborators cs\r\n";
    sqlText = sqlText + "inner join collaborator c on c.id = cs.id\r\n";
    sqlText = sqlText + "where cs.is_dismiss != 1";
    macroRows = ArraySelectAll(XQuery("sql:" + sqlText));
    return macroRows;
}

/*
* Город -- custom_elem "sity". Нужен для drill-down из "Процент обученных" по городу.
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
* "Дней отработано" (days_worked) по каждому сотруднику -- DATEDIFF(day, hire_date,
* GETDATE()) считается в SQL (T-SQL DATEDIFF/GETDATE), а не вычитанием дат в движке
* выборки (та арифметика на платформе не подтверждена). hire_date -- верхнеуровневое
* поле collaborator (та же позиция, что position_name/org_name/fullname), не custom_elem.
* @returns {Object[]}   -   Массив {id, days_worked}.
*/
function GetHireDateRows()
{
    var sqlText, rows;
    sqlText = "";
    sqlText = sqlText + "select cs.id,\r\n";
    sqlText = sqlText + "       DATEDIFF(day, c.data.value('(*/hire_date)[1]', 'datetime'), GETDATE()) as days_worked\r\n";
    sqlText = sqlText + "from collaborators cs\r\n";
    sqlText = sqlText + "inner join collaborator c on c.id = cs.id\r\n";
    sqlText = sqlText + "where cs.is_dismiss != 1";
    rows = ArraySelectAll(XQuery("sql:" + sqlText));
    return rows;
}

/*
* Сортирует массив строк по возрастанию числового поля "id" -- подготовка для
* BinarySearchById(). Объект-словарь с динамическим ключом на этой платформе не
* поддерживается, поэтому поиск -- через сортировку + бинарный поиск, а не хэш-мапу.
* @param {Object[]} rows
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
* Сортирует массив чисел по возрастанию -- подготовка для IdArrayContainsSorted().
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
* Находит "дней отработано" (days_worked, см. GetHireDateRows()) конкретного сотрудника;
* -1, если не найдено (нет строки / hire_date не задан) -- защитный фолбэк в
* BuildStatusColor().
* @param {Object[]} sortedHireRows   -   Результат GetHireDateRows() + SortRowsById().
* @param {number} collaboratorID
* @returns {number}
*/
function FindDaysWorked(sortedHireRows, collaboratorID)
{
    var row;
    row = BinarySearchById(sortedHireRows, collaboratorID);
    return (row != undefined ? OptInt(row.days_worked, -1) : -1);
}

/*
* Сортирует dateRows по collaborator_id (у этих строк своего "id" нет, ключевое поле --
* collaborator_id, см. GetCompletionDateRows()).
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
    var sqlText, rows;
    sqlText = "";
    sqlText = sqlText + "select cs.id,\r\n";
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_mir_codes'']/value)[1]', 'varchar(max)') as mir_codes\r\n";
    sqlText = sqlText + "from collaborators cs\r\n";
    sqlText = sqlText + "inner join collaborator c on c.id = cs.id\r\n";
    sqlText = sqlText + "where cs.is_dismiss != 1";
    rows = ArraySelectAll(XQuery("sql:" + sqlText));
    return rows;
}

/*
* Разбивает sText по ЛЮБОЙ строке-разделителю sDelim -- замена .split() (не работает на
* этой платформе), через StrOptSubStrPos()/StrRangePos().
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
    if (row == undefined)
    {
        return false;
    }
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
* Матчинг "*текст*"/"текст*"/"*текст" без regex (не поддерживаются этим движком).
* @param {string} sPattern
* @returns {string[]}
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
* @param {string} sPattern
* @param {string} sText
* @param {boolean} bIgnoreCase   -   true -- игнорировать регистр (у StrOptSubStrPos --
*                                    ИМЕННО true игнорирует регистр, обратное названию).
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
* хотя бы под один паттерн из списка (ИЛИ).
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
* Одна "положительная" ось аудитории (f_position_names/f_org_names/f_subdivision_names).
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
* Ось мир-кода -- у сотрудника может быть несколько кодов (ExtractMirCodes()) --
* совпадение, если хотя бы один код сотрудника подходит хотя бы под один паттерн программы.
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
* Главная функция аудитории -- проверяется один раз на пару (сотрудник x матрица).
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

    // ИСКЛЮЧЕНИЯ -- хватает ОДНОГО совпадения по ЛЮБОЙ оси, чтобы исключить сотрудника.
    if (AxisExcludeMatches(matrixRow.f_position_names_exclude, sPosition)) { return false; }
    if (AxisExcludeMatches(matrixRow.f_org_names_exclude, sOrg)) { return false; }
    if (AxisExcludeMatches(matrixRow.f_subdivision_names_exclude, sSubdivision)) { return false; }
    if (MirCodeAxisExcludeMatches(matrixRow.f_mir_code_exclude, employeeCodes)) { return false; }
    if (AxisExcludeMatches(matrixRow.f_collaborator_statuses_exclude, sStatus)) { return false; }

    // ОСНОВНЫЕ ОСИ -- "И" между ЗАПОЛНЕННЫМИ осями (пустая ось = без ограничения).
    if (!AxisMatches(matrixRow.f_position_names, sPosition)) { return false; }
    if (!AxisMatches(matrixRow.f_org_names, sOrg)) { return false; }
    if (!AxisMatches(matrixRow.f_subdivision_names, sSubdivision)) { return false; }
    if (!MirCodeAxisMatches(matrixRow.f_mir_code, employeeCodes)) { return false; }

    return true;
}

/*
* Находит минимальную дату прохождения по каждому сотруднику и программе.
* @param {number[]} programIds
* @returns {Object[]}
*/
function GetCompletionDateRows(programIds)
{
    var sqlText, dateRows;
    sqlText = "";
    sqlText = sqlText + "select ec.collaborator_id, e.education_method_id, min(ec.start_date) as first_date\r\n";
    sqlText = sqlText + "from event_collaborators ec\r\n";
    sqlText = sqlText + "join events e on e.id = ec.event_id\r\n";
    sqlText = sqlText + "where e.education_method_id in (" + ArrayMerge(programIds, "This", ",") + ")\r\n";
    sqlText = sqlText + "group by ec.collaborator_id, e.education_method_id";
    dateRows = ArraySelectAll(XQuery("sql:" + sqlText));
    return dateRows;
}

/*
* Ищет дату прохождения конкретного сотрудника по конкретной программе. Бинарный поиск
* первой строки этого сотрудника (по sortedDateRows), дальше короткий линейный проход
* только по строкам этого сотрудника (их мало) в поисках нужной программы.
* @param {Object[]} sortedDateRows   -   Массив, УЖЕ отсортированный через SortDateRowsByCollaboratorId().
* @param {number} collaboratorID
* @param {number} programID
* @returns {string}
*/
function FindCompletionDate(sortedDateRows, collaboratorID, programID)
{
    var lo, hi, mid, midId, iTarget, iProgram, startIdx, i, n;

    iTarget = Int(collaboratorID);
    iProgram = Int(programID);

    // Бинарный поиск ЛЮБОЙ строки этого сотрудника, затем сдвигаемся к ПЕРВОЙ такой строке
    // (могут быть дубликаты collaborator_id -- по одной строке на каждую пройденную программу).
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
            hi = mid - 1; // ищем более ранний индекс с тем же collaborator_id
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
* Ищет макрорегион конкретного сотрудника.
* @param {Object[]} sortedMacroRows   -   Массив, УЖЕ отсортированный через SortRowsById().
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
* Собирает строки отчёта для ОДНОГО сотрудника -- по одной строке на каждую программу
* матрицы.
* @param {Object} collaborator
* @param {Object[]} macroSorted
* @param {Object[]} citySorted
* @param {Object[]} dateSorted
* @param {Object[]} hireSorted     -   GetHireDateRows() + SortRowsById().
* @param {Object[]} programNames   -   Массив {id, name, delay_days, days}, собран в Run() из taskRows.
* @param {number[]} programIds
* @param {boolean} bInAudience     -   Результат CollaboratorMatchesMatrixAudience() для ЭТОГО
*                                      сотрудника и ТЕКУЩЕЙ матрицы -- один на всех программ.
* @returns {Object[]}
*/
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
        row.status_color = BuildStatusColor(bPassed, iDaysWorked, programTiming.delayDays, programTiming.days);

        rows.push(row);
    }
    return rows;
}

/*
* Статус цвета по п.6 ТЗ (7 статусов): различает "не пройдено" ПО СРОКУ прохождения
* (delay_days -- "старт через", days -- "продолжительность" задачи education_method,
* см. FindProgramTiming()) относительно даты приёма сотрудника (hire_date, см.
* GetHireDateRows()/FindDaysWorked()):
*   - bPassed=true                                                          -> "green"
*   - bPassed=false, iDaysWorked < iDelayDays ("срок ещё не наступил")      -> "white"
*   - bPassed=false, iDelayDays <= iDaysWorked < iDelayDays+iDurationDays
*     ("срок наступил, но ещё не истёк")                                    -> "white"
*   - bPassed=false, iDaysWorked >= iDelayDays+iDurationDays ("срок истёк") -> "red"
* iDaysWorked == -1 -- нет hire_date у сотрудника -- защитный фолбэк на бинарную
* логику (green/red по bPassed).
* ПРОВЕРИТЬ РЕАЛЬНЫМ ТЕСТОМ: "white" как значение status_color (view:"color") отдельно
* не проверялось -- подтверждены только "green"/"red".
* @param {boolean} bPassed
* @param {number} iDaysWorked    -   Дней с даты приёма (hire_date) до сегодня, или -1.
* @param {number} iDelayDays     -   "Старт через" (delay_days) задачи education_method.
* @param {number} iDurationDays  -   "Продолжительность" (days) задачи education_method.
* @returns {string}  -  "white" | "green" | "red".
*/
function BuildStatusColor(bPassed, iDaysWorked, iDelayDays, iDurationDays)
{
    if (bPassed)
    {
        return "green";
    }
    if (iDaysWorked < 0)
    {
        // Защитный фолбэк -- нет данных по hire_date.
        return "red";
    }
    if (iDaysWorked >= (OptInt(iDelayDays, 0) + OptInt(iDurationDays, 0)))
    {
        return "red";
    }
    return "white";
}

/*
* Резолвит id программы в название через кэш programNames (собран в Run() из taskRows).
* @param {Object[]} programNames   -   Массив {id, name, delay_days, days}.
* @param {number} iProgramId
* @returns {string}
*/
function FindProgramName(programNames, iProgramId)
{
    var row;
    row = ArrayOptFind(programNames, "Int(This.id) == Int(iProgramId)");
    return (row != undefined ? String(row.name) : "id=" + iProgramId);
}

/*
* Находит delay_days ("старт через") и days ("продолжительность") учебной программы.
* Оба читаются через OptInt(..., 0) -- days может ОТСУТСТВОВАТЬ в XML, если
* продолжительность не указана; тогда порог "просрочки" равен просто delay_days.
* @param {Object[]} programNames   -   Массив {id, name, delay_days, days}.
* @param {number} iProgramId
* @returns {Object}   -   {delayDays: number, days: number}.
*/
function FindProgramTiming(programNames, iProgramId)
{
    var row;
    row = ArrayOptFind(programNames, "Int(This.id) == Int(iProgramId)");
    return (row != undefined ? { delayDays: OptInt(row.delay_days, 0), days: OptInt(row.days, 0) } : { delayDays: 0, days: 0 });
}

/*
* Ищет матрицу по id (бинарный поиск в bulk-результате GetCompoundProgramRows()),
* проверяет f_matrix_active, собирает programIds.
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
        throw ("У модульной программы [" + matrixRow.name + "] не найдено ни одной задачи с типом [Учебная программа] (education_method)");
    }

    return { matrixRow: matrixRow, programIds: programIds };
}

/*
* Точка входа. Собирает данные ОДНОГО из 4 отчётов-представлений ТЭП, в зависимости от
* result_type.
* @returns {void}
*/
function Run()
{
    LogAlert(2, "Run(). НАЧАЛО");
    var sResultType, sFullUrl, matrixId;
    var iProgramFilter, sMacroregionFilter, sMirCodeFilter, iPositionFilter, sCityFilter;
    var matrixContext, matrixRow, allProgramRows, taskRows;
    var programIds, programNames, collaboratorRows, macroRows, mirCodeRows, cityRows, dateRows, statusRows, hireRows;
    var macroSorted, citySorted, dateSorted;
    var mirCodeSorted, statusSorted, hireSorted;
    var collaboratorReportRows, i, j, taskRowForName, bInAudience;
    var filteredProgramIds, filteredCollaboratorRows, allowedPositionIds, filteredResultRows;

    ERROR = 0;
    MESSAGE = "";
    RESULT = [];

    try
    {
        sFullUrl = GetRequestUrlSafe();
        LogAlert(1, "Run(). Request.Url = [" + sFullUrl + "]");

        // result_type: сначала из URL (пользователь может переключать сам), иначе
        // старое фиксированное значение LPE, иначе "total".
        sResultType = GetQueryParam(sFullUrl, "result_type");
        if (sResultType == "")
        {
            try
            {
                sResultType = String(result_type);
            }
            catch (_exResultTypeParam)
            {
                sResultType = "";
            }
        }
        if (sResultType == "" || sResultType == "undefined")
        {
            sResultType = "total";
        }
        if (sResultType != "total" && sResultType != "plan" && sResultType != "fact" && sResultType != "mandatory")
        {
            throw ("Неизвестный result_type=[" + sResultType + "] -- ожидается одно из: total, plan, fact, mandatory");
        }
        LogAlert(1, "Run(). sResultType=[" + sResultType + "]");

        matrixId = OptInt(GetQueryParam(sFullUrl, "matrix_id"), 0);
        iProgramFilter = OptInt(GetQueryParam(sFullUrl, "program_id"), 0);
        sMacroregionFilter = GetQueryParam(sFullUrl, "macroregion");
        sMirCodeFilter = GetQueryParam(sFullUrl, "mir_code");
        iPositionFilter = OptInt(GetQueryParam(sFullUrl, "position_common_id"), 0);
        // Необязательный фильтр по городу (custom_elem "sity") -- для drill-down из
        // "Процент обученных".
        sCityFilter = GetQueryParam(sFullUrl, "city");

        LogAlert(1, "Run(). matrixId=" + matrixId + " programFilter=" + iProgramFilter
            + " macroregionFilter=[" + sMacroregionFilter + "] mirCodeFilter=[" + sMirCodeFilter
            + "] positionCommonIdFilter=" + iPositionFilter + " cityFilter=[" + sCityFilter + "]");

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

        // Имена программ и их тайминг (delay_days/days) берутся напрямую из taskRows --
        // без обращения к документам.
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

        // mirCodeRows нужен и для аудитории матрицы, и для ручного фильтра по мир-коду
        // ниже -- грузим один раз здесь и переиспользуем.
        mirCodeRows = GetMirCodeRows();
        mirCodeSorted = SortRowsById(mirCodeRows);

        statusRows = GetStatusRows();
        statusSorted = SortRowsById(statusRows);

        hireRows = GetHireDateRows();
        hireSorted = SortRowsById(hireRows);

        // Дальше -- РУЧНЫЕ фильтры пользователя.
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
            LogAlert(1, "Run(). После ручного фильтра по типовой должности осталось сотрудников: " + ArrayCount(collaboratorRows));
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
            LogAlert(1, "Run(). После ручного фильтра по макрорегиону осталось сотрудников: " + ArrayCount(collaboratorRows));
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
            LogAlert(1, "Run(). После ручного фильтра по мир-коду осталось сотрудников: " + ArrayCount(collaboratorRows));
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
            LogAlert(1, "Run(). После фильтра по городу осталось сотрудников: " + ArrayCount(collaboratorRows));
        }

        dateRows = GetCompletionDateRows(programIds);
        dateSorted = SortDateRowsByCollaboratorId(dateRows);

        // Аудитория матрицы проверяется ОДИН РАЗ НА СОТРУДНИКА, ДО построения его строк --
        // сотрудники не из аудитории не попадают в RESULT вообще. Применяется ко всем 4
        // режимам одинаково, включая "fact".
        RESULT = [];
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
                RESULT.push(collaboratorReportRows[j]);
            }
        }
        LogAlert(1, "Run(). Строк (сотрудник x программа) после фильтра аудитории матрицы, до result_type: " + ArrayCount(RESULT));

        // Финальное разбиение по result_type. "total"/"plan" -- без изменений (План = Общее).
        if (sResultType == "fact")
        {
            filteredResultRows = [];
            for (i = 0; i < ArrayCount(RESULT); i++)
            {
                if (RESULT[i].completion_date != "")
                {
                    filteredResultRows.push(RESULT[i]);
                }
            }
            RESULT = filteredResultRows;
        }
        else if (sResultType == "mandatory")
        {
            filteredResultRows = [];
            for (i = 0; i < ArrayCount(RESULT); i++)
            {
                if (RESULT[i].completion_date == "")
                {
                    filteredResultRows.push(RESULT[i]);
                }
            }
            RESULT = filteredResultRows;
        }

        LogAlert(2, "Run(). Готово. result_type=" + sResultType + ", строк отчёта: " + ArrayCount(RESULT));
    }
    catch (_ex)
    {
        ERROR = 1;
        MESSAGE = ExtractUserError(_ex);
        LogAlert(4, "Run(). ОШИБКА: " + MESSAGE);
    }
    LogAlert(2, "Run(). КОНЕЦ");
}

//-------------------------------------------------------------------------
//              Область основного кода
//-------------------------------------------------------------------------

Run();
