
// HREDU-183. ТЭП_общее_кол-во / ТЭП_план / ТЭП_факт / ТЭП_обязательно -- выборка для
// Табличных данных. Один файл, четыре режима через параметр result_type -- по образцу
// education_accept_event_card (там тоже один result_type переключает поведение одной
// выборки). Построено на основе HREDU-181_vostok_polny_spisok_draft.js -- та же матрица/
// программы/сотрудники/даты/фильтры, плюс новое: аудитория матрицы и разбиение по
// результату (см. ниже).
//
// ТЗ (HREDU-183, полный текст прислан пользователем 10.09.2026):
//   Общее кол-во сотрудников -- все, кто подходят под матрицу (должность + мир-код).
//   План -- кол-во сотрудников которые подходят под матрицу и у кого уже наступил
//     период прохождения тренинга.
//   Факт -- кол-во сотрудников, прошедших тренинг (НЕ ЗАВИСИМО от условий матрицы).
//   Обязательно к прохождению -- все, кто подходят под матрицу и при этом ещё не
//     проходили тренинг.
//   % обученных -- факт/план (уточнено с пользователем 10.09.2026 -- в тексте ТЗ
//     написано "план/факт", но по смыслу метрики и по факту подтверждения это факт/план;
//     сам % не входит в эту выборку -- это отдельный лёгкий расчёт для "Процент обученных",
//     не список сотрудников).
//
// РЕШЕНИЯ, ПРИНЯТЫЕ С ПОЛЬЗОВАТЕЛЕМ (10.09.2026), ЧАСТИЧНО ПЕРЕСМОТРЕНЫ (17.09.2026,
// HREDU-215 "Правки 1" -- см. ниже):
//   1. Аудитория (должность + мир-код) -- РЕАЛИЗУЕМ. ИЗМЕНЕНО (17.09.2026, HREDU-215
//      "Правки 1", по прямому указанию тим-лида пользователя -- "ошиблись в архитектуре"):
//      РАНЬШЕ поля position_common_id/mir_code_id жили НА САМОЙ МАТРИЦЕ (cc_learning_matrice),
//      аудитория была ОДНА на всю матрицу. ТЕПЕРЬ эти поля УДАЛЕНЫ с типа документа
//      "Матрицы обучения" и ДОБАВЛЕНЫ на тип документа "Элементы матриц обучения"
//      (cc_learning_matrice_element, у которых уже были education_method_id/
//      start_study_period/end_study_period) -- значит аудитория теперь СВОЯ У КАЖДОГО
//      ЭЛЕМЕНТА (по факту -- у каждой программы, а если у одной программы несколько
//      элементов с разными position_common_id -- у неё НЕСКОЛЬКО аудиторий, объединяемых
//      через ИЛИ). См. BuildProgramAudienceIndex()/CollaboratorInProgramAudience() ниже.
//      Это МАНДАТОРНОЕ условие "кому вообще адресована ЭТА программа" -- отдельная вещь
//      от РУЧНЫХ фильтров пользователя (те же имена полей, но разный смысл): ручные
//      фильтры дополнительно СУЖАЮТ то, что уже прошло через аудиторию, а не заменяют её.
//   2. "Период прохождения тренинга" (нужен для честного "План") -- НЕ РЕАЛИЗОВАН. На
//      элементе есть start_study_period=1/end_study_period=4 (числа, не даты -- см.
//      диагностику), но неясно: единицы измерения и от какой даты сотрудника отсчитывать.
//      Пользователь решил не тратить на это время сейчас -- УПРОЩЕНИЕ: План = Общее (без
//      доп. фильтра по периоду). Это совпадает с тем, что мы уже видели на тестовых
//      данных пользователя раньше в этом тикете (план и общее количество были равны).
//      ОТКРЫТЫЙ ВОПРОС, вернуться при необходимости -- аналогично открытым вопросам
//      №1-3 в HREDU-181_vostok_polny_spisok_draft.js.
//   3. Факт -- ПЕРЕСМОТРЕНО ПОВТОРНО (17.09.2026, тот же день, что и п.1, но отдельным
//      уточнением от пользователя -- см. AskUserQuestion): раньше "не зависимо от условий
//      матрицы" понималось БУКВАЛЬНО -- фильтр аудитории для "fact" вообще не применялся
//      (сотрудник мог быть кем угодно по должности). Реальный тест (матрица
//      "Менеджер"/"Стандарт менеджер", город Воронеж) показал, что это даёт СТРАННЫЙ
//      результат -- в "Факт" попадали сотрудники СОВСЕМ ДРУГИХ должностей (Экономисты),
//      просто когда-то прошедшие ту же программу по не связанной с этой матрицей причине
//      (программа -- общий каталог, не привязана к конкретной матрице). ПОДТВЕРЖДЕНО
//      пользователем: "Факт" ТЕПЕРЬ ТОЖЕ должен ограничиваться аудиторией (должность+
//      мир-код ХОТЯ БЫ ОДНОГО элемента этой программы) -- "не зависимо от условий
//      матрицы" означает "не зависимо от ПЕРИОДА" (см. п.2 -- он всё равно не
//      реализован), а НЕ "вообще без каких-либо условий". Базовый пул для факта -- все
//      активные сотрудники (как и раньше), ручные фильтры пользователя применяются, ПЛЮС
//      теперь и аудитория программы -- см. п. "б" ниже.
//   4. Обязательно -- аудитория программы (как Общее) МИНУС те, кто прошёл (т.е. строки
//      с пустой датой прохождения).
//
// ИТОГОВАЯ АРХИТЕКТУРА: сначала строим ряды "сотрудник x программа" ТОЧНО как в
// HREDU-181 (программы матрицы, активные сотрудники, дата прохождения, все 4 ручных
// фильтра из URL). Разница только в ДВУХ местах:
//   а) ИЗМЕНЕНО (17.09.2026, дважды в один день -- см. п.3 выше): аудитория применяется
//      НЕ к общему пулу сотрудников ДО построения строк (раньше -- GetMatrixAudienceCollaboratorRows(),
//      убрана), а К КАЖДОЙ ГОТОВОЙ СТРОКЕ (сотрудник x программа) ПОСЛЕ построения --
//      потому что у разных программ в одной и той же строке-сотруднике может быть РАЗНАЯ
//      аудитория (см. row.in_audience в BuildReportRows(), фильтр в Run() сразу после
//      сборки RESULT). Применяется ко ВСЕМ 4 режимам ОДИНАКОВО, включая "fact" (было
//      исключение для "fact" -- убрано по итогам теста, см. п.3);
//   б) ПОСЛЕ того, как готовые строки (с completion_date) собраны -- для "fact" оставляем
//      только строки с НЕпустой датой, для "mandatory" -- только с ПУСТОЙ датой,
//      для "total"/"plan" -- оставляем все строки без изменений.
//
// Параметр result_type -- один из: "total" | "plan" | "fact" | "mandatory".
//   ИЗМЕНЕНО (10.09.2026): раньше читался ТОЛЬКО как фиксированное значение на вкладке
//   "Параметры" отдельного виджета (по виджету на режим). Теперь читается СНАЧАЛА из
//   URL (как остальные фильтры) -- это открывает дорогу к переключению режима самим
//   пользователем на фронтенде (вкладки/ссылки/поле в модалке -- способ ещё
//   обсуждается). Если в URL параметра нет -- запасной путь: старое фиксированное
//   значение LPE (обратная совместимость с уже настроенными виджетами), иначе "total".
//   Подробности -- в начале Run().
// Фильтры (matrix_id, macroregion, mir_code, position_common_id, program_id) читаются
// ТАК ЖЕ, как в HREDU-181 -- из Request.Url (см. HREDU-183_diagnostic_get_params.js) --
// это работает надёжно в контексте ВЫБОРКИ (не удалённого действия, там был другой
// механизм через {{curEnv.curEnvUrl}}, здесь он не нужен).
//
// ВАЖНО про RESULT: как и в HREDU-181 -- RESULT это ПРЯМО массив строк, без обёртки.

DEBUG = true;              // На проде поставить false после тестирования
LOG_NAME = "agent";        // TODO: уточнить после создания документа в админке
CUR_OBJECT_ID = 0;         // TODO: заполнить ID документа выборки после её создания в админке

//-------------------------------------------------------------------------
//              Область функций
//-------------------------------------------------------------------------

function LogAlert(typeLog, message)
{
    // ЗАЩИЩЕНО (10.09.2026, по мотивам реальной поломки): если LOG_NAME/CUR_OBJECT_ID
    // ещё не настроены (например CUR_OBJECT_ID=0 -- заглушка "TODO: заполнить после
    // создания документа в админке"), вызов может упасть с ошибкой -- а LogAlert()
    // вызывается ДО главного try/catch в Run(), поэтому необработанное исключение тут
    // рушило ВЕСЬ Run() целиком: RESULT никогда не устанавливался, таблица оставалась
    // пустой БЕЗ какого-либо сообщения об ошибке (именно это и произошло на реальном
    // тесте). Логирование -- вспомогательная вещь, её сбой не должен ронять сам отчёт.
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
// ВРЕМЕННАЯ ДИАГНОСТИКА. Когда причина тормозов найдена -- этот блок и все вызовы
// PerfStart()/PerfCheckpoint() ниже можно спокойно удалить, на остальную логику файла
// это никак не влияет (отдельный флаг PERF_DEBUG, отдельные функции -- ничего не
// переиспользуется в основном коде отчёта).
//
// КАК ЧИТАТЬ: PerfStart() -- один раз в начале Run(), дальше PerfCheckpoint("название
// шага") после каждого интересующего шага (SQL/XQuery-запрос, tools.open_doc(), тяжёлый
// цикл). Каждый вызов -- это alert() с ТРЕМЯ цифрами: текущее время, сколько прошло С
// ПРЕДЫДУЩЕЙ точки (это и есть время именно ЭТОГО шага) и сколько прошло С САМОГО НАЧАЛА
// (общее время на данный момент). Если "с предыдущей точки" у конкретного шага большое --
// значит тормозит именно он. В названии каждого шага ниже (см. Run()) явно помечено,
// SQL/БД это или ЧИСТЫЙ КОД -- так сразу видно, куда смотреть: в СУБД или в сам скрипт.
//
// ЧЕСТНО ПРО ТОЧНОСТЬ: подсчёт секунд ((dTo - dFrom) * 86400) -- ЭКСПЕРИМЕНТАЛЬНЫЙ.
// Арифметика вычитания дат на этой платформе раньше в проекте нигде не проверялась (в
// отличие от Date()/StrDate()/DateOffset() -- они точно рабочие, см. FindCompletionDate()
// в этом же файле). Точно известно: сравнение дат (">=") работает (см.
// HREDU-176_integration_final_working.js), а DateOffset(date, секунды) прибавляет к дате
// секунды -- это похоже на тип, где дата хранится числом (целая часть -- дни, дробная --
// доля суток), поэтому вычитание, скорее всего, даёт разницу В ДНЯХ, а *86400 переводит
// в секунды. НО если цифра в alert'е выглядит абсурдно (отрицательная, огромная, "?") --
// НЕ доверяй числу секунд, доверяй RAW-таймштампам ("время сейчас: ...") -- разницу по
// ним можно посчитать вручную, Date()/StrDate() -- точно рабочие функции. Вся арифметика
// обёрнута в try/catch -- если вычитание дат не поддерживается, вместо числа увидишь "?".
//
// И сами alert() тоже обёрнуты в try/catch -- если alert() почему-то недоступен в
// контексте исполнения выборки, замер просто промолчит, а не уронит отчёт.

PERF_DEBUG = true; // поставь false, чтобы быстро выключить весь этот блок целиком
gPerfStartTime = undefined;
gPerfLastTime = undefined;

/*
 * Начинает замер -- запоминает "сейчас" как точку отсчёта. Вызвать ОДИН РАЗ в начале Run().
 * @returns {void}
 */
function PerfStart()
{
    if (!PERF_DEBUG) { return; }
    gPerfStartTime = PerfNowSafe();
    gPerfLastTime = gPerfStartTime;
    PerfAlertSafe("[ЗАМЕР] СТАРТ. Время: " + PerfFormatTimestamp(gPerfStartTime));
}

/*
 * Точка замера -- alert() с текущим временем, временем ЭТОГО шага (с предыдущей точки) и
 * временем с начала (с PerfStart()). См. подробности в шапке блока выше.
 * @param {string} sLabel   -   Название шага, например "GetActiveCollaboratorRows() -- SQL".
 * @returns {void}
 */
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

/*
 * Достаёт полный URL текущей страницы (см. HREDU-181_vostok_polny_spisok_draft.js --
 * тот же приём, подтверждён диагностикой). В контексте ВЫБОРКИ Request.Url надёжен.
 * @returns {string}
 */
function GetRequestUrlSafe()
{
    try
    {
        return String(Request.Url);
    }
    catch (_ex)
    {
        return "";
    }
}

/*
 * Вырезает значение GET-параметра из полного URL строки -- без regex, без методов
 * строк (их нет в этом движке), через штатный API платформы (StrOptSubStrPos/
 * StrRangePos/StrLen), декодирование через UrlDecode(). Идентична версии в
 * HREDU-181_vostok_polny_spisok_draft.js.
 * @param {string} sUrl
 * @param {string} sParamName
 * @returns {string}
 */
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
        if (iParamPos == undefined)
        {
            return "";
        }
        iValueStart = iParamPos + StrLen(sQMarkMarker);
    }

    iAmpPos = StrOptSubStrPos(sUrl, "&", false, iValueStart);
    iValueEnd = (iAmpPos != undefined ? iAmpPos : iUrlLen);

    sRawValue = StrRangePos(sUrl, iValueStart, iValueEnd);

    try
    {
        return UrlDecode(sRawValue);
    }
    catch (_exDecode)
    {
        return sRawValue;
    }
}

/*
 * Находит все записи cc_learning_matrice с указанным названием (см. открытый вопрос №1
 * в HREDU-181_vostok_polny_spisok_draft.js -- матрица может быть "размножена").
 * @param {string} matrixName
 * @returns {Object[]}
 */
function GetMatrixRows(matrixName)
{
    LogAlert(1, "GetMatrixRows(). НАЧАЛО. matrixName=" + matrixName);
    var matrixRows;
    matrixRows = ArraySelectAll(XQuery("for $elem in cc_learning_matrices where $elem/name = " + XQueryLiteral(matrixName) + " return $elem"));
    LogAlert(1, "GetMatrixRows(). Найдено записей: " + ArrayCount(matrixRows));
    LogAlert(1, "GetMatrixRows(). КОНЕЦ");
    return matrixRows;
}

/*
 * Находит активные элементы (программы) для указанных записей матрицы.
 * @param {number[]} matrixIds
 * @returns {Object[]}
 */
function GetMatrixElementRows(matrixIds)
{
    LogAlert(1, "GetMatrixElementRows(). НАЧАЛО");
    var elementRows;
    elementRows = ArraySelectAll(XQuery("for $elem in cc_learning_matrice_elements where MatchSome($elem/cc_learning_matrice_id, (" + ArrayMerge(matrixIds, "This", ",") + ")) and $elem/is_active=true() return $elem"));
    LogAlert(1, "GetMatrixElementRows(). Найдено элементов: " + ArrayCount(elementRows));
    LogAlert(1, "GetMatrixElementRows(). КОНЕЦ");
    return elementRows;
}

/*
 * Собирает уникальный список ID программ (education_method) -- ТОЛЬКО с элементов.
 * ИЗМЕНЕНО (17.09.2026, HREDU-215 "Правки 1", по прямому указанию тим-лида): у типа
 * документов "Матрицы обучения" (cc_learning_matrice) поле education_method_id УДАЛЕНО
 * (как и mir_code_id/position_common_id/start_study_period/end_study_period -- см. новую
 * структуру матрицы, шапка задачи) -- теперь ЭТИ поля есть ТОЛЬКО у "Элементов матриц
 * обучения" (cc_learning_matrice_element). Раньше эта функция объединяла
 * education_method_id матрицы И её элементов (на случай, если у матрицы тоже было такое
 * поле, см. историю правки 15.09.2026) -- теперь у матрицы этого поля больше нет вообще,
 * читаем ТОЛЬКО из elementRows.
 * @param {Object[]} elementRows
 * @returns {number[]}
 */
function GetProgramIds(elementRows)
{
    LogAlert(1, "GetProgramIds(). НАЧАЛО");
    var elementProgramIds, allProgramIds, programIds, i;
    elementProgramIds = ArrayExtract(elementRows, "OptInt(This.education_method_id, 0)");
    allProgramIds = [];
    for (i = 0; i < ArrayCount(elementProgramIds); i++)
    {
        if (Int(elementProgramIds[i]) > 0) { allProgramIds.push(elementProgramIds[i]); }
    }
    programIds = ArraySelectDistinct(allProgramIds, "This");
    LogAlert(1, "GetProgramIds(). Уникальных программ (после отбрасывания пустых education_method_id): " + ArrayCount(programIds));
    LogAlert(1, "GetProgramIds(). КОНЕЦ");
    return programIds;
}

/*
 * Строит справочник { id, title } по программам обучения.
 * @param {number[]} programIds
 * @returns {Object[]}
 */
function GetProgramTitles(programIds)
{
    LogAlert(1, "GetProgramTitles(). НАЧАЛО");
    var titles, i, programID, educationMethodDoc;
    titles = [];
    for (i = 0; i < ArrayCount(programIds); i++)
    {
        programID = programIds[i];
        educationMethodDoc = tools.open_doc(programID).TopElem;
        titles.push({ id: String(programID), title: String(educationMethodDoc.name) });
    }
    LogAlert(1, "GetProgramTitles(). КОНЕЦ");
    return titles;
}

/*
 * Читает всех действующих сотрудников (базовый пул, ДО аудитории матрицы и ДО ручных
 * фильтров).
 * @returns {Object[]}
 */
function GetActiveCollaboratorRows()
{
    LogAlert(1, "GetActiveCollaboratorRows(). НАЧАЛО");
    var collaboratorRows;
    collaboratorRows = ArraySelectAll(XQuery("for $elem in collaborators where $elem/is_dismiss=false() return $elem"));
    LogAlert(1, "GetActiveCollaboratorRows(). Найдено сотрудников: " + ArrayCount(collaboratorRows));
    LogAlert(1, "GetActiveCollaboratorRows(). КОНЕЦ");
    return collaboratorRows;
}

/*
 * Находит ID документов коллекции "positions", у которых position_common_id совпадает
 * с переданным ID (см. подробное объяснение схемы в HREDU-181_vostok_polny_spisok_draft.js).
 * @param {number} iCommonPositionFilter
 * @returns {number[]}
 */
function GetPositionIdsByCommonPosition(iCommonPositionFilter)
{
    LogAlert(1, "GetPositionIdsByCommonPosition(). НАЧАЛО. iCommonPositionFilter=" + iCommonPositionFilter);
    var positionRows, positionIds, i;
    positionRows = ArraySelectAll(XQuery("for $elem in positions where $elem/position_common_id = " + iCommonPositionFilter + " return $elem"));
    positionIds = [];
    for (i = 0; i < ArrayCount(positionRows); i++)
    {
        positionIds.push(Int(positionRows[i].id));
    }
    LogAlert(1, "GetPositionIdsByCommonPosition(). Найдено конкретных должностей: " + ArrayCount(positionIds));
    LogAlert(1, "GetPositionIdsByCommonPosition(). КОНЕЦ");
    return positionIds;
}

/*
 * Проверяет вхождение числа в массив чисел обычным циклом (без ArraySelect()/строк-
 * выражений со ссылкой на внешние переменные -- см. объяснение в HREDU-181).
 * @param {number[]} idArray
 * @param {number} value
 * @returns {boolean}
 */
function IdArrayContains(idArray, value)
{
    var i;
    for (i = 0; i < ArrayCount(idArray); i++)
    {
        if (Int(idArray[i]) == Int(value))
        {
            return true;
        }
    }
    return false;
}

/*
 * Достаёт макрорегион (custom_elem f_2ewj) по всем действующим сотрудникам одним SQL-запросом.
 * @returns {Object[]}
 */
function GetMacroregionRows()
{
    LogAlert(1, "GetMacroregionRows(). НАЧАЛО");
    var sqlText, macroRows;
    sqlText = "";
    sqlText = sqlText + "select cs.id,\r\n";
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_2ewj'']/value)[1]', 'varchar(max)') as macroregion\r\n";
    sqlText = sqlText + "from collaborators cs\r\n";
    sqlText = sqlText + "inner join collaborator c on c.id = cs.id\r\n";
    sqlText = sqlText + "where cs.is_dismiss != 1";
    macroRows = ArraySelectAll(XQuery("sql:" + sqlText));
    LogAlert(1, "GetMacroregionRows(). Строк: " + ArrayCount(macroRows));
    LogAlert(1, "GetMacroregionRows(). КОНЕЦ");
    return macroRows;
}

/*
 * ДОБАВЛЕНО (14.09.2026, для drill-down из "Процент обученных"): город -- custom_elem
 * "sity" (имя технического поля подтверждено пользователем реальным XML документа
 * collaborator -- см. HREDU-182_procent_obuchennyh.js). Нужен, чтобы клик по конкретной
 * строке-городу в таблице "Процент обученных" открывал список ТОЛЬКО по этому городу,
 * а не по всей матрице.
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

/*
 * НАЙДЕНО ЗАМЕРОМ ПРОИЗВОДИТЕЛЬНОСТИ (18.09.2026, реальный тест пользователя, см.
 * PerfCheckpoint() в Run()): "Ручной фильтр по макрорегиону" занял ~3 сек, "Фильтр по
 * городу" ~1 сек -- ПРИ ЭТОМ все SQL/XQuery-запросы заняли КАЖДЫЙ 0-1 сек. Т.е. БД тут ни
 * при чём -- тормозил ИМЕННО ЭТОТ КОД. Причина: FindCity()/FindMacroregion()/
 * FindCompletionDate() делали ArrayOptFind() -- ЛИНЕЙНЫЙ поиск по ВСЕМУ массиву
 * (macroRows/cityRows/dateRows -- это ВСЕ активные сотрудники компании) -- внутри цикла по
 * каждому сотруднику. Классическая O(n^2)-ловушка.
 *
 * ПЕРВАЯ ПОПЫТКА ФИКСА (18.09.2026, ОТКАЧЕНА В ТОТ ЖЕ ДЕНЬ): строили индекс через
 * обычный объект-словарь с динамическим ключом (object[String(id)]). РЕАЛЬНЫЙ ТЕСТ
 * пользователя это СЛОМАЛ -- платформа выдала "Unknown object property:
 * 6088495934734023244_6696477179940732741" -- т.е. динамический доступ к свойству
 * объекта по ВЫЧИСЛЯЕМОМУ строковому ключу (object[stringKey], в отличие от
 * object.fixedName) НЕ ПОДДЕРЖИВАЕТСЯ этим движком. Это подтверждённый факт, а не
 * гипотеза -- как в своё время не поддержались regex/.indexOf/.substring.
 *
 * ИТОГОВЫЙ ФИКС (18.09.2026, ВТОРАЯ ПОПЫТКА): вместо объекта-словаря -- БИНАРНЫЙ ПОИСК по
 * массиву, ПРЕДВАРИТЕЛЬНО ОТСОРТИРОВАННОМУ через ArraySort(). Использует ТОЛЬКО
 * конструкции, уже подтверждённые в проекте: доступ к элементу массива по ЧИСЛОВОМУ
 * индексу (array[i] -- используется вообще везде в этом файле), ArraySort() (подтверждена
 * РЕАЛЬНО РАБОТАЮЩЕЙ в HREDU-176_integration_final_working.js: "ArraySort(snilsPersons,
 * ...HiringDate..., "+")"), сравнения и арифметика. Сложность: O(n log n) на сортировку
 * (один раз) + O(log n) на каждый поиск -- гораздо лучше O(n^2), хоть и не идеальный O(1),
 * зато не опирается на непроверенный (и теперь уже опровергнутый) механизм.
 *
 * ЧЕСТНО ПРО РИСК №2: ArraySort() подтверждена РАБОЧЕЙ в другом файле этого же проекта --
 * это МНОГО надёжнее, чем "непроверенная гипотеза" (какой была динамическая индексация),
 * но не 100% гарантия именно в ЭТОМ контексте (выборка, а не удалённое действие). ОБЯЗАТЕЛЬНО
 * проверь на реальных данных: (а) что отчёт снова показывает те же строки, что и раньше;
 * (б) что новые PerfCheckpoint() для "сортировки" в логе быстрые, а сами фильтры больше НЕ
 * занимают 3 сек/1 сек. Если ArraySort() тоже поведёт себя не так, как ожидается -- пришли
 * мне точный текст ошибки, откатимся на самый надёжный (хоть и медленный) вариант --
 * исходный ArrayOptFind() -- и будем думать дальше.
 * @param {Object[]} rows   -   Строки с полем "id" (macroRows/cityRows).
 * @returns {Object[]}       -   Тот же массив строк, отсортированный по возрастанию id.
 */
function SortRowsById(rows)
{
    return ArraySort(rows, "Int(This.id)", "+");
}

/*
 * Бинарный поиск строки с полем "id" == targetId в МАССИВЕ, ОТСОРТИРОВАННОМ ПО ВОЗРАСТАНИЮ
 * id (см. SortRowsById()). Использует только доступ к массиву по числовому индексу --
 * никакого object[computedKey], см. "ЧЕСТНО ПРО РИСК" выше.
 * @param {Object[]} sortedRows   -   Массив, УЖЕ отсортированный через SortRowsById().
 * @param {number} targetId
 * @returns {Object}               -   Найденная строка или undefined.
 */
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

/*
 * НАЙДЕНО (18.09.2026, ЧЕТВЁРТЫЙ раунд замера -- уже ПОСЛЕ фикса macro/city/date/mirCode
 * бинарным поиском в HREDU-182_procent_obuchennyh.js): реальный тест там показал, что "Цикл
 * total/mandatory" всё ещё занимает ~14 сек. Подозреваемый источник -- IdArrayContains()
 * (см. ниже): вызывается из CollaboratorInProgramAudience() (проверка должности сотрудника)
 * И из ручного фильтра по должности -- В ОБОИХ МЕСТАХ на КАЖДОГО сотрудника, линейным
 * перебором по списку position_id для "общей должности" (GetPositionIdsByCommonPosition()) --
 * список может быть большим (десятки-сотни конкретных должностей на одну "общую"), то есть
 * тот же O(n x m)-паттерн, что был у macro/city/date/mirCode, просто на ДРУГОМ поле.
 * Идентичный фикс здесь -- сортируем список один раз сразу после получения, ищем бинарным
 * поиском.
 * @param {number[]} idArray
 * @returns {number[]}
 */
function SortIdArray(idArray)
{
    return ArraySort(idArray, "Int(This)", "+");
}

/*
 * Бинарный поиск значения в МАССИВЕ ЧИСЕЛ (не объектов), ОТСОРТИРОВАННОМ через
 * SortIdArray(). Идентична версии в HREDU-182_procent_obuchennyh.js.
 * @param {number[]} sortedIdArray
 * @param {number} value
 * @returns {boolean}
 */
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

/*
 * Находит город конкретного сотрудника; "(без города)" если поле пустое/не найдено --
 * та же условность, что в HREDU-182_procent_obuchennyh.js.
 * ИЗМЕНЕНО (18.09.2026, см. "НАЙДЕНО ЗАМЕРОМ ПРОИЗВОДИТЕЛЬНОСТИ" выше): теперь принимает
 * ОТСОРТИРОВАННЫЙ по id массив (см. SortRowsById()) и ищет БИНАРНЫМ ПОИСКОМ, а не линейно.
 * @param {Object[]} sortedCityRows
 * @param {number} collaboratorID
 * @returns {string}
 */
function FindCity(sortedCityRows, collaboratorID)
{
    var cityRow, sCity;
    cityRow = BinarySearchById(sortedCityRows, collaboratorID);
    sCity = (cityRow != undefined && cityRow.sity != undefined ? String(cityRow.sity) : "");
    return (sCity != "" ? sCity : "(без города)");
}

/*
 * Сортирует dateRows по collaborator_id (не по id -- у этих строк нет своего "id",
 * ключевое поле здесь -- collaborator_id, см. GetCompletionDateRows()).
 * @param {Object[]} dateRows
 * @returns {Object[]}
 */
function SortDateRowsByCollaboratorId(dateRows)
{
    return ArraySort(dateRows, "Int(This.collaborator_id)", "+");
}

/*
 * Достаёт сырое значение мир-кодов (custom_elem f_mir_codes) по всем действующим
 * сотрудникам одним SQL-запросом.
 * @returns {Object[]}
 */
function GetMirCodeRows()
{
    LogAlert(1, "GetMirCodeRows(). НАЧАЛО");
    var sqlText, rows;
    sqlText = "";
    sqlText = sqlText + "select cs.id,\r\n";
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_mir_codes'']/value)[1]', 'varchar(max)') as mir_codes\r\n";
    sqlText = sqlText + "from collaborators cs\r\n";
    sqlText = sqlText + "inner join collaborator c on c.id = cs.id\r\n";
    sqlText = sqlText + "where cs.is_dismiss != 1";
    rows = ArraySelectAll(XQuery("sql:" + sqlText));
    LogAlert(1, "GetMirCodeRows(). Строк: " + ArrayCount(rows));
    LogAlert(1, "GetMirCodeRows(). КОНЕЦ");
    return rows;
}

/*
 * Разбирает сырое значение f_mir_codes ("#LASK#17#|#LASM#17#...") в массив кодов без процентов.
 * @param {string} rawValue
 * @returns {string[]}
 */
function ExtractMirCodes(rawValue)
{
    var parts, fields, codes, i;
    codes = [];
    parts = ArrayDirect(ArraySelect(String(rawValue).split("|"), "This != ''"));
    for (i = 0; i < ArrayCount(parts); i++)
    {
        fields = ArrayDirect(ArraySelect(String(parts[i]).split("#"), "This != ''"));
        if (ArrayCount(fields) > 0)
        {
            codes.push(String(fields[0]));
        }
    }
    return codes;
}

/*
 * Проверяет, есть ли у сотрудника указанный мир-код среди любых его мир-кодов.
 * НАЙДЕНО (18.09.2026, замер на "Процент обученных" ПОСЛЕ фикса macro/city/date бинарным
 * поиском): реальный тест показал провал в 31 сек ИМЕННО на шаге "Цикл total/mandatory" --
 * виновата была ЭТА функция: CollaboratorInProgramAudience() (см. ниже) вызывает её на
 * каждую пару сотрудник x программа-с-мир-кодовым-сегментом, а она делала линейный
 * ArrayOptFind() по mirCodeRows -- ТАКОМУ ЖЕ полному массиву по ВСЕМ активным сотрудникам
 * компании, как macroRows/cityRows/dateRows до фикса -- та же O(n^2)-ловушка, пропущенная
 * в первых двух раундах правки ЭТОГО файла (тестовая матрица ТЭП не содержала элементов с
 * непустым мир-кодовым сегментом, поэтому баг здесь не проявился, хотя код был тот же).
 * Исправлено идентично остальным трём полям -- бинарный поиск по mirCodeRows,
 * отсортированному через SortRowsById() один раз в Run().
 * @param {Object[]} sortedMirCodeRows
 * @param {number} collaboratorID
 * @param {string} mirCodeFilter
 * @returns {boolean}
 */
function CollaboratorHasMirCode(sortedMirCodeRows, collaboratorID, mirCodeFilter)
{
    var row, codes;
    row = BinarySearchById(sortedMirCodeRows, collaboratorID);
    if (row == undefined)
    {
        return false;
    }
    codes = ExtractMirCodes(row.mir_codes);
    return (ArrayOptFind(codes, "String(This) == String(mirCodeFilter)") != undefined);
}

/*
 * Резолвит ID объекта cc_mir_codes в его текстовый код -- нужно, чтобы превратить
 * mir_code_id САМОЙ МАТРИЦЫ (аудитория) в текст для сравнения с f_mir_codes сотрудника
 * (та же функция, что в HREDU-183_filtry_modal_shag1.js).
 * @param {number} iMirCodeID
 * @returns {string}
 */
function ResolveMirCodeText(iMirCodeID)
{
    if (OptInt(iMirCodeID, 0) <= 0)
    {
        return "";
    }
    try
    {
        return String(tools.open_doc(Int(iMirCodeID)).TopElem.name);
    }
    catch (_ex)
    {
        return "";
    }
}

// УБРАНО (17.09.2026, HREDU-215 "Правки 1"): GetMatrixAudienceCollaboratorRows() читала
// АУДИТОРИЮ (должность + мир-код) с полей position_common_id/mir_code_id САМОЙ МАТРИЦЫ
// (cc_learning_matrice) -- этих полей у типа документа больше нет, тим-лид перенёс их на
// "Элементы матриц обучения" (cc_learning_matrice_element, у которых уже были
// education_method_id/start_study_period/end_study_period). Значит аудитория теперь НЕ
// ОДНА НА ВСЮ МАТРИЦУ, а СВОЯ У КАЖДОГО ЭЛЕМЕНТА (то есть, по факту, у каждой программы --
// см. пример из задачи: два элемента с ОДНИМ и тем же education_method_id, но РАЗНЫМИ
// position_common_id -- это значит "эту программу должны пройти две разные аудитории").
// Заменено на BuildProgramAudienceIndex()/CollaboratorInProgramAudience() ниже -- аудитория
// считается ОТДЕЛЬНО ДЛЯ КАЖДОЙ ПРОГРАММЫ (по всем активным элементам с этим
// education_method_id, через ИЛИ -- сотрудник входит в аудиторию программы, если подходит
// ХОТЯ БЫ ПОД ОДИН из её элементов), и применяется НЕ к общему пулу сотрудников ДО
// построения строк (как раньше), а К КАЖДОЙ СТРОКЕ (сотрудник x программа) ПОСЛЕ
// построения -- см. Run() и BuildReportRows() ниже.

/*
 * Строит по каждой программе (education_method_id) список "сегментов аудитории" --
 * одна запись на КАЖДЫЙ активный элемент матрицы с этим education_method_id. Сотрудник
 * входит в аудиторию программы, если подходит ХОТЯ БЫ ПОД ОДИН сегмент (ИЛИ между
 * элементами, И между должностью/мир-кодом ВНУТРИ одного элемента -- та же логика "0/пусто
 * = без ограничения по этой оси", что была в старой GetMatrixAudienceCollaboratorRows()).
 * allowedPositionIds считается ОДИН РАЗ НА СЕГМЕНТ здесь (не на каждого сотрудника ниже) --
 * иначе был бы N+1 запросов в GetPositionIdsByCommonPosition() на каждой строке отчёта.
 * @param {Object[]} elementRows   -   Активные элементы матрицы (is_active=1 уже в выборке).
 * @param {number[]} programIds
 * @returns {Object[]}   -   Массив { programId, segments: [{ positionCommonId, allowedPositionIds, mirCodeText }] }.
 *                           allowedPositionIds ОТСОРТИРОВАН (см. SortIdArray() -- ИЗМЕНЕНО
 *                           18.09.2026, ЧЕТВЁРТЫЙ раунд) для бинарного поиска в
 *                           CollaboratorInProgramAudience() ниже.
 */
function BuildProgramAudienceIndex(elementRows, programIds)
{
    LogAlert(1, "BuildProgramAudienceIndex(). НАЧАЛО");
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
                // ИЗМЕНЕНО (18.09.2026, ЧЕТВЁРТЫЙ раунд -- см. комментарий над
                // SortIdArray()): список считается один раз на сегмент (как и раньше), но
                // теперь ЕЩЁ И сортируется один раз здесь -- дальше по нему бинарный поиск
                // на каждого сотрудника, а не линейный перебор.
                allowedPositionIds = (iPos > 0 ? SortIdArray(GetPositionIdsByCommonPosition(iPos)) : []);
                sMirCodeText = ResolveMirCodeText(iMirCode);
                segments.push({ positionCommonId: iPos, allowedPositionIds: allowedPositionIds, mirCodeText: sMirCodeText });
            }
        }
        index.push({ programId: Int(programId), segments: segments });
    }
    LogAlert(1, "BuildProgramAudienceIndex(). КОНЕЦ. Программ в индексе: " + ArrayCount(index));
    return index;
}

/*
 * ДОБАВЛЕНО (18.09.2026, ЧЕТВЁРТЫЙ раунд, диагностика -- см. комментарий над
 * SortIdArray()): считает суммарное число сегментов и суммарный/максимальный размер
 * allowedPositionIds по всем сегментам -- чтобы ПОДТВЕРДИТЬ или ОПРОВЕРГНУТЬ гипотезу про
 * большие списки должностей. Идентична версии в HREDU-182_procent_obuchennyh.js. Только
 * счёт, никакого Date()/арифметики времени.
 * @param {Object[]} audienceIndex
 * @returns {string}
 */
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

/*
 * Находит сегменты аудитории конкретной программы в индексе, построенном
 * BuildProgramAudienceIndex().
 * @param {Object[]} audienceIndex
 * @param {number} programId
 * @returns {Object[]}
 */
function FindProgramAudienceSegments(audienceIndex, programId)
{
    var row;
    row = ArrayOptFind(audienceIndex, "Int(This.programId) == Int(programId)");
    return (row != undefined ? row.segments : []);
}

/*
 * Проверяет, входит ли сотрудник в аудиторию программы -- подходит ли он ХОТЯ БЫ ПОД
 * ОДИН из её сегментов (элементов матрицы).
 * @param {Object} collaboratorRow
 * @param {Object[]} segments        -   Результат FindProgramAudienceSegments().
 * @param {Object[]} sortedMirCodeRows   -   Результат GetMirCodeRows() + SortRowsById() (см. Run()).
 * @returns {boolean}
 */
function CollaboratorInProgramAudience(collaboratorRow, segments, sortedMirCodeRows)
{
    var i, seg, positionOk, mirCodeOk;
    for (i = 0; i < ArrayCount(segments); i++)
    {
        seg = segments[i];
        positionOk = (seg.positionCommonId <= 0 || IdArrayContainsSorted(seg.allowedPositionIds, OptInt(collaboratorRow.position_id, 0)));
        mirCodeOk = (seg.mirCodeText == "" || CollaboratorHasMirCode(sortedMirCodeRows, Int(collaboratorRow.id), seg.mirCodeText));
        if (positionOk && mirCodeOk)
        {
            return true;
        }
    }
    return false;
}

/*
 * Находит минимальную дату прохождения по каждому сотруднику и программе.
 * @param {number[]} programIds
 * @returns {Object[]}
 */
function GetCompletionDateRows(programIds)
{
    LogAlert(1, "GetCompletionDateRows(). НАЧАЛО");
    var sqlText, dateRows;
    sqlText = "";
    sqlText = sqlText + "select ec.collaborator_id, e.education_method_id, min(ec.start_date) as first_date\r\n";
    sqlText = sqlText + "from event_collaborators ec\r\n";
    sqlText = sqlText + "join events e on e.id = ec.event_id\r\n";
    sqlText = sqlText + "where e.education_method_id in (" + ArrayMerge(programIds, "This", ",") + ")\r\n";
    sqlText = sqlText + "group by ec.collaborator_id, e.education_method_id";
    dateRows = ArraySelectAll(XQuery("sql:" + sqlText));
    LogAlert(1, "GetCompletionDateRows(). Строк: " + ArrayCount(dateRows));
    LogAlert(1, "GetCompletionDateRows(). КОНЕЦ");
    return dateRows;
}

/*
 * Ищет дату прохождения конкретного сотрудника по конкретной программе.
 * ИЗМЕНЕНО (18.09.2026, см. "НАЙДЕНО ЗАМЕРОМ ПРОИЗВОДИТЕЛЬНОСТИ" над SortRowsById()):
 * dateRows У ОДНОГО сотрудника обычно немного строк (по числу программ, которые он вообще
 * когда-либо проходил -- НЕ пропорционально общему числу сотрудников компании), поэтому
 * здесь -- БИНАРНЫЙ ПОИСК первой строки этого сотрудника (по sortedDateRows,
 * отсортированному через SortDateRowsByCollaboratorId()), а дальше короткий линейный
 * проход ТОЛЬКО по строкам ЭТОГО сотрудника (их мало) в поисках нужной программы -- без
 * object[computedKey], см. объяснение риска выше.
 * @param {Object[]} sortedDateRows   -   Массив, УЖЕ отсортированный через SortDateRowsByCollaboratorId().
 * @param {number} collaboratorID
 * @param {number} programID
 * @returns {string}
 */
function FindCompletionDate(sortedDateRows, collaboratorID, programID)
{
    var lo, hi, mid, midId, iTarget, iProgram, startIdx, i, n;

    iTarget = Int(collaboratorID);
    iProgram = Int(programID);

    // Бинарный поиск ЛЮБОЙ строки этого сотрудника, затем сдвигаемся к ПЕРВОЙ такой строке
    // (могут быть дубликаты collaborator_id -- по одной строке на каждую пройденную программу).
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
            hi = mid - 1; // ищем более ранний индекс с тем же collaborator_id
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

/*
 * Ищет макрорегион конкретного сотрудника.
 * ИЗМЕНЕНО (18.09.2026, см. "НАЙДЕНО ЗАМЕРОМ ПРОИЗВОДИТЕЛЬНОСТИ" над SortRowsById()) --
 * бинарный поиск по отсортированному массиву вместо линейного ArrayOptFind().
 * @param {Object[]} sortedMacroRows   -   Массив, УЖЕ отсортированный через SortRowsById().
 * @param {number} collaboratorID
 * @returns {string}
 */
function FindMacroregion(sortedMacroRows, collaboratorID)
{
    var macroRow;
    macroRow = BinarySearchById(sortedMacroRows, collaboratorID);
    return (macroRow != undefined && macroRow.macroregion != undefined ? String(macroRow.macroregion) : "");
}

/*
 * Ищет название программы по её ID.
 * @param {Object[]} programTitles
 * @param {number} programID
 * @returns {string}
 */
function FindProgramTitle(programTitles, programID)
{
    var titleRow;
    titleRow = ArrayOptFind(programTitles, "String(This.id) == String(programID)");
    return (titleRow != undefined ? String(titleRow.title) : "");
}

/*
 * Собирает строки отчёта для ОДНОГО сотрудника -- по одной строке на каждую программу
 * матрицы (те же 6 полей, что в HREDU-181, ПЛЮС город -- см. ниже).
 * ИЗМЕНЕНО (16.09.2026, по просьбе пользователя): добавлено поле row.city -- отдельная
 * колонка "Город" рядом с "Макрорегион". Считается тем же способом, что и раньше
 * использовался ТОЛЬКО для фильтрации (FindCity()/GetCityRows(), см. выше) -- теперь
 * используется ещё и для отображения. В LPE-настройках виджета "Табличные данные" для
 * этой выборки нужно добавить колонку, привязанную к полю "city" (рядом с "macroregion") --
 * из кода это не настраивается, RESULT здесь -- просто массив строк без описания колонок
 * (см. "ВАЖНО про RESULT" в шапке файла).
 * ИЗМЕНЕНО (17.09.2026, HREDU-215 "Правки 1"): добавлено служебное поле row.in_audience --
 * входит ли ЭТОТ сотрудник в аудиторию ИМЕННО ЭТОЙ программы (проверяется по индексу
 * BuildProgramAudienceIndex(), т.к. аудитория теперь своя у каждого элемента/программы,
 * а не одна на всю матрицу -- см. комментарий над BuildProgramAudienceIndex() выше). Это
 * НЕ колонка для отображения (в LPE её просто не привязывают ни к чему) -- используется
 * только внутри Run() для финальной фильтрации total/plan/mandatory (для fact аудитория
 * не применяется вообще, как и раньше -- "не зависимо от условий матрицы").
 * ИЗМЕНЕНО (18.09.2026, ЗАМЕР ПРОИЗВОДИТЕЛЬНОСТИ, дважды в один день -- см. подробности
 * над SortRowsById()): параметры macroRows/cityRows/dateRows переименованы в
 * macroSorted/citySorted/dateSorted -- это массивы, ОТСОРТИРОВАННЫЕ через SortRowsById()/
 * SortDateRowsByCollaboratorId(), для БИНАРНОГО поиска внутри FindMacroregion()/FindCity()/
 * FindCompletionDate() вместо линейного ArrayOptFind(). (Первая версия правки использовала
 * объект-словарь с динамическим ключом -- реальный тест пользователя показал, что такой
 * доступ не поддерживается платформой ("Unknown object property") -- откачено на бинарный
 * поиск, см. подробности над SortRowsById().)
 * @param {Object} collaborator
 * @param {Object[]} macroSorted
 * @param {Object[]} citySorted
 * @param {Object[]} dateSorted
 * @param {Object[]} programTitles
 * @param {number[]} programIds
 * @param {Object[]} audienceIndex
 * @param {Object[]} mirCodeSorted   -   Отсортированный через SortRowsById() (см. Run()).
 * @returns {Object[]}
 */
function BuildReportRows(collaborator, macroSorted, citySorted, dateSorted, programTitles, programIds, audienceIndex, mirCodeSorted)
{
    var rows, row, i, programID, segments;
    rows = [];
    for (i = 0; i < ArrayCount(programIds); i++)
    {
        programID = programIds[i];
        segments = FindProgramAudienceSegments(audienceIndex, programID);
        row = new Object();
        row.fullname = String(collaborator.fullname);
        row.position_name = String(collaborator.position_name);
        row.subdivision_name = String(collaborator.position_parent_name);
        row.macroregion = FindMacroregion(macroSorted, Int(collaborator.id));
        row.city = FindCity(citySorted, Int(collaborator.id));
        row.program_name = FindProgramTitle(programTitles, programID);
        row.completion_date = FindCompletionDate(dateSorted, Int(collaborator.id), programID);
        row.in_audience = CollaboratorInProgramAudience(collaborator, segments, mirCodeSorted);
        rows.push(row);
    }
    return rows;
}

/*
 * Резолвит выбранную матрицу (matrix_id) в список ID программ обучения И в сами строки
 * элементов матрицы (elementRows нужны отдельно от programIds -- см. HREDU-215 "Правки 1":
 * аудитория (должность+мир-код) теперь считается ПО ЭЛЕМЕНТАМ, см. BuildProgramAudienceIndex()
 * выше, поэтому одних programIds уже недостаточно).
 * ПЕРЕИМЕНОВАНО (17.09.2026, было ResolveProgramIds -- возвращала только programIds):
 * теперь возвращает объект { programIds, elementRows }.
 * @param {number} matrixId
 * @param {string} matrixName
 * @returns {Object}   -   { programIds: number[], elementRows: Object[] }.
 */
function ResolveMatrixContext(matrixId, matrixName)
{
    LogAlert(1, "ResolveMatrixContext(). НАЧАЛО. matrixId=" + matrixId);
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

    LogAlert(1, "ResolveMatrixContext(). КОНЕЦ");
    return { programIds: programIds, elementRows: elementRows };
}

/*
 * Точка входа. Собирает данные ОДНОГО из 4 отчётов-представлений ТЭП, в зависимости от
 * result_type -- см. подробности архитектуры и принятых решений в шапке файла.
 * @returns {void}
 */
function Run()
{
    // ИСПРАВЛЕНО (14.09.2026, по мотивам реальной поломки в HREDU-183_tep_reports.js с
    // CUR_OBJECT_ID): ссылка на "голый" глобал result_type ЗДЕСЬ, ДО try/catch, была
    // рискованной -- result_type теперь необязательный LPE-параметр (см. изменение
    // 10.09.2026, чтение сначала из URL), и если он вообще не привязан у виджета,
    // обращение к необъявленному глобалу могло бы упасть необработанным исключением
    // ДО того, как основной try/catch успел бы его поймать -- Run() оборвался бы молча,
    // как уже было один раз с LogAlert()/CUR_OBJECT_ID. Убрал ссылку на result_type
    // из этой самой первой строки -- реальное значение (sResultType) вычисляется и
    // логируется чуть ниже, УЖЕ внутри try/catch.
    LogAlert(2, "Run(). НАЧАЛО");
    var sResultType, sFullUrl, matrixId, matrixDoc, matrixName;
    var iProgramFilter, sMacroregionFilter, sMirCodeFilter, iPositionFilter, sCityFilter;
    var matrixContext, elementRows, audienceIndex;
    var programIds, programTitles, collaboratorRows, macroRows, mirCodeRows, cityRows, dateRows;
    var macroSorted, citySorted, dateSorted; // ДОБАВЛЕНО (18.09.2026, ЗАМЕР ПРОИЗВОДИТЕЛЬНОСТИ) -- см. SortRowsById()/SortDateRowsByCollaboratorId()
    var mirCodeSorted; // ДОБАВЛЕНО (18.09.2026, ВТОРАЯ правка -- см. комментарий над CollaboratorHasMirCode()): та же O(n^2)-ловушка нашлась и в мир-кодах, пропущенная в первом раунде
    var collaboratorReportRows, i, j;
    var filteredProgramIds, filteredCollaboratorRows, allowedPositionIds, filteredResultRows;

    ERROR = 0;
    MESSAGE = "";
    RESULT = [];

    try
    {
        PerfStart();

        sFullUrl = GetRequestUrlSafe();
        LogAlert(1, "Run(). Request.Url = [" + sFullUrl + "]");

        // ИЗМЕНЕНО (10.09.2026, вынос режима на фронтенд + английские имена): раньше
        // result_type был ТОЛЬКО фиксированным параметром выборки -- задавался один раз
        // в LPE "Параметры" каждого из 4 виджетов, пользователь не мог его поменять сам.
        // Теперь сначала читаем result_type из URL -- так же, как остальные фильтры --
        // если он там есть, режим может переключать сам пользователь (например через
        // поле в модалке фильтров или ссылки-вкладки на странице). Если в URL параметра
        // нет -- запасной путь: старое фиксированное значение из LPE (обратная
        // совместимость с уже настроенными виджетами), а если и его нет -- дефолт "total".
        // Имена режимов ТЕПЕРЬ НА АНГЛИЙСКОМ (было "obshee"/"fakt"/"obyazatelno" --
        // транслит с русского, неудобно читать): total | plan | fact | mandatory.
        // Если у виджетов в админке result_type ещё настроен старыми именами -- их нужно
        // переименовать (obshee->total, fakt->fact, obyazatelno->mandatory, plan остаётся).
        sResultType = GetQueryParam(sFullUrl, "result_type");
        if (sResultType == "")
        {
            try
            {
                sResultType = String(result_type);
            }
            catch (_exResultTypeParam)
            {
                sResultType = "";
            }
        }
        if (sResultType == "" || sResultType == "undefined")
        {
            sResultType = "total";
        }
        if (sResultType != "total" && sResultType != "plan" && sResultType != "fact" && sResultType != "mandatory")
        {
            throw ("Неизвестный result_type=[" + sResultType + "] -- ожидается одно из: total, plan, fact, mandatory");
        }
        LogAlert(1, "Run(). sResultType=[" + sResultType + "]");

        matrixId = OptInt(GetQueryParam(sFullUrl, "matrix_id"), 0);
        iProgramFilter = OptInt(GetQueryParam(sFullUrl, "program_id"), 0);
        sMacroregionFilter = GetQueryParam(sFullUrl, "macroregion");
        sMirCodeFilter = GetQueryParam(sFullUrl, "mir_code");
        iPositionFilter = OptInt(GetQueryParam(sFullUrl, "position_common_id"), 0);
        // ДОБАВЛЕНО (14.09.2026, drill-down из "Процент обученных"): необязательный
        // фильтр по городу (custom_elem "sity") -- если ссылка со страницы "Процент
        // обученных" пришла для конкретного города, здесь сузим список ТОЛЬКО до него.
        // Если параметра нет -- ведёт себя как раньше (без изменений для существующих
        // ссылок без city).
        sCityFilter = GetQueryParam(sFullUrl, "city");

        LogAlert(1, "Run(). matrixId=" + matrixId + " programFilter=" + iProgramFilter
            + " macroregionFilter=[" + sMacroregionFilter + "] mirCodeFilter=[" + sMirCodeFilter
            + "] positionCommonIdFilter=" + iPositionFilter + " cityFilter=[" + sCityFilter + "]");

        PerfCheckpoint("Разбор Request.Url и всех фильтров -- ЧИСТЫЙ КОД, без SQL");

        if (matrixId == 0)
        {
            throw ("Не передан matrix_id -- выбранная пользователем матрица обучения");
        }

        matrixDoc = tools.open_doc(matrixId).TopElem;
        matrixName = String(matrixDoc.name);
        PerfCheckpoint("tools.open_doc(matrixId) -- открытие документа матрицы -- БД/документ");

        // ИЗМЕНЕНО (17.09.2026, HREDU-215 "Правки 1"): ResolveProgramIds() переименована в
        // ResolveMatrixContext() и теперь возвращает ещё и elementRows (не только
        // programIds) -- нужны для построения аудитории ПО ЭЛЕМЕНТАМ ниже (BuildProgramAudienceIndex()).
        matrixContext = ResolveMatrixContext(matrixId, matrixName);
        programIds = matrixContext.programIds;
        elementRows = matrixContext.elementRows;
        PerfCheckpoint("ResolveMatrixContext() -- GetMatrixRows()+GetMatrixElementRows()+GetProgramIds() -- SQL/XQuery");

        if (iProgramFilter > 0)
        {
            filteredProgramIds = [];
            for (i = 0; i < ArrayCount(programIds); i++)
            {
                if (Int(programIds[i]) == iProgramFilter)
                {
                    filteredProgramIds.push(programIds[i]);
                }
            }
            programIds = filteredProgramIds;
            if (ArrayCount(programIds) == 0)
            {
                throw ("Программа [" + iProgramFilter + "] не найдена среди программ выбранной матрицы");
            }
        }

        programTitles = GetProgramTitles(programIds);
        PerfCheckpoint("GetProgramTitles() -- N x tools.open_doc() по программам матрицы -- БД/документы (потенциальный N+1)");

        // ИЗМЕНЕНО (17.09.2026, HREDU-215 "Правки 1"): раньше здесь фильтровался ОБЩИЙ
        // пул сотрудников по ОДНОЙ аудитории матрицы (GetMatrixAudienceCollaboratorRows(),
        // убрана). Теперь у КАЖДОЙ программы (элемента) СВОЯ аудитория -- строим индекс
        // один раз (по текущему, уже возможно суженному фильтром program_id, списку
        // programIds), а применяем его ПОЗЖЕ, к каждой ГОТОВОЙ СТРОКЕ отчёта (сотрудник x
        // программа) -- см. блок после сборки RESULT ниже. Аудитория по-прежнему не
        // применяется для "fact" вообще (см. "РЕШЕНИЯ" в шапке файла).
        audienceIndex = BuildProgramAudienceIndex(elementRows, programIds);
        // ДОБАВЛЕНО (18.09.2026, ЧЕТВЁРТЫЙ раунд -- см. комментарий над SortIdArray()):
        // размеры сегментов/списков должностей -- чтобы подтвердить/опровергнуть гипотезу.
        PerfCheckpoint("BuildProgramAudienceIndex() -- аудитория по элементам, внутри может быть SQL (GetPositionIdsByCommonPosition() на сегмент). " + SummarizeAudienceIndex(audienceIndex));

        collaboratorRows = GetActiveCollaboratorRows();
        PerfCheckpoint("GetActiveCollaboratorRows() -- SQL/XQuery по всем активным сотрудникам");

        // ИЗМЕНЕНО (17.09.2026): mirCodeRows раньше грузился ЛЕНИВО -- либо внутри
        // GetMatrixAudienceCollaboratorRows() (для аудитории матрицы, убрана), либо здесь
        // ниже при ручном фильтре по мир-коду. Теперь он нужен ВСЕГДА -- для аудитории
        // КАЖДОЙ программы (CollaboratorInProgramAudience(), см. BuildReportRows() ниже) --
        // грузим один раз здесь и переиспользуем и для ручного фильтра по мир-коду.
        mirCodeRows = GetMirCodeRows();
        PerfCheckpoint("GetMirCodeRows() -- SQL по мир-кодам сотрудников");
        // ДОБАВЛЕНО (18.09.2026, ВТОРАЯ правка -- см. комментарий над
        // CollaboratorHasMirCode()): реальный тест на "Процент обученных" показал провал
        // в 31 сек именно из-за линейного поиска по mirCodeRows -- та же ловушка есть и
        // здесь (тестовая матрица ТЭП её просто не проявила). Сортируем один раз.
        mirCodeSorted = SortRowsById(mirCodeRows);
        PerfCheckpoint("SortRowsById(mirCodeRows) -- сортировка для бинарного поиска по мир-кодам -- ЧИСТЫЙ КОД, O(n log n)");

        // Дальше -- РУЧНЫЕ фильтры пользователя, точно как в HREDU-181 (без изменений).
        if (iPositionFilter > 0)
        {
            // ИЗМЕНЕНО (18.09.2026, ЧЕТВЁРТЫЙ раунд -- см. комментарий над
            // SortIdArray()): сортируем один раз, ищем бинарным поиском вместо линейного
            // перебора на каждого сотрудника.
            allowedPositionIds = SortIdArray(GetPositionIdsByCommonPosition(iPositionFilter));
            filteredCollaboratorRows = [];
            for (i = 0; i < ArrayCount(collaboratorRows); i++)
            {
                if (IdArrayContainsSorted(allowedPositionIds, OptInt(collaboratorRows[i].position_id, 0)))
                {
                    filteredCollaboratorRows.push(collaboratorRows[i]);
                }
            }
            collaboratorRows = filteredCollaboratorRows;
            LogAlert(1, "Run(). После ручного фильтра по типовой должности осталось сотрудников: " + ArrayCount(collaboratorRows));
            PerfCheckpoint("Ручной фильтр по типовой должности (GetPositionIdsByCommonPosition() + цикл) -- SQL + ЧИСТЫЙ КОД");
        }

        macroRows = GetMacroregionRows();
        PerfCheckpoint("GetMacroregionRows() -- SQL по макрорегионам сотрудников");
        // ДОБАВЛЕНО (18.09.2026, ЗАМЕР ПРОИЗВОДИТЕЛЬНОСТИ): строим индекс ОДИН РАЗ сразу
        // после SQL -- дальше и ручной фильтр ниже, и BuildReportRows() (там пробегаем по
        // ВСЕМ отфильтрованным сотрудникам ещё раз) используют O(1)-поиск по индексу
        // вместо O(n) ArrayOptFind() -- см. подробное объяснение над SortRowsById().
        macroSorted = SortRowsById(macroRows);
        PerfCheckpoint("SortRowsById(macroRows) -- сортировка для бинарного поиска по макрорегионам -- ЧИСТЫЙ КОД, O(n log n)");
        if (sMacroregionFilter != "")
        {
            filteredCollaboratorRows = [];
            for (i = 0; i < ArrayCount(collaboratorRows); i++)
            {
                if (FindMacroregion(macroSorted, Int(collaboratorRows[i].id)) == sMacroregionFilter)
                {
                    filteredCollaboratorRows.push(collaboratorRows[i]);
                }
            }
            collaboratorRows = filteredCollaboratorRows;
            LogAlert(1, "Run(). После ручного фильтра по макрорегиону осталось сотрудников: " + ArrayCount(collaboratorRows));
            PerfCheckpoint("Ручной фильтр по макрорегиону (цикл по collaboratorRows, теперь через индекс) -- ЧИСТЫЙ КОД");
        }

        if (sMirCodeFilter != "")
        {
            // mirCodeSorted уже загружен и отсортирован выше (грузится теперь всегда, не только тут).
            filteredCollaboratorRows = [];
            for (i = 0; i < ArrayCount(collaboratorRows); i++)
            {
                if (CollaboratorHasMirCode(mirCodeSorted, Int(collaboratorRows[i].id), sMirCodeFilter))
                {
                    filteredCollaboratorRows.push(collaboratorRows[i]);
                }
            }
            collaboratorRows = filteredCollaboratorRows;
            LogAlert(1, "Run(). После ручного фильтра по мир-коду осталось сотрудников: " + ArrayCount(collaboratorRows));
            PerfCheckpoint("Ручной фильтр по мир-коду (CollaboratorHasMirCode() в цикле) -- ЧИСТЫЙ КОД");
        }

        // ИЗМЕНЕНО (16.09.2026, колонка "Город"): cityRows теперь грузится ВСЕГДА (было --
        // только если sCityFilter != "") -- иначе для отображения города колонкой (не
        // только для фильтрации) данных бы не было, когда фильтр по городу не применён.
        cityRows = GetCityRows();
        PerfCheckpoint("GetCityRows() -- SQL по городам сотрудников");
        citySorted = SortRowsById(cityRows);
        PerfCheckpoint("SortRowsById(cityRows) -- сортировка для бинарного поиска по городам -- ЧИСТЫЙ КОД, O(n log n)");

        // ДОБАВЛЕНО (14.09.2026, drill-down из "Процент обученных"): необязательный
        // фильтр по городу -- см. GetCityRows()/FindCity() выше.
        if (sCityFilter != "")
        {
            filteredCollaboratorRows = [];
            for (i = 0; i < ArrayCount(collaboratorRows); i++)
            {
                if (FindCity(citySorted, Int(collaboratorRows[i].id)) == sCityFilter)
                {
                    filteredCollaboratorRows.push(collaboratorRows[i]);
                }
            }
            collaboratorRows = filteredCollaboratorRows;
            LogAlert(1, "Run(). После фильтра по городу осталось сотрудников: " + ArrayCount(collaboratorRows));
            PerfCheckpoint("Фильтр по городу (FindCity() в цикле, теперь через индекс) -- ЧИСТЫЙ КОД");
        }

        dateRows = GetCompletionDateRows(programIds);
        PerfCheckpoint("GetCompletionDateRows() -- SQL по датам прохождения программ");
        dateSorted = SortDateRowsByCollaboratorId(dateRows);
        PerfCheckpoint("SortDateRowsByCollaboratorId(dateRows) -- сортировка для бинарного поиска по датам -- ЧИСТЫЙ КОД, O(n log n)");

        RESULT = [];
        for (i = 0; i < ArrayCount(collaboratorRows); i++)
        {
            collaboratorReportRows = BuildReportRows(collaboratorRows[i], macroSorted, citySorted, dateSorted, programTitles, programIds, audienceIndex, mirCodeSorted);
            for (j = 0; j < ArrayCount(collaboratorReportRows); j++)
            {
                RESULT.push(collaboratorReportRows[j]);
            }
        }
        LogAlert(1, "Run(). Строк (сотрудник x программа) до фильтра аудитории/result_type: " + ArrayCount(RESULT));
        // ДОБАВЛЕНО (18.09.2026, ЧЕТВЁРТЫЙ раунд): точные числа для этого шага -- см.
        // комментарий над SortIdArray().
        PerfCheckpoint("Цикл построения строк (BuildReportRows() x сотрудников x программ) -- ЧИСТЫЙ КОД, без SQL. Сотрудников: " + ArrayCount(collaboratorRows) + "; программ: " + ArrayCount(programIds) + "; пар всего: " + (ArrayCount(collaboratorRows) * ArrayCount(programIds)));

        // ИЗМЕНЕНО (17.09.2026, повторное уточнение с пользователем в тот же день): раньше
        // здесь стояло "если result_type != fact" -- т.е. для "Факт" аудитория вообще НЕ
        // проверялась (буквальное прочтение старого ТЗ "не зависимо от условий матрицы").
        // Реальный тест пользователя (матрица "Менеджер"/"Стандарт менеджер", город
        // Воронеж) показал, что это даёт СТРАННЫЙ результат -- в "Факт" попадали
        // сотрудники СОВСЕМ ДРУГИХ должностей (Экономисты), просто когда-то прошедшие ту
        // же программу по любой другой причине (программа -- общий каталог, не привязана
        // к конкретной матрице). Пользователь подтвердил (см. AskUserQuestion 17.09.2026):
        // "Факт" ТЕПЕРЬ ТОЖЕ ограничивается аудиторией -- должен подходить ХОТЯ БЫ ПОД
        // ОДИН элемент этой программы по должности+мир-коду (row.in_audience, см.
        // BuildReportRows() выше) -- НО, как и раньше, БЕЗ ограничения по периоду/датам
        // элемента (start_study_period/end_study_period по-прежнему не реализованы, см.
        // "РЕШЕНИЯ" пункт 2 в шапке файла) -- т.е. "не зависимо от условий матрицы"
        // теперь означает именно "не зависимо от периода", а не "не зависимо вообще ни от
        // чего". Поэтому условие "если != fact" убрано -- фильтр по аудитории применяется
        // ко ВСЕМ 4 режимам одинаково.
        filteredResultRows = [];
        for (i = 0; i < ArrayCount(RESULT); i++)
        {
            if (RESULT[i].in_audience)
            {
                filteredResultRows.push(RESULT[i]);
            }
        }
        RESULT = filteredResultRows;
        LogAlert(1, "Run(). После фильтра аудитории по элементам матрицы (для ВСЕХ режимов, включая fact) осталось строк: " + ArrayCount(RESULT));
        PerfCheckpoint("Фильтр по row.in_audience (цикл по RESULT) -- ЧИСТЫЙ КОД");

        // НОВОЕ (10.09.2026): финальное разбиение по result_type -- см. "ИТОГОВАЯ
        // АРХИТЕКТУРА" в шапке файла. "total"/"plan" -- без изменений (План = Общее,
        // см. "РЕШЕНИЯ" пункт 2).
        if (sResultType == "fact")
        {
            filteredResultRows = [];
            for (i = 0; i < ArrayCount(RESULT); i++)
            {
                if (RESULT[i].completion_date != "")
                {
                    filteredResultRows.push(RESULT[i]);
                }
            }
            RESULT = filteredResultRows;
        }
        else if (sResultType == "mandatory")
        {
            filteredResultRows = [];
            for (i = 0; i < ArrayCount(RESULT); i++)
            {
                if (RESULT[i].completion_date == "")
                {
                    filteredResultRows.push(RESULT[i]);
                }
            }
            RESULT = filteredResultRows;
        }

        PerfCheckpoint("Финальное разбиение по result_type (" + sResultType + ") -- ЧИСТЫЙ КОД");
        LogAlert(2, "Run(). Готово. result_type=" + sResultType + ", строк отчёта: " + ArrayCount(RESULT));
        PerfCheckpoint("Run() -- ГОТОВО (успех), строк отчёта: " + ArrayCount(RESULT));
    }
    catch (_ex)
    {
        ERROR = 1;
        MESSAGE = ExtractUserError(_ex);
        LogAlert(4, "Run(). ОШИБКА: " + MESSAGE);
        PerfCheckpoint("Run() -- ОШИБКА: " + MESSAGE);
    }
    LogAlert(2, "Run(). КОНЕЦ");
}

//-------------------------------------------------------------------------
//              Область основного кода
//-------------------------------------------------------------------------

Run();
