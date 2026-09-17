EnableLog('matrix_filters_7683109153292052868', true);
function alert(_string) {
    LogEvent('matrix_filters_7683109153292052868', _string);
    return _string;
}

// =====================================================================
// HREDU-183. Шаг 1: модальное окно с фильтрами.
//
//
// Поля фильтра:
//   matrix_id           -- foreign_elem, catalog: "cc_learning_matrice".
//   macroregion          -- ИЗМЕНЕНО (16.09.2026): было select со списком из
//                           GetMacroregionEntries() (SQL DISTINCT, убрана) -- теперь
//                           обычное текстовое поле, значение вводится вручную "как есть".
//   mir_code_id          -- foreign_elem, catalog: "cc_mir_code" -- резолвится в текст
//                           через ResolveMirCodeText().
//   position_common_id   -- foreign_elem, catalog: "position_common".
//   program_id           -- foreign_elem, catalog: "education_method".
//   city                 -- ДОБАВЛЕНО (16.09.2026): текстовое поле, значение вводится
//                           вручную "как есть" (либо приходит по умолчанию из URL, если
//                           попали сюда кликом по строке в "Процент обученных").
//

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

// УБРАНО (16.09.2026): GetMacroregionEntries() строила SQL DISTINCT список значений для
// select-поля "Макрорегион" -- больше не нужна, т.к. поле стало обычным текстовым вводом
// (см. изменение поля macroregion в oForm.form_fields ниже).

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
* ИСПРАВЛЕНО (10.09.2026, аварийное -- "GetFE Error ... objects/0000/00.xml"): поля
* matrix_id/mir_code_id/position_common_id/program_id -- это picker'ы типа
* "foreign_elem". Когда фильтр НЕ выбран, наш код (ветка "apply") кладёт в URL
* буквально "...=0" (дефолт OptInt(x, 0)). При повторном открытии модалки мы читаем
* это "0" из URL и подставляем в value: picker-поля -- а платформа воспринимает "0"
* НЕ как "ничего не выбрано", а как РЕАЛЬНЫЙ ID документа, и пытается открыть
* документ №0 (x-local://wt_data/objects/0000/00.xml) -- такого не существует, отсюда
* нативная ошибка платформы "GetFE Error returned: ... End of file (OpenDoc(),
* wt\web\lpapi.html, line 1289)" при повторном открытии модалки (после первого
* "Применить" без position_common_id/program_id). Раньше эти поля просто не получали
* value: (было "" по умолчанию), поэтому баг не проявлялся -- он появился ИМЕННО из-за
* фичи "запоминание фильтров". Фикс: "0" из URL для полей-picker'ов всегда превращаем
* обратно в "" перед тем, как класть в value:.
* @param {string} sValue
* @returns {string}
*/
function SanitizeIdFieldValue(sValue)
{
    if (sValue == "0" || sValue == undefined)
    {
        return "";
    }
    return sValue;
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
    oForm.title = "Выбор отчета";
    oForm.message = null;

    // ДОБАВЛЕНО (10.09.2026, запоминание фильтров): читаем текущий URL страницы --
    // если фильтры туда уже попали через предыдущий "Применить", используем их как
    // значения по умолчанию вместо "". Если Request недоступен (sModalPageUrl == "") --
    // GetQueryParam() всё равно вернёт "" на любое имя, ничего не ломается.
    // ВАЖНО (17.09.2026): ЭТО ЧТЕНИЕ НЕ УДАЛЕНО, хотя соответствующие поля теперь скрыты
    // (см. правку выше) -- sDefaultXxx по-прежнему нужны, чтобы молча "прокинуть" текущие
    // значения фильтров дальше в ветке "apply" (см. там).
    DebugAlert("3b. Читаем текущий URL страницы для восстановления фильтров (сначала параметр cur_page_url, потом Request.Url)");
    sModalPageUrl = GetCurPageUrlSafe();
    DebugAlert("3b2. Итоговый URL, который используем: [" + sModalPageUrl + "]");
    sDefaultMatrixID = SanitizeIdFieldValue(GetQueryParam(sModalPageUrl, "matrix_id"));
    sDefaultMacroregion = GetQueryParam(sModalPageUrl, "macroregion");
    sDefaultMirCodeID = SanitizeIdFieldValue(GetQueryParam(sModalPageUrl, "mir_code_id"));
    sDefaultPositionCommonID = SanitizeIdFieldValue(GetQueryParam(sModalPageUrl, "position_common_id"));
    sDefaultProgramID = SanitizeIdFieldValue(GetQueryParam(sModalPageUrl, "program_id"));

    // ИЗМЕНЕНО (16.09.2026): "city" раньше был только сквозным параметром (проезжал через
    // модалку, не будучи полем формы) -- теперь, по прямой просьбе пользователя, это
    // САМОСТОЯТЕЛЬНОЕ текстовое поле формы (см. oForm.form_fields ниже), которое можно
    // вручную ввести/поправить в модалке, а не только получить кликом по строке в
    // "Процент обученных" (см. BuildTepLink() в HREDU-182_procent_obuchennyh.js). Значение
    // по умолчанию всё равно читаем из текущего URL -- так что если сюда попали по клику
    // (city уже в адресе), поле будет заполнено само.
    // ЗАКОММЕНТИРОВАНО (17.09.2026): само ПОЛЕ формы для city теперь скрыто (см. правку
    // выше) -- чтение sDefaultCity ниже НЕ УДАЛЕНО, используется в ветке "apply" как
    // сквозное значение (city снова "просто проезжает" через модалку, как было до
    // 16.09.2026).
    sDefaultCity = GetQueryParam(sModalPageUrl, "city");

    // ДОБАВЛЕНО (10.09.2026, режим ТЭП-отчёта): "result_type" -- НЕ фильтр сотрудников,
    // а переключатель того, КАКОЙ из 4 отчётов ТЭП показывать (Общее/План/Факт/
    // Обязательно, см. HREDU-183_tep_reports.js). Раньше это было фиксированное
    // значение на вкладке "Параметры" отдельного виджета (нужно было 4 разных виджета) --
    // теперь пользователь выбирает режим прямо в этой модалке, вместе с остальными
    // фильтрами, и он же попадает в URL -- значит на странице достаточно ОДНОГО виджета
    // "Табличные данные" (HREDU-183_tep_reports.js сам переключает поведение по URL).
    // Дефолт -- "total" (Общее), а не "" -- select-полю нужно совпадающее значение
    // из entries ниже, иначе платформа может повести себя непредсказуемо (по аналогии
    // с историей про "0" для picker-полей, см. SanitizeIdFieldValue выше).
    sDefaultResultType = GetQueryParam(sModalPageUrl, "result_type");
    if (sDefaultResultType == "")
    {
        sDefaultResultType = "total";
    }

    DebugAlert("3c. Значения по умолчанию из URL: matrix_id=[" + sDefaultMatrixID + "] macroregion=[" + sDefaultMacroregion
        + "] mir_code_id=[" + sDefaultMirCodeID + "] position_common_id=[" + sDefaultPositionCommonID
        + "] program_id=[" + sDefaultProgramID + "] result_type=[" + sDefaultResultType + "] city=[" + sDefaultCity + "]");

    oForm.form_fields = [
        /* ЗАКОММЕНТИРОВАНО (17.09.2026, по прямой просьбе пользователя): "убери все
         * фильтры именно в модальном окне, кроме фильтра выбора режимов отчета, только не
         * удаляй всю эту логику, а просто закомментируй вывод этих фильтров". Эти 6 полей
         * (matrix_id/macroregion/city/mir_code_id/position_common_id/program_id) больше не
         * отображаются -- см. ветку "apply" ниже, там теперь используется sDefaultXxx
         * вместо "" вторым аргументом у getFormField(), чтобы значения этих фильтров
         * по-прежнему брались из текущего URL и просто "проезжали" дальше без изменений.

        {
            name: "matrix_id",
            label: "Матрица обучения",
            title: "Выберите матрицу обучения",
            type: "foreign_elem",
            value: sDefaultMatrixID,
            mandatory: true,
            multiple: false,
            catalog: "cc_learning_matrice",
            query_qual: ""
        },
        {
            // ИЗМЕНЕНО (16.09.2026, по прямой просьбе пользователя): было select со
            // списком значений из GetMacroregionEntries() (SQL DISTINCT, функция теперь не
            // используется и убрана из файла) -- стало обычное текстовое поле, значение
            // берётся "как есть" и сравнивается с макрорегионом сотрудника без выбора из
            // списка. Заодно убрано visibility: false -- раньше поле было СКРЫТО в форме
            // (не было видно и недоступно для ручного ввода), что для текстового поля,
            // которое как раз и предназначено для ручного ввода, было бы бессмысленно.
            // ПРОВЕРИТЬ НА РЕАЛЬНОЙ ФОРМЕ: type: "string" -- по аналогии с "select"/
            // "foreign_elem" в этом же файле это предполагаемое имя типа для однострочного
            // текстового поля в display_form, экспериментально на этой форме ещё не
            // проверялось -- если поле не отрисуется как текстовый инпут (например,
            // вообще не появится или появится как что-то другое), сообщи, поправим тип.
            name: "macroregion",
            label: "Макрорегион",
            type: "string",
            value: sDefaultMacroregion,
            mandatory: false
        },
        {
            // ДОБАВЛЕНО (16.09.2026, по прямой просьбе пользователя): текстовое поле
            // "Город" -- раньше в модалке такого фильтра вообще не было (город попадал в
            // URL только кликом по строке в "Процент обученных"). Теперь можно ввести
            // город вручную -- фильтрация в HREDU-183_tep_reports.js по нему уже была
            // реализована раньше (см. sCityFilter/FindCity()/GetCityRows() в том файле,
            // не менялось), не хватало только самого поля ввода здесь. Тот же тип "string",
            // что и у macroregion выше -- см. комментарий там же про непроверенность типа.
            name: "city",
            label: "Город",
            type: "string",
            value: sDefaultCity,
            mandatory: false
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
        },
        */
        {
            // ИСПРАВЛЕНО (14.09.2026): mandatory БЫЛ true -- но эту же модалку теперь
            // вешаем ещё и на страницы, которые result_type вообще НЕ используют
            // ("Восток полный список", "Процент обученных" -- см. HREDU-182_procent_obuchennyh.js,
            // он этот параметр не читает). Обязательный выбор непонятного поля на
            // странице, где оно ни на что не влияет, -- плохой UX. Сделано необязательным,
            // дефолт "Общее кол-во" (total) -- странице, которой всё равно, лишний
            // параметр в URL не мешает.
            // ЕДИНСТВЕННОЕ ОСТАВШЕЕСЯ ВИДИМЫМ ПОЛЕ (17.09.2026, см. правку в шапке файла).
            name: "result_type",
            label: "Выберите нужный отчет из списка",
            type: "select",
            value: sDefaultResultType,
            entries: [
                { name: "Общее кол-во", value: "total" },
                { name: "План", value: "plan" },
                { name: "Факт", value: "fact" },
                { name: "Обязательно к прохождению", value: "mandatory" }
            ],
            mandatory: false
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
            // ИЗМЕНЕНО (17.09.2026, поля выше скрыты -- см. правку в шапке файла): для
            // matrix_id/macroregion/mir_code_id/position_common_id/program_id/city вторым
            // аргументом getFormField() теперь передаётся НЕ "", а sDefaultXxx -- т.к. у
            // этих полей больше нет формы, откуда взять отправленное значение (aFormFields
            // для них будет пустым), getFormField() ВСЕГДА возвращал бы "" (и, например,
            // matrix_id намертво обнулялся бы при каждом "Применить"). sDefaultXxx --
            // то самое значение, что было прочитано из ТЕКУЩЕГО URL страницы ДО открытия
            // формы (см. выше) -- так эти 6 фильтров молча "проезжают" без изменений.
            // result_type -- ЕДИНСТВЕННОЕ поле, которое реально осталось в форме, поэтому
            // для него по-прежнему читаем то, что пользователь выбрал (fallback "total" --
            // как и было, на случай пустого значения).
            iMatrixID = OptInt(getFormField("matrix_id", sDefaultMatrixID), 0);
            sMacroregion = String(getFormField("macroregion", sDefaultMacroregion));
            iMirCodeID = OptInt(getFormField("mir_code_id", sDefaultMirCodeID), 0);
            iPositionCommonID = OptInt(getFormField("position_common_id", sDefaultPositionCommonID), 0);
            iProgramID = OptInt(getFormField("program_id", sDefaultProgramID), 0);
            sResultType = String(getFormField("result_type", "total"));
            sCity = String(getFormField("city", sDefaultCity));
            DebugAlert("7b. matrix_id=" + iMatrixID + " macroregion=[" + sMacroregion + "] mir_code_id=" + iMirCodeID + " position_common_id=" + iPositionCommonID + " program_id=" + iProgramID + " result_type=[" + sResultType + "] city=[" + sCity + "]");

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
            sCleanBaseUrl = RemoveQueryParam(sCleanBaseUrl, "result_type");
            // ДОБАВЛЕНО (16.09.2026): стираем старое значение city явно перед тем, как
            // дописать его снова ниже -- та же логика, что и для остальных 6 параметров.
            // Без этого при повторном "Применить" город мог бы задвоиться в query string
            // (старое значение осталось бы как "хвост", новое добавилось бы отдельно).
            sCleanBaseUrl = RemoveQueryParam(sCleanBaseUrl, "city");
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
                program_id: String(iProgramID),
                result_type: sResultType,
                // ДОБАВЛЕНО (16.09.2026): явно пробрасываем city дальше -- не полагаемся
                // на то, что он "сам выживет" как нетронутый хвост в sCleanBaseUrl.
                // sDefaultCity прочитан выше (до switch), из ТЕКУЩЕГО URL страницы --
                // значит если сюда попали, кликнув по строке города в "Процент обученных"
                // (city уже в URL), а потом открыли эту модалку и нажали "Применить" не
                // трогая город, он не потеряется.
                city: sCity
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
