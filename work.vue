// =====================================================================
// HREDU-183. Диагностика v3: как в ВЫБОРКЕ прочитать GET-параметр из URL.
//
// ИТОГИ v2 (09.09.2026, реальный прогон через redirect с ?matrix_id=...):
//   matrix_id (голая переменная)              -- "matrix_id not defined"
//   Request.matrix_id                          -- "Unknown object property: matrix_id"
//   Request.Q("matrix_id")                     -- "Unknown method: Q()"
//   Request.GetParam("matrix_id")              -- "Unknown method: GetParam()"
//   Request.GetOptProperty("matrix_id")        -- "Unknown method: GetOptProperty()"
//   GET.matrix_id                              -- "GET not defined"
//   PARAMETERS.GetOptProperty("matrix_id")     -- "PARAMETERS not defined"
//   ScopeWVars.matrix_id                       -- "ScopeWVars not defined"
//
// ВЫВОД: единственный реально существующий объект в этом списке -- Request (у него
// "Unknown object property"/"Unknown method", а не "not defined" -- то есть сам объект
// есть, просто не те имена свойств/методов). Все остальные глобалы (matrix_id, GET,
// PARAMETERS, ScopeWVars) в контексте ВЫБОРКИ попросту не существуют -- похоже,
// PARAMETERS/ScopeWVars это специфика УДАЛЁННЫХ ДЕЙСТВИЙ (там PARAMETERS точно
// работал -- в filtry_modal_shag1.js), а не выборок.
//
// Вместо дальнейшего перебора наугад -- в этой версии:
//   1) перечисляем СВОИМИ СИЛАМИ, какие свойства/методы реально есть у Request
//      (та же техника, что сработала для полей коллекции positions в HREDU-181);
//   2) пробуем ещё несколько типовых кандидатов (QueryString, GetProperty, Params,
//      квадратные скобки Request["matrix_id"]).
//
// КАК ЗАПУСТИТЬ: так же, как v2 -- привязать выборкой к Табличным данным на странице
// matrix_test и открыть страницу ЧЕРЕЗ модалку (кнопка "Применить"), чтобы в URL был
// реальный query string. Поля результата в админке: id (integer), method (string),
// value (string) -- как и раньше, не меняются.
// =====================================================================

DEBUG = true;

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
iRowId = 0;

try
{
    // --- Часть 1: перечисляем свойства/методы Request через for-in ---------------
    DebugAlert("1. Начинаем перебор свойств Request через for-in");
    sEnumReport = "";
    try
    {
        for (fld in Request)
        {
            try
            {
                sEnumReport = sEnumReport + String(fld) + "\r\n";
            }
            catch (_exFld)
            {
                sEnumReport = sEnumReport + "<не читается: " + ExtractUserError(_exFld) + ">\r\n";
            }
        }
    }
    catch (_exEnum)
    {
        sEnumReport = "for-in по Request упал целиком: " + ExtractUserError(_exEnum);
    }
    iRowId = iRowId + 1;
    RESULT.push({ id: iRowId, method: "for (fld in Request) -- список того, что перечислилось", value: (sEnumReport != "" ? sEnumReport : "-- пусто, перебор ничего не дал --") });
    DebugAlert("1. Готово, длина отчёта: " + sEnumReport.length);

    // --- Часть 2: пробуем прочитать значение по каждому имени из перебора --------
    // (если for-in что-то дал -- fld это, скорее всего, ИМЯ свойства, пробуем Request[fld])
    DebugAlert("2. Пробуем Request[fld] для каждого перечисленного имени, ищем matrix_id");
    sFoundReport = "";
    try
    {
        for (fld in Request)
        {
            try
            {
                if (String(fld) == "matrix_id")
                {
                    sFoundReport = sFoundReport + "НАШЛИ! Request[\"matrix_id\"] = [" + String(Request[fld]) + "]\r\n";
                }
            }
            catch (_exFld2)
            {
                // пропускаем
            }
        }
    }
    catch (_exEnum2)
    {
        sFoundReport = "перебор с поиском matrix_id упал: " + ExtractUserError(_exEnum2);
    }
    iRowId = iRowId + 1;
    RESULT.push({ id: iRowId, method: "поиск matrix_id среди перечисленных имён Request", value: (sFoundReport != "" ? sFoundReport : "-- matrix_id среди перечисленных имён не найден --") });
    DebugAlert("2. Готово");

    // --- Часть 3: квадратные скобки напрямую по известному имени -----------------
    DebugAlert("3. Пробуем Request[\"matrix_id\"] напрямую");
    try
    {
        RESULT.push({ id: (iRowId = iRowId + 1), method: "Request[\"matrix_id\"]", value: String(Request["matrix_id"]) });
    }
    catch (_ex3)
    {
        RESULT.push({ id: (iRowId = iRowId + 1), method: "Request[\"matrix_id\"]", value: "-- ОШИБКА: " + ExtractUserError(_ex3) + " --" });
    }
    DebugAlert("3. Готово");

    // --- Часть 4: классические ASP-подобные варианты ------------------------------
    DebugAlert("4. Пробуем Request.QueryString(\"matrix_id\")");
    try
    {
        RESULT.push({ id: (iRowId = iRowId + 1), method: "Request.QueryString(\"matrix_id\")", value: String(Request.QueryString("matrix_id")) });
    }
    catch (_ex4)
    {
        RESULT.push({ id: (iRowId = iRowId + 1), method: "Request.QueryString(\"matrix_id\")", value: "-- ОШИБКА: " + ExtractUserError(_ex4) + " --" });
    }
    DebugAlert("4. Готово");

    DebugAlert("5. Пробуем Request.QueryString (без вызова, как свойство-строка)");
    try
    {
        RESULT.push({ id: (iRowId = iRowId + 1), method: "Request.QueryString (свойство)", value: String(Request.QueryString) });
    }
    catch (_ex5)
    {
        RESULT.push({ id: (iRowId = iRowId + 1), method: "Request.QueryString (свойство)", value: "-- ОШИБКА: " + ExtractUserError(_ex5) + " --" });
    }
    DebugAlert("5. Готово");

    DebugAlert("6. Пробуем Request.GetProperty(\"matrix_id\") (без Opt)");
    try
    {
        RESULT.push({ id: (iRowId = iRowId + 1), method: "Request.GetProperty(\"matrix_id\")", value: String(Request.GetProperty("matrix_id")) });
    }
    catch (_ex6)
    {
        RESULT.push({ id: (iRowId = iRowId + 1), method: "Request.GetProperty(\"matrix_id\")", value: "-- ОШИБКА: " + ExtractUserError(_ex6) + " --" });
    }
    DebugAlert("6. Готово");

    DebugAlert("7. Пробуем Request.Property(\"matrix_id\")");
    try
    {
        RESULT.push({ id: (iRowId = iRowId + 1), method: "Request.Property(\"matrix_id\")", value: String(Request.Property("matrix_id")) });
    }
    catch (_ex7)
    {
        RESULT.push({ id: (iRowId = iRowId + 1), method: "Request.Property(\"matrix_id\")", value: "-- ОШИБКА: " + ExtractUserError(_ex7) + " --" });
    }
    DebugAlert("7. Готово");

    DebugAlert("8. Пробуем Request.Params");
    try
    {
        RESULT.push({ id: (iRowId = iRowId + 1), method: "Request.Params (свойство)", value: String(Request.Params) });
    }
    catch (_ex8)
    {
        RESULT.push({ id: (iRowId = iRowId + 1), method: "Request.Params (свойство)", value: "-- ОШИБКА: " + ExtractUserError(_ex8) + " --" });
    }
    DebugAlert("8. Готово");

    DebugAlert("9. Пробуем Request.Url / Request.URL / Request.CurUrl (полный адрес, чтобы хотя бы вытащить строку вручную)");
    try
    {
        RESULT.push({ id: (iRowId = iRowId + 1), method: "Request.Url (свойство)", value: String(Request.Url) });
    }
    catch (_ex9)
    {
        RESULT.push({ id: (iRowId = iRowId + 1), method: "Request.Url (свойство)", value: "-- ОШИБКА: " + ExtractUserError(_ex9) + " --" });
    }
    DebugAlert("9. Готово");

    DebugAlert("10. Все варианты проверены, строк в RESULT: " + ArrayCount(RESULT));
}
catch (_exMain)
{
    DebugAlert("ОШИБКА ВЕРХНЕГО УРОВНЯ: " + ExtractUserError(_exMain));
    RESULT = [{ id: 0, method: "ОШИБКА ВЕРХНЕГО УРОВНЯ", value: ExtractUserError(_exMain) }];
}

COLUMNS = [
    { "data": "id", "editable": true, "hidden": true, "sortable": false },
    { "data": "method", "title": "Способ / что проверяем", "type": "string", "editable": false, "sortable": false, "multiline": true, "width": "40%" },
    { "data": "value", "title": "Результат", "type": "string", "editable": false, "sortable": false, "multiline": true, "width": "60%" }
];
