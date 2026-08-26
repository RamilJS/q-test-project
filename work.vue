
// HREDU-174. Откат: удаление кастомного поля f_position_common_id у сотрудников.
// Требования тимлида изменились -- связь с типовой должностью должна храниться
// не на collaborator (в custom_elems), а на position (в штатном поле position_common_id).
// Этот скрипт нужен один раз, чтобы убрать то, что было проставлено по старой схеме.

//-------------------------------------------------------------------------
//              Область констант
//-------------------------------------------------------------------------

DEBUG = false;              // Включает подробные логи уровня 1 [DEBUG] -- на проде и в репозитории должно быть false
LOG_NAME = "agent";         // Системный лог для агентов сервера
CUR_OBJECT_ID = oData.id;   // ID текущего документа агента
TEST_TOP_N = 0;             // Ограничение выборки сотрудников для тестового прогона. 0 = без ограничения
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
    tools.call_code_library_method("vtbl_log_lib", "LogAlert", [LOG_NAME, typeLog, CUR_OBJECT_ID, message, DEBUG]);
}

/*
 * Читает из БД тех же действующих сотрудников, на которых ранее (по старой схеме)
 * проставлялось custom_elems.f_position_common_id. Если TEST_TOP_N > 0, ограничивает
 * выборку первыми N записями (для тестового прогона).
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
 * Удаляет у сотрудника кастомное поле f_position_common_id, если оно есть
 * (по образцу, присланному тимлидом). Ничего не делает, если поля нет.
 * BindToDb() не вызывается -- документ уже открыт через tools.open_doc()
 * и уже привязан к БД.
 * @param {Object} collaboratorRow     -   Строка { id, position_name } сотрудника.
 * @returns {void}
 */
function RemoveCommonPositionCustomElem(collaboratorRow)
{
    LogAlert(1, "RemoveCommonPositionCustomElem(). НАЧАЛО");
    var oDoc;
    oDoc = tools.open_doc(collaboratorRow.id);
    oDoc.TopElem.custom_elems.DeleteChildren("This.name=='f_position_common_id'");
    oDoc.Save();
    LogAlert(1, "RemoveCommonPositionCustomElem(). КОНЕЦ");
}

/*
 * Точка входа агента. Проходит по действующим сотрудникам и удаляет у каждого
 * кастомное поле f_position_common_id, если оно было проставлено ранее.
 * @returns {void}
 */
function Run()
{
    LogAlert(2, "Run(). НАЧАЛО");
    try
    {
        var collaboratorRows, i, collaboratorRow, processedCount;
        collaboratorRows = GetActiveCollaboratorPositions();
        processedCount = 0;
        for (i = 0; i < collaboratorRows.length; i++)
        {
            collaboratorRow = collaboratorRows[i];
            try
            {
                RemoveCommonPositionCustomElem(collaboratorRow);
            }
            catch (employeeError)
            {
                LogAlert(3, "Run(). Не удалось удалить f_position_common_id у сотрудника ID=" + collaboratorRow.id + ": " + employeeError.message);
            }
            processedCount = processedCount + 1;
            if (processedCount % PROGRESS_LOG_STEP == 0)
            {
                LogAlert(2, "Run(). Прогресс: обработано " + processedCount + " из " + collaboratorRows.length);
            }
        }
        LogAlert(2, "Run(). Обработка завершена: всего сотрудников " + collaboratorRows.length);
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
