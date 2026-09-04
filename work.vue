// HREDU-181. Восток_полный_список -- удалённое действие для выборки данных отчёта.
// Черновик: БЕЗ фильтра по видимости (подчинённость/HR) -- пока отдаёт всех активных
// сотрудников, без фильтра по position_common_id/mir_code_id самой матрицы.
//
// Параметр удалённого действия: matrix_id -- id одной из записей cc_learning_matrice.
//
// ПОДТВЕРЖДЕНО диагностикой (HREDU-181_diagnostic_learning_matrice_names.js, прогон
// пользователем 04.09.2026): имена коллекций 'cc_learning_matrices' и
// 'cc_learning_matrice_elements' -- верные, поля совпадают с ожиданиями из HREDU-184/180.
//
// ПОДТВЕРЖДЕНО ПОЛНОСТЬЮ end-to-end тестовым прогоном (HREDU-181_test_run_vostok_polny_spisok.js,
// пользователь, 04.09.2026) на двух реальных матрицах ("Матрица тест" / программа "Ассесcмент";
// "Матрица тест новый" / программа "Основы лизинга", 1824 совпадения дат из 2312 сотрудников,
// даты и ФИО визуально сверены и совпадают с ожиданиями). По пути найден и исправлен реальный
// баг: фильтр ec.is_collaborator = 1 в GetCompletionDateRows() отсекал вообще все строки, т.к.
// это поле в реальных данных всегда NULL -- убран (см. комментарий у самой функции).
//
// ВСЁ ЕЩЁ ОТКРЫТО:
//   1. Логика "матрица размножена на несколько записей с одинаковым name" (одна запись
//      cc_learning_matrice на комбинацию должность/мир-код) -- предположение из HREDU-181/184,
//      тимлидом напрямую не подтверждено, проверить нечем (реальные матрицы заведёт
//      администратор позже). НЕ БЛОКИРУЕТ: GetMatrixRows()/GetProgramIds() ниже работают
//      одинаково корректно в обоих случаях -- берём ВСЕ записи с данным name и объединяем
//      программы со всех найденных записей и их элементов.
//   2. У самой записи cc_learning_matrice есть собственное поле education_method_id (помимо
//      того, что программы также приходят через её cc_learning_matrice_element). В тестовой
//      записи оно совпало со значением у единственного элемента -- похоже на "зеркалирование",
//      но на одной записи не доказать. Поэтому GetProgramIds() берёт программы И с матрицы,
//      И с элементов, с дедупликацией -- не портит, если предположение верно, подстраховывает,
//      если нет.
//   3. Нужно уточнить у тимлида: похоже, что position_common_id и mir_code_id на
//      cc_learning_matrice -- это условие "к каким сотрудникам применяется матрица" (по
//      аналогии со старой compound_program и её f_position_names/f_mir_code). Если так, то
//      GetActiveCollaboratorRows() должна фильтровать по position.position_common_id
//      (см. HREDU-174) и мир-коду через cc_collaborator_mircode (см. HREDU-176/178/179), а не
//      отдавать вообще всех активных, как сейчас. Отдельный вопрос от видимости по
//      подчинённости/HR -- этот про то, какие строки вообще должны попадать в отчёт.
//
// TODO при заведении документа remote_action в админке: заполнить LOG_NAME и CUR_OBJECT_ID
// ниже реальными значениями (Сервис >> Показать в XML / Копировать ID документа).

//-------------------------------------------------------------------------
//              Область констант
//-------------------------------------------------------------------------

DEBUG = false;              // Включает подробные логи уровня 1 [DEBUG] -- на проде и в репозитории должно быть false
LOG_NAME = "agent";         // TODO: уточнить после создания документа в админке -- пока по аналогии с серверными агентами
CUR_OBJECT_ID = 0;          // TODO: заполнить ID документа remote_action после его создания в админке

//-------------------------------------------------------------------------
//              Область функций
//-------------------------------------------------------------------------

/*
 * Управляет логированием. Уровни: 1-[DEBUG], 2-[INFO], 3-[WARN], 4-[ERROR].
 * @param {number} typeLog     -   Уровень логов.
 * @param {string} message     -   Сообщение для логов.
 * @returns {void}
 */
function LogAlert(typeLog, message)
{
    tools.call_code_library_method("vtbl_log_lib", "LogAlert", [LOG_NAME, typeLog, CUR_OBJECT_ID, message, DEBUG]);
}

/*
 * Читает параметр удалённого действия, при отсутствии возвращает значение по умолчанию.
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
 * Находит все записи cc_learning_matrice с указанным названием (см. открытый вопрос №1 в шапке файла).
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
 * и с её элементов (см. открытый вопрос №2 в шапке файла).
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
 * Читает всех действующих сотрудников. ПОКА без фильтра по подчинённости/HR и без
 * фильтра по position_common_id/mir_code_id матрицы -- см. открытый вопрос №3 в шапке файла.
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
 * Достаёт макрорегион (custom_elem f_2ewj) по всем действующим сотрудникам одним SQL-запросом
 * (по образцу живого настраиваемого отчёта "Отчет проверки незаполненых полей для матриц обучения").
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
 * Находит минимальную дату прохождения (start_date мероприятия) по каждому сотруднику
 * и программе. Без фильтра по статусу мероприятия -- по указанию тимлида, не усложняем.
 * ВАЖНО: фильтр по ec.is_collaborator НЕ используется -- диагностикой (тестовый прогон
 * 04.09.2026, программа 6499763079148161049) подтверждено, что это поле в реальных данных
 * всегда NULL (27 из 27 строк), из-за чего "= 1" отсекал вообще все строки. Сам факт
 * наличия строки в event_collaborators уже означает участие сотрудника в мероприятии.
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
 * @returns {string}                -   Дата в виде строки или "" если не найдена.
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
 * @param {Object} collaborator     -   Документ сотрудника (из GetActiveCollaboratorRows()).
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
 * Резолвит выбранную матрицу (matrix_id) в список ID программ обучения: находит все
 * записи cc_learning_matrice с тем же названием и объединяет программы с них и их
 * элементов (см. открытые вопросы №1-2 в шапке файла). Бросает исключение, если матрица
 * или её программы не найдены.
 * @param {number} matrixId     -   ID записи cc_learning_matrice, выбранной пользователем.
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
 * Точка входа удалённого действия. Собирает данные отчёта "Восток_полный_список" по
 * выбранной матрице обучения: список программ-колонок и строки сотрудников с датами
 * прохождения. ПОКА без фильтра по видимости (подчинённость/HR) и без фильтра сотрудников
 * по position_common_id/mir_code_id матрицы -- см. открытые вопросы в шапке файла.
 * @returns {void}
 */
function Run()
{
    LogAlert(2, "Run(). НАЧАЛО");
    var matrixId, programIds, collaboratorRows, macroRows, dateRows, i;

    ERROR = 0;
    MESSAGE = "";
    RESULT = new Object();
    RESULT.columns = [];
    RESULT.rows = [];

    try
    {
        matrixId = OptInt(GetParam("matrix_id", "0"), 0);
        LogAlert(1, "Run(). matrixId=" + matrixId);

        if (matrixId == 0)
        {
            throw ("Не передан matrix_id -- выбранная пользователем матрица обучения");
        }

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
