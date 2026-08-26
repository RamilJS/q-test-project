// HREDU-174. Агент создания и проставления типовых должностей.
 
//-------------------------------------------------------------------------
//              Область констант
//-------------------------------------------------------------------------
 
DEBUG = false;              // Включает подробные логи уровня 1 [DEBUG] -- на проде и в репозитории должно быть false
LOG_NAME = "agent";         // Системный лог для агентов сервера (отдельный server_agent не заводим)
CUR_OBJECT_ID = oData.id;   // ID текущего документа агента
TEST_TOP_N = 5;             // Ограничение выборки должностей для тестового прогона. 0 = без ограничения (боевой режим)
PROGRESS_LOG_STEP = 100;    // Через сколько обработанных должностей писать в лог прогресс выполнения
 
//-------------------------------------------------------------------------
//              Область функций
//-------------------------------------------------------------------------
 
/*
 * Выводит сообщение в логи.
 * @param {number} typeLog     -   Уровень логов. 1 - [DEBUG], 2 - [INFO], 3 - [WARN], 4 - [ERROR].
 * @param {string} message     -   Сообщение для логов.
 * @returns {void}
 */
function LogAlert(typeLog, message)
{
    tools.call_code_library_method("vtbl_log_lib", "LogAlert", [LOG_NAME, typeLog, CUR_OBJECT_ID, message, DEBUG]);
}
 
/*
 * Приводит "сырое" название должности к нормализованному виду для сравнения и дедупликации:
 * подставляет пустую строку вместо undefined/null, обрезает пробелы по краям и схлопывает
 * повторы пробелов/табов/переносов строк внутри строки.
 * @param {string} rawName     -   Исходное название должности (может быть undefined/null).
 * @returns {string}
 */
function NormalizePositionName(rawName)
{
    LogAlert(1, "NormalizePositionName(). НАЧАЛО");
    var safeRawName, normalizedName;
    safeRawName = "" + rawName;
    if (safeRawName == "undefined")
    {
        safeRawName = "";
    }
    if (safeRawName == "null")
    {
        safeRawName = "";
    }
    normalizedName = Trim(UnifySpaces(safeRawName));
    LogAlert(1, "NormalizePositionName(). КОНЕЦ");
    return normalizedName;
}
 
/*
 * Проверяет, содержится ли указанное значение в массиве строк.
 * @param {string[]} values    -   Массив строк для поиска.
 * @param {string} target      -   Искомое значение.
 * @returns {boolean}
 */
function ArrayContainsString(values, target)
{
    LogAlert(1, "ArrayContainsString(). НАЧАЛО");
    var i, result;
    result = false;
    for (i = 0; i < values.length; i++)
    {
        if (values[i] == target)
        {
            result = true;
        }
    }
    LogAlert(1, "ArrayContainsString(). КОНЕЦ");
    return result;
}
 
/*
 * Ищет ID типовой должности по нормализованному названию в списке пар { name, id }.
 * @param {Object[]} commonPositionPairs   -   Список пар { name, id } типовых должностей.
 * @param {string} name                    -   Нормализованное название должности.
 * @returns {?number}                      -   ID типовой должности либо null, если не найдена.
 */
function FindPositionIdByName(commonPositionPairs, name)
{
    LogAlert(1, "FindPositionIdByName(). НАЧАЛО");
    var i, result;
    result = null;
    for (i = 0; i < commonPositionPairs.length; i++)
    {
        if (commonPositionPairs[i].name == name)
        {
            result = commonPositionPairs[i].id;
        }
    }
    LogAlert(1, "FindPositionIdByName(). КОНЕЦ");
    return result;
}
 
/*
 * Строит список пар { name, id } из строк справочника типовых должностей,
 * нормализуя название каждой типовой должности.
 * @param {Object[]} commonPositionRows    -   Строки { id, name } из position_commons.
 * @returns {Object[]}                     -   Список пар { name, id }.
 */
function BuildCommonPositionPairs(commonPositionRows)
{
    LogAlert(1, "BuildCommonPositionPairs(). НАЧАЛО");
    var pairs, i, row, normalizedName;
    pairs = [];
    for (i = 0; i < commonPositionRows.length; i++)
    {
        row = commonPositionRows[i];
        normalizedName = NormalizePositionName(row.name);
        pairs.push({ name: normalizedName, id: row.id });
    }
    LogAlert(1, "BuildCommonPositionPairs(). КОНЕЦ");
    return pairs;
}
 
/*
 * Читает из БД должности (position), реально занятые действующим сотрудником и не
 * закрытые. Направление связи и фильтры -- по образцу, присланному тимлидом:
 * от позиции к сотруднику через positions.basic_collaborator_id (а не от
 * сотрудника к позиции через collaborators.position_id), плюс проверка
 * ps.is_position_finished, а не только is_dismiss у сотрудника.
 * Если TEST_TOP_N > 0, ограничивает выборку первыми N записями (для тестового прогона).
 * ПРЕДПОЛОЖЕНИЕ (проверить при первом прогоне): ps.name -- реальная колонка
 * представления positions (по аналогии с position_commons.name).
 * @returns {Object[]}     -   Массив строк { position_id, position_name }.
 */
function GetActivePositionsInUse()
{
    LogAlert(1, "GetActivePositionsInUse(). НАЧАЛО");
    var topClause, sqlText, rawRows, positionRows, row;
    topClause = "";
    if (TEST_TOP_N > 0)
    {
        topClause = "top " + TEST_TOP_N + " ";
    }
    sqlText = "";
    sqlText = sqlText + "select distinct " + topClause + "ps.id as position_id, ps.name as position_name\r\n";
    sqlText = sqlText + "from positions ps\r\n";
    sqlText = sqlText + "join collaborators cs on ps.basic_collaborator_id = cs.id\r\n";
    sqlText = sqlText + "where cs.is_dismiss <> 1\r\n";
    sqlText = sqlText + "  and ps.is_position_finished <> 1\r\n";
    sqlText = sqlText + "  and ps.name is not null\r\n";
    sqlText = sqlText + "  and ltrim(rtrim(ps.name)) != ''\r\n";
    rawRows = XQuery("sql:" + sqlText);
    positionRows = [];
    for (row in rawRows)
    {
        positionRows.push(row);
    }
    LogAlert(1, "GetActivePositionsInUse(). Найдено должностей: " + positionRows.length);
    LogAlert(1, "GetActivePositionsInUse(). КОНЕЦ");
    return positionRows;
}
 
/*
 * Читает из БД все существующие типовые должности (справочник position_commons).
 * @returns {Object[]}     -   Массив строк { id, name }.
 */
function GetExistingCommonPositionNames()
{
    LogAlert(1, "GetExistingCommonPositionNames(). НАЧАЛО");
    var sqlText, rawRows, commonPositionRows, row;
    sqlText = "";
    sqlText = sqlText + "select pc.id, pc.name\r\n";
    sqlText = sqlText + "from position_commons pc\r\n";
    rawRows = XQuery("sql:" + sqlText);
    commonPositionRows = [];
    for (row in rawRows)
    {
        commonPositionRows.push(row);
    }
    LogAlert(1, "GetExistingCommonPositionNames(). Найдено типовых должностей: " + commonPositionRows.length);
    LogAlert(1, "GetExistingCommonPositionNames(). КОНЕЦ");
    return commonPositionRows;
}
 
/*
 * Создаёт новую типовую должность (объект position_common) с указанным названием.
 * @param {string} name     -   Нормализованное название новой типовой должности.
 * @returns {Object}        -   TopElem созданного документа (содержит id).
 */
function CreateCommonPosition(name)
{
    LogAlert(1, "CreateCommonPosition(). НАЧАЛО");
    var oNewDoc, oNewDocTE;
    oNewDoc = tools.new_doc_by_name("position_common");
    oNewDocTE = oNewDoc.TopElem;
    oNewDocTE.name = name;
    oNewDoc.BindToDb(DefaultDb);
    oNewDoc.Save();
    LogAlert(1, "CreateCommonPosition(). Создана типовая должность: " + name);
    LogAlert(1, "CreateCommonPosition(). КОНЕЦ");
    return oNewDocTE;
}
 
/*
 * Проверяет список нормализованных названий должностей и создаёт недостающие
 * типовые должности в справочнике position_commons.
 * @param {string[]} positionNames         -   Уникальные нормализованные названия должностей.
 * @param {Object[]} commonPositionPairs   -   Текущий список пар { name, id } типовых должностей.
 * @returns {Object[]}                     -   Обновлённый список пар { name, id } (с учётом созданных).
 */
function EnsureCommonPositionsExist(positionNames, commonPositionPairs)
{
    LogAlert(1, "EnsureCommonPositionsExist(). НАЧАЛО");
    var i, name, commonPosition;
    for (i = 0; i < positionNames.length; i++)
    {
        name = positionNames[i];
        if (FindPositionIdByName(commonPositionPairs, name) == null)
        {
            commonPosition = CreateCommonPosition(name);
            commonPositionPairs.push({ name: name, id: commonPosition.id });
        }
    }
    LogAlert(1, "EnsureCommonPositionsExist(). КОНЕЦ");
    return commonPositionPairs;
}
 
/*
 * Проставляет должности (position) ссылку на её типовую должность -- в штатное
 * поле position_common_id (поле уже существует на объекте position, заводить
 * его через админку не требуется).
 * ВАЖНО: если у позиции position_common_id уже заполнен (не обязательно нашим
 * агентом -- поле используется и другой, более старой логикой в системе),
 * агент его НЕ перезаписывает и пропускает позицию. Это защита от случайной
 * порчи ранее проставленных вручную/другим процессом значений, которые могут
 * не совпадать с простым сопоставлением по имени.
 * @param {Object} positionRow              -   Строка { position_id, position_name }.
 * @param {Object[]} commonPositionPairs    -   Список пар { name, id } типовых должностей.
 * @returns {void}
 */
function AssignCommonPositionToPosition(positionRow, commonPositionPairs)
{
    LogAlert(1, "AssignCommonPositionToPosition(). НАЧАЛО");
    var oDoc, normalizedName, commonPositionId;
    oDoc = tools.open_doc(positionRow.position_id);
    if (oDoc.TopElem.position_common_id.HasValue)
    {
        LogAlert(2, "AssignCommonPositionToPosition(). У позиции ID=" + positionRow.position_id + " [" + positionRow.position_name + "] position_common_id уже заполнен, пропуск");
    }
    else
    {
        normalizedName = NormalizePositionName(positionRow.position_name);
        commonPositionId = FindPositionIdByName(commonPositionPairs, normalizedName);
        if (commonPositionId == null)
        {
            LogAlert(3, "AssignCommonPositionToPosition(). Не найден commonPositionId для должности ID=" + positionRow.position_id + " [" + normalizedName + "], пропуск");
        }
        else
        {
            oDoc.TopElem.position_common_id = commonPositionId;
            oDoc.Save();
            if (TEST_TOP_N > 0)
            {
                LogAlert(2, "AssignCommonPositionToPosition(). ТЕСТ: position_id=" + positionRow.position_id + " [" + normalizedName + "] -> position_common_id=" + commonPositionId);
            }
        }
    }
    LogAlert(1, "AssignCommonPositionToPosition(). КОНЕЦ");
}
 
/*
 * Точка входа агента. Читает должности, реально занятые действующими сотрудниками,
 * дополняет справочник типовых должностей недостающими записями и проставляет
 * каждой должности ссылку на её типовую должность.
 * @returns {void}
 */
function Run()
{
    LogAlert(2, "Run(). НАЧАЛО");
    try
    {
        var positionRows, positionNames, i, normalizedName, commonPositionPairs, positionRow, processedCount;
        positionRows = GetActivePositionsInUse();
        positionNames = [];
        for (i = 0; i < positionRows.length; i++)
        {
            try
            {
                normalizedName = NormalizePositionName(positionRows[i].position_name);
                if (normalizedName != "" && !ArrayContainsString(positionNames, normalizedName))
                {
                    positionNames.push(normalizedName);
                }
            }
            catch (normalizeError)
            {
                LogAlert(3, "Run(). Не удалось нормализовать должность ID=" + positionRows[i].position_id + " [" + positionRows[i].position_name + "]: " + normalizeError.message);
            }
        }
        commonPositionPairs = BuildCommonPositionPairs(GetExistingCommonPositionNames());
        commonPositionPairs = EnsureCommonPositionsExist(positionNames, commonPositionPairs);
        processedCount = 0;
        for (i = 0; i < positionRows.length; i++)
        {
            positionRow = positionRows[i];
            try
            {
                AssignCommonPositionToPosition(positionRow, commonPositionPairs);
            }
            catch (positionError)
            {
                LogAlert(3, "Run(). Не удалось проставить типовую должность для позиции ID=" + positionRow.position_id + ": " + positionError.message);
            }
            processedCount = processedCount + 1;
            if (processedCount % PROGRESS_LOG_STEP == 0)
            {
                LogAlert(2, "Run(). Прогресс: обработано " + processedCount + " из " + positionRows.length);
            }
        }
        LogAlert(2, "Run(). Обработка завершена: всего должностей " + positionRows.length + ", уникальных названий " + positionNames.length);
    }
    catch (error)
    {
        LogAlert(4, "Run(). КРИТИЧЕСКАЯ ОШИБКА:\n" + error);
    }
    LogAlert(2, "Run(). КОНЕЦ");
}
 
//-------------------------------------------------------------------------
//              Область основного кода
//-------------------------------------------------------------------------
 
Run();
 
