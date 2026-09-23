sLogName = 'HREDU_182_TEST_manager_sql_round6_23092026';
EnableLog(sLogName, true);
function alert(sInputObj)
{
    LogEvent(sLogName, sInputObj);
    return sInputObj;
}

// =====================================================================
// ВРЕМЕННЫЙ ДИАГНОСТИЧЕСКИЙ ФАЙЛ, РАУНД 6 (23.09.2026) -- НЕ для продакшена.
// Продолжение _round5.js -- см. историю там же.
//
// РЕЗУЛЬТАТ РАУНДА 5: is_native=''true'' (правильное значение как строка) -- ВСЁ РАВНО
// 0 строк на весь запрос. Значит дело не в значении "1" vs "true" -- дело в САМОЙ ФОРМЕ
// сравнения. Позиционный предикат func_manager[3] (раунд 4, тест 2) сработал СРАЗУ и
// идеально -- значит [] на func_manager в принципе работает, просто предикаты именно
// СО СРАВНЕНИЕМ ПО ПОЛЮ (is_native=..., boss_type_id=...) -- ЛЮБЫМ значением, ЛЮБЫМ
// стилем кавычек -- валят запрос целиком.
//
// НОВАЯ ГИПОТЕЗА: is_native, возможно, хранится КАК ТИПИЗИРОВАННОЕ булево поле
// (xs:boolean по XML-схеме этой колонки), а не как обычный нетипизированный текст --
// и обычное строковое сравнение (= 'true') с типизированным булевым полем в SQL Server
// XQuery может кидать ошибку несовместимости типов (в отличие от custom_elem, который,
// похоже, нетипизированный текст -- отсюда и разница в поведении). Стандартный способ
// сравнивать булевы XML-поля в XQuery -- через функции true()/false(), а не через
// строковый литерал.
// =====================================================================

RESULT = [];
try
{
    var sqlText, rows, ramilRow, i;

    alert("1. НАЧАЛО раунда 6.");

    sqlText = "";
    sqlText = sqlText + "select cs.id,\r\n";
    sqlText = sqlText + "       c.data.value('(*/func_managers/func_manager[is_native=true()]/person_id)[1]', 'varchar(max)') as manager_id\r\n";
    sqlText = sqlText + "from collaborators cs\r\n";
    sqlText = sqlText + "inner join collaborator c on c.id = cs.id\r\n";
    sqlText = sqlText + "where cs.is_dismiss != 1";

    try
    {
        rows = ArraySelectAll(XQuery("sql:" + sqlText));
        alert("2. Запрос (is_native=true(), функция) ВЫПОЛНИЛСЯ. Строк: " + ArrayCount(rows));

        ramilRow = ArrayOptFind(rows, "String(This.id) == '7311507899656113337'");
        if (ramilRow != undefined)
        {
            alert("3. Строка Рамиля: manager_id=[" + ramilRow.manager_id + "] (ОЖИДАЕМ 6555406089169669479 -- Колесникова)");
        }
        else
        {
            alert("3. ОШИБКА: строка Рамиля НЕ найдена (если вообще есть строки).");
        }

        var iEmpty, iFilled;
        iEmpty = 0;
        iFilled = 0;
        for (i = 0; i < ArrayCount(rows); i++)
        {
            if (rows[i].manager_id == undefined || String(rows[i].manager_id) == "")
            {
                iEmpty = iEmpty + 1;
            }
            else
            {
                iFilled = iFilled + 1;
            }
        }
        alert("4. Итог: с заполненным manager_id -- " + iFilled + "; БЕЗ manager_id -- " + iEmpty + " (из " + ArrayCount(rows) + ").");
    }
    catch (_ex1)
    {
        alert("2. Запрос (true()) КИНУЛ ОШИБКУ: " + ExtractUserError(_ex1));
    }

    alert("5. КОНЕЦ раунда 6.");
}
catch (_ex)
{
    RESULT = [];
    alert("ОШИБКА ВЕРХНЕГО УРОВНЯ: " + ExtractUserError(_ex));
}
