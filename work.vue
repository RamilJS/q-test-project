// =====================================================================
// HREDU-183. Шаг 1: модальное окно с фильтрами.
//
// ИСПРАВЛЕНИЕ (09.09.2026, аварийное): предыдущая версия ЛОМАЛА весь файл целиком --
// даже стартовый display_form не открывался. Главный подозреваемый: UrlEncodeSafe()
// использовала regex-литерал (/ /g) в запасной ветке -- скриптовый движок WebTutor,
// судя по всему, НЕ поддерживает синтаксис регулярных выражений, и это ломает разбор
// (компиляцию) всего файла целиком, а не только ветку "apply", где эта функция
// реально вызывается. Regex убран (используется split/join без regex).
//
// ЗАЩИТА ОТ ПОВТОРЕНИЯ ТАКОГО ЖЕ СБОЯ: весь основной код теперь обёрнут в один
// try/catch -- если где-то ещё есть невидимая проблема, вместо "ничего не происходит"
// ты увидишь alert() с точным текстом ошибки (через ExtractUserError). Плюс по твоей
// просьбе добавлены чек-пойнты DebugAlert() на каждом шаге -- если DEBUG = true, они
// покажут alert() на каждой стадии, чтобы точно видеть, до какого места код доходит.
// Когда всё заработает и надоест -- поставь DEBUG = false, чек-пойнты замолчат, но
// try/catch-защита останется (она не зависит от DEBUG).
//
// Устроено по образцу рабочего "Удаленное действие кнопки" (визард создания
// заявки на подбор):
//   - PARAMETERS.GetOptProperty("form_fields") / ("form_fields_default") --
//     JSON-массивы полей формы, читаются через getParam()/getFormField().
//   - oForm.command = "display_form" -- команда показать модальное окно.
//   - Кнопки с submit_type определяют, что произойдёт при нажатии (обрабатывается
//     через switch(sSubmitType) ниже).
//
// Параметры удалённого действия (настраиваются в LPE у кнопки): form_fields --
// обычно пусто; form_fields_default -- обычно [].
//
// Поля фильтра:
//   matrix_id           -- foreign_elem, catalog: "cc_learning_matrice".
//   macroregion          -- select, список из GetMacroregionEntries() (SQL DISTINCT).
//   mir_code_id          -- foreign_elem, catalog: "cc_mir_code" -- резолвится в текст
//                           через ResolveMirCodeText().
//   position_common_id   -- foreign_elem, catalog: "position_common".
//   program_id           -- foreign_elem, catalog: "education_method".
//
// ШАГ "apply" (09.09.2026) -- redirect на страницу отчёта с фильтрами в query string.
//
// ПЕРЕНОСИМОСТЬ (10.09.2026): раньше redirect шёл на ЗАХАРДКОЖЕННЫЙ адрес (тестовую
// страницу matrix_test) -- значит одну и ту же модалку нельзя было повесить на другую
// страницу без правки кода. Теперь redirect идёт на ТУ ЖЕ страницу, откуда модалку
// открыли (через cur_page_url/Request.Url, см. GetCleanTargetUrl-логику в ветке
// "apply" -- старые значения фильтров сначала стираются через RemoveQueryParam(),
// потом дописываются новые). Значит ЭТУ ЖЕ модалку (без изменений в коде) можно
// вешать кнопкой на любую новую страницу -- в том числе на страницы 4 ТЭП-отчётов
// (HREDU-183_tep_reports.js) -- она сама вернёт на ту страницу, откуда её открыли,
// с новыми фильтрами.
//
// ПОДТВЕРЖДЕНО (09.09.2026, реальный тест): контракт oForm.command="close_form" +
// oForm.confirm_result={command:"redirect",url:...} работает, редирект происходит.
// Табличные данные читают GET-параметры не через подстановку в UI (в Env/Context её
// нет), а сама выборка читает их из Request.Url и парсит вручную -- см.
// HREDU-183_diagnostic_get_params.js.
//
// ИСПРАВЛЕНО (09.09.2026, кодирование): раньше кодировали через encodeURIComponent()
// (стандартный JS -- UTF-8: кириллица уезжала как %D0%A3%D0%A4%D0%9E). Но родная
// функция платформы для декодирования на стороне выборки -- UrlDecode() -- судя по
// примеру в документации (%E0%EF%F0%EE%EB -> "апрол"), ждёт ОДНОБАЙТНУЮ кодировку,
// не UTF-8: декодирование UTF-8-строки через неё дало бы кракозябру. Поэтому теперь
// кодируем ТОЖЕ родной функцией платформы -- UrlEncodeQuery(obj) -- она сама собирает
// "имя1=значение1&имя2=значение2&..." из объекта, в той же схеме, что понимает
// UrlDecode() на другом конце.
//
// ДОБАВЛЕНО (10.09.2026, запоминание фильтров): после ЛЮБОЙ перезагрузки страницы
// модалка при открытии (ветка "step_0") показывала пустые поля -- пользователю
// приходилось выбирать все фильтры заново, даже если он просто обновил страницу.
// Решение -- та же техника, что уже подтверждена в выборке отчёта (см.
// HREDU-181_vostok_polny_spisok_draft.js): САМА МОДАЛКА при открытии читает
// Request.Url текущей страницы (куда фильтры уже попали через предыдущий "Применить")
// и подставляет их как значения полей ПО УМОЛЧАНИЮ, вместо хардкода value: "". Раз
// сама страница и есть источник состояния (через query string), отдельное хранение
// (LOCAL-переменные, куки, что-то ещё) не нужно.
//
// НЮАНС с мир-кодом: в URL для отчёта передаётся ТЕКСТОВЫЙ код (mir_code=LAMA,
// нужен отчёту для фильтрации сотрудников), а полю-picker'у mir_code_id для
// восстановления нужен ID документа cc_mir_code, не текст -- обратно текст в ID без
// дополнительного похода в базу не превратить надёжно (могут быть тёзки по названию).
// Поэтому в query string теперь дублируем ОБА значения: mir_code (текст, для отчёта,
// как и раньше) и mir_code_id (ID, только для восстановления поля в модалке) -- отчёт
// (HREDU-181_vostok_polny_spisok_draft.js) продолжает читать mir_code как раньше,
// его трогать не пришлось.
// =====================================================================

DEBUG = true;

/*
 * Чек-пойнт для отладки -- alert() с номером шага, только если DEBUG = true.
 * Обёрнут в try/catch, чтобы сама отладочная печать не могла обрушить скрипт,
 * если в каком-то контексте alert()/LogAlert недоступны.
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

function getParam(sName, sDefault) {
    var sValue = PARAMETERS.GetOptProperty(sName);
    if (sDefault != undefined && (sValue == undefined || sValue == "")) {
        sValue = sDefault;
    }
    return sValue;
}

function getFormField(sName, sDefault) {
    var sValue = ArrayOptFind(aFormFields, ("This.name == " + XQueryLiteral(sName)));
    sValue = (sValue != undefined ? sValue.value : sValue);
    if (sDefault != undefined && (sValue == undefined || sValue == "")) {
        sValue = sDefault;
    }
    return sValue;
}

function getFormFieldDefault(sName, sDefault) {
    var sValue = ArrayOptFind(aFormFieldsDef, ("This.name == " + XQueryLiteral(sName)));
    sValue = (sValue != undefined ? sValue.value : sValue);
    if (sDefault != undefined && (sValue == undefined || sValue == "")) {
        sValue = sDefault;
    }
    return sValue;
}

/*
 * Строит список entries для select-поля "Макрорегион" -- DISTINCT по реальным
 * значениям custom_elem f_2ewj у активных сотрудников (не хардкод).
 * @returns {Object[]}   -   Массив {name, value}, первый пункт -- "Все".
 */
function GetMacroregionEntries()
{
    var sqlText, rows, entries, i, sVal;
    entries = [{ name: "Все", value: "" }];
    try
    {
        sqlText = "";
        sqlText = sqlText + "select distinct c.data.value('(*/custom_elems/custom_elem[name=''f_2ewj'']/value)[1]', 'varchar(max)') as macroregion\r\n";
        sqlText = sqlText + "from collaborators cs\r\n";
        sqlText = sqlText + "inner join collaborator c on c.id = cs.id\r\n";
        sqlText = sqlText + "where cs.is_dismiss != 1";
        rows = ArraySelectAll(XQuery("sql:" + sqlText));
        for (i = 0; i < ArrayCount(rows); i++)
        {
            sVal = String(rows[i].macroregion);
            if (sVal != "")
            {
                entries.push({ name: sVal, value: sVal });
            }
        }
    }
    catch (_ex)
    {
        entries = [{ name: "-- ошибка загрузки списка: " + ExtractUserError(_ex) + " --", value: "" }];
    }
    return entries;
}

/*
 * Резолвит ID объекта cc_mir_codes (то, что реально возвращает foreign_elem) в его
 * текстовый код (то, что реально лежит в custom_elem f_mir_codes у сотрудников).
 * @param {number} iMirCodeID   -   ID документа cc_mir_codes.
 * @returns {string}            -   Код (например "LASK") или "" если не найден/не выбран.
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

/*
 * Достаёт полный URL текущей страницы (с фильтрами, которые туда попали через
 * предыдущий "Применить"). ИСПРАВЛЕНО (10.09.2026, реальный тест показал пустые
 * значения): Request.Url надёжно сработал в ВЫБОРКЕ (см. HREDU-181_vostok_polny_spisok_draft.js,
 * HREDU-183_diagnostic_get_params.js), но в контексте УДАЛЁННОГО ДЕЙСТВИЯ (эта модалка
 * вызывается по кнопке через ajax) он, судя по всему, отражает адрес самого ajax-запроса
 * к серверу, а не видимый адрес страницы в браузере -- это другой контекст выполнения,
 * та же история, что уже была с PARAMETERS/ScopeWVars (доступны только в одном из двух
 * контекстов, не в обоих).
 *
 * Способ 1 (предпочтительный): параметр удалённого действия "cur_page_url", привязанный
 * в LPE к подстановке {{curEnv.curEnvUrl}} ("Полный URL страницы" -- см. список Env, что
 * ты присылал раньше). НАСТРОЙ этот параметр в LPE у кнопки: добавь параметр с именем
 * cur_page_url, тип "Текст с подстановками", значение {{curEnv.curEnvUrl}}.
 * Способ 2 (запасной): Request.Url -- оставлен на случай, если способ 1 почему-то не
 * настроен или тоже не сработает.
 * @returns {string}
 */
function GetCurPageUrlSafe()
{
    var sUrl;

    sUrl = getParam("cur_page_url", "");
    if (sUrl != "")
    {
        return sUrl;
    }

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
 * Вырезает значение GET-параметра из полного URL строки -- та же функция, что уже
 * подтверждена диагностикой и используется в выборке отчёта (HREDU-183_diagnostic_get_params.js,
 * HREDU-181_vostok_polny_spisok_draft.js). Без regex и без методов строк (.indexOf/.substring
 * здесь не существуют) -- через штатный строковый API платформы.
 * @param {string} sUrl         -   Полный URL (например Request.Url).
 * @param {string} sParamName   -   Имя параметра, например "matrix_id".
 * @returns {string}             -   Значение параметра или "" если не найден.
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
 * ДОБАВЛЕНО (10.09.2026, переносимость на разные страницы): убирает из URL старое
 * значение указанного GET-параметра (если оно там есть), не трогая остальную часть
 * адреса. Нужно, чтобы модалка могла делать redirect на ТУ ЖЕ страницу, на которой её
 * открыли (а не на захардкоженный адрес) -- сначала стираем старые фильтры из текущего
 * URL, потом дописываем новые (см. "ПЕРЕНОСИМОСТЬ" в шапке файла). Без regex -- через
 * тот же штатный строковый API, что и GetQueryParam().
 * @param {string} sUrl
 * @param {string} sParamName
 * @returns {string}   -   URL без этого параметра (если параметра не было -- вернёт как есть).
 */
function RemoveQueryParam(sUrl, sParamName)
{
    var sAmpMarker, sQMarkMarker, iMarkerPos, iValueStart, iAmpPos, iUrlLen, sBefore, sAfter;

    iUrlLen = StrLen(sUrl);

    // Случай 1: параметр не первый -- ищем "&имя=" и убираем его целиком вместе со
    // значением, до следующего "&" или до конца строки.
    sAmpMarker = "&" + sParamName + "=";
    iMarkerPos = StrOptSubStrPos(sUrl, sAmpMarker, false);
    if (iMarkerPos != undefined)
    {
        iValueStart = iMarkerPos + StrLen(sAmpMarker);
        iAmpPos = StrOptSubStrPos(sUrl, "&", false, iValueStart);
        sBefore = StrRangePos(sUrl, 0, iMarkerPos);
        sAfter = (iAmpPos != undefined ? StrRangePos(sUrl, iAmpPos, iUrlLen) : "");
        return sBefore + sAfter;
    }

    // Случай 2: параметр первый сразу после "?" -- "?" оставляем, а если следом шёл
    // "&" следующего параметра -- он становится новой границей после "?".
    sQMarkMarker = "?" + sParamName + "=";
    iMarkerPos = StrOptSubStrPos(sUrl, sQMarkMarker, false);
    if (iMarkerPos != undefined)
    {
        iValueStart = iMarkerPos + StrLen(sQMarkMarker);
        iAmpPos = StrOptSubStrPos(sUrl, "&", false, iValueStart);
        sBefore = StrRangePos(sUrl, 0, iMarkerPos + 1); // включая сам "?"
        sAfter = (iAmpPos != undefined ? StrRangePos(sUrl, iAmpPos + 1, iUrlLen) : "");
        return sBefore + sAfter;
    }

    // Параметра не было -- ничего менять не нужно.
    return sUrl;
}

DebugAlert("0. Файл начал выполняться");

try
{
    DebugAlert("1. Читаем form_fields/form_fields_default");
    aFormFields = ParseJson(getParam("form_fields", "[]"));
    aFormFieldsDef = ParseJson(getParam("form_fields_default", "[]"));
    sSubmitType = getFormField("__submit_type__", getFormFieldDefault("__submit_type__", "step_0"));
    DebugAlert("2. sSubmitType = [" + sSubmitType + "]");

    oForm = new Object();
    oForm.command = "display_form";
    oForm.height = 320;
    oForm.title = "Фильтры отчёта (Восток)";
    oForm.message = null;

    DebugAlert("3. Строим GetMacroregionEntries()");
    aMacroregionEntries = GetMacroregionEntries();
    DebugAlert("4. GetMacroregionEntries() построен, пунктов: " + ArrayCount(aMacroregionEntries));

    // ДОБАВЛЕНО (10.09.2026, запоминание фильтров): читаем текущий URL страницы --
    // если фильтры туда уже попали через предыдущий "Применить", используем их как
    // значения по умолчанию вместо "". Если Request недоступен (sModalPageUrl == "") --
    // GetQueryParam() всё равно вернёт "" на любое имя, ничего не ломается.
    DebugAlert("3b. Читаем текущий URL страницы для восстановления фильтров (сначала параметр cur_page_url, потом Request.Url)");
    sModalPageUrl = GetCurPageUrlSafe();
    DebugAlert("3b2. Итоговый URL, который используем: [" + sModalPageUrl + "]");
    sDefaultMatrixID = GetQueryParam(sModalPageUrl, "matrix_id");
    sDefaultMacroregion = GetQueryParam(sModalPageUrl, "macroregion");
    sDefaultMirCodeID = GetQueryParam(sModalPageUrl, "mir_code_id");
    sDefaultPositionCommonID = GetQueryParam(sModalPageUrl, "position_common_id");
    sDefaultProgramID = GetQueryParam(sModalPageUrl, "program_id");
    DebugAlert("3c. Значения по умолчанию из URL: matrix_id=[" + sDefaultMatrixID + "] macroregion=[" + sDefaultMacroregion
        + "] mir_code_id=[" + sDefaultMirCodeID + "] position_common_id=[" + sDefaultPositionCommonID
        + "] program_id=[" + sDefaultProgramID + "]");

    oForm.form_fields = [
        {
            name: "matrix_id",
            label: "Матрица обучения *",
            title: "Выберите матрицу обучения",
            type: "foreign_elem",
            value: sDefaultMatrixID,
            mandatory: true,
            multiple: false,
            catalog: "cc_learning_matrice",
            query_qual: ""
        },
        {
            name: "macroregion",
            label: "Макрорегион",
            type: "select",
            value: sDefaultMacroregion,
            entries: aMacroregionEntries,
            mandatory: false,
            visibility: false
        },
        {
            name: "mir_code_id",
            label: "Мир-код",
            title: "Выберите мир-код",
            type: "foreign_elem",
            value: sDefaultMirCodeID,
            mandatory: false,
            multiple: false,
            catalog: "cc_mir_code",
            query_qual: ""
        },
        {
            name: "position_common_id",
            label: "Типовая должность",
            title: "Выберите типовую должность",
            type: "foreign_elem",
            value: sDefaultPositionCommonID,
            mandatory: false,
            multiple: false,
            catalog: "position_common",
            query_qual: ""
        },
        {
            name: "program_id",
            label: "Учебная программа",
            title: "Выберите учебную программу",
            type: "foreign_elem",
            value: sDefaultProgramID,
            mandatory: false,
            multiple: false,
            catalog: "education_method",
            query_qual: ""
        }
    ];
    DebugAlert("5. oForm.form_fields собран, полей: " + ArrayCount(oForm.form_fields));

    for (oField in oForm.form_fields)
    {
        oField.value = getFormField(oField.name, oField.value);
    }
    DebugAlert("6. Значения полей проставлены из aFormFields");

    oForm.buttons = [];
    oForm.no_buttons = false;

    switch (sSubmitType)
    {
        case "apply":
        {
            DebugAlert("7a. Ветка apply -- читаем значения полей");
            iMatrixID = OptInt(getFormField("matrix_id", ""), 0);
            sMacroregion = String(getFormField("macroregion", ""));
            iMirCodeID = OptInt(getFormField("mir_code_id", ""), 0);
            iPositionCommonID = OptInt(getFormField("position_common_id", ""), 0);
            iProgramID = OptInt(getFormField("program_id", ""), 0);
            DebugAlert("7b. matrix_id=" + iMatrixID + " macroregion=[" + sMacroregion + "] mir_code_id=" + iMirCodeID + " position_common_id=" + iPositionCommonID + " program_id=" + iProgramID);

            sMirCodeText = ResolveMirCodeText(iMirCodeID);
            DebugAlert("7c. mir_code резолвлен в текст: [" + sMirCodeText + "]");

            // ИЗМЕНЕНО (10.09.2026, ПЕРЕНОСИМОСТЬ на разные страницы): раньше redirect шёл
            // на захардкоженный тестовый адрес -- значит эту же модалку нельзя было
            // повесить на другую страницу (например, на страницы ТЭП-отчётов) без правки
            // кода. Теперь берём ТЕКУЩУЮ страницу (sModalPageUrl, уже прочитан выше для
            // восстановления значений полей), стираем из неё старые значения фильтров
            // (если модалку открывали не в первый раз) и дописываем новые -- так одна и
            // та же модалка работает на любой странице, куда её повесят, и всегда
            // возвращает на ту же страницу, откуда её открыли.
            sCleanBaseUrl = sModalPageUrl;
            sCleanBaseUrl = RemoveQueryParam(sCleanBaseUrl, "matrix_id");
            sCleanBaseUrl = RemoveQueryParam(sCleanBaseUrl, "macroregion");
            sCleanBaseUrl = RemoveQueryParam(sCleanBaseUrl, "mir_code_id");
            sCleanBaseUrl = RemoveQueryParam(sCleanBaseUrl, "mir_code");
            sCleanBaseUrl = RemoveQueryParam(sCleanBaseUrl, "position_common_id");
            sCleanBaseUrl = RemoveQueryParam(sCleanBaseUrl, "program_id");
            DebugAlert("7c2. Текущая страница без старых фильтров: [" + sCleanBaseUrl + "]");

            // Родная функция платформы -- сама собирает "имя1=значение1&имя2=значение2&..."
            // и кодирует значения в той же схеме, что понимает UrlDecode() на стороне выборки.
            // mir_code_id ДОБАВЛЕН (10.09.2026) -- отчёту не нужен (он фильтрует по тексту
            // mir_code, как и раньше), нужен ТОЛЬКО модалке, чтобы при следующем открытии
            // восстановить значение picker'а по ID, а не по тексту (см. "НЮАНС с мир-кодом"
            // в шапке файла).
            oQueryParams = {
                matrix_id: String(iMatrixID),
                macroregion: sMacroregion,
                mir_code: sMirCodeText,
                mir_code_id: String(iMirCodeID),
                position_common_id: String(iPositionCommonID),
                program_id: String(iProgramID)
            };
            sQueryString = UrlEncodeQuery(oQueryParams);

            // Разделитель зависит от того, остался ли в sCleanBaseUrl хоть один "?"
            // (страница почти наверняка сохранит свой собственный параметр вроде
            // mode=... -- мы стираем только 6 фильтров, не всю строку запроса).
            sSeparator = (StrOptSubStrPos(sCleanBaseUrl, "?", false) != undefined ? "&" : "?");
            sFullUrl = sCleanBaseUrl + sSeparator + sQueryString;
            DebugAlert("7d. Итоговый URL redirect: " + sFullUrl);

            oForm = {
                command: "close_form",
                confirm_result: {
                    command: "redirect",
                    url: sFullUrl
                }
            };

            // ЗАПАСНОЙ ВАРИАНТ (проверочный alert вместо редиректа) -- если после починки
            // регэкспа модалка открывается, но именно redirect не срабатывает -- раскомментируй
            // этот блок вместо oForm выше, чтобы отдельно проверить, что сами значения полей
            // (picker'ы/select) верны, независимо от механизма передачи в отчёт:
            //
            // oForm = {
            //     command: "alert",
            //     msg: ("Выбранные фильтры (проверочный вывод):<br/><pre>" + sFullUrl + "</pre>"),
            //     title: "Фильтры применены (пока без связи с отчётом)"
            // };

            DebugAlert("7e. Ветка apply завершена, RESULT будет = close_form/redirect");
            break;
        }
        case "step_0":
        default:
        {
            DebugAlert("8. Ветка step_0/default -- добавляем кнопки Применить/Отмена");
            oForm.buttons.push(
                { name: "submit", submit_type: "apply", label: "Применить", type: "submit" },
                { name: "cancel", label: "Отмена", type: "cancel" }
            );
            break;
        }
    }

    RESULT = oForm;
    DebugAlert("9. RESULT успешно собран, sSubmitType был [" + sSubmitType + "]");
}
catch (_exMain)
{
    // ГЛАВНАЯ ЗАЩИТА (09.09.2026): если где-то в коде выше вылетит ЛЮБАЯ ошибка --
    // вместо "ничего не происходит"/пустого падения покажем alert с точным текстом.
    RESULT = {
        command: "alert",
        msg: ("Ошибка в модалке фильтров (HREDU-183_filtry_modal_shag1.js):<br/><pre>" + ExtractUserError(_exMain) + "</pre>"),
        title: "ОШИБКА"
    };
}
