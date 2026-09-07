// HREDU-181. Восток_полный_список -- удалённое действие для выборки данных отчёта.
// Черновик: БЕЗ фильтра по видимости (подчинённость/HR) -- пока отдаёт всех активных
// сотрудников, без фильтра по position_common_id/mir_code_id самой матрицы.
//
// Параметры удалённого действия:
//   matrix_id       -- обязательный, id одной из записей cc_learning_matrice.
//   program_id      -- опциональный, id одной программы (education_method) из числа программ
//                       выбранной матрицы -- сужает отчёт до одной программы вместо всех.
//   macroregion     -- опциональный, точное значение макрорегиона (custom_elem f_2ewj).
//   mir_code        -- опциональный, код мир-кода (например "LASK") -- сотрудник попадает в
//                       отчёт, если этот код есть у него СРЕДИ ЛЮБЫХ его мир-кодов (не только
//                       основного/с наибольшим процентом, см. getMirCodeObject() в примере
//                       education_accept_event_card).
//   position_name   -- опциональный, точное совпадение по названию должности.


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
 * Строит справочник { id, title } по программам обучения (education_method) -- используется
 * как справочник для подстановки человекочитаемого названия программы в поле program_name
 * каждой строки отчёта (длинный формат, см. "ФОРМАТ ОТЧЁТА" в шапке файла). ВАЖНО: это уже не
 * список колонок виджета -- при "длинном" формате колонки статичны и не зависят от программ.
 * @param {number[]} programIds     -   ID программ (education_method).
 * @returns {Object[]}              -   Массив { id, title }.
 */
function GetProgramTitles(programIds)
{
    LogAlert(1, "GetProgramTitles(). НАЧАЛО");
    var titles, i, programID, educationMethodDoc;
    titles = [];
    for (i = 0; i < ArrayCount(programIds); i++)
    {
        programID = programIds[i];
        educationMethodDoc = tools.open_doc(programID).TopElem;
        titles.push({ id: String(programID), title: String(educationMethodDoc.name) });
    }
    LogAlert(1, "GetProgramTitles(). КОНЕЦ");
    return titles;
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
 * Достаёт сырое значение мир-кодов (custom_elem f_mir_codes) по всем действующим сотрудникам
 * одним SQL-запросом -- тот же паттерн, что и GetMacroregionRows(). Разбор строки -- в
 * ExtractMirCodes(). Вызывается только когда реально пришёл фильтр mir_code (см. Run()) --
 * не нужен для самих строк отчёта, только для фильтрации.
 * @returns {Object[]}      -   Массив { id, mir_codes } (mir_codes -- сырая строка вида "#LASK#17#|#LASM#17#").
 */
function GetMirCodeRows()
{
    LogAlert(1, "GetMirCodeRows(). НАЧАЛО");
    var sqlText, rows;
    sqlText = "";
    sqlText = sqlText + "select cs.id,\r\n";
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_mir_codes'']/value)[1]', 'varchar(max)') as mir_codes\r\n";
    sqlText = sqlText + "from collaborators cs\r\n";
    sqlText = sqlText + "inner join collaborator c on c.id = cs.id\r\n";
    sqlText = sqlText + "where cs.is_dismiss != 1";
    rows = ArraySelectAll(XQuery("sql:" + sqlText));
    LogAlert(1, "GetMirCodeRows(). Строк: " + ArrayCount(rows));
    LogAlert(1, "GetMirCodeRows(). КОНЕЦ");
    return rows;
}

/*
 * Разбирает сырое значение f_mir_codes ("#LASK#17#|#LASM#17#...") в массив кодов без процентов.
 * По образцу getMirCodeObject() из education_accept_event_card, но нам не нужны ни проценты,
 * ни руководитель мир-кода -- только сами коды, для фильтра "есть ли у сотрудника такой код".
 * @param {string} rawValue     -   Сырое значение custom_elem f_mir_codes.
 * @returns {string[]}
 */
function ExtractMirCodes(rawValue)
{
    var parts, fields, codes, i;
    codes = [];
    parts = ArrayDirect(ArraySelect(String(rawValue).split("|"), "This != ''"));
    for (i = 0; i < ArrayCount(parts); i++)
    {
        fields = ArrayDirect(ArraySelect(String(parts[i]).split("#"), "This != ''"));
        if (ArrayCount(fields) > 0)
        {
            codes.push(String(fields[0]));
        }
    }
    return codes;
}

/*
 * Проверяет, есть ли у сотрудника указанный мир-код -- СРЕДИ ЛЮБЫХ его мир-кодов, не только
 * основного/с наибольшим процентом (так решили для фильтра -- см. параметры в шапке файла).
 * @param {Object[]} mirCodeRows    -   Результат GetMirCodeRows().
 * @param {number} collaboratorID   -   ID сотрудника.
 * @param {string} mirCodeFilter    -   Искомый мир-код.
 * @returns {boolean}
 */
function CollaboratorHasMirCode(mirCodeRows, collaboratorID, mirCodeFilter)
{
    var row, codes;
    row = ArrayOptFind(mirCodeRows, "Int(This.id) == Int(collaboratorID)");
    if (row == undefined)
    {
        return false;
    }
    codes = ExtractMirCodes(row.mir_codes);
    return (ArrayOptFind(codes, "String(This) == String(mirCodeFilter)") != undefined);
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
 * Ищет название программы по её ID в справочнике, построенном GetProgramTitles().
 * @param {Object[]} programTitles  -   Результат GetProgramTitles().
 * @param {number} programID        -   ID программы.
 * @returns {string}
 */
function FindProgramTitle(programTitles, programID)
{
    var titleRow;
    titleRow = ArrayOptFind(programTitles, "String(This.id) == String(programID)");
    return (titleRow != undefined ? String(titleRow.title) : "");
}

/*
 * Собирает строки отчёта для ОДНОГО сотрудника -- по одной строке на каждую программу матрицы
 * (длинный формат, см. "ФОРМАТ ОТЧЁТА" в шапке файла): 4 обязательных поля сотрудника + название
 * программы + дата прохождения. Ровно под статический список колонок стандартного виджета LPE
 * "Табличные данные" (fullname, position_name, subdivision_name, macroregion, program_name,
 * completion_date -- 6 полей, без зависимости от числа программ в матрице).
 * @param {Object} collaborator     -   Документ сотрудника (из GetActiveCollaboratorRows()).
 * @param {Object[]} macroRows      -   Результат GetMacroregionRows().
 * @param {Object[]} dateRows       -   Результат GetCompletionDateRows().
 * @param {Object[]} programTitles  -   Результат GetProgramTitles().
 * @param {number[]} programIds     -   ID программ (education_method).
 * @returns {Object[]}              -   Массив строк отчёта (по числу программ в матрице).
 */
function BuildReportRows(collaborator, macroRows, dateRows, programTitles, programIds)
{
    var rows, row, i, programID;
    rows = [];
    for (i = 0; i < ArrayCount(programIds); i++)
    {
        programID = programIds[i];
        row = new Object();
        row.fullname = String(collaborator.fullname);
        row.position_name = String(collaborator.position_name);
        row.subdivision_name = String(collaborator.position_parent_name);
        row.macroregion = FindMacroregion(macroRows, Int(collaborator.id));
        row.program_name = FindProgramTitle(programTitles, programID);
        row.completion_date = FindCompletionDate(dateRows, Int(collaborator.id), programID);
        rows.push(row);
    }
    return rows;
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
 * выбранной матрице обучения в ДЛИННОМ формате (1 строка на сотрудника+программу, см.
 * "ФОРМАТ ОТЧЁТА" в шапке файла) -- под стандартный виджет LPE "Табличные данные" со
 * статическим списком колонок. Поддерживает опциональные ручные фильтры: program_id,
 * macroregion, mir_code, position_name (см. описание параметров в шапке файла). ПОКА без
 * автоматического фильтра сотрудников по position_common_id/mir_code_id САМОЙ МАТРИЦЫ --
 * см. открытый вопрос №3 в шапке файла (это отдельный механизм, не путать с ручными фильтрами).
 *
 * ВАЖНО про RESULT: для "общей коллекции" (как у education_accept_event_card) виджет
 * ожидает, что RESULT -- это ПРЯМО массив строк, а не объект-обёртка (см. "ИСПРАВЛЕНО" в
 * шапке файла). Параметры читаются как обычные глобальные переменные (по образцу event_id
 * в education_accept_event_card), а не через PARAMETERS.
 * @returns {void}
 */
function Run()
{
    LogAlert(2, "Run(). НАЧАЛО");
    var matrixId, iProgramFilter, sMacroregionFilter, sMirCodeFilter, sPositionFilter;
    var programIds, programTitles, collaboratorRows, macroRows, mirCodeRows, dateRows, collaboratorReportRows, i, j;

    ERROR = 0;
    MESSAGE = "";
    RESULT = [];

    try
    {
        matrixId = OptInt(matrix_id, 0);
        iProgramFilter = OptInt(program_id, 0);
        sMacroregionFilter = String(macroregion);
        sMirCodeFilter = String(mir_code);
        sPositionFilter = String(position_name);
        LogAlert(1, "Run(). matrixId=" + matrixId + " programFilter=" + iProgramFilter
            + " macroregionFilter=[" + sMacroregionFilter + "] mirCodeFilter=[" + sMirCodeFilter
            + "] positionFilter=[" + sPositionFilter + "]");

        if (matrixId == 0)
        {
            throw ("Не передан matrix_id -- выбранная пользователем матрица обучения");
        }

        programIds = ResolveProgramIds(matrixId);

        if (iProgramFilter > 0)
        {
            programIds = ArraySelect(programIds, "Int(This) == iProgramFilter");
            if (ArrayCount(programIds) == 0)
            {
                throw ("Программа [" + iProgramFilter + "] не найдена среди программ выбранной матрицы");
            }
        }

        programTitles = GetProgramTitles(programIds);
        collaboratorRows = GetActiveCollaboratorRows();

        if (sPositionFilter != "")
        {
            collaboratorRows = ArraySelect(collaboratorRows, "String(This.position_name) == sPositionFilter");
            LogAlert(1, "Run(). После фильтра по должности осталось сотрудников: " + ArrayCount(collaboratorRows));
        }

        macroRows = GetMacroregionRows();
        if (sMacroregionFilter != "")
        {
            collaboratorRows = ArraySelect(collaboratorRows, "FindMacroregion(macroRows, Int(This.id)) == sMacroregionFilter");
            LogAlert(1, "Run(). После фильтра по макрорегиону осталось сотрудников: " + ArrayCount(collaboratorRows));
        }

        if (sMirCodeFilter != "")
        {
            mirCodeRows = GetMirCodeRows();
            collaboratorRows = ArraySelect(collaboratorRows, "CollaboratorHasMirCode(mirCodeRows, Int(This.id), sMirCodeFilter)");
            LogAlert(1, "Run(). После фильтра по мир-коду осталось сотрудников: " + ArrayCount(collaboratorRows));
        }

        dateRows = GetCompletionDateRows(programIds);
        //alert("programIds=" + tools.object_to_text(programIds, 'json') + "\r\ndateRows.count=" + ArrayCount(dateRows) + "\r\ndateRows=" + tools.object_to_text(dateRows, 'json')); // временно для отладки -- убрать перед сдачей

        for (i = 0; i < ArrayCount(collaboratorRows); i++)
        {
            collaboratorReportRows = BuildReportRows(collaboratorRows[i], macroRows, dateRows, programTitles, programIds);
            for (j = 0; j < ArrayCount(collaboratorReportRows); j++)
            {
                RESULT.push(collaboratorReportRows[j]);
            }
        }

        LogAlert(2, "Run(). Готово. Сотрудников: " + ArrayCount(collaboratorRows) + ", программ: " + ArrayCount(programIds) + ", строк отчёта: " + ArrayCount(RESULT));
        //alert(tools.object_to_text(RESULT, 'json')); // временно для отладки -- посмотреть, что реально вернул скрипт; убрать перед сдачей
    }
    catch (_ex)
    {
        ERROR = 1;
        MESSAGE = ExtractUserError(_ex);
        LogAlert(4, "Run(). ОШИБКА: " + MESSAGE);
        //alert("ОШИБКА: " + MESSAGE); // временно для отладки; убрать перед сдачей
    }
    LogAlert(2, "Run(). КОНЕЦ");
}

//-------------------------------------------------------------------------
//              Область основного кода
//-------------------------------------------------------------------------

Run();
