// =====================================================================
// HREDU-183. Диагностика: как в ВЫБОРКЕ прочитать GET-параметр из URL страницы
// (matrix_id, macroregion и т.д.), который прилетает через redirect из модалки
// фильтров (HREDU-183_filtry_modal_shag1.js).
//
// Контекст: в selections.md (раздел 11б, "GET-параметры адресной строки") сказано,
// что такие параметры читаются в коде выборки через объект Request, а НЕ через
// подстановки в UI привязки параметра (Env/Context там просто нет отдельного пункта
// под каждый твой параметр -- ты уже проверил, там только фиксированный список вроде
// curEnv.nowDate/curHost/curSite). Но точный синтаксис обращения к Request в этом
// документе помечен как непроверенный (на слух с вебинара, кандидат "Request.Q(...)").
// Вместо того чтобы гадать и тратить твои раунды тестов -- пробуем сразу несколько
// вариантов и смотрим, какой реально сработает.
//
// КАК ЗАПУСТИТЬ (ВАЖНО, отличается от предыдущих диагностик):
//   Предыдущие диагностические файлы ты гонял как тестовый remote_action/скрипт-агент --
//   для ЭТОГО файла так не получится, потому что GET-параметры реально появляются только
//   при настоящей навигации браузера по URL с query string. Поэтому:
//   1. На странице https://als-devwt.vl.vtb/view_doc.html?mode=matrix_test добавь (или
//      возьми существующий, если он там уже есть для теста) виджет "Табличные данные".
//   2. Привяжи к нему этот файл как выборку (временно, только для теста).
//   3. Открой страницу ЧЕРЕЗ модалку фильтров -- нажми "Применить", чтобы в адресной
//      строке оказался реальный URL с matrix_id=...&macroregion=...&mir_code=...
//      (тот самый, что ты уже прислал в примере).
//   4. Посмотри, что показывает таблица -- каждая строка ниже -- это один способ прочитать
//      параметр matrix_id, и его результат (значение или текст ошибки). Пришли мне то,
//      что увидишь -- по этому сразу станет ясно, какой способ реально работает в этой
//      версии платформы, и дальше пропишем его во все параметры (macroregion, mir_code,
//      position_common_id, program_id) сразу правильно, без повторного гадания.
//
// Поле id у каждой строки результата -- обязательное требование к выборкам, иначе
// редактор страниц не сможет использовать результат вообще (см. selections.md, раздел 4).
// =====================================================================

/*
 * Пытается прочитать значение через переданную функцию-попытку; если падает --
 * возвращает текст ошибки вместо самого значения, чтобы строка диагностики не
 * прерывала остальные попытки.
 * @param {Function} fnTry   -   Функция без аргументов, которая пытается прочитать значение.
 * @returns {string}          -   Строковое представление значения либо текст ошибки.
 */
function TryRead(fnTry)
{
    try
    {
        var vValue = fnTry();
        if (vValue == undefined)
        {
            return "-- undefined (переменная/поле не существует или пусто) --";
        }
        return String(vValue);
    }
    catch (_ex)
    {
        return "-- ОШИБКА: " + ExtractUserError(_ex) + " --";
    }
}

RESULT = [];

RESULT.push({ id: 1, method: "matrix_id (голая глобальная переменная, как event_id в education_accept_event_card)", value: TryRead(function () { return matrix_id; }) });
RESULT.push({ id: 2, method: "Request.matrix_id", value: TryRead(function () { return Request.matrix_id; }) });
RESULT.push({ id: 3, method: "Request.Q(\"matrix_id\")", value: TryRead(function () { return Request.Q("matrix_id"); }) });
RESULT.push({ id: 4, method: "Request.GetParam(\"matrix_id\")", value: TryRead(function () { return Request.GetParam("matrix_id"); }) });
RESULT.push({ id: 5, method: "Request.GetOptProperty(\"matrix_id\")", value: TryRead(function () { return Request.GetOptProperty("matrix_id"); }) });
RESULT.push({ id: 6, method: "GET.matrix_id", value: TryRead(function () { return GET.matrix_id; }) });
RESULT.push({ id: 7, method: "PARAMETERS.GetOptProperty(\"matrix_id\")", value: TryRead(function () { return PARAMETERS.GetOptProperty("matrix_id"); }) });
RESULT.push({ id: 8, method: "ScopeWVars.matrix_id", value: TryRead(function () { return ScopeWVars.matrix_id; }) });

COLUMNS = [
    { "data": "id", "editable": true, "hidden": true, "sortable": false },
    { "data": "method", "title": "Способ прочитать matrix_id", "type": "string", "editable": false, "sortable": false, "width": "60%" },
    { "data": "value", "title": "Результат", "type": "string", "editable": false, "sortable": false, "width": "40%" }
];
