EnableLog('HREDU-183_7683878110140100214', true);
function alert(_string) {
    LogEvent('HREDU-183_7683878110140100214', _string);
    return _string;
}
 
 
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

// =====================================================================
// ИСТОЧНИК ДАННЫХ HREDU-237 (29.09.2026, по прямому указанию пользователя -- та же смена
// архитектуры, что уже сделана и подтверждена реальным тестом в HREDU-182_procent_obuchennyh.js,
// теперь переносится сюда). ЧИТАТЬ ПЕРЕД ВСЕЙ ИСТОРИЕЙ ВЫШЕ -- она описывает ПРЕЖНЮЮ (СЕЙЧАС
// ЗАМЕНЁННУЮ) модель данных на кастомных каталогах cc_learning_matrice/
// cc_learning_matrice_element -- оставлена как есть для истории решений (не переписана и не
// удалена), но actual источник данных теперь ДРУГОЙ:
//
//   БЫЛО (до 29.09.2026): cc_learning_matrice (матрица) + cc_learning_matrice_element
//   (элементы, каждый со своей аудиторией: position_common_id/mir_code_id, объединяемых по
//   education_method_id через BuildProgramAudienceIndex()/CollaboratorInProgramAudience()).
//
//   СТАЛО: коробочная сущность WebSoft "Модульные программы" (compound_program) --
//   matrix_id из URL теперь id документа compound_program, а не cc_learning_matrice. Внутри --
//   вложенная коллекция programs/program (задачи), нам нужны ТОЛЬКО задачи с type ==
//   "education_method", эл. курсы (type == "course") игнорируются. Аудитория (должность/
//   мир-код/орг-структура/подразделение + статус-исключения) теперь ОДНА НА ВСЮ МОДУЛЬНУЮ
//   ПРОГРАММУ (custom_elems самого compound_program), а НЕ своя у каждого элемента, как было --
//   значит проверка аудитории теперь делается ОДИН РАЗ НА ПАРУ (сотрудник x матрица), а НЕ на
//   каждую пару (сотрудник x программа), как раньше (см. CollaboratorMatchesMatrixAudience()
//   ниже и её использование в Run() -- ПЕРЕД циклом по programIds, а не внутри него). Значения
//   этих полей -- СВОБОДНЫЙ ТЕКСТ с wildcard-паттернами через "*" (не FK-ссылки на каталоги,
//   как было у position_common_id/mir_code_id) -- см. WildcardMatch().
//
//   "Факт" (дата прохождения через GetCompletionDateRows()) НЕ ЗАТРОНУТ этой сменой
//   архитектуры -- education_method_id общее понятие в обеих моделях.
//
//   Названия программ теперь берутся напрямую из taskRows (задачи compound_program уже
//   содержат собственное поле name) -- БЕЗ N x tools.open_doc() (см. GetProgramTitles(),
//   УБРАНА, замена -- programNames, собирается в Run()).
//
//   ПОДТВЕРЖДЕНО РЕАЛЬНЫМ ТЕСТОМ (28-29.09.2026, HREDU-182_procent_obuchennyh.js): таблицы
//   compound_programs/compound_program, техника c.data.value() для custom_elems, массовое
//   чтение вложенных задач через SQL .nodes(), гейт видимости и сама аудитория
//   (CollaboratorMatchesMatrixAudience()) -- всё это уже отработало на реальном тесте (295
//   строк город x программа для матрицы 6946532362879990227, без ошибок). Здесь -- перенос
//   ТЕХ ЖЕ проверенных функций, БЕЗ изменений в их логике.
//
//   НЕ ПЕРЕНЕСЕНО (по решению пользователя, 29.09.2026 -- "удаленное действие по этим
//   отчетам на твое усмотрение"): гейт видимости по роли/иерархии (IsUorApMember()/
//   GetManagerIdRows() и т.д. из HREDU-182_procent_obuchennyh.js) -- в ТЗ HREDU-183 такого
//   требования не было, здесь не добавлялся, чтобы не выйти за рамки текущей задачи (только
//   смена источника данных). Если понадобится -- отдельным шагом, по образцу соседнего файла.
// =====================================================================
 
DEBUG = true;              // На проде поставить false после тестирования
LOG_NAME = "agent";        // TODO: уточнить после создания документа в админке
CUR_OBJECT_ID = 7683878110140100214;         // TODO: заполнить ID документа выборки после её создания в админке
 
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
* УБРАНО (29.09.2026, HREDU-237): GetMatrixRows(matrixName)/GetMatrixElementRows(matrixIds) --
* искали матрицу по имени в cc_learning_matrices/cc_learning_matrice_elements. Заменены на
* GetCompoundProgramRows()/GetEducationMethodTaskRows() ниже -- та же техника, что уже
* подтверждена реальным тестом в HREDU-182_procent_obuchennyh.js (см. "ИСТОЧНИК ДАННЫХ
* HREDU-237" в шапке файла).
*/

/*
 * ДОБАВЛЕНО (29.09.2026, HREDU-237). Безопасно интерпретирует текстовое "булево" значение
 * custom_elem (f_matrix_active и т.п.). Идентична версии из HREDU-182_procent_obuchennyh.js,
 * где уже подтверждена реальным тестом (fallback на try/catch -- на случай, если tools_web
 * недоступен в контексте ВЫБОРКИ).
 * @param {string} sValue
 * @returns {boolean}
 */
function IsActiveText(sValue)
{
    try { return tools_web.is_true(sValue); }
    catch (_ex) { return (String(sValue) == "true" || String(sValue) == "1"); }
}

/*
 * ДОБАВЛЕНО (29.09.2026, HREDU-237). Массовое чтение самих модульных программ
 * (compound_program) -- id, название и ВСЕ custom_elems аудитории -- ОДНИМ SQL-запросом по
 * ВСЕМ документам сразу. Идентична версии из HREDU-182_procent_obuchennyh.js.
 * @returns {Object[]}   -   {id, name, f_matrix_active, f_position_names, f_position_names_exclude,
 *                            f_mir_code, f_mir_code_exclude, f_org_names, f_org_names_exclude,
 *                            f_subdivision_names, f_subdivision_names_exclude, f_subdivision_child,
 *                            f_collaborator_statuses_exclude}.
 */
function GetCompoundProgramRows()
{
    var sqlText;
    sqlText = "";
    sqlText = sqlText + "select cs.id,\r\n";
    sqlText = sqlText + "       c.data.value('(*/name)[1]', 'varchar(max)') as name,\r\n";
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_matrix_active'']/value)[1]', 'varchar(max)') as f_matrix_active,\r\n";
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
    return ArraySelectAll(XQuery("sql:" + sqlText));
}

/*
 * ДОБАВЛЕНО (29.09.2026, HREDU-237). Массовое чтение задач типа "Учебная программа"
 * (education_method) из ВСЕХ модульных программ ОДНИМ SQL-запросом через XML .nodes().
 * Идентична версии из HREDU-182_procent_obuchennyh.js.
 * @returns {Object[]}   -   {matrix_id, object_id, education_method_id, ptype, delay_days, pname}.
 */
function GetEducationMethodTaskRows()
{
    var sqlText;
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
    return ArraySelectAll(XQuery("sql:" + sqlText));
}

/*
* ПЕРЕПИСАНО (29.09.2026, HREDU-237): раньше собирала programIds из elementRows (элементы
* матрицы cc_learning_matrice_element). Теперь -- из taskRows (задачи compound_program, уже
* отфильтрованные по type=education_method в самом SQL), только для ОДНОЙ конкретной матрицы
* (matrixId) -- taskRows читаются сразу по ВСЕМ матрицам одним запросом, фильтр по matrixId
* делаем здесь же, в JS. Идентична версии из HREDU-182_procent_obuchennyh.js.
* @param {Object[]} taskRows
* @param {number} matrixId
* @returns {number[]}
*/
function GetProgramIds(taskRows, matrixId)
{
    var ids, i;
    ids = [];
    for (i = 0; i < ArrayCount(taskRows); i++)
    {
        if (Int(taskRows[i].matrix_id) == Int(matrixId) && OptInt(taskRows[i].education_method_id, 0) > 0)
        {
            ids.push(Int(taskRows[i].education_method_id));
        }
    }
    return ArraySelectDistinct(ids, "This");
}

// УБРАНО (29.09.2026, HREDU-237): GetProgramTitles(programIds) -- строила справочник
// {id, title} через N x tools.open_doc(programID) (открывала документ КАЖДОЙ программы).
// Больше не нужна -- задачи compound_program (taskRows, см. GetEducationMethodTaskRows() выше)
// уже содержат собственное поле name (pname) для каждой программы, справочник programNames
// теперь строится прямо из taskRows одним циклом в Run(), без единого tools.open_doc().
 
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
 * ИСПРАВЛЕНО (30.09.2026, проактивно -- не через отдельный real-платформенный тест именно
 * этой функции, а по аналогии с уже ПОДТВЕРЖДЁННЫМ реальным тестом фактом: .indexOf()/
 * .substring() -- нативные строковые методы JS -- не работают на этой платформе, RUNTIME-
 * ошибка при вызове, не синтаксическая). .split() -- та же категория метода, поэтому вместо
 * него используем уже проверенный (см. SplitByStar() выше) приём через StrOptSubStrPos()/
 * StrRangePos(). Разбивает sText по ЛЮБОЙ строке-разделителю sDelim (не regex, точная
 * подстрока). ПОКА НЕ проверено отдельным реальным тестом именно с непустым
 * f_position_names/f_mir_code -- см. общую находку про .split() в шапке файла.
 * @param {string} sText
 * @param {string} sDelim
 * @returns {string[]}
 */
function SplitByDelimiter(sText, sDelim)
{
    var parts, iLen, iDelimLen, iStart, iPos;
    parts = [];
    if (sText == undefined || sText == "") { return parts; }
    iLen = StrLen(sText);
    iDelimLen = StrLen(sDelim);
    iStart = 0;
    while (true)
    {
        iPos = StrOptSubStrPos(sText, sDelim, false, iStart);
        if (iPos == undefined)
        {
            parts.push(StrRangePos(sText, iStart, iLen));
            break;
        }
        parts.push(StrRangePos(sText, iStart, iPos));
        iStart = iPos + iDelimLen;
    }
    return parts;
}

function ExtractMirCodes(rawValue)
{
    var parts, fields, codes, i;
    codes = [];
    parts = ArrayDirect(ArraySelect(SplitByDelimiter(String(rawValue), "|"), "This != ''"));
    for (i = 0; i < ArrayCount(parts); i++)
    {
        fields = ArrayDirect(ArraySelect(SplitByDelimiter(String(parts[i]), "#"), "This != ''"));
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
 
// УБРАНО (29.09.2026, HREDU-237): ResolveMirCodeText(iMirCodeID) -- резолвила mir_code_id
// (FK на cc_mir_codes) в текст. Больше не нужна -- f_mir_code у compound_program уже
// свободный текст (не FK), см. "ИСТОЧНИК ДАННЫХ HREDU-237" в шапке файла.
//
// УБРАНО (29.09.2026, HREDU-237): BuildProgramAudienceIndex()/SummarizeAudienceIndex()/
// FindProgramAudienceSegments()/CollaboratorInProgramAudience() -- в старой модели аудитория
// была своя у КАЖДОГО ЭЛЕМЕНТА (программы внутри матрицы, через position_common_id/
// mir_code_id). В новой -- ОДНА НА ВСЮ МАТРИЦУ (custom_elems compound_program) -- поэтому
// проверка теперь ОДИН РАЗ НА ПАРУ (сотрудник x матрица), а не на каждую пару (сотрудник x
// программа). Заменены на CollaboratorMatchesMatrixAudience() ниже -- перенесена БЕЗ изменений
// в логике из HREDU-182_procent_obuchennyh.js (уже подтверждена реальным тестом там).

/*
 * ДОБАВЛЕНО (29.09.2026, HREDU-237). Статус сотрудника -- custom_elem "CurrentState" на
 * карточке collaborator. Нужен для f_collaborator_statuses_exclude у модульной программы.
 * Идентична версии из HREDU-182_procent_obuchennyh.js.
 * @returns {Object[]}   -   Массив {id, status}.
 */
function GetStatusRows()
{
    var sqlText;
    sqlText = "";
    sqlText = sqlText + "select cs.id,\r\n";
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''CurrentState'']/value)[1]', 'varchar(max)') as status\r\n";
    sqlText = sqlText + "from collaborators cs\r\n";
    sqlText = sqlText + "inner join collaborator c on c.id = cs.id\r\n";
    sqlText = sqlText + "where cs.is_dismiss != 1";
    return ArraySelectAll(XQuery("sql:" + sqlText));
}

/*
 * ДОБАВЛЕНО (29.09.2026, HREDU-237). Матчинг "* текст *"/"текст*"/"*текст" БЕЗ regex --
 * идентична версии из HREDU-182_procent_obuchennyh.js (см. там же историю находок про
 * StrOptSubStrPos()).
 */
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

/*
 * @param {string} sPattern
 * @param {string} sText
 * @param {boolean} bIgnoreCase   -   true -- игнорировать регистр (см. находку про
 *                                    StrOptSubStrPos в HREDU-182_procent_obuchennyh.js --
 *                                    ИМЕННО true игнорирует регистр, обратное названию).
 * @returns {boolean}
 */
function WildcardMatch(sPattern, sText, bIgnoreCase)
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
        iFoundPos = StrOptSubStrPos(sText, sSeg, bIgnoreCase, iSearchPos);
        if (iFoundPos == undefined) { return false; }
        if (i == 0 && !bLeadingStar && iFoundPos != 0) { return false; }
        iSearchPos = iFoundPos + StrLen(sSeg);
    }
    if (!bTrailingStar && iSearchPos != iTextLen) { return false; }
    return true;
}

/*
 * Проверяет текст против списка паттернов, разделённых ";". true, если текст подходит ХОТЯ БЫ
 * ПОД ОДИН паттерн из списка (ИЛИ).
 * @param {string} sPatternsList
 * @param {string} sText
 * @param {boolean} bIgnoreCase
 * @returns {boolean}
 */
function MatchAnySemicolonPattern(sPatternsList, sText, bIgnoreCase)
{
    var patterns, i;
    if (sPatternsList == undefined || sPatternsList == "") { return false; }
    patterns = ArraySelect(SplitByDelimiter(String(sPatternsList), ";"), "This != ''");
    for (i = 0; i < ArrayCount(patterns); i++)
    {
        if (WildcardMatch(patterns[i], sText, bIgnoreCase)) { return true; }
    }
    return false;
}

/*
 * Одна "положительная" ось аудитории (f_position_names/f_org_names/f_subdivision_names).
 * ПУСТОЙ список паттернов = ось не задана = БЕЗ ОГРАНИЧЕНИЯ по этой оси.
 * @param {string} sPatternsList
 * @param {string} sSingleValue
 * @returns {boolean}
 */
function AxisMatches(sPatternsList, sSingleValue)
{
    if (sPatternsList == undefined || String(sPatternsList) == "") { return true; }
    return MatchAnySemicolonPattern(sPatternsList, sSingleValue, true);
}

/*
 * Ось-ИСКЛЮЧЕНИЕ (f_position_names_exclude и т.п.) -- ПУСТОЙ список = никого не исключаем.
 * @param {string} sPatternsList
 * @param {string} sSingleValue
 * @returns {boolean}
 */
function AxisExcludeMatches(sPatternsList, sSingleValue)
{
    if (sPatternsList == undefined || String(sPatternsList) == "") { return false; }
    return MatchAnySemicolonPattern(sPatternsList, sSingleValue, true);
}

/*
 * Ось мир-кода -- у сотрудника МОЖЕТ БЫТЬ НЕСКОЛЬКО кодов (ExtractMirCodes()) -- совпадение,
 * если ХОТЯ БЫ ОДИН код сотрудника подходит ХОТЯ БЫ ПОД ОДИН паттерн программы.
 * @param {string} sPatternsList
 * @param {string[]} employeeCodes
 * @returns {boolean}
 */
function MirCodeAxisMatches(sPatternsList, employeeCodes)
{
    var i;
    if (sPatternsList == undefined || String(sPatternsList) == "") { return true; }
    for (i = 0; i < ArrayCount(employeeCodes); i++)
    {
        if (MatchAnySemicolonPattern(sPatternsList, employeeCodes[i], true)) { return true; }
    }
    return false;
}

function MirCodeAxisExcludeMatches(sPatternsList, employeeCodes)
{
    var i;
    if (sPatternsList == undefined || String(sPatternsList) == "") { return false; }
    for (i = 0; i < ArrayCount(employeeCodes); i++)
    {
        if (MatchAnySemicolonPattern(sPatternsList, employeeCodes[i], true)) { return true; }
    }
    return false;
}

/*
 * ГЛАВНАЯ функция аудитории HREDU-237 -- заменяет CollaboratorInProgramAudience() (УБРАНА).
 * Проверяется ОДИН РАЗ НА ПАРУ (сотрудник x матрица) -- см. блок выше. Перенесена БЕЗ
 * изменений в логике из HREDU-182_procent_obuchennyh.js, где уже подтверждена реальным тестом
 * (29.09.2026, гейт + аудитория прошли без ошибок на матрице 6946532362879990227).
 * @param {Object} collaboratorRow      -   Строка из GetActiveCollaboratorRows().
 * @param {Object} matrixRow            -   Строка из GetCompoundProgramRows() -- аудитория ЭТОЙ матрицы.
 * @param {Object[]} sortedMirCodeRows  -   GetMirCodeRows() + SortRowsById().
 * @param {Object[]} sortedStatusRows   -   GetStatusRows() + SortRowsById().
 * @returns {boolean}
 */
function CollaboratorMatchesMatrixAudience(collaboratorRow, matrixRow, sortedMirCodeRows, sortedStatusRows)
{
    var sPosition, sOrg, sSubdivision, sStatus, employeeCodes, mirCodeRow, statusRow;

    sPosition = String(collaboratorRow.position_name);
    sOrg = String(collaboratorRow.org_name);
    sSubdivision = String(collaboratorRow.position_parent_name);

    mirCodeRow = BinarySearchById(sortedMirCodeRows, Int(collaboratorRow.id));
    employeeCodes = (mirCodeRow != undefined ? ExtractMirCodes(mirCodeRow.mir_codes) : []);

    statusRow = BinarySearchById(sortedStatusRows, Int(collaboratorRow.id));
    sStatus = (statusRow != undefined && statusRow.status != undefined ? String(statusRow.status) : "");

    // ИСКЛЮЧЕНИЯ -- хватает ОДНОГО совпадения по ЛЮБОЙ оси, чтобы исключить сотрудника.
    if (AxisExcludeMatches(matrixRow.f_position_names_exclude, sPosition)) { return false; }
    if (AxisExcludeMatches(matrixRow.f_org_names_exclude, sOrg)) { return false; }
    if (AxisExcludeMatches(matrixRow.f_subdivision_names_exclude, sSubdivision)) { return false; }
    if (MirCodeAxisExcludeMatches(matrixRow.f_mir_code_exclude, employeeCodes)) { return false; }
    if (AxisExcludeMatches(matrixRow.f_collaborator_statuses_exclude, sStatus)) { return false; }

    // ОСНОВНЫЕ ОСИ -- "И" между ЗАПОЛНЕННЫМИ осями (пустая ось = без ограничения).
    if (!AxisMatches(matrixRow.f_position_names, sPosition)) { return false; }
    if (!AxisMatches(matrixRow.f_org_names, sOrg)) { return false; }
    if (!AxisMatches(matrixRow.f_subdivision_names, sSubdivision)) { return false; }
    if (!MirCodeAxisMatches(matrixRow.f_mir_code, employeeCodes)) { return false; }

    return true;
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
 
// УБРАНО (29.09.2026, HREDU-237): FindProgramTitle(programTitles, programID) -- искала
// название программы в справочнике, построенном через GetProgramTitles() (N x tools.open_doc(),
// УБРАНА). Заменена на FindProgramName(programNames, iProgramId) ниже (см. рядом с
// ResolveMatrixContext()) -- источник имён теперь taskRows, без единого tools.open_doc().
 
/*
* Собирает строки отчёта для ОДНОГО сотрудника -- по одной строке на каждую программу
* матрицы (те же 6 полей, что в HREDU-181, ПЛЮС город -- см. ниже).
* ИЗМЕНЕНО (16.09.2026, по просьбе пользователя): добавлено поле row.city -- отдельная
* колонка "Город" рядом с "Макрорегион".
* ИЗМЕНЕНО (29.09.2026, HREDU-237): раньше принимала audienceIndex/mirCodeSorted и считала
* row.in_audience ПО КАЖДОЙ ПРОГРАММЕ (CollaboratorInProgramAudience(), УБРАНА) -- в новой
* модели аудитория ОДНА НА ВСЮ МАТРИЦУ, поэтому проверяется ОДИН РАЗ НА СОТРУДНИКА В Run()
* (CollaboratorMatchesMatrixAudience()), а сюда приходит уже готовым булевым результатом
* (bInAudience) -- одинаковым для всех строк этого сотрудника. programTitles/FindProgramTitle
* заменены на programNames/FindProgramName (имена программ теперь из taskRows, а не из
* N x tools.open_doc() -- см. "ИСТОЧНИК ДАННЫХ HREDU-237" в шапке файла).
* @param {Object} collaborator
* @param {Object[]} macroSorted
* @param {Object[]} citySorted
* @param {Object[]} dateSorted
* @param {Object[]} programNames   -   Массив {id, name}, собран в Run() из taskRows.
* @param {number[]} programIds
* @param {boolean} bInAudience     -   Результат CollaboratorMatchesMatrixAudience() для ЭТОГО
*                                      сотрудника и ТЕКУЩЕЙ матрицы -- один на всех программ.
* @returns {Object[]}
*/
function BuildReportRows(collaborator, macroSorted, citySorted, dateSorted, programNames, programIds, bInAudience)
{
    var rows, row, i, programID;
    rows = [];
    for (i = 0; i < ArrayCount(programIds); i++)
    {
        programID = programIds[i];
        row = new Object();
        row.fullname = String(collaborator.fullname);
        row.position_name = String(collaborator.position_name);
        row.subdivision_name = String(collaborator.position_parent_name);
        row.macroregion = FindMacroregion(macroSorted, Int(collaborator.id));
        row.city = FindCity(citySorted, Int(collaborator.id));
        row.program_name = FindProgramName(programNames, programID);
        row.completion_date = FindCompletionDate(dateSorted, Int(collaborator.id), programID);
        row.in_audience = bInAudience;
        rows.push(row);
    }
    return rows;
}

/*
* Резолвит id программы в текст через уже готовый кэш (см. Run() -- programNames строится
* ОДИН РАЗ на все уникальные programIds, прямо из taskRows -- см. GetEducationMethodTaskRows()).
* ПЕРЕИМЕНОВАНА (29.09.2026, HREDU-237, было FindProgramTitle -- принимала programTitles,
* собранные через N x tools.open_doc()).
* @param {Object[]} programNames   -   Массив {id, name}.
* @param {number} iProgramId
* @returns {string}
*/
function FindProgramName(programNames, iProgramId)
{
    var row;
    row = ArrayOptFind(programNames, "Int(This.id) == Int(iProgramId)");
    return (row != undefined ? String(row.name) : "id=" + iProgramId);
}

/*
* ПЕРЕПИСАНО (29.09.2026, HREDU-237): раньше искала матрицу по ИМЕНИ (GetMatrixRows(matrixName),
* УБРАНА) двумя отдельными XQuery-запросами. Теперь ищет ОДНИМ бинарным поиском по id в уже
* загруженном bulk-SQL результате (GetCompoundProgramRows()) -- отличать "нет вообще" от
* "деактивирована" можно по ОДНОМУ и тому же результату, без повторного запроса. Идентична
* версии из HREDU-182_procent_obuchennyh.js (там же подтверждена реальным тестом).
* @param {number} matrixId
* @param {Object[]} allProgramRows   -   Результат GetCompoundProgramRows().
* @param {Object[]} taskRows         -   Результат GetEducationMethodTaskRows() (по ВСЕМ матрицам).
* @returns {Object}   -   { matrixRow: Object, programIds: number[] }.
*/
function ResolveMatrixContext(matrixId, allProgramRows, taskRows)
{
    LogAlert(1, "ResolveMatrixContext(). НАЧАЛО. matrixId=" + matrixId);
    var matrixRow, programIds;

    matrixRow = ArrayOptFind(allProgramRows, "Int(This.id) == Int(matrixId)");
    if (matrixRow == undefined)
    {
        throw ("Не найдено модульной программы (compound_program) с id=" + matrixId);
    }
    if (!IsActiveText(matrixRow.f_matrix_active))
    {
        throw ("Модульная программа [" + matrixRow.name + "] (id=" + matrixId + ") деактивирована (f_matrix_active) -- отчёт недоступен для деактивированных программ.");
    }

    programIds = GetProgramIds(taskRows, matrixId);
    if (ArrayCount(programIds) == 0)
    {
        throw ("У модульной программы [" + matrixRow.name + "] не найдено ни одной задачи с типом [Учебная программа] (education_method)");
    }

    LogAlert(1, "ResolveMatrixContext(). КОНЕЦ");
    return { matrixRow: matrixRow, programIds: programIds };
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
    var sResultType, sFullUrl, matrixId;
    var iProgramFilter, sMacroregionFilter, sMirCodeFilter, iPositionFilter, sCityFilter;
    var matrixContext, matrixRow, allProgramRows, taskRows;
    var programIds, programNames, collaboratorRows, macroRows, mirCodeRows, cityRows, dateRows, statusRows;
    var macroSorted, citySorted, dateSorted; // ДОБАВЛЕНО (18.09.2026, ЗАМЕР ПРОИЗВОДИТЕЛЬНОСТИ) -- см. SortRowsById()/SortDateRowsByCollaboratorId()
    var mirCodeSorted, statusSorted; // statusSorted -- ДОБАВЛЕНО (29.09.2026, HREDU-237, статус для f_collaborator_statuses_exclude)
    var collaboratorReportRows, i, j, taskRowForName, bInAudience;
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

        // ИЗМЕНЕНО (29.09.2026, HREDU-237): раньше здесь был tools.open_doc(matrixId) только
        // чтобы достать matrixName, и ResolveMatrixContext() искала матрицу ПОВТОРНО, по имени.
        // Теперь имя матрицы уже приходит в составе allProgramRows (GetCompoundProgramRows()) --
        // отдельный tools.open_doc() не нужен, см. "ИСТОЧНИК ДАННЫХ HREDU-237" в шапке файла.
        allProgramRows = GetCompoundProgramRows();
        PerfCheckpoint("GetCompoundProgramRows() -- SQL по всем compound_program (custom_elems аудитории)");
        taskRows = GetEducationMethodTaskRows();
        PerfCheckpoint("GetEducationMethodTaskRows() -- SQL (.nodes()) по всем задачам типа education_method во всех compound_program");

        matrixContext = ResolveMatrixContext(matrixId, allProgramRows, taskRows);
        matrixRow = matrixContext.matrixRow;
        programIds = matrixContext.programIds;
        PerfCheckpoint("ResolveMatrixContext() -- поиск матрицы по id + is_active + сбор programIds -- ЧИСТЫЙ КОД (данные уже загружены выше)");

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

        // ИЗМЕНЕНО (29.09.2026, HREDU-237): раньше -- N x tools.open_doc() (GetProgramTitles(),
        // УБРАНА). Теперь имена программ берутся напрямую из taskRows (задачи compound_program
        // уже содержат собственное поле name/pname) -- без единого обращения к документам.
        programNames = [];
        for (j = 0; j < ArrayCount(programIds); j++)
        {
            taskRowForName = ArrayOptFind(taskRows, "Int(This.matrix_id) == Int(matrixId) && Int(This.education_method_id) == Int(programIds[j])");
            programNames.push({
                id: Int(programIds[j]),
                name: (taskRowForName != undefined && taskRowForName.pname != undefined ? String(taskRowForName.pname) : "id=" + programIds[j])
            });
        }
        PerfCheckpoint("Сбор programNames из taskRows -- ЧИСТЫЙ КОД, без SQL/tools.open_doc()");

        // УБРАНО (29.09.2026, HREDU-237): раньше здесь строился audienceIndex
        // (BuildProgramAudienceIndex(), УБРАНА) -- аудитория ПО КАЖДОЙ ПРОГРАММЕ/элементу.
        // Аудитория теперь ОДНА НА ВСЮ МАТРИЦУ (matrixRow, уже получен выше из
        // ResolveMatrixContext()) -- проверяется ОДИН РАЗ НА СОТРУДНИКА, см.
        // CollaboratorMatchesMatrixAudience() в цикле построения RESULT ниже.

        collaboratorRows = GetActiveCollaboratorRows();
        PerfCheckpoint("GetActiveCollaboratorRows() -- SQL/XQuery по всем активным сотрудникам");

        // mirCodeRows нужен ВСЕГДА -- и для аудитории матрицы (CollaboratorMatchesMatrixAudience(),
        // см. цикл построения RESULT ниже), и для ручного фильтра по мир-коду ниже. Грузим один
        // раз здесь и переиспользуем.
        mirCodeRows = GetMirCodeRows();
        PerfCheckpoint("GetMirCodeRows() -- SQL по мир-кодам сотрудников");
        // НАЙДЕНО (18.09.2026, реальный тест на "Процент обученных" -- см. историю выше):
        // линейный поиск по mirCodeRows -- та же O(n^2)-ловушка, что и у macro/city/date.
        // Сортируем один раз.
        mirCodeSorted = SortRowsById(mirCodeRows);
        PerfCheckpoint("SortRowsById(mirCodeRows) -- сортировка для бинарного поиска по мир-кодам -- ЧИСТЫЙ КОД, O(n log n)");

        // ДОБАВЛЕНО (29.09.2026, HREDU-237): статус сотрудника (CurrentState) -- нужен для
        // f_collaborator_statuses_exclude (аудитория матрицы).
        statusRows = GetStatusRows();
        PerfCheckpoint("GetStatusRows() -- SQL по статусам сотрудников (CurrentState)");
        statusSorted = SortRowsById(statusRows);
        PerfCheckpoint("SortRowsById(statusRows) -- сортировка для бинарного поиска по статусам -- ЧИСТЫЙ КОД, O(n log n)");

        // Дальше -- РУЧНЫЕ фильтры пользователя, точно как в HREDU-181 (без изменений).
        if (iPositionFilter > 0)
        {
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

        cityRows = GetCityRows();
        PerfCheckpoint("GetCityRows() -- SQL по городам сотрудников");
        citySorted = SortRowsById(cityRows);
        PerfCheckpoint("SortRowsById(cityRows) -- сортировка для бинарного поиска по городам -- ЧИСТЫЙ КОД, O(n log n)");

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

        // ИЗМЕНЕНО (29.09.2026, HREDU-237): раньше строки строились ДЛЯ ВСЕХ сотрудников пула,
        // а аудитория (своя у каждой программы/элемента) применялась ПОСЛЕ, отдельным проходом
        // по готовому RESULT (row.in_audience, фильтр ниже -- УБРАН). Теперь аудитория ОДНА НА
        // ВСЮ МАТРИЦУ -- проверяется ОДИН РАЗ НА СОТРУДНИКА, ДО построения его строк
        // (CollaboratorMatchesMatrixAudience()) -- сотрудники не из аудитории просто не попадают
        // в RESULT вообще, без отдельного фильтра постфактум. Применяется ко ВСЕМ 4 режимам
        // одинаково, включая "fact" -- ТО ЖЕ решение пользователя (17.09.2026), что и раньше,
        // логика не поменялась, поменялся только момент проверки.
        RESULT = [];
        for (i = 0; i < ArrayCount(collaboratorRows); i++)
        {
            bInAudience = CollaboratorMatchesMatrixAudience(collaboratorRows[i], matrixRow, mirCodeSorted, statusSorted);
            if (!bInAudience)
            {
                continue;
            }
            collaboratorReportRows = BuildReportRows(collaboratorRows[i], macroSorted, citySorted, dateSorted, programNames, programIds, bInAudience);
            for (j = 0; j < ArrayCount(collaboratorReportRows); j++)
            {
                RESULT.push(collaboratorReportRows[j]);
            }
        }
        LogAlert(1, "Run(). Строк (сотрудник x программа) после фильтра аудитории матрицы, до result_type: " + ArrayCount(RESULT));
        PerfCheckpoint("Цикл построения строк (CollaboratorMatchesMatrixAudience() + BuildReportRows() x сотрудников x программ) -- ЧИСТЫЙ КОД, без SQL. Сотрудников в пуле: " + ArrayCount(collaboratorRows) + "; программ: " + ArrayCount(programIds));

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
