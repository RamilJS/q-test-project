sLogName = 'HREDU_182_TEST_manager_sql';
EnableLog(sLogName, true);
function alert(sInputObj)
{
    LogEvent(sLogName, sInputObj);
    return sInputObj;
}

// =====================================================================
// ВРЕМЕННЫЙ ДИАГНОСТИЧЕСКИЙ ФАЙЛ, РАУНД 2 (23.09.2026) -- НЕ для продакшена, удалить
// после теста. Продолжение HREDU-182_test_manager_sql.js -- см. историю там же.
//
// РЕЗУЛЬТАТ РАУНДА 1 (реальный тест, 23.09.2026):
//   - Контрольный запрос БЕЗ предиката (func_manager БЕЗ [is_native=...]) -- ОТРАБОТАЛ,
//     2312 строк. Для Рамиля вернул manager_id=6808456539733239590, что в hex -- ровно
//     0x5E7C7F266F7C2726 (Щапов, его ПЕРВАЯ, is_native=0, функциональная запись) --
//     подтверждает и порядок записей в документе, и то, что PERSON_ID В ОТВЕТЕ SQL
//     ПРИХОДИТ ОБЫЧНЫМ ДЕСЯТИЧНЫМ ЧИСЛОМ (а НЕ строкой "0x..." как в самом XML при
//     экспорте/просмотре документа) -- конвертация hex->десятичное НЕ понадобится.
//   - Оба варианта С ПРЕДИКАТОМ ([is_native=1] без кавычек, [is_native="1"] в двойных
//     кавычках) -- вернули ВООБЩЕ НОЛЬ СТРОК на весь запрос (не просто NULL в поле --
//     именно 0 строк, при том же JOIN/WHERE, что в контроле дал 2312) -- значит запрос
//     ПАДАЕТ ЦЕЛИКОМ на уровне SQL Server из-за синтаксиса предиката, и эта ошибка
//     почему-то не долетает до нашего try/catch как исключение (просто пустой массив).
//
// ГИПОТЕЗА РАУНДА 2: в РАБОЧЕМ предикате в этом же файле (GetCityRows() и др.,
// custom_elem[name=''sity'']) используются ДВОЙНЫЕ ОДИНАРНЫЕ кавычки (T-SQL-
// экранирование одинарной кавычки ВНУТРИ строкового литерала) -- а не "без кавычек" и
// не обычные двойные кавычки, которые пробовались в раунде 1. Вариант Г ниже -- точная
// копия ЭТОГО же проверенного стиля, просто сравниваем is_native со строкой ''1''
// вместо name со строкой ''sity''.
//
// ЗАПАСНОЙ ВАРИАНТ (Вариант Д): если is_native всё равно не заработает -- у ВСЕХ 4
// сотрудников по цепочке (Рамиль, Колесникова, Соловьева, Казначеев) поле boss_type_id
// РОВНО У is_native=1 записи было ОДНИМ И ТЕМ ЖЕ значением -- 0x55555555555555AA
// (похоже на служебную "sentinel"-константу платформы для типа "непосредственный
// руководитель"). Пробуем ту же ''..''-кавычку, но сравнение по boss_type_id вместо
// is_native -- альтернативный способ найти ТУ ЖЕ запись, если is_native почему-то не
// читается.
// =====================================================================

RESULT = [];
try
{
    var sqlTextD, sqlTextE, rowsD, rowsE, ramilRowD, ramilRowE;

    alert("1. НАЧАЛО раунда 2.");

    // -----------------------------------------------------------------
    // Вариант Г: is_native как СТРОКА, в ДВОЙНЫХ ОДИНАРНЫХ кавычках -- точная копия
    // проверенного стиля из GetCityRows()/GetMacroregionRows()/GetMirCodeRows().
    // -----------------------------------------------------------------
    sqlTextD = "";
    sqlTextD = sqlTextD + "select cs.id,\r\n";
    sqlTextD = sqlTextD + "       c.data.value('(*/func_managers/func_manager[is_native=''1'']/person_id)[1]', 'varchar(max)') as manager_id\r\n";
    sqlTextD = sqlTextD + "from collaborators cs\r\n";
    sqlTextD = sqlTextD + "inner join collaborator c on c.id = cs.id\r\n";
    sqlTextD = sqlTextD + "where cs.is_dismiss != 1";

    try
    {
        rowsD = ArraySelectAll(XQuery("sql:" + sqlTextD));
        alert("2D. Вариант Г (''1'', двойные одинарные кавычки) ВЫПОЛНИЛСЯ. Строк: " + ArrayCount(rowsD));
        ramilRowD = ArrayOptFind(rowsD, "String(This.id) == '7311507899656113337'");
        if (ramilRowD != undefined)
        {
            alert("3D. Строка Рамиля (Вариант Г): manager_id=[" + ramilRowD.manager_id + "] (ОЖИДАЕМ 6555406089169669479 -- это Колесникова, дес. от 0x5AF97B1B27640D67)");
        }
        else
        {
            alert("3D. ОШИБКА: строка Рамиля НЕ найдена в результате Варианта Г.");
        }
    }
    catch (_exD)
    {
        alert("2D. Вариант Г КИНУЛ ОШИБКУ: " + ExtractUserError(_exD));
    }

    // -----------------------------------------------------------------
    // Вариант Д: ЗАПАСНОЙ -- сравнение по boss_type_id (той же ''..''-кавычкой), а не
    // по is_native, на случай если is_native всё равно не сработает.
    // -----------------------------------------------------------------
    sqlTextE = "";
    sqlTextE = sqlTextE + "select cs.id,\r\n";
    sqlTextE = sqlTextE + "       c.data.value('(*/func_managers/func_manager[boss_type_id=''0x55555555555555AA'']/person_id)[1]', 'varchar(max)') as manager_id\r\n";
    sqlTextE = sqlTextE + "from collaborators cs\r\n";
    sqlTextE = sqlTextE + "inner join collaborator c on c.id = cs.id\r\n";
    sqlTextE = sqlTextE + "where cs.is_dismiss != 1";

    try
    {
        rowsE = ArraySelectAll(XQuery("sql:" + sqlTextE));
        alert("2E. Вариант Д (boss_type_id) ВЫПОЛНИЛСЯ. Строк: " + ArrayCount(rowsE));
        ramilRowE = ArrayOptFind(rowsE, "String(This.id) == '7311507899656113337'");
        if (ramilRowE != undefined)
        {
            alert("3E. Строка Рамиля (Вариант Д): manager_id=[" + ramilRowE.manager_id + "] (ОЖИДАЕМ 6555406089169669479 -- Колесникова)");
        }
        else
        {
            alert("3E. ОШИБКА: строка Рамиля НЕ найдена в результате Варианта Д.");
        }
    }
    catch (_exE)
    {
        alert("2E. Вариант Д КИНУЛ ОШИБКУ: " + ExtractUserError(_exE));
    }

    RESULT = (rowsD != undefined ? rowsD : (rowsE != undefined ? rowsE : []));
    alert("4. КОНЕЦ раунда 2. RESULT = результат первого удавшегося варианта (" + ArrayCount(RESULT) + " строк).");
}
catch (_ex)
{
    RESULT = [];
    alert("ОШИБКА ВЕРХНЕГО УРОВНЯ: " + ExtractUserError(_ex));
}
