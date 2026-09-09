// =====================================================================
// HREDU-183. Диагностика: как в ВЫБОРКЕ прочитать GET-параметр из URL страницы
// (matrix_id, macroregion и т.д.), который прилетает через redirect из модалки
// фильтров (HREDU-183_filtry_modal_shag1.js).
//
// ИСПРАВЛЕНО (09.09.2026): предыдущая версия падала с "JS Syntax Error, Unknown
// Source, line 67". Подозреваемая причина: TryRead(function () {...}) передавал
// АНОНИМНУЮ ФУНКЦИЮ КАК АРГУМЕНТ (closure-колбэк) -- судя по всему, этот скриптовый
// движок такое не поддерживает (мы уже находили, что он не понимает regex-литералы;
// во всех твоих рабочих файлах функции всегда именованные и вызываются напрямую,
// нигде не передаются "как значение"). Переписано без единой анонимной функции --
// каждый вариант чтения параметра проверяется отдельным прямым try/catch.
//
// Контекст: в selections.md (раздел 11б, "GET-параметры адресной строки") сказано,
// что такие параметры читаются в коде выборки через объект Request, а НЕ через
// подстановки в UI привязки параметра (в Env/Context ты уже проверил -- там нет
// отдельного пункта под каждый твой параметр, только фиксированный список вроде
// curEnv.nowDate/curHost/curSite). Но точный синтаксис обращения к Request в этом
// документе помечен как непроверенный (на слух с вебинара, кандидат "Request.Q(...)").
// Вместо гадания -- пробуем сразу несколько вариантов и смотрим, какой сработает.
//
// КАК ЗАПУСТИТЬ (важно, отличается от предыдущих диагностик):
//   GET-параметры реально появляются только при настоящей навигации браузера по URL
//   с query string, поэтому тестовым remote_action/скрипт-агентом это не прогнать.
//   1. На странице https://als-devwt.vl.vtb/view_doc.html?mode=matrix_test привяжи
//      этот файл как выборку к виджету "Табличные данные" (временно, только для теста).
//      Поля результата (см. отдельно): id (integer), method (string), value (string).
//   2. Открой страницу ЧЕРЕЗ модалку фильтров -- нажми "Применить", чтобы в адресной
//      строке оказался реальный URL с matrix_id=...&macroregion=...&mir_code=...
//   3. Если DEBUG = true (см. ниже) -- по пути будут всплывать алерты-чекпойнты, по
//      ним видно, до какого места код доходит. В САМОЙ таблице появятся 8 строк — по
//      одной на каждый способ прочитать matrix_id, с результатом или текстом ошибки.
//   4. Пришли мне и алерты (если были), и что показала таблица.
//
// Поле id у каждой строки результата -- обязательное требование к выборкам (см.
// selections.md, раздел 4), иначе редактор страниц не сможет использовать результат.
// =====================================================================

DEBUG = true;

/*
 * Чек-пойнт для отладки -- alert() с номером шага, только если DEBUG = true.
 * Обёрнут в try/catch, чтобы сама отладочная печать не могла обрушить скрипт.
 * @param {string} sStep
 */
function DebugAlert(sStep)
{
    if (!DEBUG)
    {
        return;
    }
    try
    {
        alert("[DEBUG] " + sStep);
    }
    catch (_exDebug)
    {
        // ничего -- отладочная печать не должна ронять основной код
    }
}

DebugAlert("0. Файл начал выполняться");

RESULT = [];

try
{
    DebugAlert("1. Пробуем вариант 1: matrix_id (голая глобальная переменная)");
    try
    {
        if (matrix_id == undefined)
        {
            RESULT.push({ id: 1, method: "matrix_id (голая глобальная переменная)", value: "-- undefined --" });
        }
        else
        {
            RESULT.push({ id: 1, method: "matrix_id (голая глобальная переменная)", value: String(matrix_id) });
        }
    }
    catch (_ex1)
    {
        RESULT.push({ id: 1, method: "matrix_id (голая глобальная переменная)", value: "-- ОШИБКА: " + ExtractUserError(_ex1) + " --" });
    }
    DebugAlert("1. Готово: " + RESULT[0].value);

    DebugAlert("2. Пробуем вариант 2: Request.matrix_id");
    try
    {
        RESULT.push({ id: 2, method: "Request.matrix_id", value: String(Request.matrix_id) });
    }
    catch (_ex2)
    {
        RESULT.push({ id: 2, method: "Request.matrix_id", value: "-- ОШИБКА: " + ExtractUserError(_ex2) + " --" });
    }
    DebugAlert("2. Готово: " + RESULT[1].value);

    DebugAlert("3. Пробуем вариант 3: Request.Q(\"matrix_id\")");
    try
    {
        RESULT.push({ id: 3, method: "Request.Q(\"matrix_id\")", value: String(Request.Q("matrix_id")) });
    }
    catch (_ex3)
    {
        RESULT.push({ id: 3, method: "Request.Q(\"matrix_id\")", value: "-- ОШИБКА: " + ExtractUserError(_ex3) + " --" });
    }
    DebugAlert("3. Готово: " + RESULT[2].value);

    DebugAlert("4. Пробуем вариант 4: Request.GetParam(\"matrix_id\")");
    try
    {
        RESULT.push({ id: 4, method: "Request.GetParam(\"matrix_id\")", value: String(Request.GetParam("matrix_id")) });
    }
    catch (_ex4)
    {
        RESULT.push({ id: 4, method: "Request.GetParam(\"matrix_id\")", value: "-- ОШИБКА: " + ExtractUserError(_ex4) + " --" });
    }
    DebugAlert("4. Готово: " + RESULT[3].value);

    DebugAlert("5. Пробуем вариант 5: Request.GetOptProperty(\"matrix_id\")");
    try
    {
        RESULT.push({ id: 5, method: "Request.GetOptProperty(\"matrix_id\")", value: String(Request.GetOptProperty("matrix_id")) });
    }
    catch (_ex5)
    {
        RESULT.push({ id: 5, method: "Request.GetOptProperty(\"matrix_id\")", value: "-- ОШИБКА: " + ExtractUserError(_ex5) + " --" });
    }
    DebugAlert("5. Готово: " + RESULT[4].value);

    DebugAlert("6. Пробуем вариант 6: GET.matrix_id");
    try
    {
        RESULT.push({ id: 6, method: "GET.matrix_id", value: String(GET.matrix_id) });
    }
    catch (_ex6)
    {
        RESULT.push({ id: 6, method: "GET.matrix_id", value: "-- ОШИБКА: " + ExtractUserError(_ex6) + " --" });
    }
    DebugAlert("6. Готово: " + RESULT[5].value);

    DebugAlert("7. Пробуем вариант 7: PARAMETERS.GetOptProperty(\"matrix_id\")");
    try
    {
        RESULT.push({ id: 7, method: "PARAMETERS.GetOptProperty(\"matrix_id\")", value: String(PARAMETERS.GetOptProperty("matrix_id")) });
    }
    catch (_ex7)
    {
        RESULT.push({ id: 7, method: "PARAMETERS.GetOptProperty(\"matrix_id\")", value: "-- ОШИБКА: " + ExtractUserError(_ex7) + " --" });
    }
    DebugAlert("7. Готово: " + RESULT[6].value);

    DebugAlert("8. Пробуем вариант 8: ScopeWVars.matrix_id");
    try
    {
        RESULT.push({ id: 8, method: "ScopeWVars.matrix_id", value: String(ScopeWVars.matrix_id) });
    }
    catch (_ex8)
    {
        RESULT.push({ id: 8, method: "ScopeWVars.matrix_id", value: "-- ОШИБКА: " + ExtractUserError(_ex8) + " --" });
    }
    DebugAlert("8. Готово: " + RESULT[7].value);

    DebugAlert("9. Все варианты проверены, строк в RESULT: " + ArrayCount(RESULT));
}
catch (_exMain)
{
    // ГЛАВНАЯ ЗАЩИТА: если что-то падает даже за пределами отдельных try/catch выше --
    // не оставляем RESULT пустым/битым, а кладём туда одну строку с текстом ошибки.
    DebugAlert("ОШИБКА ВЕРХНЕГО УРОВНЯ: " + ExtractUserError(_exMain));
    RESULT = [{ id: 0, method: "ОШИБКА ВЕРХНЕГО УРОВНЯ", value: ExtractUserError(_exMain) }];
}

COLUMNS = [
    { "data": "id", "editable": true, "hidden": true, "sortable": false },
    { "data": "method", "title": "Способ прочитать matrix_id", "type": "string", "editable": false, "sortable": false, "width": "60%" },
    { "data": "value", "title": "Результат", "type": "string", "editable": false, "sortable": false, "width": "40%" }
];
