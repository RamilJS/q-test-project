sLogName = 'HREDU_182_TEST_manager_sql';
EnableLog(sLogName, true);
function alert(sInputObj)
{
    LogEvent(sLogName, sInputObj);
    return sInputObj;
}


RESULT = [];
try
{
    var sqlText1, sqlText2, sqlText3, rows1, rows2, rows3, row, i;

    alert("1. НАЧАЛО раунда 3.");

    // -----------------------------------------------------------------
    // Тест 1: предикат is_native=''1'', но только на 4 заведомо "хороших" id.
    // -----------------------------------------------------------------
    sqlText1 = "";
    sqlText1 = sqlText1 + "select cs.id,\r\n";
    sqlText1 = sqlText1 + "       c.data.value('(*/func_managers/func_manager[is_native=''1'']/person_id)[1]', 'varchar(max)') as manager_id\r\n";
    sqlText1 = sqlText1 + "from collaborators cs\r\n";
    sqlText1 = sqlText1 + "inner join collaborator c on c.id = cs.id\r\n";
    sqlText1 = sqlText1 + "where cs.id in (7311507899656113337, 6555406089169669479, 7595032916280367128, 7527496053871281461)";

    try
    {
        rows1 = ArraySelectAll(XQuery("sql:" + sqlText1));
        alert("2.1. Тест 1 (сужено до 4 id) ВЫПОЛНИЛСЯ. Строк: " + ArrayCount(rows1));
        for (i = 0; i < ArrayCount(rows1); i++)
        {
            alert("3.1." + i + ". id=" + rows1[i].id + " manager_id=[" + rows1[i].manager_id + "]");
        }
    }
    catch (_ex1)
    {
        alert("2.1. Тест 1 КИНУЛ ОШИБКУ: " + ExtractUserError(_ex1));
    }

    // -----------------------------------------------------------------
    // Тест 2: .exist() с тем же предикатом, на ВСЕЙ таблице -- просто булево флаг,
    // без value()/varchar. Считаем, у скольких сотрудников есть is_native=1 запись.
    // -----------------------------------------------------------------
    sqlText2 = "";
    sqlText2 = sqlText2 + "select cs.id,\r\n";
    sqlText2 = sqlText2 + "       c.data.exist('*/func_managers/func_manager[is_native=''1'']') as has_native\r\n";
    sqlText2 = sqlText2 + "from collaborators cs\r\n";
    sqlText2 = sqlText2 + "inner join collaborator c on c.id = cs.id\r\n";
    sqlText2 = sqlText2 + "where cs.is_dismiss != 1";

    try
    {
        rows2 = ArraySelectAll(XQuery("sql:" + sqlText2));
        alert("2.2. Тест 2 (.exist() с предикатом, вся таблица) ВЫПОЛНИЛСЯ. Строк: " + ArrayCount(rows2));
        row = ArrayOptFind(rows2, "String(This.id) == '7311507899656113337'");
        if (row != undefined)
        {
            alert("3.2. Рамиль: has_native=[" + row.has_native + "]");
        }
    }
    catch (_ex2)
    {
        alert("2.2. Тест 2 (.exist() с предикатом) КИНУЛ ОШИБКУ: " + ExtractUserError(_ex2));
    }

    // -----------------------------------------------------------------
    // Тест 3: .exist() БЕЗ предиката -- просто "есть ли у сотрудника func_managers
    // вообще". Если тут меньше 2312 -- вот источник "плохих" строк.
    // -----------------------------------------------------------------
    sqlText3 = "";
    sqlText3 = sqlText3 + "select cs.id,\r\n";
    sqlText3 = sqlText3 + "       c.data.exist('*/func_managers/func_manager') as has_any_manager\r\n";
    sqlText3 = sqlText3 + "from collaborators cs\r\n";
    sqlText3 = sqlText3 + "inner join collaborator c on c.id = cs.id\r\n";
    sqlText3 = sqlText3 + "where cs.is_dismiss != 1";

    try
    {
        rows3 = ArraySelectAll(XQuery("sql:" + sqlText3));
        alert("2.3. Тест 3 (.exist() без предиката) ВЫПОЛНИЛСЯ. Строк: " + ArrayCount(rows3));
        var iWithout, iWith;
        iWithout = 0;
        iWith = 0;
        for (i = 0; i < ArrayCount(rows3); i++)
        {
            if (String(rows3[i].has_any_manager) == "0" || rows3[i].has_any_manager == undefined)
            {
                iWithout = iWithout + 1;
                if (iWithout <= 5)
                {
                    alert("3.3. БЕЗ func_manager вообще: id=" + rows3[i].id);
                }
            }
            else
            {
                iWith = iWith + 1;
            }
        }
        alert("4. Тест 3 итог: с func_manager -- " + iWith + "; БЕЗ func_manager вообще -- " + iWithout + " (из " + ArrayCount(rows3) + " активных).");
    }
    catch (_ex3)
    {
        alert("2.3. Тест 3 (.exist() без предиката) КИНУЛ ОШИБКУ: " + ExtractUserError(_ex3));
    }

    RESULT = (rows1 != undefined ? rows1 : []);
    alert("5. КОНЕЦ раунда 3.");
}
catch (_ex)
{
    RESULT = [];
    alert("ОШИБКА ВЕРХНЕГО УРОВНЯ: " + ExtractUserError(_ex));
}
