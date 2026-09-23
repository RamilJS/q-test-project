sLogName = 'HREDU_182_TEST_manager_sql_23092026';
EnableLog(sLogName, true);
function alert(sInputObj)
{
    LogEvent(sLogName, sInputObj);
    return sInputObj;
}


RESULT = [];
try
{
    var sqlTextA, sqlTextB, sqlTextC, rowsA, rowsB, rowsC, ramilRowA, ramilRowB, ramilRowC, i;

    alert("1. НАЧАЛО. Проверяем 3 варианта SQL-запроса по func_manager/person_id.");

    // -----------------------------------------------------------------
    // Вариант А: предикат БЕЗ кавычек (is_native как число).
    // -----------------------------------------------------------------
    sqlTextA = "";
    sqlTextA = sqlTextA + "select cs.id,\r\n";
    sqlTextA = sqlTextA + "       c.data.value('(*/func_managers/func_manager[is_native=1]/person_id)[1]', 'varchar(max)') as manager_id\r\n";
    sqlTextA = sqlTextA + "from collaborators cs\r\n";
    sqlTextA = sqlTextA + "inner join collaborator c on c.id = cs.id\r\n";
    sqlTextA = sqlTextA + "where cs.is_dismiss != 1";

    try
    {
        rowsA = ArraySelectAll(XQuery("sql:" + sqlTextA));
        alert("2A. Вариант А (без кавычек) ВЫПОЛНИЛСЯ. Строк: " + ArrayCount(rowsA));
        ramilRowA = ArrayOptFind(rowsA, "String(This.id) == '7311507899656113337'");
        if (ramilRowA != undefined)
        {
            alert("3A. Строка Рамиля (Вариант А): manager_id=[" + ramilRowA.manager_id + "] (длина: " + StrLen(String(ramilRowA.manager_id)) + ")");
        }
        else
        {
            alert("3A. ОШИБКА: строка Рамиля НЕ найдена в результате Варианта А.");
        }
    }
    catch (_exA)
    {
        alert("2A. Вариант А (без кавычек) КИНУЛ ОШИБКУ: " + ExtractUserError(_exA));
    }

    // -----------------------------------------------------------------
    // Вариант Б: предикат В кавычках (is_native как строка "1").
    // -----------------------------------------------------------------
    sqlTextB = "";
    sqlTextB = sqlTextB + "select cs.id,\r\n";
    sqlTextB = sqlTextB + "       c.data.value('(*/func_managers/func_manager[is_native=\"1\"]/person_id)[1]', 'varchar(max)') as manager_id\r\n";
    sqlTextB = sqlTextB + "from collaborators cs\r\n";
    sqlTextB = sqlTextB + "inner join collaborator c on c.id = cs.id\r\n";
    sqlTextB = sqlTextB + "where cs.is_dismiss != 1";

    try
    {
        rowsB = ArraySelectAll(XQuery("sql:" + sqlTextB));
        alert("2B. Вариант Б (в кавычках) ВЫПОЛНИЛСЯ. Строк: " + ArrayCount(rowsB));
        ramilRowB = ArrayOptFind(rowsB, "String(This.id) == '7311507899656113337'");
        if (ramilRowB != undefined)
        {
            alert("3B. Строка Рамиля (Вариант Б): manager_id=[" + ramilRowB.manager_id + "] (длина: " + StrLen(String(ramilRowB.manager_id)) + ")");
        }
        else
        {
            alert("3B. ОШИБКА: строка Рамиля НЕ найдена в результате Варианта Б.");
        }
    }
    catch (_exB)
    {
        alert("2B. Вариант Б (в кавычках) КИНУЛ ОШИБКУ: " + ExtractUserError(_exB));
    }

    // -----------------------------------------------------------------
    // Вариант В: КОНТРОЛЬНЫЙ -- БЕЗ предиката вообще (первый func_manager в документе,
    // каким бы ни было его is_native) -- чтобы убедиться, что предикат в А/Б реально
    // что-то фильтрует, а не просто случайно совпадает с первым узлом.
    // -----------------------------------------------------------------
    sqlTextC = "";
    sqlTextC = sqlTextC + "select cs.id,\r\n";
    sqlTextC = sqlTextC + "       c.data.value('(*/func_managers/func_manager/person_id)[1]', 'varchar(max)') as manager_id\r\n";
    sqlTextC = sqlTextC + "from collaborators cs\r\n";
    sqlTextC = sqlTextC + "inner join collaborator c on c.id = cs.id\r\n";
    sqlTextC = sqlTextC + "where cs.is_dismiss != 1";

    try
    {
        rowsC = ArraySelectAll(XQuery("sql:" + sqlTextC));
        alert("2C. Вариант В (контроль, без предиката) ВЫПОЛНИЛСЯ. Строк: " + ArrayCount(rowsC));
        ramilRowC = ArrayOptFind(rowsC, "String(This.id) == '7311507899656113337'");
        if (ramilRowC != undefined)
        {
            alert("3C. Строка Рамиля (Вариант В, контроль): manager_id=[" + ramilRowC.manager_id + "]");
            if (ramilRowA != undefined && String(ramilRowA.manager_id) == String(ramilRowC.manager_id))
            {
                alert("4. ВНИМАНИЕ: Вариант А совпал с контролем В -- похоже, предикат [is_native=1] БЕЗ КАВЫЧЕК НЕ фильтрует (взял первый func_manager без разбора is_native). У Рамиля первые записи func_manager -- is_native=0, значит если А == В, то А, скорее всего, ОШИБОЧНО вернул функционального руководителя, а не непосредственного.");
            }
            else if (ramilRowA != undefined)
            {
                alert("4. ХОРОШО: Вариант А ОТЛИЧАЕТСЯ от контроля В -- предикат [is_native=1] похоже реально фильтрует.");
            }
        }
        else
        {
            alert("3C. ОШИБКА: строка Рамиля НЕ найдена в результате Варианта В.");
        }

        // Первые 5 строк контрольного варианта -- просто посмотреть, что вообще
        // приходит (пустые/NULL manager_id тоже важно увидеть -- например у сотрудников
        // без единой записи func_manager вообще).
        for (i = 0; i < ArrayCount(rowsC) && i < 5; i++)
        {
            alert("5." + i + ". id=" + rowsC[i].id + " manager_id(контроль)=[" + rowsC[i].manager_id + "]");
        }
    }
    catch (_exC)
    {
        alert("2C. Вариант В (контроль) КИНУЛ ОШИБКУ: " + ExtractUserError(_exC));
    }

    RESULT = (rowsA != undefined ? rowsA : (rowsB != undefined ? rowsB : []));
    alert("6. КОНЕЦ теста. RESULT = результат первого удавшегося варианта (" + ArrayCount(RESULT) + " строк), для просмотра в виджете если понадобится.");
}
catch (_ex)
{
    RESULT = [];
    alert("ОШИБКА ВЕРХНЕГО УРОВНЯ: " + ExtractUserError(_ex));
}
