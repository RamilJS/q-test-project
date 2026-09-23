sLogName = 'HREDU_182_TEST_manager_sql';
EnableLog(sLogName, true);
function alert(sInputObj)
{
    LogEvent(sLogName, sInputObj);
    return sInputObj;
}

// =====================================================================
// ВРЕМЕННЫЙ ДИАГНОСТИЧЕСКИЙ ФАЙЛ, РАУНД 5 (23.09.2026) -- НЕ для продакшена.
// Продолжение _round4.js -- см. историю там же.
//
// НАЙДЕНО (реальный тест, раунд 4): is_native в СЫРОЙ XML хранится как ТЕКСТ
// "true"/"false", а НЕ "1"/"0", как показывал экспорт документа при просмотре карточки
// сотрудника -- та же ловушка "экспорт ≠ то, что в БД", что уже была с id группы
// УОРиАП. ВСЕ предыдущие раунды (1-3) сравнивали с "1", которого в данных не было
// НИГДЕ -- отсюда предикат не находил совпадений вообще ни у одного сотрудника, и
// платформа (по неясной пока причине, отдельная находка) схлопывала это в 0 строк
// вместо 2312 строк с пустым manager_id. Позиционный предикат func_manager[3] в
// раунде 4 сработал СРАЗУ и правильно -- значит с синтаксисом предиката на func_manager
// всё было в порядке всё это время, дело было ИСКЛЮЧИТЕЛЬНО в сравниваемом значении.
//
// ЭТОТ ФАЙЛ: тот же предикат, что и в раунде 2 (Вариант Г, стиль ''..'' кавычек,
// проверенный на custom_elem[name=''sity'']), но со значением ''true'' вместо ''1'' --
// на ВСЕЙ таблице активных сотрудников (2312), с той же точечной проверкой на Рамиле,
// что и во всех прошлых раундах.
// =====================================================================

RESULT = [];
try
{
    var sqlText, rows, ramilRow, i;

    alert("1. НАЧАЛО раунда 5.");

    sqlText = "";
    sqlText = sqlText + "select cs.id,\r\n";
    sqlText = sqlText + "       c.data.value('(*/func_managers/func_manager[is_native=''true'']/person_id)[1]', 'varchar(max)') as manager_id\r\n";
    sqlText = sqlText + "from collaborators cs\r\n";
    sqlText = sqlText + "inner join collaborator c on c.id = cs.id\r\n";
    sqlText = sqlText + "where cs.is_dismiss != 1";

    rows = ArraySelectAll(XQuery("sql:" + sqlText));
    alert("2. Запрос (is_native=''true'') ВЫПОЛНИЛСЯ. Строк: " + ArrayCount(rows));

    ramilRow = ArrayOptFind(rows, "String(This.id) == '7311507899656113337'");
    if (ramilRow != undefined)
    {
        alert("3. Строка Рамиля: manager_id=[" + ramilRow.manager_id + "] (ОЖИДАЕМ 6555406089169669479 -- Колесникова)");
    }
    else
    {
        alert("3. ОШИБКА: строка Рамиля НЕ найдена.");
    }

    // Ещё точечно проверим Колесникову (её manager_id должен быть Соловьева,
    // 7595032916280367128) и Соловьеву (её manager_id должен быть Казначеев,
    // 7527496053871281461) -- всю цепочку сразу, раз уж мы её знаем по карточкам.
    var kolesnikovaRow, solovievaRow, kaznacheevRow;
    kolesnikovaRow = ArrayOptFind(rows, "String(This.id) == '6555406089169669479'");
    if (kolesnikovaRow != undefined)
    {
        alert("4. Колесникова: manager_id=[" + kolesnikovaRow.manager_id + "] (ОЖИДАЕМ 7595032916280367128 -- Соловьева)");
    }
    solovievaRow = ArrayOptFind(rows, "String(This.id) == '7595032916280367128'");
    if (solovievaRow != undefined)
    {
        alert("5. Соловьева: manager_id=[" + solovievaRow.manager_id + "] (ОЖИДАЕМ 7527496053871281461 -- Казначеев)");
    }
    kaznacheevRow = ArrayOptFind(rows, "String(This.id) == '7527496053871281461'");
    if (kaznacheevRow != undefined)
    {
        alert("6. Казначеев: manager_id=[" + kaznacheevRow.manager_id + "] (ОЖИДАЕМ 7527496053871281461 -- САМ НА СЕБЯ, маркер вершины иерархии)");
    }

    // Сколько сотрудников вообще НЕ имеют is_native=''true'' записи (manager_id пустой) --
    // если таких много, это тоже важно знать заранее (например, новые сотрудники без
    // назначенного руководителя, или сами руководители верхнего уровня).
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
    alert("7. Итог: с заполненным manager_id -- " + iFilled + "; БЕЗ manager_id (пусто) -- " + iEmpty + " (из " + ArrayCount(rows) + ").");

    RESULT = rows;
    alert("8. КОНЕЦ раунда 5. RESULT = все строки, для просмотра в виджете если понадобится.");
}
catch (_ex)
{
    RESULT = [];
    alert("ОШИБКА: " + ExtractUserError(_ex));
}
