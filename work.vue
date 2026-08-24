// =====================================================================
// HREDU-174. Агент создания и проставления типовых должностей
// -----------------------------------------------------------------------
// v3 - SQL-часть подтверждена в SSMS тимлидом, интегрирована в агент.
//
// ЧТО УЖЕ РЕАЛЬНОЕ (не заглушка):
//   - LogAlert() -- через tools.call_code_library_method/vtbl_common_lib
//   - GetActiveCollaboratorPositions() и GetExistingCommonPositionNames() --
//     настоящие SQL-запросы (только SELECT) через Binary()/AppendStr()/
//     XQuery("sql:"+...), по образцу примера тимлида. Таблицы и поля
//     подтверждены запросами к sys.tables/sys.columns:
//       collaborators.is_dismiss, collaborators.position_name,
//       position_commons.name
//
// ЧТО ВСЁ ЕЩЁ TODO (и почему):
//   1. CreateCommonPosition() / AssignCommonPositionToEmployee() --
//      это запись данных. Пользователь явно запретил делать это через
//      прямой SQL DML -- создание/сохранение объектов должно идти через
//      платформенный объектный API (CreateObject/SaveObject или как он
//      реально называется). Примера такого вызова мне пока не присылали
//      (только для чтения через XQuery) -- нужен пример из любого
//      агента/библиотеки, которая реально создаёт или сохраняет объект.
//   2. На collaborators до сих пор НЕТ поля-связи с типовой должностью
//      (проверено sys.columns) -- его создание через админку это
//      отдельный подготовительный шаг перед тем, как AssignCommonPosition
//      ToEmployee() вообще сможет что-то сохранить. Пока имя поля -- TODO
//      "position_common_id" по аналогии с наименованием в других тикетах.
//
// ОТКРЫТЫЕ БИЗНЕС-ВОПРОСЫ (решить с тимлидом/бизнесом, не блокируют код):
//   - Разночтения вроде "Администратор БД" / "Администратор баз данных",
//     "Бизнес-менеджер" / "Бизнес менеджер" -- по букве ТЗ считаются
//     РАЗНЫМИ типовыми должностями (агент их не сливает). Опечатки вроде
//     "Cпециалист" с латинской C, "Ведуший специалист" -- туда же.
//   - Тестовые/служебные записи ("Директор для теста", "Сотрудник 1-6" и
//     т.п.) агент тоже превратит в типовые должности -- ничего не фильтрует.
// =====================================================================

// ==================== ОБЛАСТЬ КОНСТАНТ ====================

DEBUG = false;                       // Включает подробные логи для дебага. На проде и в репо - всегда false
LOG_NAME = "agent";                  // Системный лог агентов, отдельный лог не создаём
CUR_OBJECT_ID = 0;                   // TODO: заменить на реальный ID агента (админка -> объект -> Сервис -> Копировать ID документа)

// ==================== ОБЛАСТЬ ФУНКЦИЙ ====================

/*
 * Выводит сообщение в логи.
 * @param {string} typeLog      -   Уровень логов. 1 - [DEBUG], 2 - [INFO], 3 - [WARN], 4 - [ERROR].
 * @param {string} message      -   Сообщение для логов
 * @returns {void}
 */
function LogAlert(typeLog, message)
{
    tools.call_code_library_method("vtbl_common_lib", "LogAlert", [LOG_NAME, typeLog, CUR_OBJECT_ID, message, DEBUG]);
}

/*
 * Обрезает пробелы по краям строки. Без regex и без String.trim() --
 * похоже, платформа их не поддерживает (была ошибка JS Syntax Error
 * именно на regex-литерале /\s+/g), поэтому только базовые charAt/substring.
 * @param {string} value        -   Исходная строка
 * @returns {string}
 */
function TrimSpaces(value)
{
    var start, end;

    start = 0;
    while (start < value.length && value.charAt(start) === " ")
    {
        start = start + 1;
    }

    end = value.length;
    while (end > start && value.charAt(end - 1) === " ")
    {
        end = end - 1;
    }

    return value.substring(start, end);
}

/*
 * Убирает лишние пробелы по краям и схлопывает повторы пробелов внутри названия.
 * Без regex -- разбивает по пробелу (split принимает строку-разделитель,
 * это не regex) и собирает обратно через одинарный пробел.
 * @param {string} rawName      -   Исходное название должности
 * @returns {string}
 */
function NormalizePositionName(rawName)
{
    var value, parts, normalizedName, i;

    value = TrimSpaces(rawName || "");
    parts = value.split(" ");
    normalizedName = "";

    for (i = 0; i < parts.length; i++)
    {
        if (parts[i] !== "")
        {
            normalizedName = normalizedName === "" ? parts[i] : normalizedName + " " + parts[i];
        }
    }

    return normalizedName;
}

/*
 * Возвращает id и название должности всех действующих сотрудников.
 * Реальный SQL, только чтение (SELECT) -- подтверждён в SSMS.
 * @returns {object[]}
 */
function GetActiveCollaboratorPositions()
{
    LogAlert(1, "GetActiveCollaboratorPositions(). НАЧАЛО");
    var bSqlStr, rows;

    bSqlStr = new Binary();
    bSqlStr.AppendStr("select id, position_name\r\n");
    bSqlStr.AppendStr("from collaborators\r\n");
    bSqlStr.AppendStr("where is_dismiss = 0\r\n");
    bSqlStr.AppendStr("  and position_name is not null\r\n");
    bSqlStr.AppendStr("  and ltrim(rtrim(position_name)) != ''\r\n");

    rows = XQuery("sql:" + bSqlStr.GetStr());

    LogAlert(1, "GetActiveCollaboratorPositions(). Найдено сотрудников: " + rows.length);
    LogAlert(1, "GetActiveCollaboratorPositions(). КОНЕЦ");
    return rows;
}

/*
 * Возвращает названия всех уже существующих типовых должностей.
 * Реальный SQL, только чтение (SELECT) -- подтверждён в SSMS.
 * @returns {object[]}
 */
function GetExistingCommonPositionNames()
{
    LogAlert(1, "GetExistingCommonPositionNames(). НАЧАЛО");
    var bSqlStr, rows;

    bSqlStr = new Binary();
    bSqlStr.AppendStr("select id, name\r\n");
    bSqlStr.AppendStr("from position_commons\r\n");

    rows = XQuery("sql:" + bSqlStr.GetStr());

    LogAlert(1, "GetExistingCommonPositionNames(). Найдено типовых должностей: " + rows.length);
    LogAlert(1, "GetExistingCommonPositionNames(). КОНЕЦ");
    return rows;
}

/*
 * Строит соответствие "нормализованное название -> ID" из строк position_commons.
 * @param {object[]} commonPositionRows  -   Строки из GetExistingCommonPositionNames()
 * @returns {object}
 */
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

/*
 * Создаёт новую типовую должность с указанным названием.
 * TODO: платформенный объектный API для СОЗДАНИЯ объекта пока не подтверждён
 * (были только примеры чтения через XQuery). Раскрыть API из примера
 * агента/библиотеки, которая реально пишет данные, и заменить заглушку.
 * @param {string} name         -   Название типовой должности
 * @returns {object}
 */
function CreateCommonPosition(name)
{
    LogAlert(1, "CreateCommonPosition(). НАЧАЛО");
    var commonPosition;

    // TODO: заменить на реальное создание объекта "Типовая должность" (position_commons)
    commonPosition = CreateObject("position_common", { name: name });
    LogAlert(1, "CreateCommonPosition(). Создана типовая должность: " + name);

    LogAlert(1, "CreateCommonPosition(). КОНЕЦ");
    return commonPosition;
}

/*
 * Гарантирует, что для каждого названия должности существует типовая должность,
 * и возвращает соответствие "название -> ID типовой должности".
 * @param {string[]} positionNames              -   Уникальные названия должностей сотрудников
 * @param {object} existingCommonPositionIdsByName  -   Уже существующие типовые должности
 * @returns {object}
 */
function EnsureCommonPositionsExist(positionNames, existingCommonPositionIdsByName)
{
    LogAlert(1, "EnsureCommonPositionsExist(). НАЧАЛО");
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
    return commonPositionIdsByName;
}

/*
 * Проставляет сотруднику ссылку на типовую должность по названию его должности.
 * TODO: платформенный объектный API для СОХРАНЕНИЯ объекта пока не подтверждён.
 * TODO: поля-связи с типовой должностью на collaborators пока не существует физически --
 * его создание через админку это отдельный шаг до того, как это заработает.
 * @param {object} collaboratorRow          -   Строка из GetActiveCollaboratorPositions() {id, position_name}
 * @param {object} commonPositionIdsByName  -   Соответствие "название -> ID типовой должности"
 * @returns {void}
 */
function AssignCommonPositionToEmployee(collaboratorRow, commonPositionIdsByName)
{
    LogAlert(1, "AssignCommonPositionToEmployee(). НАЧАЛО");
    var normalizedName, commonPositionId;

    normalizedName = NormalizePositionName(collaboratorRow.position_name);
    commonPositionId = commonPositionIdsByName[normalizedName];

    // TODO: заменить на реальное сохранение объекта "Сотрудник" (collaborators),
    // поле-связь пока называется предположительно position_common_id
    SaveObject("collaborator", collaboratorRow.id, { position_common_id: commonPositionId });

    LogAlert(1, "AssignCommonPositionToEmployee(). КОНЕЦ");
}

/*
 * Точка входа агента.
 * @returns {void}
 */
function Run()
{
    LogAlert(2, "Run(). НАЧАЛО");

    try
    {
        var collaboratorRows, positionNamesSeen, positionNames, i, normalizedName, existingCommonPositionIdsByName, commonPositionIdsByName, collaboratorRow;

        collaboratorRows = GetActiveCollaboratorPositions();

        positionNamesSeen = {};
        positionNames = [];
        for (i = 0; i < collaboratorRows.length; i++)
        {
            normalizedName = NormalizePositionName(collaboratorRows[i].position_name);
            if (normalizedName !== "" && !positionNamesSeen[normalizedName])
            {
                positionNamesSeen[normalizedName] = true;
                positionNames.push(normalizedName);
            }
        }

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
            }
        }
    }
    catch (error)
    {
        LogAlert(4, "Run(). КРИТИЧЕСКАЯ ОШИБКА:\n" + error);
    }

    LogAlert(2, "Run(). КОНЕЦ");
}

// ==================== ОБЛАСТЬ ОСНОВНОГО КОДА ====================

Run();
