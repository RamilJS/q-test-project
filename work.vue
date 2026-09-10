// =====================================================================
// HREDU-183. Диагностика: поля "аудитории матрицы" (position_common_id/mir_code_id
// на самой cc_learning_matrice -- см. ТЗ: "Общее кол-во сотрудников -- все, кто
// подходят под матрицу (должность + мир-код)") и поля "периода прохождения тренинга"
// (нужно для "План" -- "у кого уже наступил период прохождения тренинга") -- такого
// поля мы ещё НЕ находили ни на матрице, ни на её элементах, ни где-либо ещё.
//
// Проверяем ОБЕ коллекции сразу: саму матрицу (cc_learning_matrice) и её элементы
// (cc_learning_matrice_element, по одному на программу в составе матрицы) -- срок
// может быть общим для всей матрицы, а может быть свой у каждой программы.
//
// Техника -- та же, что уже сработала для полей "positions" (HREDU-181_diagnostic_positions_fields.js):
// общий перебор через for-in (работает по МАССИВАМ строк из XQuery, не по любым объектам --
// это уже проверено на Request и не сработало там, но сработало на строках из ArraySelectAll)
// + точечная проверка кандидатов по имени напрямую через точку (без скобочной нотации --
// Request["x"] уже показал себя ненадёжно, скобочный доступ к произвольному свойству по
// имени переменной здесь может не работать как в обычном JS).
//
// matrixId ниже -- реальный ID матрицы из предыдущих логов ("Матрица тест 3").
//
// Как запустить: как тестовый remote_action/скрипт-агент. Пришли мне вывод alert() целиком.
// =====================================================================

/*
 * Общий перебор полей строки через for-in -- по аналогии с диагностикой positions.
 * @param {Object} row
 * @returns {string}
 */
function DumpFieldsGeneric(row)
{
    var report, fld;
    report = "";
    try
    {
        for (fld in row)
        {
            try
            {
                report = report + "  " + fld.Name + " = " + String(fld) + "\r\n";
            }
            catch (_exField)
            {
                report = report + "  <поле не читается: " + ExtractUserError(_exField) + ">\r\n";
            }
        }
    }
    catch (_exLoop)
    {
        report = report + "  [перебор полей упал целиком: " + ExtractUserError(_exLoop) + "]\r\n";
    }
    return report;
}

function Run()
{
    var matrixId, matrixRows, matrixRow, elementRows, elementRow, report;

    matrixId = 7682761831139375285; // "Матрица тест 3" -- известный ID из прошлых логов

    report = "";

    // --- Матрица (cc_learning_matrice) -------------------------------------------
    matrixRows = ArraySelectAll(XQuery("for $elem in cc_learning_matrices where $elem/id = " + matrixId + " return $elem"));
    if (ArrayCount(matrixRows) == 0)
    {
        alert("Матрица с id=" + matrixId + " не найдена -- нужен другой реальный ID для диагностики");
        return;
    }
    matrixRow = matrixRows[0];

    report = report + "===== МАТРИЦА (cc_learning_matrice, id=" + matrixId + ") =====\r\n";
    report = report + "-- общий перебор полей (for-in) --\r\n";
    report = report + DumpFieldsGeneric(matrixRow);
    report = report + "-- точечные кандидаты --\r\n";

    try { report = report + "  position_common_id = [" + String(matrixRow.position_common_id) + "]\r\n"; } catch (_e1) { report = report + "  position_common_id -- ОШИБКА: " + ExtractUserError(_e1) + "\r\n"; }
    try { report = report + "  position_id = [" + String(matrixRow.position_id) + "]\r\n"; } catch (_e2) { report = report + "  position_id -- ОШИБКА: " + ExtractUserError(_e2) + "\r\n"; }
    try { report = report + "  mir_code_id = [" + String(matrixRow.mir_code_id) + "]\r\n"; } catch (_e3) { report = report + "  mir_code_id -- ОШИБКА: " + ExtractUserError(_e3) + "\r\n"; }
    try { report = report + "  mir_code = [" + String(matrixRow.mir_code) + "]\r\n"; } catch (_e4) { report = report + "  mir_code -- ОШИБКА: " + ExtractUserError(_e4) + "\r\n"; }
    try { report = report + "  start_date = [" + String(matrixRow.start_date) + "]\r\n"; } catch (_e5) { report = report + "  start_date -- ОШИБКА: " + ExtractUserError(_e5) + "\r\n"; }
    try { report = report + "  due_date = [" + String(matrixRow.due_date) + "]\r\n"; } catch (_e6) { report = report + "  due_date -- ОШИБКА: " + ExtractUserError(_e6) + "\r\n"; }
    try { report = report + "  deadline_date = [" + String(matrixRow.deadline_date) + "]\r\n"; } catch (_e7) { report = report + "  deadline_date -- ОШИБКА: " + ExtractUserError(_e7) + "\r\n"; }
    try { report = report + "  period_start = [" + String(matrixRow.period_start) + "]\r\n"; } catch (_e8) { report = report + "  period_start -- ОШИБКА: " + ExtractUserError(_e8) + "\r\n"; }
    try { report = report + "  period_start_date = [" + String(matrixRow.period_start_date) + "]\r\n"; } catch (_e9) { report = report + "  period_start_date -- ОШИБКА: " + ExtractUserError(_e9) + "\r\n"; }
    try { report = report + "  training_period = [" + String(matrixRow.training_period) + "]\r\n"; } catch (_e10) { report = report + "  training_period -- ОШИБКА: " + ExtractUserError(_e10) + "\r\n"; }
    try { report = report + "  period = [" + String(matrixRow.period) + "]\r\n"; } catch (_e11) { report = report + "  period -- ОШИБКА: " + ExtractUserError(_e11) + "\r\n"; }
    try { report = report + "  term = [" + String(matrixRow.term) + "]\r\n"; } catch (_e12) { report = report + "  term -- ОШИБКА: " + ExtractUserError(_e12) + "\r\n"; }
    try { report = report + "  assign_date = [" + String(matrixRow.assign_date) + "]\r\n"; } catch (_e13) { report = report + "  assign_date -- ОШИБКА: " + ExtractUserError(_e13) + "\r\n"; }
    try { report = report + "  period_days = [" + String(matrixRow.period_days) + "]\r\n"; } catch (_e14) { report = report + "  period_days -- ОШИБКА: " + ExtractUserError(_e14) + "\r\n"; }

    alert("Шаг 1/2 (МАТРИЦА) готов, длина отчёта: " + report.length);

    // --- Элемент матрицы (cc_learning_matrice_element) -----------------------------
    elementRows = ArraySelectAll(XQuery("for $elem in cc_learning_matrice_elements where $elem/cc_learning_matrice_id = " + matrixId + " return $elem"));
    report = report + "\r\n===== ЭЛЕМЕНТ МАТРИЦЫ (cc_learning_matrice_element), найдено элементов: " + ArrayCount(elementRows) + " =====\r\n";

    if (ArrayCount(elementRows) > 0)
    {
        elementRow = elementRows[0];
        report = report + "-- общий перебор полей ПЕРВОГО элемента (for-in) --\r\n";
        report = report + DumpFieldsGeneric(elementRow);
        report = report + "-- точечные кандидаты --\r\n";

        try { report = report + "  position_common_id = [" + String(elementRow.position_common_id) + "]\r\n"; } catch (_f1) { report = report + "  position_common_id -- ОШИБКА: " + ExtractUserError(_f1) + "\r\n"; }
        try { report = report + "  mir_code_id = [" + String(elementRow.mir_code_id) + "]\r\n"; } catch (_f2) { report = report + "  mir_code_id -- ОШИБКА: " + ExtractUserError(_f2) + "\r\n"; }
        try { report = report + "  start_date = [" + String(elementRow.start_date) + "]\r\n"; } catch (_f3) { report = report + "  start_date -- ОШИБКА: " + ExtractUserError(_f3) + "\r\n"; }
        try { report = report + "  due_date = [" + String(elementRow.due_date) + "]\r\n"; } catch (_f4) { report = report + "  due_date -- ОШИБКА: " + ExtractUserError(_f4) + "\r\n"; }
        try { report = report + "  deadline_date = [" + String(elementRow.deadline_date) + "]\r\n"; } catch (_f5) { report = report + "  deadline_date -- ОШИБКА: " + ExtractUserError(_f5) + "\r\n"; }
        try { report = report + "  period_start = [" + String(elementRow.period_start) + "]\r\n"; } catch (_f6) { report = report + "  period_start -- ОШИБКА: " + ExtractUserError(_f6) + "\r\n"; }
        try { report = report + "  period_start_date = [" + String(elementRow.period_start_date) + "]\r\n"; } catch (_f7) { report = report + "  period_start_date -- ОШИБКА: " + ExtractUserError(_f7) + "\r\n"; }
        try { report = report + "  training_period = [" + String(elementRow.training_period) + "]\r\n"; } catch (_f8) { report = report + "  training_period -- ОШИБКА: " + ExtractUserError(_f8) + "\r\n"; }
        try { report = report + "  period = [" + String(elementRow.period) + "]\r\n"; } catch (_f9) { report = report + "  period -- ОШИБКА: " + ExtractUserError(_f9) + "\r\n"; }
        try { report = report + "  term = [" + String(elementRow.term) + "]\r\n"; } catch (_f10) { report = report + "  term -- ОШИБКА: " + ExtractUserError(_f10) + "\r\n"; }
        try { report = report + "  assign_date = [" + String(elementRow.assign_date) + "]\r\n"; } catch (_f11) { report = report + "  assign_date -- ОШИБКА: " + ExtractUserError(_f11) + "\r\n"; }
        try { report = report + "  period_days = [" + String(elementRow.period_days) + "]\r\n"; } catch (_f12) { report = report + "  period_days -- ОШИБКА: " + ExtractUserError(_f12) + "\r\n"; }
        try { report = report + "  education_method_id = [" + String(elementRow.education_method_id) + "]\r\n"; } catch (_f13) { report = report + "  education_method_id -- ОШИБКА: " + ExtractUserError(_f13) + "\r\n"; }
        try { report = report + "  is_active = [" + String(elementRow.is_active) + "]\r\n"; } catch (_f14) { report = report + "  is_active -- ОШИБКА: " + ExtractUserError(_f14) + "\r\n"; }
    }

    alert("--- ИТОГОВЫЙ ОТЧЁТ (длина " + report.length + " символов) ---\r\n" + report);
}

Run();
