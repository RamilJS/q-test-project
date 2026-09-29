// =====================================================================
// ВРЕМЕННЫЙ ДИАГНОСТИЧЕСКИЙ ФАЙЛ (29.09.2026) -- НЕ для продакшена.
// Механически сгенерирован из HREDU-183_tep_reports.js: alert() (уже существующая в файле
// функция, просто пишет в лог) вставлен ПОСЛЕ почти каждого исполняемого выражения (461 шт.),
// с номером и номером ИСХОДНОЙ строки в тексте самого alert()'а. Цель -- вставить этот файл
// ВМЕСТО обычного в то же самое место в админке (коллекция "Матрицы обучения. Данные для
// отчета ТЭП"), запустить и прислать лог. Если ошибка -- правда синтаксическая (падает при
// разборе всего файла) -- ни один DIAG-алерт не появится вообще, это тоже важный результат.
// Если алерты какие-то появляются -- последний увиденный "DIAG N (после исходной строки X)"
// точно покажет, где именно останавливается выполнение -- дальше чиним уже прицельно.
// Проверено: node -c и acorn (ES3/ES5/ES6) -- синтаксис самого этого файла корректен.
// После диагностики этот файл нужно удалить/не использовать -- он не заменяет
// HREDU-183_tep_reports.js на продакшене.
// =====================================================================

EnableLog('HREDU-183_7683878110140100214', true);
alert("DIAG 1 (после исходной строки 1)");
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
//   Общее кол-во сотрудников -- все, кто подходят под матрицу (должность + мир-код).
//   План -- кол-во сотрудников которые подходят под матрицу и у кого уже наступил
//     период прохождения тренинга.
//   Факт -- кол-во сотрудников, прошедших тренинг (НЕ ЗАВИСИМО от условий матрицы).
//   Обязательно к прохождению -- все, кто подходят под матрицу и при этом ещё не
//     проходили тренинг.
//   % обученных -- факт/план (уточнено с пользователем 10.09.2026 -- в тексте ТЗ
//     написано "план/факт", но по смыслу метрики и по факту подтверждения это факт/план;
//     сам % не входит в эту выборку -- это отдельный лёгкий расчёт для "Процент обученных",
//     не список сотрудников).
//
// РЕШЕНИЯ, ПРИНЯТЫЕ С ПОЛЬЗОВАТЕЛЕМ (10.09.2026), ЧАСТИЧНО ПЕРЕСМОТРЕНЫ (17.09.2026,
// HREDU-215 "Правки 1" -- см. ниже):
//   1. Аудитория (должность + мир-код) -- РЕАЛИЗУЕМ. ИЗМЕНЕНО (17.09.2026, HREDU-215
//      "Правки 1", по прямому указанию тим-лида пользователя -- "ошиблись в архитектуре"):
//      РАНЬШЕ поля position_common_id/mir_code_id жили НА САМОЙ МАТРИЦЕ (cc_learning_matrice),
//      аудитория была ОДНА на всю матрицу. ТЕПЕРЬ эти поля УДАЛЕНЫ с типа документа
//      "Матрицы обучения" и ДОБАВЛЕНЫ на тип документа "Элементы матриц обучения"
//      (cc_learning_matrice_element, у которых уже были education_method_id/
//      start_study_period/end_study_period) -- значит аудитория теперь СВОЯ У КАЖДОГО
//      ЭЛЕМЕНТА (по факту -- у каждой программы, а если у одной программы несколько
//      элементов с разными position_common_id -- у неё НЕСКОЛЬКО аудиторий, объединяемых
//      через ИЛИ). См. BuildProgramAudienceIndex()/CollaboratorInProgramAudience() ниже.
//      Это МАНДАТОРНОЕ условие "кому вообще адресована ЭТА программа" -- отдельная вещь
//      от РУЧНЫХ фильтров пользователя (те же имена полей, но разный смысл): ручные
//      фильтры дополнительно СУЖАЮТ то, что уже прошло через аудиторию, а не заменяют её.
//   2. "Период прохождения тренинга" (нужен для честного "План") -- НЕ РЕАЛИЗОВАН. На
//      элементе есть start_study_period=1/end_study_period=4 (числа, не даты -- см.
//      диагностику), но неясно: единицы измерения и от какой даты сотрудника отсчитывать.
//      Пользователь решил не тратить на это время сейчас -- УПРОЩЕНИЕ: План = Общее (без
//      доп. фильтра по периоду). Это совпадает с тем, что мы уже видели на тестовых
//      данных пользователя раньше в этом тикете (план и общее количество были равны).
//      ОТКРЫТЫЙ ВОПРОС, вернуться при необходимости -- аналогично открытым вопросам
//      №1-3 в HREDU-181_vostok_polny_spisok_draft.js.
//   3. Факт -- ПЕРЕСМОТРЕНО ПОВТОРНО (17.09.2026, тот же день, что и п.1, но отдельным
//      уточнением от пользователя -- см. AskUserQuestion): раньше "не зависимо от условий
//      матрицы" понималось БУКВАЛЬНО -- фильтр аудитории для "fact" вообще не применялся
//      (сотрудник мог быть кем угодно по должности). Реальный тест (матрица
//      "Менеджер"/"Стандарт менеджер", город Воронеж) показал, что это даёт СТРАННЫЙ
//      результат -- в "Факт" попадали сотрудники СОВСЕМ ДРУГИХ должностей (Экономисты),
//      просто когда-то прошедшие ту же программу по не связанной с этой матрицей причине
//      (программа -- общий каталог, не привязана к конкретной матрице). ПОДТВЕРЖДЕНО
//      пользователем: "Факт" ТЕПЕРЬ ТОЖЕ должен ограничиваться аудиторией (должность+
//      мир-код ХОТЯ БЫ ОДНОГО элемента этой программы) -- "не зависимо от условий
//      матрицы" означает "не зависимо от ПЕРИОДА" (см. п.2 -- он всё равно не
//      реализован), а НЕ "вообще без каких-либо условий". Базовый пул для факта -- все
//      активные сотрудники (как и раньше), ручные фильтры пользователя применяются, ПЛЮС
//      теперь и аудитория программы -- см. п. "б" ниже.
//   4. Обязательно -- аудитория программы (как Общее) МИНУС те, кто прошёл (т.е. строки
//      с пустой датой прохождения).
//
// ИТОГОВАЯ АРХИТЕКТУРА: сначала строим ряды "сотрудник x программа" ТОЧНО как в
// HREDU-181 (программы матрицы, активные сотрудники, дата прохождения, все 4 ручных
// фильтра из URL). Разница только в ДВУХ местах:
//   а) ИЗМЕНЕНО (17.09.2026, дважды в один день -- см. п.3 выше): аудитория применяется
//      НЕ к общему пулу сотрудников ДО построения строк (раньше -- GetMatrixAudienceCollaboratorRows(),
//      убрана), а К КАЖДОЙ ГОТОВОЙ СТРОКЕ (сотрудник x программа) ПОСЛЕ построения --
//      потому что у разных программ в одной и той же строке-сотруднике может быть РАЗНАЯ
//      аудитория (см. row.in_audience в BuildReportRows(), фильтр в Run() сразу после
//      сборки RESULT). Применяется ко ВСЕМ 4 режимам ОДИНАКОВО, включая "fact" (было
//      исключение для "fact" -- убрано по итогам теста, см. п.3);
//   б) ПОСЛЕ того, как готовые строки (с completion_date) собраны -- для "fact" оставляем
//      только строки с НЕпустой датой, для "mandatory" -- только с ПУСТОЙ датой,
//      для "total"/"plan" -- оставляем все строки без изменений.
//
// Параметр result_type -- один из: "total" | "plan" | "fact" | "mandatory".
//   ИЗМЕНЕНО (10.09.2026): раньше читался ТОЛЬКО как фиксированное значение на вкладке
//   "Параметры" отдельного виджета (по виджету на режим). Теперь читается СНАЧАЛА из
//   URL (как остальные фильтры) -- это открывает дорогу к переключению режима самим
//   пользователем на фронтенде (вкладки/ссылки/поле в модалке -- способ ещё
//   обсуждается). Если в URL параметра нет -- запасной путь: старое фиксированное
//   значение LPE (обратная совместимость с уже настроенными виджетами), иначе "total".
//   Подробности -- в начале Run().
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
 
DEBUG = true;              // На проде поставить false после тестирования
LOG_NAME = "agent";        // TODO: уточнить после создания документа в админке
CUR_OBJECT_ID = 7683878110140100214;         // TODO: заполнить ID документа выборки после её создания в админке
 
//-------------------------------------------------------------------------
//              Область функций
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
        alert("DIAG 2 (после исходной строки 160)");
    }
    catch (_exLog)
    {
        // ничего -- сбой логирования не должен ронять основной код
    }
    alert("DIAG 3 (после исходной строки 165)");
}
alert("DIAG 4 (после исходной строки 166)");
 
//-------------------------------------------------------------------------
//              ЗАМЕР ПРОИЗВОДИТЕЛЬНОСТИ (18.09.2026, по просьбе тим-лида
//              пользователя -- медленно грузятся страницы после смены фильтров,
//              нужно понять, тормозит БД (SQL/XQuery/tools.open_doc()) или сам код)
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
alert("DIAG 5 (после исходной строки 202)");
gPerfLastTime = undefined;
alert("DIAG 6 (после исходной строки 203)");
 
/*
* Начинает замер -- запоминает "сейчас" как точку отсчёта. Вызвать ОДИН РАЗ в начале Run().
* @returns {void}
*/
function PerfStart()
{
    if (!PERF_DEBUG) { return; }
    gPerfStartTime = PerfNowSafe();
    alert("DIAG 7 (после исходной строки 212)");
    gPerfLastTime = gPerfStartTime;
    alert("DIAG 8 (после исходной строки 213)");
    PerfAlertSafe("[ЗАМЕР] СТАРТ. Время: " + PerfFormatTimestamp(gPerfStartTime));
    alert("DIAG 9 (после исходной строки 214)");
}
alert("DIAG 10 (после исходной строки 215)");
 
/*
* Точка замера -- alert() с текущим временем, временем ЭТОГО шага (с предыдущей точки) и
* временем с начала (с PerfStart()). См. подробности в шапке блока выше.
* @param {string} sLabel   -   Название шага, например "GetActiveCollaboratorRows() -- SQL".
* @returns {void}
*/
function PerfCheckpoint(sLabel)
{
    if (!PERF_DEBUG) { return; }
    var dNow, sMsg;
    alert("DIAG 11 (после исходной строки 226)");
    dNow = PerfNowSafe();
    alert("DIAG 12 (после исходной строки 227)");
    sMsg = "[ЗАМЕР] " + sLabel + ". Время сейчас: " + PerfFormatTimestamp(dNow)
        + "; ЭТОТ шаг занял: " + PerfDiffSafe(gPerfLastTime, dNow)
        + "; всего с начала: " + PerfDiffSafe(gPerfStartTime, dNow);
        alert("DIAG 13 (после исходной строки 230)");
    PerfAlertSafe(sMsg);
    alert("DIAG 14 (после исходной строки 231)");
    gPerfLastTime = dNow;
    alert("DIAG 15 (после исходной строки 232)");
}
alert("DIAG 16 (после исходной строки 233)");
 
function PerfNowSafe()
{
    try { return Date(); } catch (_ex) { return undefined; }
}
alert("DIAG 17 (после исходной строки 238)");
 
function PerfFormatTimestamp(dValue)
{
    try { return (dValue != undefined ? StrDate(dValue, true) : "?"); } catch (_ex) { return "?"; }
}
alert("DIAG 18 (после исходной строки 243)");
 
function PerfDiffSafe(dFrom, dTo)
{
    try
    {
        if (dFrom == undefined || dTo == undefined) { return "?"; }
        return String(Int((dTo - dFrom) * 86400)) + " сек";
        alert("DIAG 19 (после исходной строки 250)");
    }
    catch (_ex)
    {
        return "? сек";
        alert("DIAG 20 (после исходной строки 254)");
    }
    alert("DIAG 21 (после исходной строки 255)");
}
alert("DIAG 22 (после исходной строки 256)");
 
function PerfAlertSafe(sMsg)
{
    try { alert(sMsg); } catch (_ex) { /* alert недоступен в этом контексте -- не роняем код */ }
}
alert("DIAG 23 (после исходной строки 261)");
 
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
        alert("DIAG 24 (после исходной строки 272)");
    }
    catch (_ex)
    {
        return "";
        alert("DIAG 25 (после исходной строки 276)");
    }
    alert("DIAG 26 (после исходной строки 277)");
}
alert("DIAG 27 (после исходной строки 278)");
 
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
    alert("DIAG 28 (после исходной строки 291)");
 
    iUrlLen = StrLen(sUrl);
    alert("DIAG 29 (после исходной строки 293)");
 
    sAmpMarker = "&" + sParamName + "=";
    alert("DIAG 30 (после исходной строки 295)");
    iParamPos = StrOptSubStrPos(sUrl, sAmpMarker, false);
    alert("DIAG 31 (после исходной строки 296)");
    if (iParamPos != undefined)
    {
        iValueStart = iParamPos + StrLen(sAmpMarker);
        alert("DIAG 32 (после исходной строки 299)");
    }
    else
    {
        sQMarkMarker = "?" + sParamName + "=";
        alert("DIAG 33 (после исходной строки 303)");
        iParamPos = StrOptSubStrPos(sUrl, sQMarkMarker, false);
        alert("DIAG 34 (после исходной строки 304)");
        if (iParamPos == undefined)
        {
            return "";
            alert("DIAG 35 (после исходной строки 307)");
        }
        alert("DIAG 36 (после исходной строки 308)");
        iValueStart = iParamPos + StrLen(sQMarkMarker);
        alert("DIAG 37 (после исходной строки 309)");
    }
    alert("DIAG 38 (после исходной строки 310)");
 
    iAmpPos = StrOptSubStrPos(sUrl, "&", false, iValueStart);
    alert("DIAG 39 (после исходной строки 312)");
    iValueEnd = (iAmpPos != undefined ? iAmpPos : iUrlLen);
    alert("DIAG 40 (после исходной строки 313)");
 
    sRawValue = StrRangePos(sUrl, iValueStart, iValueEnd);
    alert("DIAG 41 (после исходной строки 315)");
 
    try
    {
        return UrlDecode(sRawValue);
        alert("DIAG 42 (после исходной строки 319)");
    }
    catch (_exDecode)
    {
        return sRawValue;
        alert("DIAG 43 (после исходной строки 323)");
    }
    alert("DIAG 44 (после исходной строки 324)");
}
alert("DIAG 45 (после исходной строки 325)");
 
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
alert("DIAG 46 (после исходной строки 347)");

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
    alert("DIAG 47 (после исходной строки 360)");
    sqlText = "";
    alert("DIAG 48 (после исходной строки 361)");
    sqlText = sqlText + "select cs.id,\r\n";
    alert("DIAG 49 (после исходной строки 362)");
    sqlText = sqlText + "       c.data.value('(*/name)[1]', 'varchar(max)') as name,\r\n";
    alert("DIAG 50 (после исходной строки 363)");
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_matrix_active'']/value)[1]', 'varchar(max)') as f_matrix_active,\r\n";
    alert("DIAG 51 (после исходной строки 364)");
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_position_names'']/value)[1]', 'varchar(max)') as f_position_names,\r\n";
    alert("DIAG 52 (после исходной строки 365)");
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_position_names_exclude'']/value)[1]', 'varchar(max)') as f_position_names_exclude,\r\n";
    alert("DIAG 53 (после исходной строки 366)");
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_mir_code'']/value)[1]', 'varchar(max)') as f_mir_code,\r\n";
    alert("DIAG 54 (после исходной строки 367)");
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_mir_code_exclude'']/value)[1]', 'varchar(max)') as f_mir_code_exclude,\r\n";
    alert("DIAG 55 (после исходной строки 368)");
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_org_names'']/value)[1]', 'varchar(max)') as f_org_names,\r\n";
    alert("DIAG 56 (после исходной строки 369)");
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_org_names_exclude'']/value)[1]', 'varchar(max)') as f_org_names_exclude,\r\n";
    alert("DIAG 57 (после исходной строки 370)");
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_subdivision_names'']/value)[1]', 'varchar(max)') as f_subdivision_names,\r\n";
    alert("DIAG 58 (после исходной строки 371)");
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_subdivision_names_exclude'']/value)[1]', 'varchar(max)') as f_subdivision_names_exclude,\r\n";
    alert("DIAG 59 (после исходной строки 372)");
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_subdivision_child'']/value)[1]', 'varchar(max)') as f_subdivision_child,\r\n";
    alert("DIAG 60 (после исходной строки 373)");
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_collaborator_statuses_exclude'']/value)[1]', 'varchar(max)') as f_collaborator_statuses_exclude\r\n";
    alert("DIAG 61 (после исходной строки 374)");
    sqlText = sqlText + "from compound_programs cs\r\n";
    alert("DIAG 62 (после исходной строки 375)");
    sqlText = sqlText + "inner join compound_program c on c.id = cs.id";
    alert("DIAG 63 (после исходной строки 376)");
    return ArraySelectAll(XQuery("sql:" + sqlText));
    alert("DIAG 64 (после исходной строки 377)");
}
alert("DIAG 65 (после исходной строки 378)");

/*
 * ДОБАВЛЕНО (29.09.2026, HREDU-237). Массовое чтение задач типа "Учебная программа"
 * (education_method) из ВСЕХ модульных программ ОДНИМ SQL-запросом через XML .nodes().
 * Идентична версии из HREDU-182_procent_obuchennyh.js.
 * @returns {Object[]}   -   {matrix_id, object_id, education_method_id, ptype, delay_days, pname}.
 */
function GetEducationMethodTaskRows()
{
    var sqlText;
    alert("DIAG 66 (после исходной строки 388)");
    sqlText = "";
    alert("DIAG 67 (после исходной строки 389)");
    sqlText = sqlText + "select cs.id as matrix_id,\r\n";
    alert("DIAG 68 (после исходной строки 390)");
    sqlText = sqlText + "       t.p.value('(object_id)[1]', 'bigint') as object_id,\r\n";
    alert("DIAG 69 (после исходной строки 391)");
    sqlText = sqlText + "       t.p.value('(education_method_id)[1]', 'bigint') as education_method_id,\r\n";
    alert("DIAG 70 (после исходной строки 392)");
    sqlText = sqlText + "       t.p.value('(type)[1]', 'varchar(50)') as ptype,\r\n";
    alert("DIAG 71 (после исходной строки 393)");
    sqlText = sqlText + "       t.p.value('(delay_days)[1]', 'int') as delay_days,\r\n";
    alert("DIAG 72 (после исходной строки 394)");
    sqlText = sqlText + "       t.p.value('(name)[1]', 'varchar(max)') as pname\r\n";
    alert("DIAG 73 (после исходной строки 395)");
    sqlText = sqlText + "from compound_programs cs\r\n";
    alert("DIAG 74 (после исходной строки 396)");
    sqlText = sqlText + "inner join compound_program c on c.id = cs.id\r\n";
    alert("DIAG 75 (после исходной строки 397)");
    sqlText = sqlText + "cross apply c.data.nodes('/*/programs/program') as t(p)\r\n";
    alert("DIAG 76 (после исходной строки 398)");
    sqlText = sqlText + "where t.p.value('(type)[1]', 'varchar(50)') = 'education_method'";
    alert("DIAG 77 (после исходной строки 399)");
    return ArraySelectAll(XQuery("sql:" + sqlText));
    alert("DIAG 78 (после исходной строки 400)");
}
alert("DIAG 79 (после исходной строки 401)");

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
    alert("DIAG 80 (после исходной строки 415)");
    ids = [];
    alert("DIAG 81 (после исходной строки 416)");
    for (i = 0; i < ArrayCount(taskRows); i++)
    {
        if (Int(taskRows[i].matrix_id) == Int(matrixId) && OptInt(taskRows[i].education_method_id, 0) > 0)
        {
            ids.push(Int(taskRows[i].education_method_id));
            alert("DIAG 82 (после исходной строки 421)");
        }
        alert("DIAG 83 (после исходной строки 422)");
    }
    alert("DIAG 84 (после исходной строки 423)");
    return ArraySelectDistinct(ids, "This");
    alert("DIAG 85 (после исходной строки 424)");
}
alert("DIAG 86 (после исходной строки 425)");

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
    alert("DIAG 87 (после исходной строки 440)");
    var collaboratorRows;
    alert("DIAG 88 (после исходной строки 441)");
    collaboratorRows = ArraySelectAll(XQuery("for $elem in collaborators where $elem/is_dismiss=false() return $elem"));
    alert("DIAG 89 (после исходной строки 442)");
    LogAlert(1, "GetActiveCollaboratorRows(). Найдено сотрудников: " + ArrayCount(collaboratorRows));
    alert("DIAG 90 (после исходной строки 443)");
    LogAlert(1, "GetActiveCollaboratorRows(). КОНЕЦ");
    alert("DIAG 91 (после исходной строки 444)");
    return collaboratorRows;
    alert("DIAG 92 (после исходной строки 445)");
}
alert("DIAG 93 (после исходной строки 446)");
 
/*
* Находит ID документов коллекции "positions", у которых position_common_id совпадает
* с переданным ID (см. подробное объяснение схемы в HREDU-181_vostok_polny_spisok_draft.js).
* @param {number} iCommonPositionFilter
* @returns {number[]}
*/
function GetPositionIdsByCommonPosition(iCommonPositionFilter)
{
    LogAlert(1, "GetPositionIdsByCommonPosition(). НАЧАЛО. iCommonPositionFilter=" + iCommonPositionFilter);
    alert("DIAG 94 (после исходной строки 456)");
    var positionRows, positionIds, i;
    alert("DIAG 95 (после исходной строки 457)");
    positionRows = ArraySelectAll(XQuery("for $elem in positions where $elem/position_common_id = " + iCommonPositionFilter + " return $elem"));
    alert("DIAG 96 (после исходной строки 458)");
    positionIds = [];
    alert("DIAG 97 (после исходной строки 459)");
    for (i = 0; i < ArrayCount(positionRows); i++)
    {
        positionIds.push(Int(positionRows[i].id));
        alert("DIAG 98 (после исходной строки 462)");
    }
    alert("DIAG 99 (после исходной строки 463)");
    LogAlert(1, "GetPositionIdsByCommonPosition(). Найдено конкретных должностей: " + ArrayCount(positionIds));
    alert("DIAG 100 (после исходной строки 464)");
    LogAlert(1, "GetPositionIdsByCommonPosition(). КОНЕЦ");
    alert("DIAG 101 (после исходной строки 465)");
    return positionIds;
    alert("DIAG 102 (после исходной строки 466)");
}
alert("DIAG 103 (после исходной строки 467)");
 
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
    alert("DIAG 104 (после исходной строки 478)");
    for (i = 0; i < ArrayCount(idArray); i++)
    {
        if (Int(idArray[i]) == Int(value))
        {
            return true;
            alert("DIAG 105 (после исходной строки 483)");
        }
        alert("DIAG 106 (после исходной строки 484)");
    }
    alert("DIAG 107 (после исходной строки 485)");
    return false;
    alert("DIAG 108 (после исходной строки 486)");
}
alert("DIAG 109 (после исходной строки 487)");
 
/*
* Достаёт макрорегион (custom_elem f_2ewj) по всем действующим сотрудникам одним SQL-запросом.
* @returns {Object[]}
*/
function GetMacroregionRows()
{
    LogAlert(1, "GetMacroregionRows(). НАЧАЛО");
    alert("DIAG 110 (после исходной строки 495)");
    var sqlText, macroRows;
    alert("DIAG 111 (после исходной строки 496)");
    sqlText = "";
    alert("DIAG 112 (после исходной строки 497)");
    sqlText = sqlText + "select cs.id,\r\n";
    alert("DIAG 113 (после исходной строки 498)");
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_2ewj'']/value)[1]', 'varchar(max)') as macroregion\r\n";
    alert("DIAG 114 (после исходной строки 499)");
    sqlText = sqlText + "from collaborators cs\r\n";
    alert("DIAG 115 (после исходной строки 500)");
    sqlText = sqlText + "inner join collaborator c on c.id = cs.id\r\n";
    alert("DIAG 116 (после исходной строки 501)");
    sqlText = sqlText + "where cs.is_dismiss != 1";
    alert("DIAG 117 (после исходной строки 502)");
    macroRows = ArraySelectAll(XQuery("sql:" + sqlText));
    alert("DIAG 118 (после исходной строки 503)");
    LogAlert(1, "GetMacroregionRows(). Строк: " + ArrayCount(macroRows));
    alert("DIAG 119 (после исходной строки 504)");
    LogAlert(1, "GetMacroregionRows(). КОНЕЦ");
    alert("DIAG 120 (после исходной строки 505)");
    return macroRows;
    alert("DIAG 121 (после исходной строки 506)");
}
alert("DIAG 122 (после исходной строки 507)");
 
/*
* ДОБАВЛЕНО (14.09.2026, для drill-down из "Процент обученных"): город -- custom_elem
* "sity" (имя технического поля подтверждено пользователем реальным XML документа
* collaborator -- см. HREDU-182_procent_obuchennyh.js). Нужен, чтобы клик по конкретной
* строке-городу в таблице "Процент обученных" открывал список ТОЛЬКО по этому городу,
* а не по всей матрице.
* @returns {Object[]}   -   Массив {id, sity}.
*/
function GetCityRows()
{
    LogAlert(1, "GetCityRows(). НАЧАЛО");
    alert("DIAG 123 (после исходной строки 519)");
    var sqlText, rows;
    alert("DIAG 124 (после исходной строки 520)");
    sqlText = "";
    alert("DIAG 125 (после исходной строки 521)");
    sqlText = sqlText + "select cs.id,\r\n";
    alert("DIAG 126 (после исходной строки 522)");
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''sity'']/value)[1]', 'varchar(max)') as sity\r\n";
    alert("DIAG 127 (после исходной строки 523)");
    sqlText = sqlText + "from collaborators cs\r\n";
    alert("DIAG 128 (после исходной строки 524)");
    sqlText = sqlText + "inner join collaborator c on c.id = cs.id\r\n";
    alert("DIAG 129 (после исходной строки 525)");
    sqlText = sqlText + "where cs.is_dismiss != 1";
    alert("DIAG 130 (после исходной строки 526)");
    rows = ArraySelectAll(XQuery("sql:" + sqlText));
    alert("DIAG 131 (после исходной строки 527)");
    LogAlert(1, "GetCityRows(). Строк: " + ArrayCount(rows));
    alert("DIAG 132 (после исходной строки 528)");
    LogAlert(1, "GetCityRows(). КОНЕЦ");
    alert("DIAG 133 (после исходной строки 529)");
    return rows;
    alert("DIAG 134 (после исходной строки 530)");
}
alert("DIAG 135 (после исходной строки 531)");
 
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
* @param {Object[]} rows   -   Строки с полем "id" (macroRows/cityRows).
* @returns {Object[]}       -   Тот же массив строк, отсортированный по возрастанию id.
*/
function SortRowsById(rows)
{
    return ArraySort(rows, "Int(This.id)", "+");
    alert("DIAG 136 (после исходной строки 572)");
}
alert("DIAG 137 (после исходной строки 573)");
 
/*
* Бинарный поиск строки с полем "id" == targetId в МАССИВЕ, ОТСОРТИРОВАННОМ ПО ВОЗРАСТАНИЮ
* id (см. SortRowsById()). Использует только доступ к массиву по числовому индексу --
* никакого object[computedKey], см. "ЧЕСТНО ПРО РИСК" выше.
* @param {Object[]} sortedRows   -   Массив, УЖЕ отсортированный через SortRowsById().
* @param {number} targetId
* @returns {Object}               -   Найденная строка или undefined.
*/
function BinarySearchById(sortedRows, targetId)
{
    var lo, hi, mid, midId, iTarget;
    alert("DIAG 138 (после исходной строки 585)");
    iTarget = Int(targetId);
    alert("DIAG 139 (после исходной строки 586)");
    lo = 0;
    alert("DIAG 140 (после исходной строки 587)");
    hi = ArrayCount(sortedRows) - 1;
    alert("DIAG 141 (после исходной строки 588)");
    while (lo <= hi)
    {
        mid = Int((lo + hi) / 2);
        alert("DIAG 142 (после исходной строки 591)");
        midId = Int(sortedRows[mid].id);
        alert("DIAG 143 (после исходной строки 592)");
        if (midId == iTarget) { return sortedRows[mid]; }
        else if (midId < iTarget) { lo = mid + 1; }
        else { hi = mid - 1; }
    }
    alert("DIAG 144 (после исходной строки 596)");
    return undefined;
    alert("DIAG 145 (после исходной строки 597)");
}
alert("DIAG 146 (после исходной строки 598)");
 
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
    alert("DIAG 147 (после исходной строки 616)");
}
alert("DIAG 148 (после исходной строки 617)");
 
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
    alert("DIAG 149 (после исходной строки 628)");
    iTarget = Int(value);
    alert("DIAG 150 (после исходной строки 629)");
    lo = 0;
    alert("DIAG 151 (после исходной строки 630)");
    hi = ArrayCount(sortedIdArray) - 1;
    alert("DIAG 152 (после исходной строки 631)");
    while (lo <= hi)
    {
        mid = Int((lo + hi) / 2);
        alert("DIAG 153 (после исходной строки 634)");
        midVal = Int(sortedIdArray[mid]);
        alert("DIAG 154 (после исходной строки 635)");
        if (midVal == iTarget) { return true; }
        else if (midVal < iTarget) { lo = mid + 1; }
        else { hi = mid - 1; }
    }
    alert("DIAG 155 (после исходной строки 639)");
    return false;
    alert("DIAG 156 (после исходной строки 640)");
}
alert("DIAG 157 (после исходной строки 641)");
 
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
    alert("DIAG 158 (после исходной строки 654)");
    cityRow = BinarySearchById(sortedCityRows, collaboratorID);
    alert("DIAG 159 (после исходной строки 655)");
    sCity = (cityRow != undefined && cityRow.sity != undefined ? String(cityRow.sity) : "");
    alert("DIAG 160 (после исходной строки 656)");
    return (sCity != "" ? sCity : "(без города)");
    alert("DIAG 161 (после исходной строки 657)");
}
alert("DIAG 162 (после исходной строки 658)");
 
/*
* Сортирует dateRows по collaborator_id (не по id -- у этих строк нет своего "id",
* ключевое поле здесь -- collaborator_id, см. GetCompletionDateRows()).
* @param {Object[]} dateRows
* @returns {Object[]}
*/
function SortDateRowsByCollaboratorId(dateRows)
{
    return ArraySort(dateRows, "Int(This.collaborator_id)", "+");
    alert("DIAG 163 (после исходной строки 668)");
}
alert("DIAG 164 (после исходной строки 669)");
 
/*
* Достаёт сырое значение мир-кодов (custom_elem f_mir_codes) по всем действующим
* сотрудникам одним SQL-запросом.
* @returns {Object[]}
*/
function GetMirCodeRows()
{
    LogAlert(1, "GetMirCodeRows(). НАЧАЛО");
    alert("DIAG 165 (после исходной строки 678)");
    var sqlText, rows;
    alert("DIAG 166 (после исходной строки 679)");
    sqlText = "";
    alert("DIAG 167 (после исходной строки 680)");
    sqlText = sqlText + "select cs.id,\r\n";
    alert("DIAG 168 (после исходной строки 681)");
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_mir_codes'']/value)[1]', 'varchar(max)') as mir_codes\r\n";
    alert("DIAG 169 (после исходной строки 682)");
    sqlText = sqlText + "from collaborators cs\r\n";
    alert("DIAG 170 (после исходной строки 683)");
    sqlText = sqlText + "inner join collaborator c on c.id = cs.id\r\n";
    alert("DIAG 171 (после исходной строки 684)");
    sqlText = sqlText + "where cs.is_dismiss != 1";
    alert("DIAG 172 (после исходной строки 685)");
    rows = ArraySelectAll(XQuery("sql:" + sqlText));
    alert("DIAG 173 (после исходной строки 686)");
    LogAlert(1, "GetMirCodeRows(). Строк: " + ArrayCount(rows));
    alert("DIAG 174 (после исходной строки 687)");
    LogAlert(1, "GetMirCodeRows(). КОНЕЦ");
    alert("DIAG 175 (после исходной строки 688)");
    return rows;
    alert("DIAG 176 (после исходной строки 689)");
}
alert("DIAG 177 (после исходной строки 690)");
 
/*
* Разбирает сырое значение f_mir_codes ("#LASK#17#|#LASM#17#...") в массив кодов без процентов.
* @param {string} rawValue
* @returns {string[]}
*/
function ExtractMirCodes(rawValue)
{
    var parts, fields, codes, i;
    alert("DIAG 178 (после исходной строки 699)");
    codes = [];
    alert("DIAG 179 (после исходной строки 700)");
    parts = ArrayDirect(ArraySelect(String(rawValue).split("|"), "This != ''"));
    alert("DIAG 180 (после исходной строки 701)");
    for (i = 0; i < ArrayCount(parts); i++)
    {
        fields = ArrayDirect(ArraySelect(String(parts[i]).split("#"), "This != ''"));
        alert("DIAG 181 (после исходной строки 704)");
        if (ArrayCount(fields) > 0)
        {
            codes.push(String(fields[0]));
            alert("DIAG 182 (после исходной строки 707)");
        }
        alert("DIAG 183 (после исходной строки 708)");
    }
    alert("DIAG 184 (после исходной строки 709)");
    return codes;
    alert("DIAG 185 (после исходной строки 710)");
}
alert("DIAG 186 (после исходной строки 711)");
 
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
    alert("DIAG 187 (после исходной строки 732)");
    row = BinarySearchById(sortedMirCodeRows, collaboratorID);
    alert("DIAG 188 (после исходной строки 733)");
    if (row == undefined)
    {
        return false;
        alert("DIAG 189 (после исходной строки 736)");
    }
    alert("DIAG 190 (после исходной строки 737)");
    codes = ExtractMirCodes(row.mir_codes);
    alert("DIAG 191 (после исходной строки 738)");
    return (ArrayOptFind(codes, "String(This) == String(mirCodeFilter)") != undefined);
    alert("DIAG 192 (после исходной строки 739)");
}
alert("DIAG 193 (после исходной строки 740)");
 
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
    alert("DIAG 194 (после исходной строки 762)");
    sqlText = "";
    alert("DIAG 195 (после исходной строки 763)");
    sqlText = sqlText + "select cs.id,\r\n";
    alert("DIAG 196 (после исходной строки 764)");
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''CurrentState'']/value)[1]', 'varchar(max)') as status\r\n";
    alert("DIAG 197 (после исходной строки 765)");
    sqlText = sqlText + "from collaborators cs\r\n";
    alert("DIAG 198 (после исходной строки 766)");
    sqlText = sqlText + "inner join collaborator c on c.id = cs.id\r\n";
    alert("DIAG 199 (после исходной строки 767)");
    sqlText = sqlText + "where cs.is_dismiss != 1";
    alert("DIAG 200 (после исходной строки 768)");
    return ArraySelectAll(XQuery("sql:" + sqlText));
    alert("DIAG 201 (после исходной строки 769)");
}
alert("DIAG 202 (после исходной строки 770)");

/*
 * ДОБАВЛЕНО (29.09.2026, HREDU-237). Матчинг "* текст *"/"текст*"/"*текст" БЕЗ regex --
 * идентична версии из HREDU-182_procent_obuchennyh.js (см. там же историю находок про
 * StrOptSubStrPos()).
 */
function SplitByStar(sPattern)
{
    var parts, iLen, iStart, iPos;
    alert("DIAG 203 (после исходной строки 779)");
    parts = [];
    alert("DIAG 204 (после исходной строки 780)");
    iLen = StrLen(sPattern);
    alert("DIAG 205 (после исходной строки 781)");
    iStart = 0;
    alert("DIAG 206 (после исходной строки 782)");
    while (true)
    {
        iPos = StrOptSubStrPos(sPattern, "*", true, iStart);
        alert("DIAG 207 (после исходной строки 785)");
        if (iPos == undefined)
        {
            parts.push(StrRangePos(sPattern, iStart, iLen));
            alert("DIAG 208 (после исходной строки 788)");
            break;
            alert("DIAG 209 (после исходной строки 789)");
        }
        alert("DIAG 210 (после исходной строки 790)");
        parts.push(StrRangePos(sPattern, iStart, iPos));
        alert("DIAG 211 (после исходной строки 791)");
        iStart = iPos + 1;
        alert("DIAG 212 (после исходной строки 792)");
    }
    alert("DIAG 213 (после исходной строки 793)");
    return parts;
    alert("DIAG 214 (после исходной строки 794)");
}
alert("DIAG 215 (после исходной строки 795)");

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
    alert("DIAG 216 (после исходной строки 807)");
    if (sPattern == "") { return false; }
    bLeadingStar = (StrRangePos(sPattern, 0, 1) == "*");
    alert("DIAG 217 (после исходной строки 809)");
    bTrailingStar = (StrRangePos(sPattern, StrLen(sPattern) - 1, StrLen(sPattern)) == "*");
    alert("DIAG 218 (после исходной строки 810)");
    parts = SplitByStar(sPattern);
    alert("DIAG 219 (после исходной строки 811)");
    iTextLen = StrLen(sText);
    alert("DIAG 220 (после исходной строки 812)");
    iSearchPos = 0;
    alert("DIAG 221 (после исходной строки 813)");
    for (i = 0; i < ArrayCount(parts); i++)
    {
        sSeg = parts[i];
        alert("DIAG 222 (после исходной строки 816)");
        if (sSeg == "") { continue; }
        iFoundPos = StrOptSubStrPos(sText, sSeg, bIgnoreCase, iSearchPos);
        alert("DIAG 223 (после исходной строки 818)");
        if (iFoundPos == undefined) { return false; }
        if (i == 0 && !bLeadingStar && iFoundPos != 0) { return false; }
        iSearchPos = iFoundPos + StrLen(sSeg);
        alert("DIAG 224 (после исходной строки 821)");
    }
    alert("DIAG 225 (после исходной строки 822)");
    if (!bTrailingStar && iSearchPos != iTextLen) { return false; }
    return true;
    alert("DIAG 226 (после исходной строки 824)");
}
alert("DIAG 227 (после исходной строки 825)");

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
    alert("DIAG 228 (после исходной строки 837)");
    if (sPatternsList == undefined || sPatternsList == "") { return false; }
    patterns = ArraySelect(String(sPatternsList).split(";"), "This != ''");
    alert("DIAG 229 (после исходной строки 839)");
    for (i = 0; i < ArrayCount(patterns); i++)
    {
        if (WildcardMatch(patterns[i], sText, bIgnoreCase)) { return true; }
    }
    alert("DIAG 230 (после исходной строки 843)");
    return false;
    alert("DIAG 231 (после исходной строки 844)");
}
alert("DIAG 232 (после исходной строки 845)");

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
    alert("DIAG 233 (после исходной строки 857)");
}
alert("DIAG 234 (после исходной строки 858)");

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
    alert("DIAG 235 (после исходной строки 869)");
}
alert("DIAG 236 (после исходной строки 870)");

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
    alert("DIAG 237 (после исходной строки 881)");
    if (sPatternsList == undefined || String(sPatternsList) == "") { return true; }
    for (i = 0; i < ArrayCount(employeeCodes); i++)
    {
        if (MatchAnySemicolonPattern(sPatternsList, employeeCodes[i], true)) { return true; }
    }
    alert("DIAG 238 (после исходной строки 886)");
    return false;
    alert("DIAG 239 (после исходной строки 887)");
}
alert("DIAG 240 (после исходной строки 888)");

function MirCodeAxisExcludeMatches(sPatternsList, employeeCodes)
{
    var i;
    alert("DIAG 241 (после исходной строки 892)");
    if (sPatternsList == undefined || String(sPatternsList) == "") { return false; }
    for (i = 0; i < ArrayCount(employeeCodes); i++)
    {
        if (MatchAnySemicolonPattern(sPatternsList, employeeCodes[i], true)) { return true; }
    }
    alert("DIAG 242 (после исходной строки 897)");
    return false;
    alert("DIAG 243 (после исходной строки 898)");
}
alert("DIAG 244 (после исходной строки 899)");

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
    alert("DIAG 245 (после исходной строки 914)");

    sPosition = String(collaboratorRow.position_name);
    alert("DIAG 246 (после исходной строки 916)");
    sOrg = String(collaboratorRow.org_name);
    alert("DIAG 247 (после исходной строки 917)");
    sSubdivision = String(collaboratorRow.position_parent_name);
    alert("DIAG 248 (после исходной строки 918)");

    mirCodeRow = BinarySearchById(sortedMirCodeRows, Int(collaboratorRow.id));
    alert("DIAG 249 (после исходной строки 920)");
    employeeCodes = (mirCodeRow != undefined ? ExtractMirCodes(mirCodeRow.mir_codes) : []);
    alert("DIAG 250 (после исходной строки 921)");

    statusRow = BinarySearchById(sortedStatusRows, Int(collaboratorRow.id));
    alert("DIAG 251 (после исходной строки 923)");
    sStatus = (statusRow != undefined && statusRow.status != undefined ? String(statusRow.status) : "");
    alert("DIAG 252 (после исходной строки 924)");

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
    alert("DIAG 253 (после исходной строки 939)");
}
alert("DIAG 254 (после исходной строки 940)");
/*
* Находит минимальную дату прохождения по каждому сотруднику и программе.
* @param {number[]} programIds
* @returns {Object[]}
*/
function GetCompletionDateRows(programIds)
{
    LogAlert(1, "GetCompletionDateRows(). НАЧАЛО");
    alert("DIAG 255 (после исходной строки 948)");
    var sqlText, dateRows;
    alert("DIAG 256 (после исходной строки 949)");
    sqlText = "";
    alert("DIAG 257 (после исходной строки 950)");
    sqlText = sqlText + "select ec.collaborator_id, e.education_method_id, min(ec.start_date) as first_date\r\n";
    alert("DIAG 258 (после исходной строки 951)");
    sqlText = sqlText + "from event_collaborators ec\r\n";
    alert("DIAG 259 (после исходной строки 952)");
    sqlText = sqlText + "join events e on e.id = ec.event_id\r\n";
    alert("DIAG 260 (после исходной строки 953)");
    sqlText = sqlText + "where e.education_method_id in (" + ArrayMerge(programIds, "This", ",") + ")\r\n";
    alert("DIAG 261 (после исходной строки 954)");
    sqlText = sqlText + "group by ec.collaborator_id, e.education_method_id";
    alert("DIAG 262 (после исходной строки 955)");
    dateRows = ArraySelectAll(XQuery("sql:" + sqlText));
    alert("DIAG 263 (после исходной строки 956)");
    LogAlert(1, "GetCompletionDateRows(). Строк: " + ArrayCount(dateRows));
    alert("DIAG 264 (после исходной строки 957)");
    LogAlert(1, "GetCompletionDateRows(). КОНЕЦ");
    alert("DIAG 265 (после исходной строки 958)");
    return dateRows;
    alert("DIAG 266 (после исходной строки 959)");
}
alert("DIAG 267 (после исходной строки 960)");
 
/*
* Ищет дату прохождения конкретного сотрудника по конкретной программе.
* ИЗМЕНЕНО (18.09.2026, см. "НАЙДЕНО ЗАМЕРОМ ПРОИЗВОДИТЕЛЬНОСТИ" над SortRowsById()):
* dateRows У ОДНОГО сотрудника обычно немного строк (по числу программ, которые он вообще
* когда-либо проходил -- НЕ пропорционально общему числу сотрудников компании), поэтому
* здесь -- БИНАРНЫЙ ПОИСК первой строки этого сотрудника (по sortedDateRows,
* отсортированному через SortDateRowsByCollaboratorId()), а дальше короткий линейный
* проход ТОЛЬКО по строкам ЭТОГО сотрудника (их мало) в поисках нужной программы -- без
* object[computedKey], см. объяснение риска выше.
* @param {Object[]} sortedDateRows   -   Массив, УЖЕ отсортированный через SortDateRowsByCollaboratorId().
* @param {number} collaboratorID
* @param {number} programID
* @returns {string}
*/
function FindCompletionDate(sortedDateRows, collaboratorID, programID)
{
    var lo, hi, mid, midId, iTarget, iProgram, startIdx, i, n;
    alert("DIAG 268 (после исходной строки 978)");
 
    iTarget = Int(collaboratorID);
    alert("DIAG 269 (после исходной строки 980)");
    iProgram = Int(programID);
    alert("DIAG 270 (после исходной строки 981)");
 
    // Бинарный поиск ЛЮБОЙ строки этого сотрудника, затем сдвигаемся к ПЕРВОЙ такой строке
    // (могут быть дубликаты collaborator_id -- по одной строке на каждую пройденную программу).
    lo = 0;
    alert("DIAG 271 (после исходной строки 985)");
    hi = ArrayCount(sortedDateRows) - 1;
    alert("DIAG 272 (после исходной строки 986)");
    startIdx = -1;
    alert("DIAG 273 (после исходной строки 987)");
    while (lo <= hi)
    {
        mid = Int((lo + hi) / 2);
        alert("DIAG 274 (после исходной строки 990)");
        midId = Int(sortedDateRows[mid].collaborator_id);
        alert("DIAG 275 (после исходной строки 991)");
        if (midId == iTarget)
        {
            startIdx = mid;
            alert("DIAG 276 (после исходной строки 994)");
            hi = mid - 1; // ищем более ранний индекс с тем же collaborator_id
        }
        else if (midId < iTarget) { lo = mid + 1; }
        else { hi = mid - 1; }
    }
    alert("DIAG 277 (после исходной строки 999)");
 
    if (startIdx == -1) { return ""; }
 
    n = ArrayCount(sortedDateRows);
    alert("DIAG 278 (после исходной строки 1003)");
    for (i = startIdx; i < n && Int(sortedDateRows[i].collaborator_id) == iTarget; i++)
    {
        if (Int(sortedDateRows[i].education_method_id) == iProgram)
        {
            return StrDate(Date(sortedDateRows[i].first_date), false);
            alert("DIAG 279 (после исходной строки 1008)");
        }
        alert("DIAG 280 (после исходной строки 1009)");
    }
    alert("DIAG 281 (после исходной строки 1010)");
    return "";
    alert("DIAG 282 (после исходной строки 1011)");
}
alert("DIAG 283 (после исходной строки 1012)");
 
/*
* Ищет макрорегион конкретного сотрудника.
* ИЗМЕНЕНО (18.09.2026, см. "НАЙДЕНО ЗАМЕРОМ ПРОИЗВОДИТЕЛЬНОСТИ" над SortRowsById()) --
* бинарный поиск по отсортированному массиву вместо линейного ArrayOptFind().
* @param {Object[]} sortedMacroRows   -   Массив, УЖЕ отсортированный через SortRowsById().
* @param {number} collaboratorID
* @returns {string}
*/
function FindMacroregion(sortedMacroRows, collaboratorID)
{
    var macroRow;
    alert("DIAG 284 (после исходной строки 1024)");
    macroRow = BinarySearchById(sortedMacroRows, collaboratorID);
    alert("DIAG 285 (после исходной строки 1025)");
    return (macroRow != undefined && macroRow.macroregion != undefined ? String(macroRow.macroregion) : "");
    alert("DIAG 286 (после исходной строки 1026)");
}
alert("DIAG 287 (после исходной строки 1027)");
 
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
    alert("DIAG 288 (после исходной строки 1058)");
    rows = [];
    alert("DIAG 289 (после исходной строки 1059)");
    for (i = 0; i < ArrayCount(programIds); i++)
    {
        programID = programIds[i];
        alert("DIAG 290 (после исходной строки 1062)");
        row = new Object();
        alert("DIAG 291 (после исходной строки 1063)");
        row.fullname = String(collaborator.fullname);
        alert("DIAG 292 (после исходной строки 1064)");
        row.position_name = String(collaborator.position_name);
        alert("DIAG 293 (после исходной строки 1065)");
        row.subdivision_name = String(collaborator.position_parent_name);
        alert("DIAG 294 (после исходной строки 1066)");
        row.macroregion = FindMacroregion(macroSorted, Int(collaborator.id));
        alert("DIAG 295 (после исходной строки 1067)");
        row.city = FindCity(citySorted, Int(collaborator.id));
        alert("DIAG 296 (после исходной строки 1068)");
        row.program_name = FindProgramName(programNames, programID);
        alert("DIAG 297 (после исходной строки 1069)");
        row.completion_date = FindCompletionDate(dateSorted, Int(collaborator.id), programID);
        alert("DIAG 298 (после исходной строки 1070)");
        row.in_audience = bInAudience;
        alert("DIAG 299 (после исходной строки 1071)");
        rows.push(row);
        alert("DIAG 300 (после исходной строки 1072)");
    }
    alert("DIAG 301 (после исходной строки 1073)");
    return rows;
    alert("DIAG 302 (после исходной строки 1074)");
}
alert("DIAG 303 (после исходной строки 1075)");

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
    alert("DIAG 304 (после исходной строки 1088)");
    row = ArrayOptFind(programNames, "Int(This.id) == Int(iProgramId)");
    alert("DIAG 305 (после исходной строки 1089)");
    return (row != undefined ? String(row.name) : "id=" + iProgramId);
    alert("DIAG 306 (после исходной строки 1090)");
}
alert("DIAG 307 (после исходной строки 1091)");

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
    alert("DIAG 308 (после исходной строки 1106)");
    var matrixRow, programIds;
    alert("DIAG 309 (после исходной строки 1107)");

    matrixRow = ArrayOptFind(allProgramRows, "Int(This.id) == Int(matrixId)");
    alert("DIAG 310 (после исходной строки 1109)");
    if (matrixRow == undefined)
    {
        throw ("Не найдено модульной программы (compound_program) с id=" + matrixId);
        alert("DIAG 311 (после исходной строки 1112)");
    }
    alert("DIAG 312 (после исходной строки 1113)");
    if (!IsActiveText(matrixRow.f_matrix_active))
    {
        throw ("Модульная программа [" + matrixRow.name + "] (id=" + matrixId + ") деактивирована (f_matrix_active) -- отчёт недоступен для деактивированных программ.");
        alert("DIAG 313 (после исходной строки 1116)");
    }
    alert("DIAG 314 (после исходной строки 1117)");

    programIds = GetProgramIds(taskRows, matrixId);
    alert("DIAG 315 (после исходной строки 1119)");
    if (ArrayCount(programIds) == 0)
    {
        throw ("У модульной программы [" + matrixRow.name + "] не найдено ни одной задачи с типом \"Учебная программа\" (education_method)");
        alert("DIAG 316 (после исходной строки 1122)");
    }
    alert("DIAG 317 (после исходной строки 1123)");

    LogAlert(1, "ResolveMatrixContext(). КОНЕЦ");
    alert("DIAG 318 (после исходной строки 1125)");
    return { matrixRow: matrixRow, programIds: programIds };
    alert("DIAG 319 (после исходной строки 1126)");
}
alert("DIAG 320 (после исходной строки 1127)");
 
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
    alert("DIAG 321 (после исходной строки 1145)");
    var sResultType, sFullUrl, matrixId;
    alert("DIAG 322 (после исходной строки 1146)");
    var iProgramFilter, sMacroregionFilter, sMirCodeFilter, iPositionFilter, sCityFilter;
    alert("DIAG 323 (после исходной строки 1147)");
    var matrixContext, matrixRow, allProgramRows, taskRows;
    alert("DIAG 324 (после исходной строки 1148)");
    var programIds, programNames, collaboratorRows, macroRows, mirCodeRows, cityRows, dateRows, statusRows;
    alert("DIAG 325 (после исходной строки 1149)");
    var macroSorted, citySorted, dateSorted; // ДОБАВЛЕНО (18.09.2026, ЗАМЕР ПРОИЗВОДИТЕЛЬНОСТИ) -- см. SortRowsById()/SortDateRowsByCollaboratorId()
    var mirCodeSorted, statusSorted; // statusSorted -- ДОБАВЛЕНО (29.09.2026, HREDU-237, статус для f_collaborator_statuses_exclude)
    var collaboratorReportRows, i, j, taskRowForName, bInAudience;
    alert("DIAG 326 (после исходной строки 1152)");
    var filteredProgramIds, filteredCollaboratorRows, allowedPositionIds, filteredResultRows;
    alert("DIAG 327 (после исходной строки 1153)");

    ERROR = 0;
    alert("DIAG 328 (после исходной строки 1155)");
    MESSAGE = "";
    alert("DIAG 329 (после исходной строки 1156)");
    RESULT = [];
    alert("DIAG 330 (после исходной строки 1157)");

    try
    {
        PerfStart();
        alert("DIAG 331 (после исходной строки 1161)");

        sFullUrl = GetRequestUrlSafe();
        alert("DIAG 332 (после исходной строки 1163)");
        LogAlert(1, "Run(). Request.Url = [" + sFullUrl + "]");
        alert("DIAG 333 (после исходной строки 1164)");

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
        alert("DIAG 334 (после исходной строки 1178)");
        if (sResultType == "")
        {
            try
            {
                sResultType = String(result_type);
                alert("DIAG 335 (после исходной строки 1183)");
            }
            catch (_exResultTypeParam)
            {
                sResultType = "";
                alert("DIAG 336 (после исходной строки 1187)");
            }
            alert("DIAG 337 (после исходной строки 1188)");
        }
        alert("DIAG 338 (после исходной строки 1189)");
        if (sResultType == "" || sResultType == "undefined")
        {
            sResultType = "total";
            alert("DIAG 339 (после исходной строки 1192)");
        }
        alert("DIAG 340 (после исходной строки 1193)");
        if (sResultType != "total" && sResultType != "plan" && sResultType != "fact" && sResultType != "mandatory")
        {
            throw ("Неизвестный result_type=[" + sResultType + "] -- ожидается одно из: total, plan, fact, mandatory");
            alert("DIAG 341 (после исходной строки 1196)");
        }
        alert("DIAG 342 (после исходной строки 1197)");
        LogAlert(1, "Run(). sResultType=[" + sResultType + "]");
        alert("DIAG 343 (после исходной строки 1198)");

        matrixId = OptInt(GetQueryParam(sFullUrl, "matrix_id"), 0);
        alert("DIAG 344 (после исходной строки 1200)");
        iProgramFilter = OptInt(GetQueryParam(sFullUrl, "program_id"), 0);
        alert("DIAG 345 (после исходной строки 1201)");
        sMacroregionFilter = GetQueryParam(sFullUrl, "macroregion");
        alert("DIAG 346 (после исходной строки 1202)");
        sMirCodeFilter = GetQueryParam(sFullUrl, "mir_code");
        alert("DIAG 347 (после исходной строки 1203)");
        iPositionFilter = OptInt(GetQueryParam(sFullUrl, "position_common_id"), 0);
        alert("DIAG 348 (после исходной строки 1204)");
        // ДОБАВЛЕНО (14.09.2026, drill-down из "Процент обученных"): необязательный
        // фильтр по городу (custom_elem "sity") -- если ссылка со страницы "Процент
        // обученных" пришла для конкретного города, здесь сузим список ТОЛЬКО до него.
        // Если параметра нет -- ведёт себя как раньше (без изменений для существующих
        // ссылок без city).
        sCityFilter = GetQueryParam(sFullUrl, "city");
        alert("DIAG 349 (после исходной строки 1210)");

        LogAlert(1, "Run(). matrixId=" + matrixId + " programFilter=" + iProgramFilter
            + " macroregionFilter=[" + sMacroregionFilter + "] mirCodeFilter=[" + sMirCodeFilter
            + "] positionCommonIdFilter=" + iPositionFilter + " cityFilter=[" + sCityFilter + "]");
            alert("DIAG 350 (после исходной строки 1214)");

        PerfCheckpoint("Разбор Request.Url и всех фильтров -- ЧИСТЫЙ КОД, без SQL");
        alert("DIAG 351 (после исходной строки 1216)");

        if (matrixId == 0)
        {
            throw ("Не передан matrix_id -- выбранная пользователем матрица обучения");
            alert("DIAG 352 (после исходной строки 1220)");
        }
        alert("DIAG 353 (после исходной строки 1221)");

        // ИЗМЕНЕНО (29.09.2026, HREDU-237): раньше здесь был tools.open_doc(matrixId) только
        // чтобы достать matrixName, и ResolveMatrixContext() искала матрицу ПОВТОРНО, по имени.
        // Теперь имя матрицы уже приходит в составе allProgramRows (GetCompoundProgramRows()) --
        // отдельный tools.open_doc() не нужен, см. "ИСТОЧНИК ДАННЫХ HREDU-237" в шапке файла.
        allProgramRows = GetCompoundProgramRows();
        alert("DIAG 354 (после исходной строки 1227)");
        PerfCheckpoint("GetCompoundProgramRows() -- SQL по всем compound_program (custom_elems аудитории)");
        alert("DIAG 355 (после исходной строки 1228)");
        taskRows = GetEducationMethodTaskRows();
        alert("DIAG 356 (после исходной строки 1229)");
        PerfCheckpoint("GetEducationMethodTaskRows() -- SQL (.nodes()) по всем задачам типа education_method во всех compound_program");
        alert("DIAG 357 (после исходной строки 1230)");

        matrixContext = ResolveMatrixContext(matrixId, allProgramRows, taskRows);
        alert("DIAG 358 (после исходной строки 1232)");
        matrixRow = matrixContext.matrixRow;
        alert("DIAG 359 (после исходной строки 1233)");
        programIds = matrixContext.programIds;
        alert("DIAG 360 (после исходной строки 1234)");
        PerfCheckpoint("ResolveMatrixContext() -- поиск матрицы по id + is_active + сбор programIds -- ЧИСТЫЙ КОД (данные уже загружены выше)");
        alert("DIAG 361 (после исходной строки 1235)");

        if (iProgramFilter > 0)
        {
            filteredProgramIds = [];
            alert("DIAG 362 (после исходной строки 1239)");
            for (i = 0; i < ArrayCount(programIds); i++)
            {
                if (Int(programIds[i]) == iProgramFilter)
                {
                    filteredProgramIds.push(programIds[i]);
                    alert("DIAG 363 (после исходной строки 1244)");
                }
                alert("DIAG 364 (после исходной строки 1245)");
            }
            alert("DIAG 365 (после исходной строки 1246)");
            programIds = filteredProgramIds;
            alert("DIAG 366 (после исходной строки 1247)");
            if (ArrayCount(programIds) == 0)
            {
                throw ("Программа [" + iProgramFilter + "] не найдена среди программ выбранной матрицы");
                alert("DIAG 367 (после исходной строки 1250)");
            }
            alert("DIAG 368 (после исходной строки 1251)");
        }
        alert("DIAG 369 (после исходной строки 1252)");

        // ИЗМЕНЕНО (29.09.2026, HREDU-237): раньше -- N x tools.open_doc() (GetProgramTitles(),
        // УБРАНА). Теперь имена программ берутся напрямую из taskRows (задачи compound_program
        // уже содержат собственное поле name/pname) -- без единого обращения к документам.
        programNames = [];
        alert("DIAG 370 (после исходной строки 1257)");
        for (j = 0; j < ArrayCount(programIds); j++)
        {
            taskRowForName = ArrayOptFind(taskRows, "Int(This.matrix_id) == Int(matrixId) && Int(This.education_method_id) == Int(programIds[j])");
            alert("DIAG 371 (после исходной строки 1260)");
            programNames.push({
                id: Int(programIds[j]),
                name: (taskRowForName != undefined && taskRowForName.pname != undefined ? String(taskRowForName.pname) : "id=" + programIds[j])
            });
            alert("DIAG 372 (после исходной строки 1264)");
        }
        alert("DIAG 373 (после исходной строки 1265)");
        PerfCheckpoint("Сбор programNames из taskRows -- ЧИСТЫЙ КОД, без SQL/tools.open_doc()");
        alert("DIAG 374 (после исходной строки 1266)");

        // УБРАНО (29.09.2026, HREDU-237): раньше здесь строился audienceIndex
        // (BuildProgramAudienceIndex(), УБРАНА) -- аудитория ПО КАЖДОЙ ПРОГРАММЕ/элементу.
        // Аудитория теперь ОДНА НА ВСЮ МАТРИЦУ (matrixRow, уже получен выше из
        // ResolveMatrixContext()) -- проверяется ОДИН РАЗ НА СОТРУДНИКА, см.
        // CollaboratorMatchesMatrixAudience() в цикле построения RESULT ниже.

        collaboratorRows = GetActiveCollaboratorRows();
        alert("DIAG 375 (после исходной строки 1274)");
        PerfCheckpoint("GetActiveCollaboratorRows() -- SQL/XQuery по всем активным сотрудникам");
        alert("DIAG 376 (после исходной строки 1275)");

        // mirCodeRows нужен ВСЕГДА -- и для аудитории матрицы (CollaboratorMatchesMatrixAudience(),
        // см. цикл построения RESULT ниже), и для ручного фильтра по мир-коду ниже. Грузим один
        // раз здесь и переиспользуем.
        mirCodeRows = GetMirCodeRows();
        alert("DIAG 377 (после исходной строки 1280)");
        PerfCheckpoint("GetMirCodeRows() -- SQL по мир-кодам сотрудников");
        alert("DIAG 378 (после исходной строки 1281)");
        // НАЙДЕНО (18.09.2026, реальный тест на "Процент обученных" -- см. историю выше):
        // линейный поиск по mirCodeRows -- та же O(n^2)-ловушка, что и у macro/city/date.
        // Сортируем один раз.
        mirCodeSorted = SortRowsById(mirCodeRows);
        alert("DIAG 379 (после исходной строки 1285)");
        PerfCheckpoint("SortRowsById(mirCodeRows) -- сортировка для бинарного поиска по мир-кодам -- ЧИСТЫЙ КОД, O(n log n)");
        alert("DIAG 380 (после исходной строки 1286)");

        // ДОБАВЛЕНО (29.09.2026, HREDU-237): статус сотрудника (CurrentState) -- нужен для
        // f_collaborator_statuses_exclude (аудитория матрицы).
        statusRows = GetStatusRows();
        alert("DIAG 381 (после исходной строки 1290)");
        PerfCheckpoint("GetStatusRows() -- SQL по статусам сотрудников (CurrentState)");
        alert("DIAG 382 (после исходной строки 1291)");
        statusSorted = SortRowsById(statusRows);
        alert("DIAG 383 (после исходной строки 1292)");
        PerfCheckpoint("SortRowsById(statusRows) -- сортировка для бинарного поиска по статусам -- ЧИСТЫЙ КОД, O(n log n)");
        alert("DIAG 384 (после исходной строки 1293)");

        // Дальше -- РУЧНЫЕ фильтры пользователя, точно как в HREDU-181 (без изменений).
        if (iPositionFilter > 0)
        {
            allowedPositionIds = SortIdArray(GetPositionIdsByCommonPosition(iPositionFilter));
            alert("DIAG 385 (после исходной строки 1298)");
            filteredCollaboratorRows = [];
            alert("DIAG 386 (после исходной строки 1299)");
            for (i = 0; i < ArrayCount(collaboratorRows); i++)
            {
                if (IdArrayContainsSorted(allowedPositionIds, OptInt(collaboratorRows[i].position_id, 0)))
                {
                    filteredCollaboratorRows.push(collaboratorRows[i]);
                    alert("DIAG 387 (после исходной строки 1304)");
                }
                alert("DIAG 388 (после исходной строки 1305)");
            }
            alert("DIAG 389 (после исходной строки 1306)");
            collaboratorRows = filteredCollaboratorRows;
            alert("DIAG 390 (после исходной строки 1307)");
            LogAlert(1, "Run(). После ручного фильтра по типовой должности осталось сотрудников: " + ArrayCount(collaboratorRows));
            alert("DIAG 391 (после исходной строки 1308)");
            PerfCheckpoint("Ручной фильтр по типовой должности (GetPositionIdsByCommonPosition() + цикл) -- SQL + ЧИСТЫЙ КОД");
            alert("DIAG 392 (после исходной строки 1309)");
        }
        alert("DIAG 393 (после исходной строки 1310)");

        macroRows = GetMacroregionRows();
        alert("DIAG 394 (после исходной строки 1312)");
        PerfCheckpoint("GetMacroregionRows() -- SQL по макрорегионам сотрудников");
        alert("DIAG 395 (после исходной строки 1313)");
        macroSorted = SortRowsById(macroRows);
        alert("DIAG 396 (после исходной строки 1314)");
        PerfCheckpoint("SortRowsById(macroRows) -- сортировка для бинарного поиска по макрорегионам -- ЧИСТЫЙ КОД, O(n log n)");
        alert("DIAG 397 (после исходной строки 1315)");
        if (sMacroregionFilter != "")
        {
            filteredCollaboratorRows = [];
            alert("DIAG 398 (после исходной строки 1318)");
            for (i = 0; i < ArrayCount(collaboratorRows); i++)
            {
                if (FindMacroregion(macroSorted, Int(collaboratorRows[i].id)) == sMacroregionFilter)
                {
                    filteredCollaboratorRows.push(collaboratorRows[i]);
                    alert("DIAG 399 (после исходной строки 1323)");
                }
                alert("DIAG 400 (после исходной строки 1324)");
            }
            alert("DIAG 401 (после исходной строки 1325)");
            collaboratorRows = filteredCollaboratorRows;
            alert("DIAG 402 (после исходной строки 1326)");
            LogAlert(1, "Run(). После ручного фильтра по макрорегиону осталось сотрудников: " + ArrayCount(collaboratorRows));
            alert("DIAG 403 (после исходной строки 1327)");
            PerfCheckpoint("Ручной фильтр по макрорегиону (цикл по collaboratorRows, теперь через индекс) -- ЧИСТЫЙ КОД");
            alert("DIAG 404 (после исходной строки 1328)");
        }
        alert("DIAG 405 (после исходной строки 1329)");

        if (sMirCodeFilter != "")
        {
            filteredCollaboratorRows = [];
            alert("DIAG 406 (после исходной строки 1333)");
            for (i = 0; i < ArrayCount(collaboratorRows); i++)
            {
                if (CollaboratorHasMirCode(mirCodeSorted, Int(collaboratorRows[i].id), sMirCodeFilter))
                {
                    filteredCollaboratorRows.push(collaboratorRows[i]);
                    alert("DIAG 407 (после исходной строки 1338)");
                }
                alert("DIAG 408 (после исходной строки 1339)");
            }
            alert("DIAG 409 (после исходной строки 1340)");
            collaboratorRows = filteredCollaboratorRows;
            alert("DIAG 410 (после исходной строки 1341)");
            LogAlert(1, "Run(). После ручного фильтра по мир-коду осталось сотрудников: " + ArrayCount(collaboratorRows));
            alert("DIAG 411 (после исходной строки 1342)");
            PerfCheckpoint("Ручной фильтр по мир-коду (CollaboratorHasMirCode() в цикле) -- ЧИСТЫЙ КОД");
            alert("DIAG 412 (после исходной строки 1343)");
        }
        alert("DIAG 413 (после исходной строки 1344)");

        cityRows = GetCityRows();
        alert("DIAG 414 (после исходной строки 1346)");
        PerfCheckpoint("GetCityRows() -- SQL по городам сотрудников");
        alert("DIAG 415 (после исходной строки 1347)");
        citySorted = SortRowsById(cityRows);
        alert("DIAG 416 (после исходной строки 1348)");
        PerfCheckpoint("SortRowsById(cityRows) -- сортировка для бинарного поиска по городам -- ЧИСТЫЙ КОД, O(n log n)");
        alert("DIAG 417 (после исходной строки 1349)");

        if (sCityFilter != "")
        {
            filteredCollaboratorRows = [];
            alert("DIAG 418 (после исходной строки 1353)");
            for (i = 0; i < ArrayCount(collaboratorRows); i++)
            {
                if (FindCity(citySorted, Int(collaboratorRows[i].id)) == sCityFilter)
                {
                    filteredCollaboratorRows.push(collaboratorRows[i]);
                    alert("DIAG 419 (после исходной строки 1358)");
                }
                alert("DIAG 420 (после исходной строки 1359)");
            }
            alert("DIAG 421 (после исходной строки 1360)");
            collaboratorRows = filteredCollaboratorRows;
            alert("DIAG 422 (после исходной строки 1361)");
            LogAlert(1, "Run(). После фильтра по городу осталось сотрудников: " + ArrayCount(collaboratorRows));
            alert("DIAG 423 (после исходной строки 1362)");
            PerfCheckpoint("Фильтр по городу (FindCity() в цикле, теперь через индекс) -- ЧИСТЫЙ КОД");
            alert("DIAG 424 (после исходной строки 1363)");
        }
        alert("DIAG 425 (после исходной строки 1364)");

        dateRows = GetCompletionDateRows(programIds);
        alert("DIAG 426 (после исходной строки 1366)");
        PerfCheckpoint("GetCompletionDateRows() -- SQL по датам прохождения программ");
        alert("DIAG 427 (после исходной строки 1367)");
        dateSorted = SortDateRowsByCollaboratorId(dateRows);
        alert("DIAG 428 (после исходной строки 1368)");
        PerfCheckpoint("SortDateRowsByCollaboratorId(dateRows) -- сортировка для бинарного поиска по датам -- ЧИСТЫЙ КОД, O(n log n)");
        alert("DIAG 429 (после исходной строки 1369)");

        // ИЗМЕНЕНО (29.09.2026, HREDU-237): раньше строки строились ДЛЯ ВСЕХ сотрудников пула,
        // а аудитория (своя у каждой программы/элемента) применялась ПОСЛЕ, отдельным проходом
        // по готовому RESULT (row.in_audience, фильтр ниже -- УБРАН). Теперь аудитория ОДНА НА
        // ВСЮ МАТРИЦУ -- проверяется ОДИН РАЗ НА СОТРУДНИКА, ДО построения его строк
        // (CollaboratorMatchesMatrixAudience()) -- сотрудники не из аудитории просто не попадают
        // в RESULT вообще, без отдельного фильтра постфактум. Применяется ко ВСЕМ 4 режимам
        // одинаково, включая "fact" -- ТО ЖЕ решение пользователя (17.09.2026), что и раньше,
        // логика не поменялась, поменялся только момент проверки.
        RESULT = [];
        alert("DIAG 430 (после исходной строки 1379)");
        for (i = 0; i < ArrayCount(collaboratorRows); i++)
        {
            bInAudience = CollaboratorMatchesMatrixAudience(collaboratorRows[i], matrixRow, mirCodeSorted, statusSorted);
            alert("DIAG 431 (после исходной строки 1382)");
            if (!bInAudience)
            {
                continue;
                alert("DIAG 432 (после исходной строки 1385)");
            }
            alert("DIAG 433 (после исходной строки 1386)");
            collaboratorReportRows = BuildReportRows(collaboratorRows[i], macroSorted, citySorted, dateSorted, programNames, programIds, bInAudience);
            alert("DIAG 434 (после исходной строки 1387)");
            for (j = 0; j < ArrayCount(collaboratorReportRows); j++)
            {
                RESULT.push(collaboratorReportRows[j]);
                alert("DIAG 435 (после исходной строки 1390)");
            }
            alert("DIAG 436 (после исходной строки 1391)");
        }
        alert("DIAG 437 (после исходной строки 1392)");
        LogAlert(1, "Run(). Строк (сотрудник x программа) после фильтра аудитории матрицы, до result_type: " + ArrayCount(RESULT));
        alert("DIAG 438 (после исходной строки 1393)");
        PerfCheckpoint("Цикл построения строк (CollaboratorMatchesMatrixAudience() + BuildReportRows() x сотрудников x программ) -- ЧИСТЫЙ КОД, без SQL. Сотрудников в пуле: " + ArrayCount(collaboratorRows) + "; программ: " + ArrayCount(programIds));
        alert("DIAG 439 (после исходной строки 1394)");

        // НОВОЕ (10.09.2026): финальное разбиение по result_type -- см. "ИТОГОВАЯ
        // АРХИТЕКТУРА" в шапке файла. "total"/"plan" -- без изменений (План = Общее,
        // см. "РЕШЕНИЯ" пункт 2).
        if (sResultType == "fact")
        {
            filteredResultRows = [];
            alert("DIAG 440 (после исходной строки 1401)");
            for (i = 0; i < ArrayCount(RESULT); i++)
            {
                if (RESULT[i].completion_date != "")
                {
                    filteredResultRows.push(RESULT[i]);
                    alert("DIAG 441 (после исходной строки 1406)");
                }
                alert("DIAG 442 (после исходной строки 1407)");
            }
            alert("DIAG 443 (после исходной строки 1408)");
            RESULT = filteredResultRows;
            alert("DIAG 444 (после исходной строки 1409)");
        }
        else if (sResultType == "mandatory")
        {
            filteredResultRows = [];
            alert("DIAG 445 (после исходной строки 1413)");
            for (i = 0; i < ArrayCount(RESULT); i++)
            {
                if (RESULT[i].completion_date == "")
                {
                    filteredResultRows.push(RESULT[i]);
                    alert("DIAG 446 (после исходной строки 1418)");
                }
                alert("DIAG 447 (после исходной строки 1419)");
            }
            alert("DIAG 448 (после исходной строки 1420)");
            RESULT = filteredResultRows;
            alert("DIAG 449 (после исходной строки 1421)");
        }
        alert("DIAG 450 (после исходной строки 1422)");
 
        PerfCheckpoint("Финальное разбиение по result_type (" + sResultType + ") -- ЧИСТЫЙ КОД");
        alert("DIAG 451 (после исходной строки 1424)");
        LogAlert(2, "Run(). Готово. result_type=" + sResultType + ", строк отчёта: " + ArrayCount(RESULT));
        alert("DIAG 452 (после исходной строки 1425)");
        PerfCheckpoint("Run() -- ГОТОВО (успех), строк отчёта: " + ArrayCount(RESULT));
        alert("DIAG 453 (после исходной строки 1426)");
    }
    catch (_ex)
    {
        ERROR = 1;
        alert("DIAG 454 (после исходной строки 1430)");
        MESSAGE = ExtractUserError(_ex);
        alert("DIAG 455 (после исходной строки 1431)");
        LogAlert(4, "Run(). ОШИБКА: " + MESSAGE);
        alert("DIAG 456 (после исходной строки 1432)");
        PerfCheckpoint("Run() -- ОШИБКА: " + MESSAGE);
        alert("DIAG 457 (после исходной строки 1433)");
    }
    alert("DIAG 458 (после исходной строки 1434)");
    LogAlert(2, "Run(). КОНЕЦ");
    alert("DIAG 459 (после исходной строки 1435)");
}
alert("DIAG 460 (после исходной строки 1436)");
 
//-------------------------------------------------------------------------
//              Область основного кода
//-------------------------------------------------------------------------
 
Run();
alert("DIAG 461 (после исходной строки 1442)");
