sLogName = 'HREDU_237_diag_round1_28092026';
EnableLog(sLogName, true);
function alert(sInputObj)
{
    LogEvent(sLogName, sInputObj);
    return sInputObj;
}

// =====================================================================
// HREDU-237. ВРЕМЕННЫЙ ДИАГНОСТИЧЕСКИЙ ФАЙЛ, РАУНД 1 (28.09.2026) -- НЕ для продакшена.
// Прикрепить как выборку любому временному виджету (например "Табличные данные") и
// запустить -- результат смотреть в логе (EnableLog/LogEvent выше), не в самом виджете.
//
// ЦЕЛЬ -- проверить вместе с пользователем 3 вещи перед переписыванием HREDU-237:
//
//   1. Сколько всего документов compound_program в организации (чтобы понять, потянет
//      ли по производительности цикл tools.open_doc() по каждому документу, если массовое
//      чтение вложенных "programs" через SQL не заработает).
//
//   2. Массовое чтение custom_elems (f_matrix_active, f_position_names, f_mir_code и т.д.)
//      САМОГО compound_program одним SQL-запросом -- тем же проверенным приёмом
//      c.data.value(), что уже работает для collaborator.f_mir_codes (см.
//      GetMirCodeRows() в HREDU_182_filtry_percent.js). ГИПОТЕЗА (не подтверждена):
//      пара таблиц называется compound_programs (список) / compound_program (данные,
//      XML-колонка data) -- по аналогии с collaborators/collaborator. Если название
//      неверное -- SQL-запрос кинет ошибку, её текст увидим в логе и уточним в
//      следующем раунде.
//
//   3. Массовое чтение ВЛОЖЕННОЙ коллекции programs/program (это и есть "задачи"
//      модульной программы -- то, что раньше были элементы матрицы) ОДНИМ SQL-запросом
//      по ВСЕМ compound_program сразу, через SQL Server XML .nodes() (разворачивает
//      повторяющиеся XML-элементы в строки) -- ЕЩЁ НЕ ПРОВЕРЕННЫЙ приём в этом проекте
//      (c.data.value() достаёт ОДНО значение, а .nodes() -- именно "размножает" на много
//      строк, это другая техника). Если сработает -- сможем прочитать все задачи всех
//      модульных программ БЕЗ цикла tools.open_doc() по каждой -- иначе придётся
//      открывать документы по одному в JS (уже подтверждённый рабочий, но потенциально
//      медленный на больших объёмах способ -- см. oMatrixDocTE.programs в примере
//      редактора матриц, который пользователь прислал).
//
// ПОПУТНО (без привязки к реальным тестам) -- самотест WildcardMatch(): функции, которая
// будет матчить "* менеджер *"/"* руководитель" и т.п. против должности/подразделения/
// оргструктуры БЕЗ regex (regex-литералы в этом движке не поддерживаются -- см. навык
// websoft-hcm-scripting) -- через StrOptSubStrPos()/StrRangePos()/StrLen(), по образцу
// GetQueryParam(). Примеры пользователя (28.09.2026): "* менеджер *" должно совпасть со
// "Старший менеджер по продажам"; "* руководитель" (без звёздочки в конце) должно
// совпасть с "Региональный руководитель" (звёздочка = "что угодно до/после", то есть
// suffix-якорь без "*" на конце -- пример должен ОКАНЧИВАТЬСЯ на "руководитель").
// =====================================================================

// ---------------------------------------------------------------------
// WildcardMatch() -- см. описание в шапке. Разбивает паттерн по '*' (SplitByStar), затем
// ищет сегменты по порядку в тексте через StrOptSubStrPos() (тот же приём, что и в
// GetQueryParam()). Если паттерн не начинается с '*' -- первый сегмент обязан начинать
// текст. Если не заканчивается на '*' -- последний сегмент обязан заканчивать текст.
// УПРОЩЕНИЕ (не полный backtracking-матчерglob) -- жадный поиск слева направо: для
// реальных паттернов этого проекта (1-2 звёздочки на поле) этого достаточно, но если
// самотест ниже покажет проблему -- нужно будет усложнять.
// ---------------------------------------------------------------------
function SplitByStar(sPattern)
{
    var parts, iLen, iStart, iPos;
    parts = [];
    iLen = StrLen(sPattern);
    iStart = 0;
    while (true)
    {
        iPos = StrOptSubStrPos(sPattern, "*", true, iStart);
        if (iPos == undefined)
        {
            parts.push(StrRangePos(sPattern, iStart, iLen));
            break;
        }
        parts.push(StrRangePos(sPattern, iStart, iPos));
        iStart = iPos + 1;
    }
    return parts;
}

function WildcardMatch(sPattern, sText, bCaseSensitive)
{
    var parts, i, sSeg, iTextLen, iSearchPos, iFoundPos, bLeadingStar, bTrailingStar;
    if (sPattern == "") { return false; }
    bLeadingStar = (StrRangePos(sPattern, 0, 1) == "*");
    bTrailingStar = (StrRangePos(sPattern, StrLen(sPattern) - 1, StrLen(sPattern)) == "*");
    parts = SplitByStar(sPattern);
    iTextLen = StrLen(sText);
    iSearchPos = 0;
    for (i = 0; i < ArrayCount(parts); i++)
    {
        sSeg = parts[i];
        if (sSeg == "") { continue; }
        iFoundPos = StrOptSubStrPos(sText, sSeg, bCaseSensitive, iSearchPos);
        if (iFoundPos == undefined) { return false; }
        if (i == 0 && !bLeadingStar && iFoundPos != 0) { return false; }
        iSearchPos = iFoundPos + StrLen(sSeg);
    }
    if (!bTrailingStar && iSearchPos != iTextLen) { return false; }
    return true;
}

function MatchAnySemicolonPattern(sPatternsList, sText, bCaseSensitive)
{
    var patterns, i;
    if (sPatternsList == undefined || sPatternsList == "") { return false; }
    patterns = ArraySelect(String(sPatternsList).split(";"), "This != ''");
    for (i = 0; i < ArrayCount(patterns); i++)
    {
        if (WildcardMatch(patterns[i], sText, bCaseSensitive)) { return true; }
    }
    return false;
}

RESULT = [];
try
{
    var sqlText, rows, i, oneDoc, oneDocTE, cmpRow, tasksRows, distinctProgramIds, sTest;

    // -------------------------------------------------------------
    // ТЕСТ 0. Самотест WildcardMatch() -- офлайн, без БД, чтобы сразу увидеть в логе,
    // правильно ли реализована сама логика сравнения, до всех остальных тестов.
    // -------------------------------------------------------------
    alert("0.1. WildcardMatch('* менеджер *', 'Старший менеджер по продажам') = " +
        WildcardMatch("* менеджер *", "Старший менеджер по продажам", false) + " (ОЖИДАЕМ true)");
    alert("0.2. WildcardMatch('* руководитель', 'Региональный руководитель') = " +
        WildcardMatch("* руководитель", "Региональный руководитель", false) + " (ОЖИДАЕМ true)");
    alert("0.3. WildcardMatch('* руководитель', 'Руководитель отдела') = " +
        WildcardMatch("* руководитель", "Руководитель отдела", false) + " (ОЖИДАЕМ false -- 'руководитель' не в конце строки)");
    alert("0.4. WildcardMatch('менеджер*', 'Менеджер по продажам') = " +
        WildcardMatch("менеджер*", "Менеджер по продажам", false) + " (ОЖИДАЕМ true -- регистронезависимо)");
    alert("0.5. WildcardMatch('менеджер*', 'Менеджер по продажам') регистрозависимо = " +
        WildcardMatch("менеджер*", "Менеджер по продажам", true) + " (ОЖИДАЕМ false -- 'М' != 'м' с учётом регистра)");
    alert("0.6. MatchAnySemicolonPattern('LASR;LASM', 'LASM', true) = " +
        MatchAnySemicolonPattern("LASR;LASM", "LASM", true) + " (ОЖИДАЕМ true -- точное совпадение без звёздочек)");

    // -------------------------------------------------------------
    // ТЕСТ 1. Сколько всего документов compound_program.
    // -------------------------------------------------------------
    try
    {
        rows = ArraySelectAll(XQuery("for $elem in compound_programs return $elem/id"));
        alert("1. XQuery 'compound_programs' СРАБОТАЛ. Всего документов compound_program: " + ArrayCount(rows));
    }
    catch (_ex1)
    {
        alert("1. XQuery 'compound_programs' КИНУЛ ОШИБКУ: " + ExtractUserError(_ex1));
    }

    // -------------------------------------------------------------
    // ТЕСТ 2. Массовое чтение custom_elems САМОГО compound_program одним SQL-запросом.
    // Гипотеза таблиц: compound_programs (список) / compound_program (данные, XML-колонка
    // data) -- по аналогии с collaborators/collaborator. Дополнительно сверяем ОДНУ
    // случайную строку с tools.open_doc() по тому же id -- чтобы убедиться, что не
    // только запрос не упал, а ещё и значения СОВПАДАЮТ с тем, что реально в документе.
    // -------------------------------------------------------------
    try
    {
        sqlText = "";
        sqlText = sqlText + "select cs.id,\r\n";
        sqlText = sqlText + "       c.data.value('(*/name)[1]', 'varchar(max)') as prog_name,\r\n";
        sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_matrix_active'']/value)[1]', 'varchar(max)') as f_matrix_active,\r\n";
        sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_matrix_type'']/value)[1]', 'varchar(max)') as f_matrix_type,\r\n";
        sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_position_names'']/value)[1]', 'varchar(max)') as f_position_names,\r\n";
        sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_position_names_exclude'']/value)[1]', 'varchar(max)') as f_position_names_exclude,\r\n";
        sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_mir_code'']/value)[1]', 'varchar(max)') as f_mir_code,\r\n";
        sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_mir_code_exclude'']/value)[1]', 'varchar(max)') as f_mir_code_exclude,\r\n";
        sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_org_names'']/value)[1]', 'varchar(max)') as f_org_names,\r\n";
        sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_org_names_exclude'']/value)[1]', 'varchar(max)') as f_org_names_exclude,\r\n";
        sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_subdivision_names'']/value)[1]', 'varchar(max)') as f_subdivision_names,\r\n";
        sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_subdivision_names_exclude'']/value)[1]', 'varchar(max)') as f_subdivision_names_exclude,\r\n";
        sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_subdivision_child'']/value)[1]', 'varchar(max)') as f_subdivision_child,\r\n";
        sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_collaborator_statuses_exclude'']/value)[1]', 'varchar(max)') as f_collaborator_statuses_exclude\r\n";
        sqlText = sqlText + "from compound_programs cs\r\n";
        sqlText = sqlText + "inner join compound_program c on c.id = cs.id";

        rows = ArraySelectAll(XQuery("sql:" + sqlText));
        alert("2. SQL compound_programs/compound_program СРАБОТАЛ. Строк: " + ArrayCount(rows));

        if (ArrayCount(rows) > 0)
        {
            cmpRow = rows[0];
            alert("2.1. Пример первой строки (id=" + cmpRow.id + ", name=[" + cmpRow.prog_name + "]): " +
                "f_matrix_active=[" + cmpRow.f_matrix_active + "], f_matrix_type=[" + cmpRow.f_matrix_type + "], " +
                "f_position_names=[" + cmpRow.f_position_names + "], f_position_names_exclude=[" + cmpRow.f_position_names_exclude + "], " +
                "f_mir_code=[" + cmpRow.f_mir_code + "], f_mir_code_exclude=[" + cmpRow.f_mir_code_exclude + "], " +
                "f_org_names=[" + cmpRow.f_org_names + "], f_org_names_exclude=[" + cmpRow.f_org_names_exclude + "], " +
                "f_subdivision_names=[" + cmpRow.f_subdivision_names + "], f_subdivision_names_exclude=[" + cmpRow.f_subdivision_names_exclude + "], " +
                "f_subdivision_child=[" + cmpRow.f_subdivision_child + "], f_collaborator_statuses_exclude=[" + cmpRow.f_collaborator_statuses_exclude + "]");

            // Сверка с tools.open_doc() по тому же id -- совпадают ли значения.
            try
            {
                oneDoc = tools.open_doc(Int(cmpRow.id));
                oneDocTE = oneDoc.TopElem;
                sTest = String(oneDocTE.custom_elems.ObtainChildByKey("f_matrix_active").value);
                alert("2.2. Сверка tools.open_doc(" + cmpRow.id + ").f_matrix_active=[" + sTest + "] против SQL=[" + cmpRow.f_matrix_active + "] -- " +
                    (sTest == String(cmpRow.f_matrix_active) ? "СОВПАДАЕТ" : "НЕ СОВПАДАЕТ, ПРОВЕРИТЬ"));
            }
            catch (_ex2b)
            {
                alert("2.2. Сверка через tools.open_doc() КИНУЛА ОШИБКУ: " + ExtractUserError(_ex2b));
            }
        }
    }
    catch (_ex2)
    {
        alert("2. SQL compound_programs/compound_program КИНУЛ ОШИБКУ: " + ExtractUserError(_ex2));
    }

    // -------------------------------------------------------------
    // ТЕСТ 3. Массовое чтение вложенной коллекции programs/program (задачи) ОДНИМ
    // SQL-запросом по ВСЕМ compound_program сразу, через XML .nodes() (SQL Server) --
    // фильтр сразу по type='education_method' (только учебные программы, БЕЗ эл. курсов --
    // см. явное указание пользователя в задаче HREDU-237).
    // -------------------------------------------------------------
    try
    {
        sqlText = "";
        sqlText = sqlText + "select cs.id as matrix_id,\r\n";
        sqlText = sqlText + "       t.p.value('(object_id)[1]', 'bigint') as object_id,\r\n";
        sqlText = sqlText + "       t.p.value('(education_method_id)[1]', 'bigint') as education_method_id,\r\n";
        sqlText = sqlText + "       t.p.value('(type)[1]', 'varchar(50)') as ptype,\r\n";
        sqlText = sqlText + "       t.p.value('(delay_days)[1]', 'int') as delay_days,\r\n";
        sqlText = sqlText + "       t.p.value('(name)[1]', 'varchar(max)') as pname\r\n";
        sqlText = sqlText + "from compound_programs cs\r\n";
        sqlText = sqlText + "inner join compound_program c on c.id = cs.id\r\n";
        sqlText = sqlText + "cross apply c.data.nodes('/*/programs/program') as t(p)\r\n";
        sqlText = sqlText + "where t.p.value('(type)[1]', 'varchar(50)') = 'education_method'";

        tasksRows = ArraySelectAll(XQuery("sql:" + sqlText));
        alert("3. SQL .nodes() по programs/program СРАБОТАЛ. Строк (задач с типом education_method): " + ArrayCount(tasksRows));

        if (ArrayCount(tasksRows) > 0)
        {
            distinctProgramIds = ArraySelectDistinct(ArrayExtract(tasksRows, "Int(This.matrix_id)"), "This");
            alert("3.1. Из них уникальных модульных программ (matrix_id), у которых есть хотя бы одна задача education_method: " + ArrayCount(distinctProgramIds));
            alert("3.2. Пример первых 3 строк:");
            for (i = 0; i < ArrayCount(tasksRows) && i < 3; i++)
            {
                alert("3.2." + i + ". matrix_id=" + tasksRows[i].matrix_id + ", object_id=" + tasksRows[i].object_id +
                    ", education_method_id=" + tasksRows[i].education_method_id + ", type=[" + tasksRows[i].ptype + "], delay_days=" + tasksRows[i].delay_days +
                    ", name=[" + tasksRows[i].pname + "]");
            }
        }
    }
    catch (_ex3)
    {
        alert("3. SQL .nodes() по programs/program КИНУЛ ОШИБКУ: " + ExtractUserError(_ex3));
    }

    alert("ГОТОВО. Пришли, пожалуйста, весь лог целиком (все строки 0.x/1/2.x/3.x) -- по ним пойму, что подтвердилось, а что надо чинить во 2 раунде.");
}
catch (_ex)
{
    RESULT = [];
    alert("ОШИБКА ВЕРХНЕГО УРОВНЯ: " + ExtractUserError(_ex));
}
