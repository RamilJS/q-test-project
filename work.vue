
// =====================================================================
// HREDU-182. "Процент обученных" -- выборка для Табличных данных.
//
// ТЗ (из письма Антонова + "ТЗ Отчёт по матрицам", присланы пользователем 10.09
// и 14.09.2026) + РЕАЛЬНЫЙ ПРИМЕР ОТЧЁТА, присланный 14.09.2026 ("Макрорегион ВОСТОК",
// лист с колонками Город/Общее кол-во сотрудников/План/Факт/Процент/Обязательно к
// прохождению, построчно по городам + итоговая строка "Общий итог").
//
// РЕШЕНО С ПОЛЬЗОВАТЕЛЕМ (14.09.2026, через AskUserQuestion): реальный отчёт -- это
// ТАБЛИЦА С РАЗБИВКОЙ ПО ГОРОДАМ (как в примере), а НЕ одна строка показателей на всю
// матрицу (как можно было бы прочитать из ТЗ п.4 буквально -- там про "поле" в
// единственном числе, но пример явно про таблицу).
//
// ИЗМЕНЕНО (16.09.2026, по итогам разговора пользователя с тимлидом): выяснилось, что
// у одной матрицы может быть НЕСКОЛЬКО элементов (cc_learning_matrice_element), каждый
// со своей учебной программой (education_method_id) и своим периодом обучения -- см.
// переписку 15.09.2026 про каталоги "Матрицы обучения"/"Элементы матриц обучения".
// Раньше строка таблицы была на КАЖДЫЙ ГОРОД, а показатели total/plan/fact/mandatory
// СУММИРОВАЛИСЬ по всем программам матрицы в одну строку -- значит из таблицы было
// не видно, по какому именно элементу (программе) идёт расчёт. Теперь строка -- это
// ПАРА (Город, Учебная программа): если в городе есть 2 программы матрицы, у него будет
// 2 строки, по одной на каждую. Явно запрошено пользователем 16.09.2026: "сделаем ещё
// одну колонку после город - Учебная программа... чтобы было понятно по какому именно
// элементу матрицы строка отчета".
//
// Поле "город" -- custom_elem с именем "sity" (ПОДТВЕРЖДЕНО пользователем 14.09.2026,
// прислал реальный XML документа collaborator: <custom_elem><name>sity</name>
// <value>Санкт-Петербург</value></custom_elem>). ВНИМАНИЕ: имя технического поля
// именно "sity" (с опечаткой, не "city") -- это НЕ опечатка в этом файле, так
// называется реальное поле в системе.
//
// ИЗМЕНЕНО (17.09.2026, HREDU-215 "Правки 1", по прямому указанию тим-лида пользователя --
// "ошиблись в архитектуре"): поля position_common_id/mir_code_id УДАЛЕНЫ с типа документа
// "Матрицы обучения" (cc_learning_matrice) и ДОБАВЛЕНЫ на тип документа "Элементы матриц
// обучения" (cc_learning_matrice_element, у которых уже были education_method_id/
// start_study_period/end_study_period). Значит аудитория (должность+мир-код) теперь СВОЯ
// У КАЖДОЙ ПРОГРАММЫ (у каждого элемента, а если у одной программы несколько элементов с
// разными position_common_id -- у неё несколько аудиторий, объединяемых через ИЛИ), а НЕ
// ОДНА НА ВСЮ МАТРИЦУ, как было раньше. См. BuildProgramAudienceIndex()/
// CollaboratorInProgramAudience() ниже -- та же замена, что в HREDU-183_tep_reports.js.
//
// ЛОГИКА ПОДСЧЁТА (полностью повторяет HREDU-183_tep_reports.js -- см. "РЕШЕНИЯ" в его
// шапке -- только теперь всё разбито по городам вместо одного общего числа):
//   Общее (total)      -- аудитория ЭТОЙ ПРОГРАММЫ (по всем её активным элементам) +
//                          ручные фильтры пользователя, сгруппировано по городу.
//   План (plan)         -- = Общее (упрощение, период прохождения ещё не реализован,
//                          см. открытый вопрос в HREDU-183_tep_reports.js).
//   Факт (fact)          -- ИЗМЕНЕНО (17.09.2026, повторное уточнение в тот же день, что
//                          и HREDU-215 "Правки 1"): раньше считался БЕЗ ограничения
//                          аудиторией вообще (буквальное "не зависимо от условий
//                          матрицы") -- реальный тест показал, что так в "Факт" попадают
//                          сотрудники совсем других должностей (см. AskUserQuestion
//                          17.09.2026). ТЕПЕРЬ прошедшие тренинг ДОЛЖНЫ ТАКЖЕ входить в
//                          аудиторию программы (должность+мир-код хотя бы одного её
//                          элемента) -- "не зависимо от условий матрицы" означает
//                          "не зависимо от ПЕРИОДА" (см. План выше), а не вообще ни от
//                          чего. Плюс те же ручные фильтры, сгруппировано по городу.
//   Обязательно (mandatory) -- аудитория программы МИНУС прошедшие (пустая дата).
//   Процент (percent)    -- факт/план (округление до целого %, "-" если план = 0).
//     Уточнено с пользователем 10.09.2026 (см. HREDU-183_tep_reports.js) -- в тексте ТЗ
//     написано "план/факт", реально считаем факт/план, подтверждено сверкой с примером
//     (Новосибирск: 10/11=91%, Красноярск: 9/10=90%, Итог: 70/72=97% -- ВСЕ совпадают
//     ТОЛЬКО с факт/план, не план/факт).
//
// ГРУППИРОВКА ПО (ГОРОД, ПРОГРАММА) -- см. ДОПУЩЕНИЯ ниже. Строка появляется, если для
// этой пары есть хотя бы 1 человек в АУДИТОРИИ ПРОГРАММЫ (Общее > 0), ИЛИ хотя бы 1
// человек её прошёл (Факт > 0).
//
// ДОПУЩЕНИЯ (уточнить с пользователем при первом реальном прогоне):
//   1. Строки = пары (город, программа), где есть хотя бы 1 человек в АУДИТОРИИ (для
//      этой программы) ИЛИ хотя бы 1 человек её ПРОШЁЛ -- см. "ГРУППИРОВКА ПО (ГОРОД,
//      ПРОГРАММА)" выше. ИЗМЕНЕНО (17.09.2026, HREDU-215 "Правки 1" + повторное
//      уточнение о "Факт" в тот же день): раньше "Факт" не проверял аудиторию вообще --
//      значит строка могла появиться ТОЛЬКО из фактового прохода (Общее=0, Факт>0), даже
//      если прошедший вообще не подходил ни под одну должность матрицы. ТЕПЕРЬ "Факт"
//      тоже фильтруется по аудитории программы (см. ЛОГИКА ПОДСЧЁТА выше) -- значит
//      КАЖДЫЙ, кто попадает в Факт, по определению уже входит в аудиторию своей
//      программы и своего города, а значит уже посчитан в Общее для той же пары (город,
//      программа) -- строка "Общее=0, Факт>0" теперь практически не должна возникать.
//      Код-защита (создавать строку из фактового прохода только при sDate != "") тем не
//      менее оставлена как есть -- дешёвая подстраховка на случай расхождений в данных
//      (например, GetPositionIdsByCommonPosition() вернёт не совсем то, что ожидалось).
//   2. Сотрудники БЕЗ заполненного города (custom_elem "sity" пустой) -- попадают в
//      отдельную группу "(без города)", чтобы не терять данные молча.
//   3. Итоговая строка "Общий итог" -- сумма по ВСЕМ парам (город, программа), колонка
//      "Учебная программа" в ней -- "-" (относится ко всем программам сразу).
//   4. Сортировка строк (ИЗМЕНЕНО 16.09.2026) -- сначала по названию программы, затем
//      по городу внутри неё (простое сравнение строк, БЕЗ гарантии точной русской
//      локали в этом движке) -- т.е. все города одной программы идут блоком, а не
//      вперемешку город-за-городом. "Общий итог" всегда последней строкой.
//
// Параметры/фильтры (matrix_id, macroregion, mir_code, position_common_id, program_id)
// читаются ИЗ URL -- точно так же, как в HREDU-183_tep_reports.js. macroregion в этой
// выборке работает как ПРЕДФИЛЬТР (например "Восток" -- сузить список городов до
// одного макрорегиона, как в примере), а группировка идёт уже ПО ГОРОДУ внутри него.
//
// ВАЖНО про RESULT: как и в остальных выборках -- RESULT это ПРЯМО массив строк.
//
// =====================================================================
// ДОБАВЛЕНО (21.09.2026, по просьбе тим-лида пользователя): ПОПЫТКА сделать клик по
// РАЗНЫМ КОЛОНКАМ (План/Факт/Общее/Обязательно) ведущим на ТЭП-отчёт с СООТВЕТСТВУЮЩИМ
// result_type -- а не всегда с "total", как сейчас.
//
// ВАЖНО -- ЧЕСТНО О РИСКЕ ДО ЗАПУСКА: этот САМЫЙ эксперимент УЖЕ был опробован раньше
// (см. блок "ЗАКРЫТО (14.09.2026, КЛИКАБЕЛЬНОСТЬ)" ближе к концу файла, у COLUMNS) --
// тогда были поля total_link/plan_link/fact_link/mandatory_link, и реальный тест
// пользователя ПОДТВЕРДИЛ, что виджет "Табличные данные" различает клик ТОЛЬКО по
// строке целиком: какая бы колонка ни была нажата, срабатывает один и тот же "link"
// всей строки. То есть у этого виджета, по всей видимости, ЕСТЬ только один
// row-level href, и НЕТ понятия "своя ссылка у каждой ячейки/колонки" -- это не баг
// кода, а ограничение самого виджета (архитектурное, не то, что можно обойти другим
// JS в выборке).
//
// ЧТО ИМЕННО ПРОБУЕМ СЕЙЧАС (по прямой просьбе): восстанавливаем per-колоночные поля
// ссылок (total_link/plan_link/fact_link/mandatory_link, каждое с своим result_type) --
// ТЕ ЖЕ, что были убраны 14.09 как мёртвый код -- И ДОПОЛНИТЕЛЬНО пробуем указать в
// самой COLUMNS для каждой числовой колонки экспериментальное поле "link" (например,
// { "data": "plan", ..., "link": "plan_link" }) -- ЭТО НЕ ПОДТВЕРЖДЁННЫЙ параметр
// конфигурации колонки, просто гипотеза (может, у виджета такая привязка тоже
// существует, просто раньше не пробовали именно ТАК, per-column, а не per-row) -- лишнее
// поле в объекте конфигурации колонки, по опыту этого тикета, ничего не ломает, даже
// если виджет его не понимает.
//
// ЕСЛИ И ЭТО НЕ СРАБОТАЕТ (что, если честно, ОЖИДАЕМО -- см. предыдущий тест 14.09) --
// значит вывод пользователя может быть таким: "пробовали два разных подхода к
// per-колоночным ссылкам, оба раза виджет 'Табличные данные' всё равно кликает только
// по всей строке -- на уровне ЭТОГО виджета такое, похоже, невозможно". Если тим-лиду
// принципиально важна именно эта функциональность -- возможно, нужен ДРУГОЙ виджет
// (не "Табличные данные") или доработка виджета на стороне платформы, это уже вне
// зоны кода выборки.
// =====================================================================

DEBUG = true;              // На проде поставить false после тестирования
LOG_NAME = "agent";        // TODO: заполнить после создания документа в админке
CUR_OBJECT_ID = 0;         // TODO: заполнить ID документа выборки после её создания в админке (LogAlert защищена try/catch -- забытый 0 не обрушит Run())

// ИСПРАВЛЕНО (16.09.2026): было "mode=matrix_test" -- адрес тестовой страницы, оставшийся
// как TODO-заглушка. Пользователь сообщил, что переименовал реальную (production) страницу
// ТЭП-отчётов с "matrix_test" на "matrix_report". Пока здесь оставался старый адрес, клик
// по строке в "Процент обученных" вёл на СТАРУЮ (тестовую, возможно неактуальную/по-другому
// настроенную) страницу -- это, судя по всему, и есть причина, почему параметр city "не
// доезжал": на новой странице (matrix_report), куда пользователь при ручной проверке заходил
// сам через фильтры, всё работало, а клик по строке уводил на другую, старую страницу.
TEP_REPORT_PAGE_URL = "/view_doc.html?mode=matrix_report";

//-------------------------------------------------------------------------
//              Область функций
//-------------------------------------------------------------------------

function LogAlert(typeLog, message)
{
    try
    {
        tools.call_code_library_method("vtbl_log_lib", "LogAlert", [LOG_NAME, typeLog, CUR_OBJECT_ID, message, DEBUG]);
    }
    catch (_exLog)
    {
        // ничего -- сбой логирования не должен ронять основной код
    }
}

//-------------------------------------------------------------------------
//              ЗАМЕР ПРОИЗВОДИТЕЛЬНОСТИ (18.09.2026, по просьбе тим-лида
//              пользователя -- медленно грузятся страницы после смены фильтров,
//              нужно понять, тормозит БД (SQL/XQuery/tools.open_doc()) или сам код)
//-------------------------------------------------------------------------
// ВРЕМЕННАЯ ДИАГНОСТИКА, идентична версии в HREDU-183_tep_reports.js (см. там подробное
// объяснение -- как читать, риски точности подсчёта секунд, почему всё в try/catch).
// Когда причина тормозов найдена -- блок и все вызовы PerfStart()/PerfCheckpoint() ниже
// можно удалить, на остальную логику файла это не влияет.

PERF_DEBUG = true; // поставь false, чтобы быстро выключить весь этот блок целиком
gPerfStartTime = undefined;
gPerfLastTime = undefined;

function PerfStart()
{
    if (!PERF_DEBUG) { return; }
    gPerfStartTime = PerfNowSafe();
    gPerfLastTime = gPerfStartTime;
    PerfAlertSafe("[ЗАМЕР] СТАРТ. Время: " + PerfFormatTimestamp(gPerfStartTime));
}

function PerfCheckpoint(sLabel)
{
    if (!PERF_DEBUG) { return; }
    var dNow, sMsg;
    dNow = PerfNowSafe();
    sMsg = "[ЗАМЕР] " + sLabel + ". Время сейчас: " + PerfFormatTimestamp(dNow)
        + "; ЭТОТ шаг занял: " + PerfDiffSafe(gPerfLastTime, dNow)
        + "; всего с начала: " + PerfDiffSafe(gPerfStartTime, dNow);
    PerfAlertSafe(sMsg);
    gPerfLastTime = dNow;
}

function PerfNowSafe()
{
    try { return Date(); } catch (_ex) { return undefined; }
}

function PerfFormatTimestamp(dValue)
{
    try { return (dValue != undefined ? StrDate(dValue, true) : "?"); } catch (_ex) { return "?"; }
}

function PerfDiffSafe(dFrom, dTo)
{
    try
    {
        if (dFrom == undefined || dTo == undefined) { return "?"; }
        return String(Int((dTo - dFrom) * 86400)) + " сек";
    }
    catch (_ex)
    {
        return "? сек";
    }
}

function PerfAlertSafe(sMsg)
{
    try { alert(sMsg); } catch (_ex) { /* alert недоступен в этом контексте -- не роняем код */ }
}

function GetRequestUrlSafe()
{
    try { return String(Request.Url); }
    catch (_ex) { return ""; }
}

function GetQueryParam(sUrl, sParamName)
{
    var sAmpMarker, sQMarkMarker, iParamPos, iValueStart, iAmpPos, iValueEnd, sRawValue, iUrlLen;
    iUrlLen = StrLen(sUrl);
    sAmpMarker = "&" + sParamName + "=";
    iParamPos = StrOptSubStrPos(sUrl, sAmpMarker, false);
    if (iParamPos != undefined)
    {
        iValueStart = iParamPos + StrLen(sAmpMarker);
    }
    else
    {
        sQMarkMarker = "?" + sParamName + "=";
        iParamPos = StrOptSubStrPos(sUrl, sQMarkMarker, false);
        if (iParamPos == undefined) { return ""; }
        iValueStart = iParamPos + StrLen(sQMarkMarker);
    }
    iAmpPos = StrOptSubStrPos(sUrl, "&", false, iValueStart);
    iValueEnd = (iAmpPos != undefined ? iAmpPos : iUrlLen);
    sRawValue = StrRangePos(sUrl, iValueStart, iValueEnd);
    try { return UrlDecode(sRawValue); }
    catch (_exDecode) { return sRawValue; }
}

function GetMatrixRows(matrixName)
{
    return ArraySelectAll(XQuery("for $elem in cc_learning_matrices where $elem/name = " + XQueryLiteral(matrixName) + " return $elem"));
}

function GetMatrixElementRows(matrixIds)
{
    return ArraySelectAll(XQuery("for $elem in cc_learning_matrice_elements where MatchSome($elem/cc_learning_matrice_id, (" + ArrayMerge(matrixIds, "This", ",") + ")) and $elem/is_active=true() return $elem"));
}

/*
 * ИЗМЕНЕНО (17.09.2026, HREDU-215 "Правки 1"): у типа документов "Матрицы обучения"
 * (cc_learning_matrice) поле education_method_id УДАЛЕНО (тим-лид пересмотрел архитектуру --
 * см. шапку файла) -- программы теперь заданы ТОЛЬКО через элементы
 * (cc_learning_matrice_element). Раньше эта функция объединяла education_method_id
 * матрицы И её элементов (см. историю правки 15.09.2026, "Int(), Unknown source") --
 * теперь читаем ТОЛЬКО из elementRows, параметр matrixRows убран.
 */
function GetProgramIds(elementRows)
{
    var elementProgramIds, allProgramIds, i;
    elementProgramIds = ArrayExtract(elementRows, "OptInt(This.education_method_id, 0)");
    allProgramIds = [];
    for (i = 0; i < ArrayCount(elementProgramIds); i++) { if (Int(elementProgramIds[i]) > 0) { allProgramIds.push(elementProgramIds[i]); } }
    return ArraySelectDistinct(allProgramIds, "This");
}

/*
 * ПЕРЕИМЕНОВАНО (17.09.2026, было ResolveProgramIds -- возвращала только programIds):
 * теперь возвращает ещё и elementRows -- нужны отдельно для BuildProgramAudienceIndex()
 * ниже (аудитория HREDU-215 теперь считается ПО ЭЛЕМЕНТАМ, не по самой матрице).
 * @returns {Object}   -   { programIds: number[], elementRows: Object[] }.
 */
function ResolveMatrixContext(matrixId, matrixName)
{
    var matrixRows, matrixIds, elementRows, programIds;
    matrixRows = GetMatrixRows(matrixName);
    matrixIds = ArrayExtract(matrixRows, "Int(This.id)");
    if (ArrayCount(matrixIds) == 0)
    {
        throw ("Не найдено ни одной записи cc_learning_matrice с названием [" + matrixName + "]");
    }
    elementRows = GetMatrixElementRows(matrixIds);
    programIds = GetProgramIds(elementRows);
    if (ArrayCount(programIds) == 0)
    {
        throw ("У матрицы [" + matrixName + "] не найдено ни одной активной программы");
    }
    return { programIds: programIds, elementRows: elementRows };
}

function GetActiveCollaboratorRows()
{
    return ArraySelectAll(XQuery("for $elem in collaborators where $elem/is_dismiss=false() return $elem"));
}

function GetPositionIdsByCommonPosition(iCommonPositionFilter)
{
    var positionRows, positionIds, i;
    positionRows = ArraySelectAll(XQuery("for $elem in positions where $elem/position_common_id = " + iCommonPositionFilter + " return $elem"));
    positionIds = [];
    for (i = 0; i < ArrayCount(positionRows); i++) { positionIds.push(Int(positionRows[i].id)); }
    return positionIds;
}

function IdArrayContains(idArray, value)
{
    var i;
    for (i = 0; i < ArrayCount(idArray); i++)
    {
        if (Int(idArray[i]) == Int(value)) { return true; }
    }
    return false;
}

function GetMacroregionRows()
{
    var sqlText;
    sqlText = "";
    sqlText = sqlText + "select cs.id,\r\n";
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_2ewj'']/value)[1]', 'varchar(max)') as macroregion\r\n";
    sqlText = sqlText + "from collaborators cs\r\n";
    sqlText = sqlText + "inner join collaborator c on c.id = cs.id\r\n";
    sqlText = sqlText + "where cs.is_dismiss != 1";
    return ArraySelectAll(XQuery("sql:" + sqlText));
}

/*
 * НОВОЕ (14.09.2026): город -- custom_elem "sity" (имя поля подтверждено пользователем
 * реальным XML документа collaborator). Та же схема, что GetMacroregionRows()/
 * GetMirCodeRows() -- один SQL на всех активных сотрудников сразу.
 * @returns {Object[]}   -   Массив {id, sity}.
 */
function GetCityRows()
{
    LogAlert(1, "GetCityRows(). НАЧАЛО");
    var sqlText, rows;
    sqlText = "";
    sqlText = sqlText + "select cs.id,\r\n";
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''sity'']/value)[1]', 'varchar(max)') as sity\r\n";
    sqlText = sqlText + "from collaborators cs\r\n";
    sqlText = sqlText + "inner join collaborator c on c.id = cs.id\r\n";
    sqlText = sqlText + "where cs.is_dismiss != 1";
    rows = ArraySelectAll(XQuery("sql:" + sqlText));
    LogAlert(1, "GetCityRows(). Строк: " + ArrayCount(rows));
    LogAlert(1, "GetCityRows(). КОНЕЦ");
    return rows;
}

function SortRowsById(rows)
{
    return ArraySort(rows, "Int(This.id)", "+");
}

function BinarySearchById(sortedRows, targetId)
{
    var lo, hi, mid, midId, iTarget;
    iTarget = Int(targetId);
    lo = 0;
    hi = ArrayCount(sortedRows) - 1;
    while (lo <= hi)
    {
        mid = Int((lo + hi) / 2);
        midId = Int(sortedRows[mid].id);
        if (midId == iTarget) { return sortedRows[mid]; }
        else if (midId < iTarget) { lo = mid + 1; }
        else { hi = mid - 1; }
    }
    return undefined;
}

function SortIdArray(idArray)
{
    return ArraySort(idArray, "Int(This)", "+");
}

function IdArrayContainsSorted(sortedIdArray, value)
{
    var lo, hi, mid, midVal, iTarget;
    iTarget = Int(value);
    lo = 0;
    hi = ArrayCount(sortedIdArray) - 1;
    while (lo <= hi)
    {
        mid = Int((lo + hi) / 2);
        midVal = Int(sortedIdArray[mid]);
        if (midVal == iTarget) { return true; }
        else if (midVal < iTarget) { lo = mid + 1; }
        else { hi = mid - 1; }
    }
    return false;
}

function FindCity(sortedCityRows, collaboratorID)
{
    var cityRow, sCity;
    cityRow = BinarySearchById(sortedCityRows, collaboratorID);
    sCity = (cityRow != undefined && cityRow.sity != undefined ? String(cityRow.sity) : "");
    return (sCity != "" ? sCity : "(без города)");
}

function FindMacroregion(sortedMacroRows, collaboratorID)
{
    var macroRow;
    macroRow = BinarySearchById(sortedMacroRows, collaboratorID);
    return (macroRow != undefined && macroRow.macroregion != undefined ? String(macroRow.macroregion) : "");
}

function SortDateRowsByCollaboratorId(dateRows)
{
    return ArraySort(dateRows, "Int(This.collaborator_id)", "+");
}

function GetMirCodeRows()
{
    var sqlText;
    sqlText = "";
    sqlText = sqlText + "select cs.id,\r\n";
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_mir_codes'']/value)[1]', 'varchar(max)') as mir_codes\r\n";
    sqlText = sqlText + "from collaborators cs\r\n";
    sqlText = sqlText + "inner join collaborator c on c.id = cs.id\r\n";
    sqlText = sqlText + "where cs.is_dismiss != 1";
    return ArraySelectAll(XQuery("sql:" + sqlText));
}

function ExtractMirCodes(rawValue)
{
    var parts, fields, codes, i;
    codes = [];
    parts = ArrayDirect(ArraySelect(String(rawValue).split("|"), "This != ''"));
    for (i = 0; i < ArrayCount(parts); i++)
    {
        fields = ArrayDirect(ArraySelect(String(parts[i]).split("#"), "This != ''"));
        if (ArrayCount(fields) > 0) { codes.push(String(fields[0])); }
    }
    return codes;
}

function CollaboratorHasMirCode(sortedMirCodeRows, collaboratorID, mirCodeFilter)
{
    var row, codes;
    row = BinarySearchById(sortedMirCodeRows, collaboratorID);
    if (row == undefined) { return false; }
    codes = ExtractMirCodes(row.mir_codes);
    return (ArrayOptFind(codes, "String(This) == String(mirCodeFilter)") != undefined);
}

function ResolveMirCodeText(iMirCodeID)
{
    if (OptInt(iMirCodeID, 0) <= 0) { return ""; }
    try { return String(tools.open_doc(Int(iMirCodeID)).TopElem.name); }
    catch (_ex) { return ""; }
}

function ResolveProgramText(iProgramId)
{
    if (OptInt(iProgramId, 0) <= 0) { return "(без программы)"; }
    try { return String(tools.open_doc(Int(iProgramId)).TopElem.name); }
    catch (_ex) { return "id=" + iProgramId; }
}

function FindProgramName(programNames, iProgramId)
{
    var row;
    row = ArrayOptFind(programNames, "Int(This.id) == Int(iProgramId)");
    return (row != undefined ? String(row.name) : "id=" + iProgramId);
}

function BuildProgramAudienceIndex(elementRows, programIds)
{
    var index, i, j, programId, elem, segments, iPos, iMirCode, sMirCodeText, allowedPositionIds;
    index = [];
    for (i = 0; i < ArrayCount(programIds); i++)
    {
        programId = programIds[i];
        segments = [];
        for (j = 0; j < ArrayCount(elementRows); j++)
        {
            elem = elementRows[j];
            if (OptInt(elem.education_method_id, 0) == Int(programId))
            {
                iPos = OptInt(elem.position_common_id, 0);
                iMirCode = OptInt(elem.mir_code_id, 0);
                allowedPositionIds = (iPos > 0 ? SortIdArray(GetPositionIdsByCommonPosition(iPos)) : []);
                sMirCodeText = ResolveMirCodeText(iMirCode);
                segments.push({ positionCommonId: iPos, allowedPositionIds: allowedPositionIds, mirCodeText: sMirCodeText });
            }
        }
        index.push({ programId: Int(programId), segments: segments });
    }
    return index;
}

function SummarizeAudienceIndex(audienceIndex)
{
    var i, j, prog, segCount, posSum, posMax, segLen;
    segCount = 0;
    posSum = 0;
    posMax = 0;
    for (i = 0; i < ArrayCount(audienceIndex); i++)
    {
        prog = audienceIndex[i];
        segLen = ArrayCount(prog.segments);
        segCount = segCount + segLen;
        for (j = 0; j < segLen; j++)
        {
            posSum = posSum + ArrayCount(prog.segments[j].allowedPositionIds);
            if (ArrayCount(prog.segments[j].allowedPositionIds) > posMax) { posMax = ArrayCount(prog.segments[j].allowedPositionIds); }
        }
    }
    return "Сегментов всего: " + segCount + "; суммарный размер allowedPositionIds: " + posSum + "; максимальный размер одного сегмента: " + posMax;
}

function FindProgramAudienceSegments(audienceIndex, programId)
{
    var row;
    row = ArrayOptFind(audienceIndex, "Int(This.programId) == Int(programId)");
    return (row != undefined ? row.segments : []);
}

function CollaboratorInProgramAudience(collaboratorRow, segments, sortedMirCodeRows)
{
    var i, seg, positionOk, mirCodeOk;
    for (i = 0; i < ArrayCount(segments); i++)
    {
        seg = segments[i];
        positionOk = (seg.positionCommonId <= 0 || IdArrayContainsSorted(seg.allowedPositionIds, OptInt(collaboratorRow.position_id, 0)));
        mirCodeOk = (seg.mirCodeText == "" || CollaboratorHasMirCode(sortedMirCodeRows, Int(collaboratorRow.id), seg.mirCodeText));
        if (positionOk && mirCodeOk) { return true; }
    }
    return false;
}

function ApplyManualFilters(collaboratorRows, iPositionFilter, sMacroregionFilter, sMirCodeFilter, sortedMacroRows, sortedMirCodeRows)
{
    var allowedPositionIds, filteredRows, i, macroRow;

    filteredRows = collaboratorRows;

    if (iPositionFilter > 0)
    {
        allowedPositionIds = SortIdArray(GetPositionIdsByCommonPosition(iPositionFilter));
        collaboratorRows = filteredRows;
        filteredRows = [];
        for (i = 0; i < ArrayCount(collaboratorRows); i++)
        {
            if (IdArrayContainsSorted(allowedPositionIds, OptInt(collaboratorRows[i].position_id, 0))) { filteredRows.push(collaboratorRows[i]); }
        }
    }

    if (sMacroregionFilter != "")
    {
        collaboratorRows = filteredRows;
        filteredRows = [];
        for (i = 0; i < ArrayCount(collaboratorRows); i++)
        {
            macroRow = BinarySearchById(sortedMacroRows, Int(collaboratorRows[i].id));
            if (macroRow != undefined && String(macroRow.macroregion) == sMacroregionFilter) { filteredRows.push(collaboratorRows[i]); }
        }
    }

    if (sMirCodeFilter != "")
    {
        collaboratorRows = filteredRows;
        filteredRows = [];
        for (i = 0; i < ArrayCount(collaboratorRows); i++)
        {
            if (CollaboratorHasMirCode(sortedMirCodeRows, Int(collaboratorRows[i].id), sMirCodeFilter)) { filteredRows.push(collaboratorRows[i]); }
        }
    }

    return filteredRows;
}

function GetCompletionDateRows(programIds)
{
    var sqlText;
    sqlText = "";
    sqlText = sqlText + "select ec.collaborator_id, e.education_method_id, min(ec.start_date) as first_date\r\n";
    sqlText = sqlText + "from event_collaborators ec\r\n";
    sqlText = sqlText + "join events e on e.id = ec.event_id\r\n";
    sqlText = sqlText + "where e.education_method_id in (" + ArrayMerge(programIds, "This", ",") + ")\r\n";
    sqlText = sqlText + "group by ec.collaborator_id, e.education_method_id";
    return ArraySelectAll(XQuery("sql:" + sqlText));
}

function FindCompletionDate(sortedDateRows, collaboratorID, programID)
{
    var lo, hi, mid, midId, iTarget, iProgram, startIdx, i, n;

    iTarget = Int(collaboratorID);
    iProgram = Int(programID);

    lo = 0;
    hi = ArrayCount(sortedDateRows) - 1;
    startIdx = -1;
    while (lo <= hi)
    {
        mid = Int((lo + hi) / 2);
        midId = Int(sortedDateRows[mid].collaborator_id);
        if (midId == iTarget)
        {
            startIdx = mid;
            hi = mid - 1;
        }
        else if (midId < iTarget) { lo = mid + 1; }
        else { hi = mid - 1; }
    }

    if (startIdx == -1) { return ""; }

    n = ArrayCount(sortedDateRows);
    for (i = startIdx; i < n && Int(sortedDateRows[i].collaborator_id) == iTarget; i++)
    {
        if (Int(sortedDateRows[i].education_method_id) == iProgram)
        {
            return StrDate(Date(sortedDateRows[i].first_date), false);
        }
    }
    return "";
}

function GetOrCreateCityProgramAcc(acc, sCity, iProgramId, sProgramName, sMacroregion)
{
    var i;
    for (i = 0; i < ArrayCount(acc); i++)
    {
        if (acc[i].city == sCity && Int(acc[i].programId) == Int(iProgramId)) { return acc[i]; }
    }
    var newAcc;
    newAcc = { city: sCity, programId: Int(iProgramId), programName: sProgramName, macroregion: sMacroregion, total: 0, mandatory: 0, fact: 0 };
    acc.push(newAcc);
    return newAcc;
}

function FormatPercent(nFact, nPlan)
{
    var iFact, iPlan, iRounded;
    if (nPlan <= 0) { return "-"; }
    iFact = Int(nFact);
    iPlan = Int(nPlan);
    iRounded = Int((iFact * 100 + Int(iPlan / 2)) / iPlan);
    return String(iRounded) + "%";
}

function HtmlEscapeAmp(sUrl)
{
    var sResult, iPos, iUrlLen, iSearchStart;
    sResult = "";
    iSearchStart = 0;
    iUrlLen = StrLen(sUrl);
    while (true)
    {
        iPos = StrOptSubStrPos(sUrl, "&", false, iSearchStart);
        if (iPos == undefined)
        {
            sResult = sResult + StrRangePos(sUrl, iSearchStart, iUrlLen);
            break;
        }
        sResult = sResult + StrRangePos(sUrl, iSearchStart, iPos) + "&amp;";
        iSearchStart = iPos + 1;
    }
    return sResult;
}

/*
 * Строит ссылку на страницу ТЭП-отчётов с нужным набором параметров. sResultType теперь
 * ИСПОЛЬЗУЕТСЯ ПО НАЗНАЧЕНИЮ (21.09.2026) -- раньше все вызовы жёстко передавали "total"
 * (см. историю "ЗАКРЫТО (14.09.2026, КЛИКАБЕЛЬНОСТЬ)" ниже), теперь для КАЖДОЙ колонки
 * (total/plan/fact/mandatory) строится СВОЯ ссылка со своим result_type -- см.
 * per-column-эксперимент в шапке файла и в Run()/COLUMNS ниже.
 * @param {number} iMatrixId
 * @param {string} sMacroregionFilter
 * @param {string} sMirCodeFilter
 * @param {number} iPositionFilter
 * @param {number} iProgramId
 * @param {string} sResultType   -   "total"|"plan"|"fact"|"mandatory"
 * @param {string} sCity
 * @returns {string}
 */
function BuildTepLink(iMatrixId, sMacroregionFilter, sMirCodeFilter, iPositionFilter, iProgramId, sResultType, sCity)
{
    var oQueryParams, sQueryString, sSeparator;
    oQueryParams = {
        matrix_id: String(iMatrixId),
        macroregion: sMacroregionFilter,
        mir_code: sMirCodeFilter,
        position_common_id: String(iPositionFilter),
        program_id: String(iProgramId),
        result_type: sResultType,
        city: sCity
    };
    sQueryString = UrlEncodeQuery(oQueryParams);
    sSeparator = (StrOptSubStrPos(TEP_REPORT_PAGE_URL, "?", false) != undefined ? "&" : "?");
    return HtmlEscapeAmp(TEP_REPORT_PAGE_URL + sSeparator + sQueryString);
}

//-------------------------------------------------------------------------
//              Точка входа
//-------------------------------------------------------------------------

function Run()
{
    LogAlert(2, "Run(). НАЧАЛО (Процент обученных)");
    var sFullUrl, matrixId, matrixDoc, matrixName, iProgramFilter, sMacroregionFilter, sMirCodeFilter, iPositionFilter;
    var matrixContext, elementRows, audienceIndex, segments;
    var programIds, filteredProgramIds, programNames, i, j;
    var activeRows, manualFilteredRows;
    var macroRows, mirCodeRows, cityRows, dateRows;
    var macroSorted, citySorted, dateSorted;
    var mirCodeSorted;
    var acc, cityProgramAcc, sCity, sProgramName, sDate, row;
    var totalAcc, resultRows, id;
    var sMacroregionForRow, sLinkMacroregion;
    var sLinkMacro; // ДОБАВЛЕНО (21.09.2026) -- используется при сборке per-колоночных ссылок ниже

    RESULT = [];

    try
    {
        PerfStart();

        sFullUrl = GetRequestUrlSafe();
        LogAlert(1, "Run(). Request.Url = [" + sFullUrl + "]");

        matrixId = OptInt(GetQueryParam(sFullUrl, "matrix_id"), 0);
        iProgramFilter = OptInt(GetQueryParam(sFullUrl, "program_id"), 0);
        sMacroregionFilter = GetQueryParam(sFullUrl, "macroregion");
        sMirCodeFilter = GetQueryParam(sFullUrl, "mir_code");
        iPositionFilter = OptInt(GetQueryParam(sFullUrl, "position_common_id"), 0);

        LogAlert(1, "Run(). matrixId=" + matrixId + " programFilter=" + iProgramFilter
            + " macroregionFilter=[" + sMacroregionFilter + "] mirCodeFilter=[" + sMirCodeFilter
            + "] positionCommonIdFilter=" + iPositionFilter);

        PerfCheckpoint("Разбор Request.Url и всех фильтров -- ЧИСТЫЙ КОД, без SQL");

        if (matrixId == 0)
        {
            throw ("Не передан matrix_id -- выбранная пользователем матрица обучения");
        }

        matrixDoc = tools.open_doc(matrixId).TopElem;
        matrixName = String(matrixDoc.name);
        PerfCheckpoint("tools.open_doc(matrixId) -- открытие документа матрицы -- БД/документ");

        matrixContext = ResolveMatrixContext(matrixId, matrixName);
        programIds = matrixContext.programIds;
        elementRows = matrixContext.elementRows;
        PerfCheckpoint("ResolveMatrixContext() -- GetMatrixRows()+GetMatrixElementRows()+GetProgramIds() -- SQL/XQuery");

        if (iProgramFilter > 0)
        {
            filteredProgramIds = [];
            for (i = 0; i < ArrayCount(programIds); i++)
            {
                if (Int(programIds[i]) == iProgramFilter) { filteredProgramIds.push(programIds[i]); }
            }
            programIds = filteredProgramIds;
            if (ArrayCount(programIds) == 0)
            {
                throw ("Программа [" + iProgramFilter + "] не найдена среди программ выбранной матрицы");
            }
        }

        activeRows = GetActiveCollaboratorRows();
        PerfCheckpoint("GetActiveCollaboratorRows() -- SQL/XQuery по всем активным сотрудникам");
        macroRows = GetMacroregionRows();
        PerfCheckpoint("GetMacroregionRows() -- SQL по макрорегионам сотрудников");
        macroSorted = SortRowsById(macroRows);
        PerfCheckpoint("SortRowsById(macroRows) -- сортировка для бинарного поиска по макрорегионам -- ЧИСТЫЙ КОД, O(n log n)");
        cityRows = GetCityRows();
        PerfCheckpoint("GetCityRows() -- SQL по городам сотрудников");
        citySorted = SortRowsById(cityRows);
        PerfCheckpoint("SortRowsById(cityRows) -- сортировка для бинарного поиска по городам -- ЧИСТЫЙ КОД, O(n log n)");
        dateRows = GetCompletionDateRows(programIds);
        PerfCheckpoint("GetCompletionDateRows() -- SQL по датам прохождения программ");
        dateSorted = SortDateRowsByCollaboratorId(dateRows);
        PerfCheckpoint("SortDateRowsByCollaboratorId(dateRows) -- сортировка для бинарного поиска по датам -- ЧИСТЫЙ КОД, O(n log n)");
        mirCodeRows = GetMirCodeRows();
        PerfCheckpoint("GetMirCodeRows() -- SQL по мир-кодам сотрудников");
        mirCodeSorted = SortRowsById(mirCodeRows);
        PerfCheckpoint("SortRowsById(mirCodeRows) -- сортировка для бинарного поиска по мир-кодам -- ЧИСТЫЙ КОД, O(n log n)");

        programNames = [];
        for (j = 0; j < ArrayCount(programIds); j++)
        {
            programNames.push({ id: Int(programIds[j]), name: ResolveProgramText(programIds[j]) });
        }
        PerfCheckpoint("Цикл ResolveProgramText() -- N x tools.open_doc() по программам матрицы -- БД/документы (потенциальный N+1)");

        audienceIndex = BuildProgramAudienceIndex(elementRows, programIds);
        PerfCheckpoint("BuildProgramAudienceIndex() -- аудитория по элементам, внутри может быть SQL (GetPositionIdsByCommonPosition() на сегмент). " + SummarizeAudienceIndex(audienceIndex));

        manualFilteredRows = ApplyManualFilters(activeRows, iPositionFilter, sMacroregionFilter, sMirCodeFilter, macroSorted, mirCodeSorted);
        LogAlert(1, "Run(). Сотрудников после ручных фильтров (база и для total/mandatory, и для fact): " + ArrayCount(manualFilteredRows));
        PerfCheckpoint("ApplyManualFilters() -- ручные фильтры пользователя -- ЧИСТЫЙ КОД (может дёргать SQL внутри при фильтре по должности)");

        acc = [];

        for (i = 0; i < ArrayCount(manualFilteredRows); i++)
        {
            sCity = FindCity(citySorted, Int(manualFilteredRows[i].id));
            sMacroregionForRow = FindMacroregion(macroSorted, Int(manualFilteredRows[i].id));
            for (j = 0; j < ArrayCount(programIds); j++)
            {
                segments = FindProgramAudienceSegments(audienceIndex, programIds[j]);
                if (CollaboratorInProgramAudience(manualFilteredRows[i], segments, mirCodeSorted))
                {
                    sProgramName = FindProgramName(programNames, programIds[j]);
                    cityProgramAcc = GetOrCreateCityProgramAcc(acc, sCity, programIds[j], sProgramName, sMacroregionForRow);
                    sDate = FindCompletionDate(dateSorted, Int(manualFilteredRows[i].id), programIds[j]);
                    cityProgramAcc.total = cityProgramAcc.total + 1;
                    if (sDate == "") { cityProgramAcc.mandatory = cityProgramAcc.mandatory + 1; }
                }
            }
        }
        PerfCheckpoint("Цикл total/mandatory (сотрудники x программы, с проверкой аудитории) -- ЧИСТЫЙ КОД, без SQL. Сотрудников: " + ArrayCount(manualFilteredRows) + "; программ: " + ArrayCount(programIds) + "; пар всего: " + (ArrayCount(manualFilteredRows) * ArrayCount(programIds)));

        for (i = 0; i < ArrayCount(manualFilteredRows); i++)
        {
            sCity = FindCity(citySorted, Int(manualFilteredRows[i].id));
            sMacroregionForRow = FindMacroregion(macroSorted, Int(manualFilteredRows[i].id));
            for (j = 0; j < ArrayCount(programIds); j++)
            {
                segments = FindProgramAudienceSegments(audienceIndex, programIds[j]);
                sDate = FindCompletionDate(dateSorted, Int(manualFilteredRows[i].id), programIds[j]);
                if (sDate != "" && CollaboratorInProgramAudience(manualFilteredRows[i], segments, mirCodeSorted))
                {
                    sProgramName = FindProgramName(programNames, programIds[j]);
                    cityProgramAcc = GetOrCreateCityProgramAcc(acc, sCity, programIds[j], sProgramName, sMacroregionForRow);
                    cityProgramAcc.fact = cityProgramAcc.fact + 1;
                }
            }
        }
        PerfCheckpoint("Цикл fact (сотрудники x программы, с проверкой аудитории) -- ЧИСТЫЙ КОД, без SQL. Пар всего: " + (ArrayCount(manualFilteredRows) * ArrayCount(programIds)));

        var iOuter, iInner, tmpAcc;
        for (iOuter = 0; iOuter < ArrayCount(acc) - 1; iOuter++)
        {
            for (iInner = 0; iInner < ArrayCount(acc) - 1 - iOuter; iInner++)
            {
                if (acc[iInner].programName > acc[iInner + 1].programName
                    || (acc[iInner].programName == acc[iInner + 1].programName && acc[iInner].city > acc[iInner + 1].city))
                {
                    tmpAcc = acc[iInner];
                    acc[iInner] = acc[iInner + 1];
                    acc[iInner + 1] = tmpAcc;
                }
            }
        }
        PerfCheckpoint("Сортировка пузырьком (O(n^2), " + ArrayCount(acc) + " строк город x программа) -- ЧИСТЫЙ КОД");

        resultRows = [];
        id = 0;
        totalAcc = { total: 0, mandatory: 0, fact: 0 };
        for (i = 0; i < ArrayCount(acc); i++)
        {
            id = id + 1;
            row = acc[i];
            sLinkMacro = (row.macroregion != "" ? row.macroregion : sMacroregionFilter);
            resultRows.push({
                id: id,
                city: row.city,
                program: row.programName,
                total: row.total,
                plan: row.total, // План = Общее, см. "РЕШЕНИЯ" в шапке
                fact: row.fact,
                percent: FormatPercent(row.fact, row.total),
                mandatory: row.mandatory,
                // ПОДТВЕРЖДЕНО (14.09.2026, реальный тест пользователя): виджет "Табличные
                // данные" различает клик ТОЛЬКО по строке целиком -- один "link" на всю
                // строку. Оставляем row-level "link" как ФОЛЛБЭК/дефолт (ведёт в режим
                // "total") -- на случай, если per-колоночный эксперимент ниже не сработает
                // (см. шапку файла), хотя бы обычный клик по строке продолжит работать
                // так же, как и раньше.
                link: BuildTepLink(matrixId, sLinkMacro, sMirCodeFilter, iPositionFilter, row.programId, "total", row.city),
                // ДОБАВЛЕНО (21.09.2026) -- ПОПЫТКА per-колоночных ссылок по просьбе
                // тим-лида пользователя. total_link/plan_link/fact_link/mandatory_link --
                // ТЕ ЖЕ ИМЕНА полей, что были опробованы 14.09.2026 и тогда не сработали
                // (виджет кликал только по row-level "link" независимо от колонки) -- см.
                // подробное объяснение в шапке файла. Восстановлены для честной повторной
                // проверки по прямой просьбе (тим-лид хочет посмотреть сам). Если снова не
                // сработает -- значит ограничение подтверждено дважды, это ограничение
                // самого виджета "Табличные данные", а не что-то, что можно починить в
                // этой выборке.
                total_link: BuildTepLink(matrixId, sLinkMacro, sMirCodeFilter, iPositionFilter, row.programId, "total", row.city),
                plan_link: BuildTepLink(matrixId, sLinkMacro, sMirCodeFilter, iPositionFilter, row.programId, "plan", row.city),
                fact_link: BuildTepLink(matrixId, sLinkMacro, sMirCodeFilter, iPositionFilter, row.programId, "fact", row.city),
                mandatory_link: BuildTepLink(matrixId, sLinkMacro, sMirCodeFilter, iPositionFilter, row.programId, "mandatory", row.city)
            });
            totalAcc.total = totalAcc.total + row.total;
            totalAcc.mandatory = totalAcc.mandatory + row.mandatory;
            totalAcc.fact = totalAcc.fact + row.fact;
        }

        id = id + 1;
        resultRows.push({
            id: id,
            city: "Общий итог",
            program: "-",
            total: totalAcc.total,
            plan: totalAcc.total,
            fact: totalAcc.fact,
            percent: FormatPercent(totalAcc.fact, totalAcc.total),
            mandatory: totalAcc.mandatory,
            link: BuildTepLink(matrixId, sMacroregionFilter, sMirCodeFilter, iPositionFilter, 0, "total", ""),
            total_link: BuildTepLink(matrixId, sMacroregionFilter, sMirCodeFilter, iPositionFilter, 0, "total", ""),
            plan_link: BuildTepLink(matrixId, sMacroregionFilter, sMirCodeFilter, iPositionFilter, 0, "plan", ""),
            fact_link: BuildTepLink(matrixId, sMacroregionFilter, sMirCodeFilter, iPositionFilter, 0, "fact", ""),
            mandatory_link: BuildTepLink(matrixId, sMacroregionFilter, sMirCodeFilter, iPositionFilter, 0, "mandatory", "")
        });

        PerfCheckpoint("Сборка resultRows + построение BuildTepLink() x5 на каждую строку (row-level + 4 per-колоночных) -- ЧИСТЫЙ КОД");

        RESULT = resultRows;
        LogAlert(2, "Run(). Готово. Строк (город x программа): " + (ArrayCount(resultRows) - 1) + " + итоговая строка");
        PerfCheckpoint("Run() -- ГОТОВО (успех), строк: " + (ArrayCount(resultRows) - 1) + " + итоговая");
    }
    catch (_ex)
    {
        RESULT = [];
        ERROR = 1;
        MESSAGE = ExtractUserError(_ex);
        LogAlert(4, "Run(). ОШИБКА: " + MESSAGE);
        PerfCheckpoint("Run() -- ОШИБКА: " + MESSAGE);
    }
    LogAlert(2, "Run(). КОНЕЦ");
}

Run();

// ИСТОРИЯ (14.09.2026, КЛИКАБЕЛЬНОСТЬ, для справки): реальный тест пользователя ТОГДА
// подтвердил, что виджет "Табличные данные" различает клик ТОЛЬКО по строке целиком --
// независимо от того, по какой колонке кликнули, срабатывает один и тот же "link" всей
// строки. Поля total_link/plan_link/fact_link/mandatory_link были тогда УБРАНЫ как
// мёртвый код.
//
// ДОБАВЛЕНО (21.09.2026, по прямой просьбе тим-лида пользователя -- см. подробное
// объяснение эксперимента в шапке файла): поля total_link/plan_link/fact_link/
// mandatory_link ВОССТАНОВЛЕНЫ в Run() выше -- ждём результата реального теста. Ниже,
// экспериментально, у каждой числовой колонки указано ДОПОЛНИТЕЛЬНОЕ (неподтверждённое)
// свойство "link" со ссылкой на соответствующее скрытое поле -- НЕ задокументированная
// возможность виджета, просто гипотеза "а вдруг он это читает". Если виджет её не
// поддерживает -- лишнее свойство в объекте конфигурации колонки не должно ничего
// сломать (по опыту этого тикета лишние поля в объектах платформа молча игнорирует).
//
// ЕСЛИ ПОСЛЕ ЭТОГО ТЕСТА КЛИК ПО ЛЮБОЙ КОЛОНКЕ ВСЁ РАВНО ВЕДЁТ ТУДА ЖЕ, ЧТО И КЛИК ПО
// ГОРОДУ/ПРОГРАММЕ (т.е. всегда на "link", а не на "*_link" своей колонки) -- значит
// вывод для тим-лида: "пробовали дважды двумя разными способами (14.09 и 21.09), виджет
// 'Табличные данные' поддерживает только ОДНУ ссылку на всю строку, per-колоночные клики
// на уровне этой выборки/этого виджета не реализуемы".
COLUMNS = [
    { "data": "id", "editable": true, "hidden": true, "sortable": false },
    { "data": "link", "hidden": true, "editable": false, "sortable": false }, // row-level link (фоллбэк/дефолт -- режим "total")
    { "data": "total_link", "hidden": true, "editable": false, "sortable": false }, // ДОБАВЛЕНО 21.09.2026 -- эксперимент
    { "data": "plan_link", "hidden": true, "editable": false, "sortable": false },  // ДОБАВЛЕНО 21.09.2026 -- эксперимент
    { "data": "fact_link", "hidden": true, "editable": false, "sortable": false },  // ДОБАВЛЕНО 21.09.2026 -- эксперимент
    { "data": "mandatory_link", "hidden": true, "editable": false, "sortable": false }, // ДОБАВЛЕНО 21.09.2026 -- эксперимент
    { "data": "city", "title": "Город", "type": "string", "editable": false, "sortable": true },
    { "data": "program", "title": "Учебная программа", "type": "string", "editable": false, "sortable": true },
    // "link": "..." ниже -- НЕПОДТВЕРЖДЁННОЕ свойство, гипотеза (см. комментарий выше).
    { "data": "total", "title": "Общее кол-во сотрудников", "type": "integer", "editable": false, "sortable": true, "link": "total_link" },
    { "data": "plan", "title": "План", "type": "integer", "editable": false, "sortable": true, "link": "plan_link" },
    { "data": "fact", "title": "Факт", "type": "integer", "editable": false, "sortable": true, "link": "fact_link" },
    { "data": "percent", "title": "Процент", "type": "string", "editable": false, "sortable": false },
    { "data": "mandatory", "title": "Обязательно к прохождению", "type": "integer", "editable": false, "sortable": true, "link": "mandatory_link" }
];
