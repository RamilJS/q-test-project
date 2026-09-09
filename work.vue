// =====================================================================
// HREDU-183. Диагностика v5: парсим GET-параметры из Request.Url через настоящий
// строковый API этой платформы (документация datex.ru).
//
// ИТОГИ v4: .indexOf()/.substring() -- это НЕ методы строк в этом движке ("Unknown
// method: substring()"). Строковые операции здесь -- ГЛОБАЛЬНЫЕ ФУНКЦИИ (тот же
// стиль, что Trim(), String(), Int() -- ничего нового, просто мы раньше не знали
// имён). Точные сигнатуры (документация datex.ru, 09.09.2026):
//
//   StrOptSubStrPos(str, subStr [, ignoreCase [, startPos]])
//     -- позиция подстроки в строке; undefined, если не найдена (как у OptInt --
//        "Opt" в названии означает "может не найти, тогда undefined, а не ошибка").
//   StrRangePos(str, pos1, pos2)
//     -- часть строки между позициями pos1 и pos2, возвращает String.
//   StrLen(str)
//     -- длина строки.
//
// GetQueryParam() ниже: ищет "&имя=" (для параметров не первых по счёту) или
// "?имя=" (для первого параметра сразу после "?"), берёт позицию конца найденной
// подстроки через StrLen(), находит следующий "&" (или конец строки, если параметр
// последний), вырезает всё между этими двумя позициями через StrRangePos(), затем
// decodeURIComponent() (уже подтверждено рабочим -- сработал при кодировании
// кириллицы в filtry_modal).
//
// КАК ЗАПУСТИТЬ: как и раньше -- выборка на Табличных данных страницы matrix_test,
// открыть страницу через "Применить" в модалке. Поля результата: id (integer),
// method (string), value (string).
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
 * Вырезает значение GET-параметра из полного URL строки через штатный строковый API
 * платформы (StrOptSubStrPos/StrRangePos/StrLen) -- без regex и без методов строк
 * (.indexOf/.substring здесь не существуют, см. шапку файла).
 * @param {string} sUrl         -   Полный URL (например Request.Url).
 * @param {string} sParamName   -   Имя параметра, например "matrix_id".
 * @returns {string}             -   Значение параметра или "" если не найден.
 */
function GetQueryParam(sUrl, sParamName)
{
    var sAmpMarker, sQMarkMarker, iParamPos, iValueStart, iAmpPos, iValueEnd, sRawValue, iUrlLen;

    iUrlLen = StrLen(sUrl);

    // Вариант 1: параметр не первый в query string -- ищем "&имя="
    sAmpMarker = "&" + sParamName + "=";
    iParamPos = StrOptSubStrPos(sUrl, sAmpMarker, false);
    if (iParamPos != undefined)
    {
        iValueStart = iParamPos + StrLen(sAmpMarker);
    }
    else
    {
        // Вариант 2: параметр первый сразу после "?"
        sQMarkMarker = "?" + sParamName + "=";
        iParamPos = StrOptSubStrPos(sUrl, sQMarkMarker, false);
        if (iParamPos == undefined)
        {
            return "";
        }
        iValueStart = iParamPos + StrLen(sQMarkMarker);
    }

    // Конец значения -- следующий "&" после начала значения, либо конец строки.
    iAmpPos = StrOptSubStrPos(sUrl, "&", false, iValueStart);
    iValueEnd = (iAmpPos != undefined ? iAmpPos : iUrlLen);

    sRawValue = StrRangePos(sUrl, iValueStart, iValueEnd);

    // ИСПРАВЛЕНО (09.09.2026): decodeURIComponent() не декодировал кириллицу --
    // судя по всему, не существует в этом движке (сработал try/catch запасной вариант
    // молча). Заменено на родную функцию платформы UrlDecode() -- парная к UrlEncodeQuery(),
    // которой теперь кодирует модалка фильтров (HREDU-183_filtry_modal_shag1.js).
    try
    {
        return UrlDecode(sRawValue);
    }
    catch (_exDecode)
    {
        return sRawValue;
    }
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

    DebugAlert("2. Парсим matrix_id");
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

    DebugAlert("7. Дополнительно: parсим mode (должно быть 'matrix_test', первый параметр после '?') -- проверка ветки '?имя='");
    sMode = GetQueryParam(sFullUrl, "mode");
    iRowId = iRowId + 1;
    RESULT.push({ id: iRowId, method: "GetQueryParam(Request.Url, \"mode\") -- проверка первого параметра", value: (sMode != "" ? sMode : "-- пусто/не найдено --") });
    DebugAlert("7. Готово: [" + sMode + "]");

    DebugAlert("8. Всё распарсено, строк в RESULT: " + ArrayCount(RESULT));
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
