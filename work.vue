// HREDU-174. Агент создания и проставления типовых должностей.

//-------------------------------------------------------------------------
//              Область констант
//-------------------------------------------------------------------------

DEBUG = false;              // Включает подробные логи уровня 1 [DEBUG] -- на проде и в репозитории должно быть false
LOG_NAME = "agent";         // Системный лог для агентов сервера (отдельный server_agent не заводим)
CUR_OBJECT_ID = 0;          // TODO: заменить на реальный ID документа агента (Админка > объект агента > Сервис > Копировать ID документа)
TEST_TOP_N = 0;             // Ограничение выборки сотрудников для тестового прогона. 0 = без ограничения (боевой режим)
PROGRESS_LOG_STEP = 100;    // Через сколько обработанных сотрудников писать в лог прогресс выполнения

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
    tools.call_code_library_method("vtbl_common_lib", "LogAlert", [LOG_NAME, typeLog, CUR_OBJECT_ID, message, DEBUG]);
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
 * Читает из БД действующих (не уволенных) сотрудников с непустым названием должности.
 * Если TEST_TOP_N > 0, ограничивает выборку первыми N записями (для тестового прогона).
 * @returns {Object[]}     -   Массив строк { id, position_name }.
 */
function GetActiveCollaboratorPositions()
{
    LogAlert(1, "GetActiveCollaboratorPositions(). НАЧАЛО");
    var topClause, sqlText, rawRows, collaboratorRows, row;
    topClause = "";
    if (TEST_TOP_N > 0)
    {
        topClause = "top " + TEST_TOP_N + " ";
    }
    sqlText = "";
    sqlText = sqlText + "select " + topClause + "cs.id, cs.position_name\r\n";
    sqlText = sqlText + "from collaborators cs\r\n";
    sqlText = sqlText + "where cs.is_dismiss = 0\r\n";
    sqlText = sqlText + "  and cs.position_name is not null\r\n";
    sqlText = sqlText + "  and ltrim(rtrim(cs.position_name)) != ''\r\n";
    rawRows = XQuery("sql:" + sqlText);
    collaboratorRows = [];
    for (row in rawRows)
    {
        collaboratorRows.push(row);
    }
    LogAlert(1, "GetActiveCollaboratorPositions(). Найдено сотрудников: " + collaboratorRows.length);
    LogAlert(1, "GetActiveCollaboratorPositions(). КОНЕЦ");
    return collaboratorRows;
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
 * @param {string[]} positionNames         -   Уникальные нормализованные названия должностей сотрудников.
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
 * Проставляет сотруднику ссылку на его типовую должность.
 * Значение хранится в custom_elems (поле-связь foreign_elem на объекте collaborator
 * отсутствует, заводить его через админку не потребовалось).
 * ВАЖНО: BindToDb() здесь не вызывается -- документ уже открыт через tools.open_doc()
 * и уже привязан к БД; повторный вызов BindToDb() создаёт дубликат документа вместо
 * обновления существующего (проверено на практике).
 * @param {Object} collaboratorRow         -   Строка { id, position_name } действующего сотрудника.
 * @param {Object[]} commonPositionPairs   -   Список пар { name, id } типовых должностей.
 * @returns {void}
 */
function AssignCommonPositionToEmployee(collaboratorRow, commonPositionPairs)
{
    LogAlert(1, "AssignCommonPositionToEmployee(). НАЧАЛО");
    var normalizedName, commonPositionId, oDoc;
    normalizedName = NormalizePositionName(collaboratorRow.position_name);
    commonPositionId = FindPositionIdByName(commonPositionPairs, normalizedName);
    if (commonPositionId == null)
    {
        LogAlert(3, "AssignCommonPositionToEmployee(). Не найден commonPositionId для сотрудника ID=" + collaboratorRow.id + " [" + normalizedName + "], пропуск");
    }
    else
    {
        oDoc = tools.open_doc(collaboratorRow.id);
        oDoc.TopElem.custom_elems.ObtainChildByKey("f_position_common_id").value = String(commonPositionId);
        oDoc.Save();
    }
    LogAlert(1, "AssignCommonPositionToEmployee(). КОНЕЦ");
}

/*
 * Точка входа агента. Читает действующих сотрудников и их должности, дополняет
 * справочник типовых должностей недостающими записями и проставляет каждому
 * сотруднику ссылку на его типовую должность.
 * @returns {void}
 */
function Run()
{
    LogAlert(2, "Run(). НАЧАЛО");
    try
    {
        var collaboratorRows, positionNames, i, normalizedName, commonPositionPairs, collaboratorRow, processedCount;
        collaboratorRows = GetActiveCollaboratorPositions();
        positionNames = [];
        for (i = 0; i < collaboratorRows.length; i++)
        {
            try
            {
                normalizedName = NormalizePositionName(collaboratorRows[i].position_name);
                if (normalizedName != "" && !ArrayContainsString(positionNames, normalizedName))
                {
                    positionNames.push(normalizedName);
                }
            }
            catch (normalizeError)
            {
                LogAlert(3, "Run(). Не удалось нормализовать должность сотрудника ID=" + collaboratorRows[i].id + " [" + collaboratorRows[i].position_name + "]: " + normalizeError.message);
            }
        }
        commonPositionPairs = BuildCommonPositionPairs(GetExistingCommonPositionNames());
        commonPositionPairs = EnsureCommonPositionsExist(positionNames, commonPositionPairs);
        processedCount = 0;
        for (i = 0; i < collaboratorRows.length; i++)
        {
            collaboratorRow = collaboratorRows[i];
            try
            {
                AssignCommonPositionToEmployee(collaboratorRow, commonPositionPairs);
            }
            catch (employeeError)
            {
                LogAlert(3, "Run(). Не удалось проставить типовую должность сотруднику ID=" + collaboratorRow.id + ": " + employeeError.message);
            }
            processedCount = processedCount + 1;
            if (processedCount % PROGRESS_LOG_STEP == 0)
            {
                LogAlert(2, "Run(). Прогресс: обработано " + processedCount + " из " + collaboratorRows.length);
            }
        }
        LogAlert(2, "Run(). Обработка завершена: всего сотрудников " + collaboratorRows.length + ", уникальных должностей " + positionNames.length);
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
