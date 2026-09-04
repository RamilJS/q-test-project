

var TEST_MATRIX_ID = 7681616865394751377; // id записи cc_learning_matrice "Матрица тест" (из диагностики 04.09.2026)

/*
 * ТЕСТОВАЯ версия LogAlert -- печатает через alert() вместо реального журнала логов.
 * @param {number} typeLog     -   Уровень логов: 1-DEBUG, 2-INFO, 3-WARN, 4-ERROR.
 * @param {string} message     -   Сообщение для логов.
 * @returns {void}
 */
function LogAlert(typeLog, message)
{
    var prefix;
    prefix = "[?]";
    if (typeLog == 1) { prefix = "[DEBUG]"; }
    else if (typeLog == 2) { prefix = "[INFO]"; }
    else if (typeLog == 3) { prefix = "[WARN]"; }
    else if (typeLog == 4) { prefix = "[ERROR]"; }
    alert(prefix + " " + message);
}

/*
 * Читает параметр удалённого действия, при отсутствии возвращает значение по умолчанию.
 * Не используется в тестовом прогоне (matrix_id захардкожен), оставлена для полноты
 * копии логики из боевого файла.
 * @param {string} paramName        -   Имя параметра (см. PARAMETERS).
 * @param {string} defaultValue     -   Значение по умолчанию.
 * @returns {string}
 */
function GetParam(paramName, defaultValue)
{
    var paramValue;
    paramValue = PARAMETERS.GetOptProperty(paramName);
    if (defaultValue != undefined && (paramValue == undefined || paramValue == ""))
    {
        paramValue = defaultValue;
    }
    return paramValue;
}

/*
 * Находит все записи cc_learning_matrice с указанным названием.
 * @param {string} matrixName   -   Название матрицы.
 * @returns {Object[]}          -   Массив документов cc_learning_matrice.
 */
function GetMatrixRows(matrixName)
{
    LogAlert(1, "GetMatrixRows(). НАЧАЛО. matrixName=" + matrixName);
    var matrixRows;
    matrixRows = ArraySelectAll(XQuery("for $elem in cc_learning_matrices where $elem/name = " + XQueryLiteral(matrixName) + " return $elem"));
    LogAlert(1, "GetMatrixRows(). Найдено записей: " + ArrayCount(matrixRows));
    LogAlert(1, "GetMatrixRows(). КОНЕЦ");
    return matrixRows;
}

/*
 * Находит активные элементы (программы) для указанных записей матрицы.
 * @param {number[]} matrixIds  -   ID записей cc_learning_matrice.
 * @returns {Object[]}          -   Массив документов cc_learning_matrice_element.
 */
function GetMatrixElementRows(matrixIds)
{
    LogAlert(1, "GetMatrixElementRows(). НАЧАЛО");
    var elementRows;
    elementRows = ArraySelectAll(XQuery("for $elem in cc_learning_matrice_elements where MatchSome($elem/cc_learning_matrice_id, (" + ArrayMerge(matrixIds, "This", ",") + ")) and $elem/is_active=true() return $elem"));
    LogAlert(1, "GetMatrixElementRows(). Найдено элементов: " + ArrayCount(elementRows));
    LogAlert(1, "GetMatrixElementRows(). КОНЕЦ");
    return elementRows;
}

/*
 * Собирает уникальный список ID программ (education_method) -- и с самой матрицы,
 * и с её элементов.
 * @param {Object[]} matrixRows     -   Документы cc_learning_matrice.
 * @param {Object[]} elementRows    -   Документы cc_learning_matrice_element.
 * @returns {number[]}
 */
function GetProgramIds(matrixRows, elementRows)
{
    LogAlert(1, "GetProgramIds(). НАЧАЛО");
    var matrixProgramIds, elementProgramIds, allProgramIds, programIds, i;
    matrixProgramIds = ArrayExtract(matrixRows, "Int(This.education_method_id)");
    elementProgramIds = ArrayExtract(elementRows, "Int(This.education_method_id)");
    allProgramIds = [];
    for (i = 0; i < ArrayCount(matrixProgramIds); i++)
    {
        allProgramIds.push(matrixProgramIds[i]);
    }
    for (i = 0; i < ArrayCount(elementProgramIds); i++)
    {
        allProgramIds.push(elementProgramIds[i]);
    }
    programIds = ArraySelectDistinct(allProgramIds, "This");
    LogAlert(1, "GetProgramIds(). Уникальных программ: " + ArrayCount(programIds));
    LogAlert(1, "GetProgramIds(). КОНЕЦ");
    return programIds;
}

/*
 * Строит список колонок отчёта -- по одной на программу обучения.
 * @param {number[]} programIds     -   ID программ (education_method).
 * @returns {Object[]}              -   Массив { id, title }.
 */
function GetProgramColumns(programIds)
{
    LogAlert(1, "GetProgramColumns(). НАЧАЛО");
    var columns, i, programID, educationMethodDoc;
    columns = [];
    for (i = 0; i < ArrayCount(programIds); i++)
    {
        programID = programIds[i];
        educationMethodDoc = tools.open_doc(programID).TopElem;
        columns.push({ id: String(programID), title: String(educationMethodDoc.name) });
    }
    LogAlert(1, "GetProgramColumns(). КОНЕЦ");
    return columns;
}

/*
 * Читает всех действующих сотрудников.
 * @returns {Object[]}
 */
function GetActiveCollaboratorRows()
{
    LogAlert(1, "GetActiveCollaboratorRows(). НАЧАЛО");
    var collaboratorRows;
    collaboratorRows = ArraySelectAll(XQuery("for $elem in collaborators where $elem/is_dismiss=false() return $elem"));
    LogAlert(1, "GetActiveCollaboratorRows(). Найдено сотрудников: " + ArrayCount(collaboratorRows));
    LogAlert(1, "GetActiveCollaboratorRows(). КОНЕЦ");
    return collaboratorRows;
}

/*
 * Достаёт макрорегион (custom_elem f_2ewj) по всем действующим сотрудникам одним SQL-запросом.
 * @returns {Object[]}      -   Массив { id, macroregion }.
 */
function GetMacroregionRows()
{
    LogAlert(1, "GetMacroregionRows(). НАЧАЛО");
    var sqlText, macroRows;
    sqlText = "";
    sqlText = sqlText + "select cs.id,\r\n";
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_2ewj'']/value)[1]', 'varchar(max)') as macroregion\r\n";
    sqlText = sqlText + "from collaborators cs\r\n";
    sqlText = sqlText + "inner join collaborator c on c.id = cs.id\r\n";
    sqlText = sqlText + "where cs.is_dismiss != 1";
    macroRows = ArraySelectAll(XQuery("sql:" + sqlText));
    LogAlert(1, "GetMacroregionRows(). Строк: " + ArrayCount(macroRows));
    LogAlert(1, "GetMacroregionRows(). КОНЕЦ");
    return macroRows;
}

/*
 * Находит минимальную дату прохождения (start_date мероприятия) по каждому сотруднику и программе.
 * ВАЖНО: фильтр по ec.is_collaborator НЕ используется -- подтверждено диагностикой, что это
 * поле в реальных данных всегда NULL, из-за чего "= 1" отсекал вообще все строки.
 * @param {number[]} programIds     -   ID программ (education_method).
 * @returns {Object[]}              -   Массив { collaborator_id, education_method_id, first_date }.
 */
function GetCompletionDateRows(programIds)
{
    LogAlert(1, "GetCompletionDateRows(). НАЧАЛО");
    var sqlText, dateRows;
    sqlText = "";
    sqlText = sqlText + "select ec.collaborator_id, e.education_method_id, min(ec.start_date) as first_date\r\n";
    sqlText = sqlText + "from event_collaborators ec\r\n";
    sqlText = sqlText + "join events e on e.id = ec.event_id\r\n";
    sqlText = sqlText + "where e.education_method_id in (" + ArrayMerge(programIds, "This", ",") + ")\r\n";
    sqlText = sqlText + "group by ec.collaborator_id, e.education_method_id";
    dateRows = ArraySelectAll(XQuery("sql:" + sqlText));
    LogAlert(1, "GetCompletionDateRows(). Строк: " + ArrayCount(dateRows));
    LogAlert(1, "GetCompletionDateRows(). КОНЕЦ");
    return dateRows;
}

/*
 * Ищет дату прохождения конкретного сотрудника по конкретной программе.
 * @param {Object[]} dateRows       -   Результат GetCompletionDateRows().
 * @param {number} collaboratorID   -   ID сотрудника.
 * @param {number} programID        -   ID программы.
 * @returns {string}
 */
function FindCompletionDate(dateRows, collaboratorID, programID)
{
    var dateRow;
    dateRow = ArrayOptFind(dateRows, "Int(This.collaborator_id) == Int(collaboratorID) && Int(This.education_method_id) == Int(programID)");
    return (dateRow != undefined ? StrDate(Date(dateRow.first_date), false) : "");
}

/*
 * Ищет макрорегион конкретного сотрудника.
 * @param {Object[]} macroRows      -   Результат GetMacroregionRows().
 * @param {number} collaboratorID   -   ID сотрудника.
 * @returns {string}
 */
function FindMacroregion(macroRows, collaboratorID)
{
    var macroRow;
    macroRow = ArrayOptFind(macroRows, "Int(This.id) == Int(collaboratorID)");
    return (macroRow != undefined && macroRow.macroregion != undefined ? String(macroRow.macroregion) : "");
}

/*
 * Собирает одну строку отчёта для сотрудника: 4 обязательных поля + дата по каждой программе.
 * @param {Object} collaborator     -   Документ сотрудника.
 * @param {Object[]} macroRows      -   Результат GetMacroregionRows().
 * @param {Object[]} dateRows       -   Результат GetCompletionDateRows().
 * @param {number[]} programIds     -   ID программ (education_method).
 * @returns {Object}
 */
function BuildReportRow(collaborator, macroRows, dateRows, programIds)
{
    var row, i, programID;
    row = new Object();
    row.fullname = String(collaborator.fullname);
    row.position_name = String(collaborator.position_name);
    row.subdivision_name = String(collaborator.position_parent_name);
    row.macroregion = FindMacroregion(macroRows, Int(collaborator.id));
    row.dates = new Object();
    for (i = 0; i < ArrayCount(programIds); i++)
    {
        programID = programIds[i];
        row.dates.SetProperty(String(programID), FindCompletionDate(dateRows, Int(collaborator.id), programID));
    }
    return row;
}

/*
 * Резолвит выбранную матрицу (matrix_id) в список ID программ обучения.
 * @param {number} matrixId     -   ID записи cc_learning_matrice.
 * @returns {number[]}          -   ID программ (education_method).
 */
function ResolveProgramIds(matrixId)
{
    LogAlert(1, "ResolveProgramIds(). НАЧАЛО. matrixId=" + matrixId);
    var matrixDoc, matrixName, matrixRows, matrixIds, elementRows, programIds;

    matrixDoc = tools.open_doc(matrixId).TopElem;
    matrixName = String(matrixDoc.name);

    matrixRows = GetMatrixRows(matrixName);
    matrixIds = ArrayExtract(matrixRows, "Int(This.id)");
    if (ArrayCount(matrixIds) == 0)
    {
        throw ("Не найдено ни одной записи cc_learning_matrice с названием [" + matrixName + "]");
    }

    elementRows = GetMatrixElementRows(matrixIds);
    programIds = GetProgramIds(matrixRows, elementRows);
    if (ArrayCount(programIds) == 0)
    {
        throw ("У матрицы [" + matrixName + "] не найдено ни одной активной программы (cc_learning_matrice / cc_learning_matrice_element)");
    }

    LogAlert(1, "ResolveProgramIds(). КОНЕЦ");
    return programIds;
}

/*
 * Печатает одну строку RESULT.rows целиком, включая дату по каждой колонке-программе.
 * @param {Object} result       -   RESULT из Run().
 * @param {number} rowIndex     -   Индекс строки в result.rows.
 * @param {string} label        -   Префикс для строки лога.
 * @returns {void}
 */
function DumpReportRow(result, rowIndex, label)
{
    var row, j, col, dateValue;
    row = result.rows[rowIndex];
    LogAlert(2, "  " + label + "[" + rowIndex + "]: " + row.fullname + " | " + row.position_name + " | " + row.subdivision_name + " | макрорегион=" + row.macroregion);
    for (j = 0; j < ArrayCount(result.columns); j++)
    {
        col = result.columns[j];
        dateValue = row.dates.GetProperty(col.id);
        LogAlert(2, "      [" + col.title + "] = " + dateValue);
    }
}

/*
 * Печатает итоговый RESULT через alert() -- колонки, первые несколько строк подряд,
 * и отдельно -- первые несколько строк, где хотя бы одна дата реально заполнена
 * (чтобы визуально свериться, что даты подтягиваются к нужным сотрудникам).
 * @param {Object} result       -   RESULT из Run().
 * @returns {void}
 */
function DumpResult(result)
{
    var i, j, col, row, dateValue, foundWithDate, printedCount;

    LogAlert(2, "DumpResult(). Колонок: " + ArrayCount(result.columns) + ", строк: " + ArrayCount(result.rows));
    for (i = 0; i < ArrayCount(result.columns); i++)
    {
        col = result.columns[i];
        LogAlert(2, "  column[" + i + "]: id=" + col.id + " title=" + col.title);
    }

    LogAlert(2, "DumpResult(). Первые 5 строк подряд:");
    for (i = 0; i < ArrayCount(result.rows) && i < 5; i++)
    {
        DumpReportRow(result, i, "row");
    }

    LogAlert(2, "DumpResult(). Строки с заполненной датой (до 5):");
    printedCount = 0;
    for (i = 0; i < ArrayCount(result.rows) && printedCount < 5; i++)
    {
        row = result.rows[i];
        foundWithDate = false;
        for (j = 0; j < ArrayCount(result.columns); j++)
        {
            col = result.columns[j];
            dateValue = row.dates.GetProperty(col.id);
            if (dateValue != undefined && dateValue != "")
            {
                foundWithDate = true;
            }
        }
        if (foundWithDate)
        {
            DumpReportRow(result, i, "row_with_date");
            printedCount++;
        }
    }
    if (printedCount == 0)
    {
        LogAlert(3, "DumpResult(). Ни одной строки с заполненной датой не найдено!");
    }
}

/*
 * ТЕСТОВЫЙ прогон: matrix_id захардкожен в TEST_MATRIX_ID, результат печатается через alert().
 * @returns {void}
 */
function Run()
{
    LogAlert(2, "Run(). НАЧАЛО (ТЕСТОВЫЙ ПРОГОН, matrixId захардкожен)");
    var matrixId, programIds, collaboratorRows, macroRows, dateRows, i;

    ERROR = 0;
    MESSAGE = "";
    RESULT = new Object();
    RESULT.columns = [];
    RESULT.rows = [];

    try
    {
        matrixId = TEST_MATRIX_ID;
        LogAlert(1, "Run(). matrixId=" + matrixId);

        programIds = ResolveProgramIds(matrixId);

        RESULT.columns = GetProgramColumns(programIds);
        collaboratorRows = GetActiveCollaboratorRows();
        macroRows = GetMacroregionRows();
        dateRows = GetCompletionDateRows(programIds);

        for (i = 0; i < ArrayCount(collaboratorRows); i++)
        {
            RESULT.rows.push(BuildReportRow(collaboratorRows[i], macroRows, dateRows, programIds));
        }

        LogAlert(2, "Run(). Готово. Сотрудников: " + ArrayCount(RESULT.rows) + ", программ: " + ArrayCount(programIds));
        DumpResult(RESULT);
    }
    catch (_ex)
    {
        ERROR = 1;
        MESSAGE = ExtractUserError(_ex);
        LogAlert(4, "Run(). ОШИБКА: " + MESSAGE);
    }
    LogAlert(2, "Run(). ERROR=" + ERROR + " MESSAGE=" + MESSAGE);
    LogAlert(2, "Run(). КОНЕЦ");
}

Run();
