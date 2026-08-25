DEBUG = false;
LOG_NAME = "agent";
CUR_OBJECT_ID = 0;
// ОБЛАСТЬ ФУНКЦИЙ
EnableLog('ramil-agent-debug', true);
function alert(_string) {
    LogEvent('ramil-agent-debug', _string);
    return _string;
}
function LogAlert(typeLog, message)
{
    tools.call_code_library_method("vtbl_common_lib", "LogAlert", [LOG_NAME, typeLog, CUR_OBJECT_ID, message, DEBUG]);
}
function NormalizePositionName(rawName)
{
    var safeRawName;
    safeRawName = "" + rawName;
    if (safeRawName == "undefined")
    {
        safeRawName = "";
    }
    if (safeRawName == "null")
    {
        safeRawName = "";
    }
    return Trim(UnifySpaces(safeRawName));
}
function ArrayContainsString(values, target)
{
    var i;
    for (i = 0; i < values.length; i++)
    {
        if (values[i] == target)
        {
            return true;
        }
    }
    return false;
}
function FindPositionIdByName(commonPositionPairs, name)
{
    var i;
    for (i = 0; i < commonPositionPairs.length; i++)
    {
        if (commonPositionPairs[i].name == name)
        {
            return commonPositionPairs[i].id;
        }
    }
    return null;
}
function GetActiveCollaboratorPositions()
{
    LogAlert(1, "GetActiveCollaboratorPositions(). НАЧАЛО");
    alert("GetActiveCollaboratorPositions(). НАЧАЛО");
    var sqlText, rawRows, collaboratorRows, row;
    sqlText = "";
    sqlText = sqlText + "select cs.id, cs.position_name\r\n";
    sqlText = sqlText + "from collaborators cs\r\n";
    sqlText = sqlText + "where cs.is_dismiss = 0\r\n";
    sqlText = sqlText + "  and cs.position_name is not null\r\n";
    sqlText = sqlText + "  and ltrim(rtrim(cs.position_name)) != ''\r\n";
    alert("sqlText = " + sqlText);
    rawRows = XQuery("sql:" + sqlText);
    collaboratorRows = [];
    for (row in rawRows)
    {
        collaboratorRows.push(row);
    }
    alert("GetActiveCollaboratorPositions(). Найдено сотрудников: " + collaboratorRows.length);
    LogAlert(1, "GetActiveCollaboratorPositions(). Найдено сотрудников: " + collaboratorRows.length);
    LogAlert(1, "GetActiveCollaboratorPositions(). КОНЕЦ");
    alert("GetActiveCollaboratorPositions(). КОНЕЦ");
    return collaboratorRows;
}
function GetExistingCommonPositionNames()
{
    LogAlert(1, "GetExistingCommonPositionNames(). НАЧАЛО");
    alert("GetExistingCommonPositionNames(). НАЧАЛО");
    var sqlText, rawRows, commonPositionRows, row;
    sqlText = "";
    sqlText = sqlText + "select pc.id, pc.name\r\n";
    sqlText = sqlText + "from position_commons pc\r\n";
    alert("sqlText = " + sqlText);
    rawRows = XQuery("sql:" + sqlText);
    commonPositionRows = [];
    for (row in rawRows)
    {
        commonPositionRows.push(row);
    }
    alert("GetExistingCommonPositionNames(). Найдено типовых должностей: " + commonPositionRows.length);
    LogAlert(1, "GetExistingCommonPositionNames(). Найдено типовых должностей: " + commonPositionRows.length);
    LogAlert(1, "GetExistingCommonPositionNames(). КОНЕЦ");
    alert("GetExistingCommonPositionNames(). КОНЕЦ");
    return commonPositionRows;
}
function BuildCommonPositionPairs(commonPositionRows)
{
    var pairs, i, row, normalizedName;
    pairs = [];
    for (i = 0; i < commonPositionRows.length; i++)
    {
        row = commonPositionRows[i];
        normalizedName = NormalizePositionName(row.name);
        pairs.push({ name: normalizedName, id: row.id });
    }
    return pairs;
}
function CreateCommonPosition(name)
{
    LogAlert(1, "CreateCommonPosition(). НАЧАЛО");
    alert("CreateCommonPosition(). НАЧАЛО: " + name);
    var oNewDoc, oNewDocTE;
    oNewDoc = tools.new_doc_by_name("position_common");
    oNewDocTE = oNewDoc.TopElem;
    oNewDocTE.name = name;
    oNewDoc.BindToDb(DefaultDb);
    oNewDoc.Save();
    LogAlert(1, "CreateCommonPosition(). Создана типовая должность: " + name);
    LogAlert(1, "CreateCommonPosition(). КОНЕЦ");
    alert("CreateCommonPosition(). КОНЕЦ");
    return oNewDocTE;
}
function EnsureCommonPositionsExist(positionNames, commonPositionPairs)
{
    LogAlert(1, "EnsureCommonPositionsExist(). НАЧАЛО");
    alert("EnsureCommonPositionsExist(). НАЧАЛО");
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
    alert("EnsureCommonPositionsExist(). КОНЕЦ");
    return commonPositionPairs;
}
function AssignCommonPositionToEmployee(collaboratorRow, commonPositionPairs)
{
    LogAlert(1, "AssignCommonPositionToEmployee(). НАЧАЛО");
    alert("AssignCommonPositionToEmployee(). НАЧАЛО: id=" + collaboratorRow.id);
    var normalizedName, commonPositionId, oDoc;
    normalizedName = NormalizePositionName(collaboratorRow.position_name);
    commonPositionId = FindPositionIdByName(commonPositionPairs, normalizedName);
    oDoc = tools.open_doc(collaboratorRow.id);
    // TODO: подтвердить реальное имя поля-связи с типовой должностью на collaborators
    oDoc.TopElem.position_common_id = commonPositionId;
    oDoc.Save();
    LogAlert(1, "AssignCommonPositionToEmployee(). КОНЕЦ");
    alert("AssignCommonPositionToEmployee(). КОНЕЦ");
}
function Run()
{
    LogAlert(2, "Run(). НАЧАЛО");
    alert("Run(). НАЧАЛО");
    try
    {
        var collaboratorRows, positionNames, i, normalizedName, commonPositionPairs, collaboratorRow;
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
                alert("Run(). Не удалось нормализовать должность сотрудника ID=" + collaboratorRows[i].id + " [" + collaboratorRows[i].position_name + "]: " + normalizeError);
            }
        }
        alert("Run(). Уникальных названий должностей: " + positionNames.length);
        commonPositionPairs = BuildCommonPositionPairs(GetExistingCommonPositionNames());
        commonPositionPairs = EnsureCommonPositionsExist(positionNames, commonPositionPairs);
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
                alert("Run(). Не удалось проставить типовую должность сотруднику ID=" + collaboratorRow.id + ": " + employeeError.message);
            }
        }
    }
    catch (error)
    {
        LogAlert(4, "Run(). КРИТИЧЕСКАЯ ОШИБКА:\n" + error);
        alert("Run(). КРИТИЧЕСКАЯ ОШИБКА: " + error);
    }
    LogAlert(2, "Run(). КОНЕЦ");
    alert("Run(). КОНЕЦ");
}
// ОБЛАСТЬ ОСНОВНОГО КОДА
Run();
