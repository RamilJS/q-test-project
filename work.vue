// =====================================================================
// HREDU-181. Диагностика №3: точечная проверка полей коллекции "positions"
// (9986 строк, подтверждено диагностикой №2).
//
// Диагностика №2 нашла рабочее название коллекции ("positions"), но её
// собственный вызов DumpFields() для первой строки этой коллекции либо не
// показал ничего, либо потерялся в логе -- неизвестно, что именно произошло.
// Этот скрипт не полагается на общий перебор "for (fld in obj)", а сначала
// пробует прочитать конкретные поля по имени (id, position_common_id)
// напрямую -- это должно быть надёжнее. Общий дамп тоже делает, но с
// защитой: если одно поле падает, это не должно убить весь дамп остальных.
//
// Как запустить: как тестовый remote_action/скрипт-агент. Пришли мне вывод
// alert() целиком -- в этот раз, пожалуйста, весь лог, там будет несколько
// строк/блоков подряд.
// =====================================================================

function Run()
{
    var rows, row, report, fld, val;

    rows = ArraySelectAll(XQuery("for $elem in positions return $elem"));
    alert("Шаг 1. positions: строк всего = " + ArrayCount(rows));

    if (ArrayCount(rows) == 0)
    {
        alert("Коллекция positions пустая -- дальше проверять нечего");
        return;
    }

    row = rows[0];

    // Шаг 2: прямой доступ к конкретным полям по имени.
    report = "";
    try
    {
        report = report + "row.id = " + String(row.id) + " (Int=" + Int(row.id) + ")\r\n";
    }
    catch (_ex1)
    {
        report = report + "row.id -- ОШИБКА: " + ExtractUserError(_ex1) + "\r\n";
    }
    try
    {
        report = report + "row.position_common_id = [" + String(row.position_common_id) + "]\r\n";
    }
    catch (_ex2)
    {
        report = report + "row.position_common_id -- ОШИБКА: " + ExtractUserError(_ex2) + "\r\n";
    }
    try
    {
        report = report + "row.name = [" + String(row.name) + "]\r\n";
    }
    catch (_ex3)
    {
        report = report + "row.name -- ОШИБКА: " + ExtractUserError(_ex3) + "\r\n";
    }
    alert("Шаг 2. Прямой доступ к полям первой строки positions:\r\n" + report);

    // Шаг 3: общий перебор полей, но с защитой -- одно упавшее поле не должен убить остальные.
    report = "";
    try
    {
        for (fld in row)
        {
            try
            {
                report = report + fld.Name + " = " + String(fld) + "\r\n";
            }
            catch (_exField)
            {
                report = report + "<поле не читается: " + ExtractUserError(_exField) + ">\r\n";
            }
        }
    }
    catch (_exLoop)
    {
        report = report + "\r\n[перебор полей упал целиком: " + ExtractUserError(_exLoop) + "]\r\n";
    }
    alert("Шаг 3. Общий перебор полей первой строки positions (" + report.length + " символов):\r\n" + report);
}

Run();
