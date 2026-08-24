DEBUG = false;
LOG_NAME = "agent";
CUR_OBJECT_ID = 0;
NPN_DEBUG_LIMIT = 3;
npnDebugCount = 0;
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
function TrimSpaces(value)
{
    var start, end, result, verbose;
    verbose = npnDebugCount <= NPN_DEBUG_LIMIT;
    if (verbose) { alert("TS start value=[" + value + "]"); }

    try
    {
        start = 0;
        while (start < value.length && value.charAt(start) == " ")
        {
            start = start + 1;
        }
        if (verbose) { alert("TS step1 OK start=" + start); }
    }
    catch (t1)
    {
        alert("TS FAIL step1 (leading loop): " + t1);
        return value;
    }

    try
    {
        end = value.length;
        while (end > start && value.charAt(end - 1) == " ")
        {
            end = end - 1;
        }
        if (verbose) { alert("TS step2 OK end=" + end); }
    }
    catch (t2)
    {
        alert("TS FAIL step2 (trailing loop): " + t2);
        return value;
    }

    try
    {
        result = value.substring(start, end);
        if (verbose) { alert("TS step3 OK result=[" + result + "]"); }
        return result;
    }
    catch (t3)
    {
        alert("TS FAIL step3 (substring): " + t3);
        return value;
    }
}
function NormalizePositionName(rawName)
{
    var value, parts, normalizedName, i, safeRawName, verbose;
    verbose = npnDebugCount <= NPN_DEBUG_LIMIT;
    npnDebugCount = npnDebugCount + 1;
    if (verbose) { alert("NPN start rawName=[" + rawName + "]"); }

    try
    {
        safeRawName = "" + rawName;
        if (safeRawName == "undefined")
        {
            safeRawName = "";
        }
        if (safeRawName == "null")
        {
            safeRawName = "";
        }
        if (verbose) { alert("NPN step1 OK safeRawName=[" + safeRawName + "]"); }
    }
    catch (e1)
    {
        alert("NPN FAIL step1 (string convert): " + e1);
        return "";
    }

    try
    {
        value = TrimSpaces(safeRawName);
        if (verbose) { alert("NPN step2 OK value=[" + value + "]"); }
    }
    catch (e2)
    {
        alert("NPN FAIL step2 (TrimSpaces call): " + e2);
        return "";
    }

    try
    {
        parts = value.split(" ");
        if (verbose) { alert("NPN step3 OK parts.length=" + parts.length); }
    }
    catch (e3)
    {
        alert("NPN FAIL step3 (split): " + e3);
        return "";
    }

    normalizedName = "";
    for (i = 0; i < parts.length; i++)
    {
        try
        {
            if (parts[i] != "")
            {
                if (normalizedName == "")
                {
                    normalizedName = parts[i];
                }
                else
                {
                    normalizedName = normalizedName + " " + parts[i];
                }
            }
        }
        catch (e4)
        {
            alert("NPN FAIL step4 (loop) i=" + i + ": " + e4);
        }
    }
    if (verbose) { alert("NPN result=[" + normalizedName + "]"); }
    return normalizedName;
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
function BuildCommonPositionIdsByName(commonPositionRows)
{
    var commonPositionIdsByName, i, row, normalizedName;
    commonPositionIdsByName = {};
    for (i = 0; i < commonPositionRows.length; i++)
    {
        row = commonPositionRows[i];
        normalizedName = NormalizePositionName(row.name);
        commonPositionIdsByName[normalizedName] = row.id;
    }
    return commonPositionIdsByName;
}
function CreateCommonPosition(name)
{
    LogAlert(1, "CreateCommonPosition(). НАЧАЛО");
    alert("CreateCommonPosition(). НАЧАЛО: " + name);
    var commonPosition;
    // TODO: заменить на реальное создание объекта "Типовая должность" (position_commons)
    commonPosition = CreateObject("position_common", { name: name });
    LogAlert(1, "CreateCommonPosition(). Создана типовая должность: " + name);
    LogAlert(1, "CreateCommonPosition(). КОНЕЦ");
    alert("CreateCommonPosition(). КОНЕЦ");
    return commonPosition;
}
function EnsureCommonPositionsExist(positionNames, existingCommonPositionIdsByName)
{
    LogAlert(1, "EnsureCommonPositionsExist(). НАЧАЛО");
    alert("EnsureCommonPositionsExist(). НАЧАЛО");
    var commonPositionIdsByName, i, name, commonPosition;
    commonPositionIdsByName = existingCommonPositionIdsByName;
    for (i = 0; i < positionNames.length; i++)
    {
        name = positionNames[i];
        if (!commonPositionIdsByName[name])
        {
            commonPosition = CreateCommonPosition(name);
            commonPositionIdsByName[name] = commonPosition.id;
        }
    }
    LogAlert(1, "EnsureCommonPositionsExist(). КОНЕЦ");
    alert("EnsureCommonPositionsExist(). КОНЕЦ");
    return commonPositionIdsByName;
}
function AssignCommonPositionToEmployee(collaboratorRow, commonPositionIdsByName)
{
    LogAlert(1, "AssignCommonPositionToEmployee(). НАЧАЛО");
    alert("AssignCommonPositionToEmployee(). НАЧАЛО: id=" + collaboratorRow.id);
    var normalizedName, commonPositionId;
    normalizedName = NormalizePositionName(collaboratorRow.position_name);
    commonPositionId = commonPositionIdsByName[normalizedName];
    // TODO: заменить на реальное сохранение объекта "Сотрудник" (collaborators),
    // поле-связь пока называется предположительно position_common_id
    SaveObject("collaborator", collaboratorRow.id, { position_common_id: commonPositionId });
    LogAlert(1, "AssignCommonPositionToEmployee(). КОНЕЦ");
    alert("AssignCommonPositionToEmployee(). КОНЕЦ");
}
function Run()
{
    LogAlert(2, "Run(). НАЧАЛО");
    alert("Run(). НАЧАЛО");
    try
    {
        var collaboratorRows, positionNamesSeen, positionNames, i, normalizedName, existingCommonPositionIdsByName, commonPositionIdsByName, collaboratorRow;
        collaboratorRows = GetActiveCollaboratorPositions();
        positionNamesSeen = {};
        positionNames = [];
        for (i = 0; i < collaboratorRows.length; i++)
        {
            try
            {
                normalizedName = NormalizePositionName(collaboratorRows[i].position_name);
                if (normalizedName != "" && !positionNamesSeen[normalizedName])
                {
                    positionNamesSeen[normalizedName] = true;
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
        existingCommonPositionIdsByName = BuildCommonPositionIdsByName(GetExistingCommonPositionNames());
        commonPositionIdsByName = EnsureCommonPositionsExist(positionNames, existingCommonPositionIdsByName);
        for (i = 0; i < collaboratorRows.length; i++)
        {
            collaboratorRow = collaboratorRows[i];
            try
            {
                AssignCommonPositionToEmployee(collaboratorRow, commonPositionIdsByName);
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
