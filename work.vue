// =====================================================================
// HREDU-174. Агент создания и проставления типовых должностей
// -----------------------------------------------------------------------
// v2 — обновлено после получения реальных примеров кода из Confluence.
// LogAlert() теперь настоящая (через tools.call_code_library_method /
// vtbl_common_lib), формат логов и структура выверены по примерам.
//
// Всё, что помечено TODO, — это ОБРАЩЕНИЯ К ДАННЫМ (поиск/создание/
// сохранение объектов в БД). Примеров такого кода мне пока не присылали
// (были только примеры про логирование и структуру), поэтому имена
// объектов/функций для работы с данными здесь — предположение по
// названиям полей из ваших тикетов (collaborator_id, position_common_id),
// а не подтверждённый API. Нужен пример любого агента/библиотеки, которая
// реально ищет, создаёт или сохраняет объект в БД — тогда перепишу набело.
// =====================================================================

// ==================== ОБЛАСТЬ КОНСТАНТ ====================

DEBUG = false;                       // Включает подробные логи для дебага. На проде и в репо — всегда false
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
 * Убирает лишние пробелы по краям и внутри названия должности.
 * @param {string} rawName      -   Исходное название должности
 * @returns {string}
 */
function NormalizePositionName(rawName)
{
    LogAlert(1, "NormalizePositionName(). НАЧАЛО");
    var normalizedName;

    normalizedName = (rawName || "").trim().replace(/\s+/g, " ");

    LogAlert(1, "NormalizePositionName(). КОНЕЦ");
    return normalizedName;
}

/*
 * Возвращает действующих сотрудников, у которых не проставлена типовая должность.
 * @returns {object[]}
 */
function GetActiveEmployeesWithoutCommonPosition()
{
    LogAlert(1, "GetActiveEmployeesWithoutCommonPosition(). НАЧАЛО");
    var employees;

    // TODO: заменить на реальный запрос к объекту "Сотрудник" (collaborator?)
    // Условие: статус сотрудника = действующий И поле связи с типовой
    // должностью (position_common_id?) не заполнено
    employees = FindObjects("collaborator", {
        status: "active",
        position_common_id: null
    });

    LogAlert(1, "GetActiveEmployeesWithoutCommonPosition(). Найдено сотрудников: " + employees.length);
    LogAlert(1, "GetActiveEmployeesWithoutCommonPosition(). КОНЕЦ");
    return employees;
}

/*
 * Возвращает уникальные нормализованные названия должностей из списка сотрудников.
 * @param {object[]} employees  -   Список сотрудников
 * @returns {string[]}
 */
function GetUniquePositionNames(employees)
{
    LogAlert(1, "GetUniquePositionNames(). НАЧАЛО");
    var positionNames, seenNames, i, employee, normalizedName;

    positionNames = [];
    seenNames = {};

    for (i = 0; i < employees.length; i++)
    {
        employee = employees[i];
        normalizedName = NormalizePositionName(employee.positionName);

        if (normalizedName !== "" && !seenNames[normalizedName])
        {
            seenNames[normalizedName] = true;
            positionNames.push(normalizedName);
        }
    }

    LogAlert(1, "GetUniquePositionNames(). Уникальных названий: " + positionNames.length);
    LogAlert(1, "GetUniquePositionNames(). КОНЕЦ");
    return positionNames;
}

/*
 * Находит существующую типовую должность по точному названию.
 * @param {string} name         -   Название типовой должности
 * @returns {object|null}
 */
function FindCommonPositionByName(name)
{
    LogAlert(1, "FindCommonPositionByName(). НАЧАЛО");
    var commonPosition;

    // TODO: заменить на реальный запрос к каталогу "Типовая должность" (position_common?)
    commonPosition = FindObjects("position_common", { name: name })[0] || null;

    LogAlert(1, "FindCommonPositionByName(). КОНЕЦ");
    return commonPosition;
}

/*
 * Создаёт новую типовую должность с указанным названием.
 * @param {string} name         -   Название типовой должности
 * @returns {object}
 */
function CreateCommonPosition(name)
{
    LogAlert(1, "CreateCommonPosition(). НАЧАЛО");
    var commonPosition;

    // TODO: заменить на реальное создание объекта в каталоге "Типовая должность"
    commonPosition = CreateObject("position_common", { name: name });
    LogAlert(1, "CreateCommonPosition(). Создана типовая должность: " + name);

    LogAlert(1, "CreateCommonPosition(). КОНЕЦ");
    return commonPosition;
}

/*
 * Гарантирует, что для каждого названия должности существует типовая должность,
 * и возвращает соответствие "название -> ID типовой должности".
 * @param {string[]} positionNames  -   Уникальные названия должностей
 * @returns {object}
 */
function EnsureCommonPositionsExist(positionNames)
{
    LogAlert(1, "EnsureCommonPositionsExist(). НАЧАЛО");
    var commonPositionIdsByName, i, name, commonPosition;

    commonPositionIdsByName = {};

    for (i = 0; i < positionNames.length; i++)
    {
        name = positionNames[i];
        commonPosition = FindCommonPositionByName(name);

        if (!commonPosition)
        {
            commonPosition = CreateCommonPosition(name);
        }

        commonPositionIdsByName[name] = commonPosition.id;
    }

    LogAlert(1, "EnsureCommonPositionsExist(). КОНЕЦ");
    return commonPositionIdsByName;
}

/*
 * Проставляет сотруднику ссылку на типовую должность по названию его должности.
 * @param {object} employee                 -   Сотрудник
 * @param {object} commonPositionIdsByName  -   Соответствие "название -> ID типовой должности"
 * @returns {void}
 */
function AssignCommonPositionToEmployee(employee, commonPositionIdsByName)
{
    LogAlert(1, "AssignCommonPositionToEmployee(). НАЧАЛО");
    var normalizedName, commonPositionId;

    normalizedName = NormalizePositionName(employee.positionName);
    commonPositionId = commonPositionIdsByName[normalizedName];

    // TODO: заменить на реальное сохранение объекта "Сотрудник"
    SaveObject("collaborator", employee.id, { position_common_id: commonPositionId });

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
        var employees, positionNames, commonPositionIdsByName, i, employee;

        employees = GetActiveEmployeesWithoutCommonPosition();
        positionNames = GetUniquePositionNames(employees);
        commonPositionIdsByName = EnsureCommonPositionsExist(positionNames);

        for (i = 0; i < employees.length; i++)
        {
            employee = employees[i];

            try
            {
                AssignCommonPositionToEmployee(employee, commonPositionIdsByName);
            }
            catch (employeeError)
            {
                LogAlert(3, "Run(). Не удалось проставить типовую должность сотруднику ID=" + employee.id + ": " + employeeError.message);
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
