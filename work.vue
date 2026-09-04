// =====================================================================
// HREDU-181. Диагностика: как на самом деле называются XQuery-коллекции
// для каталогов cc_learning_matrice / cc_learning_matrice_element.
//
// Зачем: в HREDU-181_vostok_polny_spisok_draft.js имена коллекций
// 'cc_learning_matrices' и 'cc_learning_matrice_elements' -- это ДОГАДКА
// по аналогии с cc_mir_code -> cc_mir_codes (HREDU-176). Но там выяснилось,
// что это не автоправило платформы, а то, как коллекцию назвали руками в
// админке -- поэтому реальное имя нужно проверять, а не предполагать.
//
// Как это работает: XQuery() с несуществующим именем коллекции падает с
// ошибкой (не возвращает молча пустой список) -- это уже проверено на
// примере cc_collaborator_mircodes в HREDU-176. Значит, прогнав несколько
// кандидатов через try/catch, можно по факту "не упало" понять, какое имя
// верное -- без похода в админку.
//
// Что делать с результатом: запустить этот скрипт (например, через
// тестовый remote_action/скрипт-агент, как и HREDU-176_diagnostic_mircodes.js),
// прислать мне вывод alert() целиком.
// =====================================================================

function SafeXQuery(label, queryArg)
{
    var rows, arr, count, firstRow, dump, fld;

    alert("--- " + label + " ---");
    try
    {
        rows = XQuery(queryArg);
        arr = ArraySelectAll(rows);
        count = ArrayCount(arr);
        alert("OK: " + label + " -- count=" + count);

        // Если что-то нашлось -- распечатаем поля первой записи, чтобы заодно
        // свериться с тем, какие имена полей мы предполагаем в черновике
        // (name, cc_learning_matrice_id, education_method_id, is_active, id...).
        if (count > 0)
        {
            firstRow = ArrayOptFirstElem(arr);
            dump = "";
            for (fld in firstRow)
            {
                dump = dump + fld.Name + " = " + String(fld) + "\r\n";
            }
            alert("Поля первой записи (" + label + "):\r\n" + dump);
        }
        return true;
    }
    catch (eq)
    {
        alert("FAIL: " + label + " -- " + eq);
        return false;
    }
}

function Run()
{
    // 1) Контроль -- заведомо рабочая коллекция, чтобы убедиться, что XQuery
    //    вообще жив и тест имеет смысл.
    SafeXQuery("XQuery('collaborators')", "collaborators");

    // 2) Кандидаты для каталога "Матрицы обучения" (object_name: cc_learning_matrice)
    SafeXQuery("XQuery('cc_learning_matrices')", "cc_learning_matrices");     // текущая догадка в черновике
    SafeXQuery("XQuery('cc_learning_matrice')", "cc_learning_matrice");       // без множественного числа

    // 3) Кандидаты для каталога "Элементы матрицы обучения" (object_name: cc_learning_matrice_element)
    SafeXQuery("XQuery('cc_learning_matrice_elements')", "cc_learning_matrice_elements"); // текущая догадка в черновике
    SafeXQuery("XQuery('cc_learning_matrice_element')", "cc_learning_matrice_element");   // без множественного числа
}

Run();
