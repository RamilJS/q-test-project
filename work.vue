// =====================================================================
// HREDU-183. Диагностика v4: парсим GET-параметры вручную из Request.Url.
//
// ИТОГИ v3 (09.09.2026, реальный прогон): единственное, что реально сработало --
// Request.Url -- вернул полный адрес страницы целиком, включая query string:
//   https://als-devwt.vl.vtb:443/view_doc.html?mode=matrix_test&matrix_id=...&macroregion=...
// Всё остальное (for-in по Request, Request["x"], QueryString(), GetProperty(),
// Property(), Params) -- либо не существует, либо не тот тип объекта (for-in вообще
// требует МАССИВ в этом движке -- "Expression is not an array", поэтому по обычным
// объектам, не по массивам, for-in не работает; отсюда же и его успешная работа
// раньше -- там всегда были именно МАССИВЫ: form_fields, aFormFields и т.п.).
//
// ВЫВОД: раз именованного метода чтения одного GET-параметра нет -- читаем ВЕСЬ URL
// строкой (Request.Url) и вырезаем нужный параметр сами: ищем "?" или "&", затем имя
// параметра и "=", до следующего "&" или до конца строки. Без regex (не поддерживается),
// обычными строковыми методами (indexOf/substring/split) -- они уже точно работают в
// этом движке (encodeURIComponent сработал в filtry_modal, .length сработал в других
// диагностиках).
//
// Функция GetQueryParam() ниже -- если она подтвердится в этом прогоне, переносим её
// без изменений в настоящую выборку отчёта (ту, что стоит на Табличных данных) для
// ВСЕХ пяти фильтров: matrix_id, macroregion, mir_code, position_common_id, program_id.
//
// КАК ЗАПУСТИТЬ: так же, как v2/v3 -- выборка на Табличных данных страницы matrix_test,
// открыть страницу через "Применить" в модалке. Поля результата не меняются: id
// (integer), method (string), value (string).
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

/*
 * Вырезает значение GET-параметра из полного URL строки, без regex.
 * Ищет "имя=" сразу после "?" или "&", берёт всё до следующего "&" или до конца строки,
 * затем декодирует через decodeURIComponent (с запасным вариантом, если её нет).
 * @param {string} sUrl         -   Полный URL (например Request.Url).
 * @param {string} sParamName   -   Имя параметра, например "matrix_id".
 * @returns {string}             -   Значение параметра или "" если не найден.
 */
function GetQueryParam(sUrl, sParamName)
{
    var iQuestionPos, sQueryPart, aPairs, i, aPair, sRawValue;

    iQuestionPos = sUrl.indexOf("?");
    if (iQuestionPos < 0)
    {
        return "";
    }
    sQueryPart = sUrl.substring(iQuestionPos + 1);

    aPairs = sQueryPart.split("&");
    for (i = 0; i < ArrayCount(aPairs); i++)
    {
        aPair = aPairs[i].split("=");
        if (aPair[0] == sParamName)
        {
            sRawValue = (ArrayCount(aPair) > 1 ? aPair[1] : "");
            try
            {
                return decodeURIComponent(sRawValue);
            }
            catch (_exDecode)
            {
                return sRawValue;
            }
        }
    }
    return "";
}

DebugAlert("0. Файл начал выполняться");

RESULT = [];
iRowId = 0;

try
{
    DebugAlert("1. Читаем Request.Url");
    sFullUrl = String(Request.Url);
    DebugAlert("1. Готово, Request.Url = " + sFullUrl);
    iRowId = iRowId + 1;
    RESULT.push({ id: iRowId, method: "Request.Url (сырой)", value: sFullUrl });

    DebugAlert("2. Парсим matrix_id из Request.Url через GetQueryParam()");
    sMatrixID = GetQueryParam(sFullUrl, "matrix_id");
    iRowId = iRowId + 1;
    RESULT.push({ id: iRowId, method: "GetQueryParam(Request.Url, \"matrix_id\")", value: (sMatrixID != "" ? sMatrixID : "-- пусто/не найдено --") });
    DebugAlert("2. Готово: [" + sMatrixID + "]");

    DebugAlert("3. Парсим macroregion");
    sMacroregion = GetQueryParam(sFullUrl, "macroregion");
    iRowId = iRowId + 1;
    RESULT.push({ id: iRowId, method: "GetQueryParam(Request.Url, \"macroregion\")", value: (sMacroregion != "" ? sMacroregion : "-- пусто/не найдено --") });
    DebugAlert("3. Готово: [" + sMacroregion + "]");

    DebugAlert("4. Парсим mir_code");
    sMirCode = GetQueryParam(sFullUrl, "mir_code");
    iRowId = iRowId + 1;
    RESULT.push({ id: iRowId, method: "GetQueryParam(Request.Url, \"mir_code\")", value: (sMirCode != "" ? sMirCode : "-- пусто/не найдено --") });
    DebugAlert("4. Готово: [" + sMirCode + "]");

    DebugAlert("5. Парсим position_common_id");
    sPositionCommonID = GetQueryParam(sFullUrl, "position_common_id");
    iRowId = iRowId + 1;
    RESULT.push({ id: iRowId, method: "GetQueryParam(Request.Url, \"position_common_id\")", value: (sPositionCommonID != "" ? sPositionCommonID : "-- пусто/не найдено --") });
    DebugAlert("5. Готово: [" + sPositionCommonID + "]");

    DebugAlert("6. Парсим program_id");
    sProgramID = GetQueryParam(sFullUrl, "program_id");
    iRowId = iRowId + 1;
    RESULT.push({ id: iRowId, method: "GetQueryParam(Request.Url, \"program_id\")", value: (sProgramID != "" ? sProgramID : "-- пусто/не найдено --") });
    DebugAlert("6. Готово: [" + sProgramID + "]");

    DebugAlert("7. Всё распарсено, строк в RESULT: " + ArrayCount(RESULT));
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
