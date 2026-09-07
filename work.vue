// =====================================================================
// HREDU-181. Диагностика №2: как называется коллекция документов должности,
// чтобы можно было массово (одним XQuery, а не по одному documents.open_doc()
// на каждого из тысяч сотрудников) достать position_common_id для всех
// активных сотрудников -- аналогично тому, как GetMacroregionRows()/
// GetMirCodeRows() массово достают свои поля.
//
// Контекст: диагностика №1 (HREDU-181_diagnostic_position_common_id.js) уже
// подтвердила, что:
//   - collaborators.position_id -> ссылка на документ должности;
//   - у этого документа ЕСТЬ поле position_common_id (не под custom_elems,
//     обычное поле) -- то самое, на которое настроен фильтр "типовая
//     должность" в LPE;
//   - но сама коллекция collaborators поля position_common_id НЕ отдаёт.
//
// Что делает скрипт: пробует прочитать несколько кандидатов-названий
// коллекции через XQuery (по образцу того, как в основном файле уже
// используются "for $elem in collaborators ...", "for $elem in
// cc_learning_matrices ..." -- то есть множественное число похоже на
// принятый в этой системе стиль именования коллекций). Для первого
// кандидата, который сработает, печатает количество строк и ВСЕ поля
// первой строки.
//
// Как запустить: как тестовый remote_action/скрипт-агент, по аналогии с
// первой диагностикой. Пришли мне вывод alert() целиком.
// =====================================================================

/*
 * Печатает все поля объекта через alert().
 * @param {string} label    -   Заголовок для лога.
 * @param {Object} obj      -   Объект, поля которого нужно распечатать.
 * @returns {void}
 */
function DumpFields(label, obj)
{
    var dump, fld;
    dump = "";
    for (fld in obj)
    {
        dump = dump + fld.Name + " = " + String(fld) + "\r\n";
    }
    alert("--- " + label + " ---\r\n" + dump);
}

function Run()
{
    var candidates, i, name, rows, report;

    candidates = ["position", "positions", "cc_position", "cc_positions"];
    report = "";

    for (i = 0; i < ArrayCount(candidates); i++)
    {
        name = candidates[i];
        try
        {
            rows = ArraySelectAll(XQuery("for $elem in " + name + " return $elem"));
            report = report + name + " -- РАБОТАЕТ, строк: " + ArrayCount(rows) + "\r\n";
            if (ArrayCount(rows) > 0)
            {
                DumpFields("коллекция [" + name + "], первая строка", rows[0]);
            }
        }
        catch (_ex)
        {
            report = report + name + " -- ошибка: " + ExtractUserError(_ex) + "\r\n";
        }
    }

    alert("--- Итог проверки кандидатов ---\r\n" + report);
}

Run();
