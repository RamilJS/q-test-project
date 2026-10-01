EnableLog('matrix_filters_7685325909044809318', true);
function alert(_string) {
    LogEvent('matrix_filters_7685325909044809318', _string);
    return _string;
}

// =====================================================================
// HREDU-182. Модальное окно с фильтрами -- ТОЛЬКО для страницы "Процент обученных".
//
// РАЗДЕЛЕНО (14.09.2026, по прямому указанию пользователя): раньше одна и та же
// модалка (HREDU-183_filtry_modal_shag1.js) висела и на страницах ТЭП, и на странице
// "Процент обученных" -- но у ТЭП есть поле "Режим отчёта" (result_type), которое
// "Процент обученных" вообще не использует (HREDU-182_procent_obuchennyh.js его не
// читает). Чтобы не тащить лишнее поле на страницу, где оно не нужно, и не путать
// пользователя -- РАЗДЕЛИЛИ на два независимых удалённых действия:
//   - HREDU-183_filtry_modal_shag1.js -- ОСТАЁТСЯ БЕЗ ИЗМЕНЕНИЙ, только для 4 страниц
//     ТЭП-отчётов (там же живёт поле "Режим отчёта").
//   - HREDU_182_filtry_percent.js (этот файл) -- НОВЫЙ, для страницы "Процент
//     обученных" и, если понадобится, для "Восток полный список". Это ПОЛНАЯ КОПИЯ
//     HREDU-183_filtry_modal_shag1.js, из которой убрано ВСЁ, что относится к
//     "Режим отчёта"/result_type (поле формы, чтение из URL, запись в URL,
//     RemoveQueryParam) -- остальная логика (5 фильтров, восстановление значений,
//     переносимость на любую страницу через cur_page_url/{{curEnv.curEnvUrl}},
//     санитизация "0" для picker-полей, кодирование через UrlEncodeQuery/UrlDecode)
//     -- идентична, без изменений. См. HREDU-183_filtry_modal_shag1.js -- там вся
//     история находок и фиксов (regex ломает весь файл, function-as-value ломает весь
//     файл, .indexOf/.substring не существуют, Request.Url не работает в удалённом
//     действии, "0" в picker-поле роняет платформу) описана подробно, здесь не
//     повторяем.
//
// Поля фильтра:
//   matrix_id           -- foreign_elem, catalog: "compound_program" -- ИЗМЕНЕНО
//                           (28.09.2026, HREDU-237, см. "ИСТОЧНИК ДАННЫХ HREDU-237" ниже) --
//                           было "cc_learning_matrice".
//   macroregion          -- ИЗМЕНЕНО (16.09.2026): было select, теперь обычное текстовое
//                           поле (значение вводится вручную "как есть").
//   mir_code_id          -- foreign_elem, catalog: "cc_mir_code" -- резолвится в текст
//                           через ResolveMirCodeText(). НЕ ЗАТРОНУТО HREDU-237 -- отдельный
//                           каталог, к модульным программам не относится.
//   position_common_id   -- foreign_elem, catalog: "position_common". НЕ ЗАТРОНУТО HREDU-237.
//   program_id           -- foreign_elem, catalog: "education_method". НЕ ЗАТРОНУТО HREDU-237
//                           -- уже указывал прямо на education_method, а не через матрицу.
//   (У этой страницы фильтра по городу нет -- "Процент обученных" сама строит по одной
//   строке НА КАЖДЫЙ город, а не фильтрует их; город как фильтр нужен только на страницах
//   ТЭП, см. HREDU-183_filtry_modal_shag1.js.)
//
// =====================================================================
// ИСТОЧНИК ДАННЫХ HREDU-237 (28.09.2026, по прямому указанию пользователя -- см. подробную
// историю решения в HREDU-182_procent_obuchennyh.js, здесь только то, что касается ЭТОГО
// файла):
//
//   matrix_id теперь указывает на compound_program (коробочная сущность "Модульные
//   программы"), а не на cc_learning_matrice. Каталог пикера сменён на
//   catalog: "compound_program" -- ИМЯ КАТАЛОГА НЕ ПОДТВЕРЖДЕНО РЕАЛЬНЫМ ТЕСТОМ (по
//   аналогии с education_method/course/assessment/notification -- везде каталог назван
//   ЕДИНСТВЕННЫМ числом типа документа) -- ПРОВЕРИТЬ ПЕРВЫМ ДЕЛОМ: открыть модалку
//   фильтров, посмотреть, показывает ли пикер "Матрица обучения" реальные модульные
//   программы или падает с ошибкой "неизвестный каталог".
//
//   Мир-код теперь СВОБОДНЫЙ ТЕКСТ с wildcard-паттернами на САМОЙ compound_program
//   (custom_elem f_mir_code, например "LASR;LASM"), а не FK на cc_mir_code через элементы
//   матрицы -- GetMatrixIdsByMirCodes()/GetAllActiveMatrixElementRows()/FilterActiveMatrixIds()
//   переписаны под новую модель, см. ниже. WildcardMatch()/MatchAnySemicolonPattern()
//   скопированы из HREDU-182_procent_obuchennyh.js (включая ПОДТВЕРЖДЁННЫЙ РЕАЛЬНЫМ ТЕСТОМ
//   фикс: 3-й параметр StrOptSubStrPos работает НАОБОРОТ от названия -- true игнорирует
//   регистр).
//
//   ВАЖНО -- ЕЩЁ НЕ ПРОВЕРЕНО РЕАЛЬНЫМ ТЕСТОМ: сузит ли query_qual пикер каталога
//   "compound_program" так же, как НЕ сузил "cc_learning_matrice" (см. "ИТОГ ДИАГНОСТИКИ
//   query_qual" ниже -- это был отдельный, другой каталог, платформенный баг которого мог
//   быть специфичен именно ему, а не query_qual в принципе). Если пикер снова покажет ВСЕ
//   матрицы, несмотря на query_qual -- это ТА ЖЕ, уже известная категория проблемы (сторона
//   платформы, не эта функция) -- сообщи мне, попробуем разобраться отдельно, это НЕ
//   блокирует базовую работу фильтра (просто руководитель увидит больше матриц, чем строго
//   нужно, как и было с cc_learning_matrice).
// =====================================================================
//
// =====================================================================
// ФИЛЬТРАЦИЯ ПИКЕРА "МАТРИЦА ОБУЧЕНИЯ" ПО МИР-КОДУ (ДОБАВЛЕНО 24.09.2026, идея тим-лида
// пользователя, озвученная 22.09.2026 и подтверждённая пользователем как готовая к
// реализации 24.09.2026):
//
// ЗАДАЧА: руководитель (не входящий в УОРиАП) должен видеть в выпадающем списке поля
// "Матрица обучения" ТОЛЬКО те матрицы, у которых есть хотя бы один активный элемент
// (cc_learning_matrice_element) с мир-кодом, совпадающим с мир-кодом САМОГО руководителя
// ИЛИ с мир-кодом хотя бы одного из его подчинённых (по всей цепочке вниз, та же
// иерархия, что уже реализована и подтверждена реальными тестами в
// HREDU-182_procent_obuchennyh.js -- см. её шапку). УОРиАП по-прежнему видит ВСЕ
// матрицы без ограничений (query_qual = "").
//
// МЕХАНИЗМ ФИЛЬТРАЦИИ ПИКЕРА -- НАЙДЕН (24.09.2026, пользователь прислал 3 реальных
// РАБОЧИХ примера удалённых действий с query_qual): query_qual foreign_elem-поля -- это
// XQuery-ВЫРАЖЕНИЕ (булево условие) на переменной $elem (текущий документ каталога),
// той же формы, что и обычный where-предикат XQuery, который уже много раз проверенно
// работает в этом тикете. В присланных примерах дважды встречается ИМЕННО та функция,
// что мы уже используем в GetMatrixElementRows() (MatchSome) -- значит риск здесь
// минимальный, синтаксис уже фактически подтверждён (хоть и в других файлах):
//   query_qual: "MatchSome($elem/id,(" + ArrayMerge(idsArray, "This", ",") + "))"
// -- то есть "показывать в пикере только документы каталога, чей id входит в список".
// ПРИМЕНЯЕМ ЭТУ ЖЕ ФОРМУ для matrix_id ниже.
//
// НАЙДЕНО ПОПУТНО (24.09.2026, из тех же присланных примеров): в удалённых действиях
// (в отличие от выборки, HREDU-182_procent_obuchennyh.js, где curUserId нужно было
// вручную привязывать через LPE-параметр) текущий пользователь портала доступен как
// ГОЛАЯ ГЛОБАЛЬНАЯ ПЕРЕМЕННАЯ "curUserID" (с большой ID на конце) -- используется
// НАПРЯМУЮ, без PARAMETERS.GetOptProperty(), в трёх независимых присланных файлах.
// ЭТО НОВОЕ ДЛЯ ДАННОГО ФАЙЛА ДОПУЩЕНИЕ, НЕ ПРОВЕРЕННОЕ РЕАЛЬНЫМ ТЕСТОМ ИМЕННО В ЭТОМ
// УДАЛЁННОМ ДЕЙСТВИИ -- GetCurUserIdSafe() ниже обёрнута в try/catch по той же схеме,
// что и в выборке (на случай, если в ЭТОМ конкретном удалённом действии переменная
// почему-то не определена) -- если curUserID недоступна, fail-safe: матрица НЕ
// показывается никому, кроме УОРиАП (см. sMatrixQueryQual ниже -- значение по
// умолчанию -- sentinel "0", который не совпадёт ни с одним реальным id).
//
// СОЗНАТЕЛЬНОЕ РЕШЕНИЕ: НЕ используем существующие библиотечные методы
// tools.call_code_library_method('libVTBL', 'userIsManager'/'getUserAllSubordinates', ...)
// (тоже найдены в присланных примерах), хотя они, возможно, решают ту же задачу проще.
// Причина: мы уже построили и ПОДТВЕРДИЛИ РЕАЛЬНЫМИ ТЕСТАМИ (несколько раундов, разные
// руководители -- 66/114/0 подчинённых) собственную BFS-логику по цепочке is_native=true
// в HREDU-182_procent_obuchennyh.js -- та же самая выборка потом СТРОИТ САМ ОТЧЁТ по
// этому же списку подчинённых. Если здесь, в фильтре пикера, использовать ДРУГОЙ источник
// (библиотечную функцию, чья логика подсчёта "подчинённых" нам неизвестна и не
// протестирована), есть риск рассинхрона: пикер покажет матрицу, а сам отчёт по ней -- 0
// строк (потому что "его" библиотека и наша BFS по-разному считают, кто чей подчинённый).
// Поэтому здесь -- БУКВАЛЬНО ТЕ ЖЕ ФУНКЦИИ (GetManagerIdRows/SortRowsByManagerId/
// FindManagerRangeStart/CollectDirectSubordinateIds/GetManagerHierarchySubordinateIds),
// скопированные без изменений из HREDU-182_procent_obuchennyh.js -- гарантия, что пикер
// и отчёт всегда видят ОДИН И ТОТ ЖЕ список людей.
//
// ЧТО ДЕЛАТЬ, ЕСЛИ У РУКОВОДИТЕЛЯ+ПОДЧИНЁННЫХ НЕТ НИ ОДНОГО МИР-КОДА, СОВПАДАЮЩЕГО НИ С
// ОДНОЙ МАТРИЦЕЙ (matrixIds пуст): query_qual ставится в "MatchSome($elem/id,(0))" --
// id=0 не существует ни у одной реальной матрицы, поэтому пикер покажет ПУСТОЙ список,
// а не "все матрицы по умолчанию" -- тот же fail-safe принцип "лучше по ошибке спрятать
// данные, чем показать лишнее", что уже применяется в остальном тикете.
//
// ЭТО ПЕРВЫЙ РЕАЛЬНЫЙ ТЕСТ query_qual В ЭТОМ ТИКЕТЕ -- В ОТЛИЧИЕ ОТ ВСЕХ ПРОШЛЫХ
// ДИАГНОСТИК (которые проверялись по логам alert()), результат этого теста НУЖНО
// СМОТРЕТЬ ГЛАЗАМИ -- открыть модалку фильтров под руководителем без УОРиАП и
// посмотреть, действительно ли выпадающий список "Матрица обучения" сузился.
//
// ИТОГ ДИАГНОСТИКИ query_qual (24.09.2026, реальные тесты): наша логика вычисления
// query_qual подтверждена рабочей (проверено на каталоге collaborator -- пикер сузился
// корректно), но сам каталог cc_learning_matrice его не учитывает (ни при multiple: false,
// ни при multiple: true) -- вероятно, у него в админке платформы настроен свой кастомный
// проводник выбора. Это отдельная, ещё не решённая проблема НА СТОРОНЕ ПЛАТФОРМЫ -- код в
// этом файле её не может починить, query_qual оставлен как есть (правильно вычисленным),
// на случай если проблему с каталогом решат на стороне платформы.
// =====================================================================
//
// =====================================================================
// УЧЁТ is_active У САМОЙ МАТРИЦЫ (ДОБАВЛЕНО 24.09.2026, по прямому указанию пользователя):
//
// ДО этой правки GetMatrixIdsByMirCodes() проверяла активность ТОЛЬКО у элементов матрицы
// (cc_learning_matrice_element, см. GetAllActiveMatrixElementRows()) -- у самой матрицы
// (cc_learning_matrice) проверки не было. Деактивированная матрица с хотя бы одним
// оставшимся активным элементом всё равно попадала бы в вычисленный query_qual как
// "подходящая". ДОБАВЛЕНА FilterActiveMatrixIds() -- отдельный XQuery-запрос по
// cc_learning_matrices (is_active=true(), то же типизированное булево поле и тот же
// синтаксис, что и у элементов), которым отфильтровывается итоговый список matrixIds
// перед тем, как из него собирается query_qual. См. GetMatrixIdsByMirCodes() и
// FilterActiveMatrixIds() ниже.
// =====================================================================


DEBUG = true;

/*
* Чек-пойнт для отладки -- alert() с номером шага, только если DEBUG = true.
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
// (см. изменение поля macroregion в oForm.form_fields ниже, та же правка, что и в
// HREDU-183_filtry_modal_shag1.js).

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
* Достаёт полный URL текущей страницы. В контексте УДАЛЁННОГО ДЕЙСТВИЯ Request.Url НЕ
* отражает видимый адрес страницы -- нужен параметр "cur_page_url", привязанный в LPE
* к подстановке {{curEnv.curEnvUrl}} (см. подробности в HREDU-183_filtry_modal_shag1.js).
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
* Вырезает значение GET-параметра из полного URL строки -- без regex, без методов
* строк (.indexOf/.substring здесь не существуют), через штатный строковый API
* платформы.
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
* "0" из URL для picker-полей (foreign_elem) означает "не выбрано", НЕ реальный ID --
* платформа иначе пытается открыть несуществующий документ №0 (см. подробности в
* HREDU-183_filtry_modal_shag1.js).
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
* Убирает из URL старое значение указанного GET-параметра, не трогая остальную часть
* адреса -- нужно для переносимости (redirect на ТУ ЖЕ страницу, откуда открыли).
* @param {string} sUrl
* @param {string} sParamName
* @returns {string}   -   URL без этого параметра (если параметра не было -- вернёт как есть).
*/
function RemoveQueryParam(sUrl, sParamName)
{
    var sAmpMarker, sQMarkMarker, iMarkerPos, iValueStart, iAmpPos, iUrlLen, sBefore, sAfter;

    iUrlLen = StrLen(sUrl);

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

    return sUrl;
}

// =====================================================================
// ГЕЙТ ВИДИМОСТИ ПО МИР-КОДУ ДЛЯ ПИКЕРА "МАТРИЦА ОБУЧЕНИЯ" (ДОБАВЛЕНО 24.09.2026).
// Все функции этого блока -- БУКВАЛЬНАЯ КОПИЯ проверенного кода из
// HREDU-182_procent_obuchennyh.js (УОРиАП-гейт + BFS по иерархии + чтение мир-кодов),
// см. подробную историю находок и реальных тестов там же. Здесь НЕ повторяем историю,
// только код.
// =====================================================================

UORIAP_GROUP_ID = "7687978602560891542";
UORIAP_GROUP_XQUERY_TYPE = "groups";

// ИСПРАВЛЕНО (24.09.2026, реальный тест): изначально здесь читалась curUserID (заглавная
// D) как "голая" глобальная переменная -- по образцу 3 файлов-примеров с query_qual. На
// реальном тесте выяснилось, что это НЕ тот же механизм, что в отчёте
// (HREDU-182_procent_obuchennyh.js): там используется curUserId (строчная d) как
// LPE-параметр, который нужно ЯВНО привязать в редакторе страниц к этому конкретному
// удалённому действию/виджету -- ровно так же, как это уже сделано для выборки отчёта. По
// прямому указанию пользователя используем ТОЧНО ТУ ЖЕ переменную и тот же приём, что уже
// подтверждён рабочим в отчёте: curUserId, напрямую через try/catch, без typeof.
//
// ВАЖНО: чтобы это заработало, curUserId должен быть привязан как LPE-параметр в редакторе
// страниц для ЭТОГО удалённого действия (страница "Процент обученных" / модалка фильтров) --
// так же, как это уже сделано для выборки отчёта.
function GetCurUserIdSafe()
{
    var sRaw;
    try
    {
        sRaw = String(curUserId);
        alert("GetCurUserIdSafe(). curUserId: [" + sRaw + "]");
        if (sRaw != "" && sRaw != "0" && sRaw != "undefined" && sRaw != "null")
        {
            return curUserId;
        }
    }
    catch (_exDirect)
    {
        alert("GetCurUserIdSafe(). curUserId недоступна (параметр не привязан в LPE на этой странице/копии виджета?) -- ошибка: " + ExtractUserError(_exDirect));
    }
    alert("GetCurUserIdSafe(). curUserId не дала валидного значения -- возвращаем 0.");
    return 0;
}

function IsUorApMember(iCurUserId)
{
    var groupRows, iGroupId, groupDoc, collabList, foundRow, sRawUserId;

    sRawUserId = String(iCurUserId);
    alert("IsUorApMember(). ДИАГНОСТИКА: получен iCurUserId=[" + sRawUserId + "]");

    if (sRawUserId == "" || sRawUserId == "0" || sRawUserId == "undefined" || sRawUserId == "null")
    {
        alert("IsUorApMember(). iCurUserId пустой/нулевой -- fail-safe, доступа нет.");
        return false;
    }

    try
    {
        groupRows = ArraySelectAll(XQuery(
            "for $elem in " + UORIAP_GROUP_XQUERY_TYPE + " where $elem/id = '" + UORIAP_GROUP_ID + "' return $elem"
        ));
        if (ArrayCount(groupRows) == 0)
        {
            throw ("Группа с id=" + UORIAP_GROUP_ID + " не найдена через тип '" + UORIAP_GROUP_XQUERY_TYPE + "'");
        }
        iGroupId = Int(groupRows[0].id);
        groupDoc = tools.open_doc(iGroupId).TopElem;
        collabList = groupDoc.collaborators.collaborator;

        foundRow = ArrayOptFind(collabList, "String(This.collaborator_id) == String(iCurUserId)");
        alert("IsUorApMember(). Группа найдена (id=" + iGroupId + "), userId=" + sRawUserId + "; совпадение=" + (foundRow != undefined));
        return (foundRow != undefined);
    }
    catch (_exDoc)
    {
        alert("IsUorApMember(). ОШИБКА: " + ExtractUserError(_exDoc) + " -- fail-safe, считаем, что доступа НЕТ.");
        return false;
    }
}

function GetManagerIdRows()
{
    alert("GetManagerIdRows(). НАЧАЛО");
    var sqlText, rows;
    sqlText = "";
    sqlText = sqlText + "select cs.id,\r\n";
    sqlText = sqlText + "       c.data.value('(*/func_managers/func_manager[is_native=true()]/person_id)[1]', 'varchar(max)') as manager_id\r\n";
    sqlText = sqlText + "from collaborators cs\r\n";
    sqlText = sqlText + "inner join collaborator c on c.id = cs.id\r\n";
    sqlText = sqlText + "where cs.is_dismiss != 1";
    rows = ArraySelectAll(XQuery("sql:" + sqlText));
    alert("GetManagerIdRows(). Строк: " + ArrayCount(rows));
    alert("GetManagerIdRows(). КОНЕЦ");
    return rows;
}

function SortRowsByManagerId(managerRows)
{
    return ArraySort(managerRows, "OptInt(This.manager_id, 0)", "+");
}

function FindManagerRangeStart(sortedByManagerRows, iManagerId)
{
    var lo, hi, mid, midVal, iTarget, startIdx;
    iTarget = Int(iManagerId);
    lo = 0;
    hi = ArrayCount(sortedByManagerRows) - 1;
    startIdx = -1;
    while (lo <= hi)
    {
        mid = Int((lo + hi) / 2);
        midVal = OptInt(sortedByManagerRows[mid].manager_id, 0);
        if (midVal == iTarget)
        {
            startIdx = mid;
            hi = mid - 1;
        }
        else if (midVal < iTarget) { lo = mid + 1; }
        else { hi = mid - 1; }
    }
    return startIdx;
}

function CollectDirectSubordinateIds(sortedByManagerRows, iManagerId)
{
    var startIdx, i, n, result, iTarget, iRowId;
    result = [];
    iTarget = Int(iManagerId);
    startIdx = FindManagerRangeStart(sortedByManagerRows, iTarget);
    if (startIdx == -1) { return result; }
    n = ArrayCount(sortedByManagerRows);
    for (i = startIdx; i < n && OptInt(sortedByManagerRows[i].manager_id, 0) == iTarget; i++)
    {
        iRowId = Int(sortedByManagerRows[i].id);
        if (iRowId != iTarget) { result.push(iRowId); }
    }
    return result;
}

function GetManagerHierarchySubordinateIds(sortedByManagerRows, iCurUserId)
{
    var visitedIds, visitedSorted, frontier, nextFrontier, allSubordinates;
    var i, j, directIds, iManagerId, iSubId;

    allSubordinates = [];
    visitedIds = [Int(iCurUserId)];
    frontier = [Int(iCurUserId)];

    while (ArrayCount(frontier) > 0)
    {
        visitedSorted = SortIdArray(visitedIds);
        nextFrontier = [];
        for (i = 0; i < ArrayCount(frontier); i++)
        {
            iManagerId = frontier[i];
            directIds = CollectDirectSubordinateIds(sortedByManagerRows, iManagerId);
            for (j = 0; j < ArrayCount(directIds); j++)
            {
                iSubId = directIds[j];
                if (!IdArrayContainsSorted(visitedSorted, iSubId))
                {
                    allSubordinates.push(iSubId);
                    visitedIds.push(iSubId);
                    nextFrontier.push(iSubId);
                }
            }
        }
        frontier = nextFrontier;
    }

    return allSubordinates;
}

function SortIdArray(idArray)
{
    return ArraySort(idArray, "Int(This)", "+");
}

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

function SortRowsById(rows)
{
    return ArraySort(rows, "Int(This.id)", "+");
}

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

function GetMirCodeRows()
{
    var sqlText;
    sqlText = "";
    sqlText = sqlText + "select cs.id,\r\n";
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_mir_codes'']/value)[1]', 'varchar(max)') as mir_codes\r\n";
    sqlText = sqlText + "from collaborators cs\r\n";
    sqlText = sqlText + "inner join collaborator c on c.id = cs.id\r\n";
    sqlText = sqlText + "where cs.is_dismiss != 1";
    return ArraySelectAll(XQuery("sql:" + sqlText));
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
        if (ArrayCount(fields) > 0) { codes.push(String(fields[0])); }
    }
    return codes;
}

/*
* Собирает список ТЕКСТОВЫХ мир-кодов текущего пользователя + всех его подчинённых
* (по всей цепочке вниз) -- в один общий, без дублей, список. Один сотрудник может иметь
* НЕСКОЛЬКО мир-кодов (см. ExtractMirCodes()) -- берём их все.
* @param {number} iCurUserId
* @param {number[]} subordinateIds
* @returns {string[]}
*/
function GetRelevantMirCodes(iCurUserId, subordinateIds)
{
    var mirCodeRows, sortedMirCodeRows, relevantIds, i, row, codes, j, allCodes;

    mirCodeRows = GetMirCodeRows();
    sortedMirCodeRows = SortRowsById(mirCodeRows);

    relevantIds = [Int(iCurUserId)];
    for (i = 0; i < ArrayCount(subordinateIds); i++) { relevantIds.push(Int(subordinateIds[i])); }

    allCodes = [];
    for (i = 0; i < ArrayCount(relevantIds); i++)
    {
        row = BinarySearchById(sortedMirCodeRows, relevantIds[i]);
        if (row == undefined) { continue; }
        codes = ExtractMirCodes(row.mir_codes);
        for (j = 0; j < ArrayCount(codes); j++) { allCodes.push(codes[j]); }
    }

    return ArraySelectDistinct(allCodes, "This");
}

/*
* ДОБАВЛЕНО (28.09.2026, HREDU-237) -- см. "ИСТОЧНИК ДАННЫХ HREDU-237" в шапке файла.
* Безопасно интерпретирует текстовое "булево" значение custom_elem (f_matrix_active) --
* идентична версии в HREDU-182_procent_obuchennyh.js.
* @param {string} sValue
* @returns {boolean}
*/
function IsActiveText(sValue)
{
    try { return tools_web.is_true(sValue); }
    catch (_ex) { return (String(sValue) == "true" || String(sValue) == "1"); }
}

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
* Матчинг "* текст *" БЕЗ regex -- ИДЕНТИЧНА версии в HREDU-182_procent_obuchennyh.js
* (включая ПОДТВЕРЖДЁННЫЙ РЕАЛЬНЫМ ТЕСТОМ 28.09.2026 фикс -- 3-й параметр StrOptSubStrPos
* работает НАОБОРОТ от названия: true игнорирует регистр, false учитывает буквально).
* @param {string} sPattern
* @param {string} sText
* @param {boolean} bIgnoreCase
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
* ДОБАВЛЕНО (28.09.2026, HREDU-237). Массовое чтение id/название/is_active/f_mir_code
* ВСЕХ модульных программ (compound_program) одним SQL -- та же проверенная техника
* c.data.value(), что в HREDU-182_procent_obuchennyh.js (GetCompoundProgramRows(), там
* читает больше полей аудитории -- здесь достаточно f_mir_code, остальные оси этому
* пикеру не нужны).
*
* ОТКАЧЕНО (01.10.2026) -- см. "ОТКАЧЕНО" у GetMatrixIdsByMirCodes() ниже: добавлялась
* колонка hex_id в предположении, что id документа реально хранится как hex-строка --
* реальный тест ПОКАЗАЛ, что это НЕВЕРНО (XQuery-выражение "(star-slash id)[1]" вернул тот же
* ГОЛЫЙ ДЕСЯТИЧНЫЙ ТЕКСТ, что и cs.id, а не "0x..."). Hex в экспорте документа -- судя по
* всему, ТОЛЬКО косметика самого экспорта/просмотра, а не реальное хранимое значение (та
* же категория сюрприза, что уже была с is_native/person_id). Колонка убрана.
* @returns {Object[]}   -   {id, name, f_matrix_active, f_mir_code}.
*/
function GetCompoundProgramRowsForFilter()
{
    var sqlText;
    sqlText = "";
    sqlText = sqlText + "select cs.id,\r\n";
    sqlText = sqlText + "       c.data.value('(*/name)[1]', 'varchar(max)') as name,\r\n";
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_matrix_active'']/value)[1]', 'varchar(max)') as f_matrix_active,\r\n";
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_mir_code'']/value)[1]', 'varchar(max)') as f_mir_code\r\n";
    sqlText = sqlText + "from compound_programs cs\r\n";
    sqlText = sqlText + "inner join compound_program c on c.id = cs.id";
    return ArraySelectAll(XQuery("sql:" + sqlText));
}

/*
* ПЕРЕПИСАНО (28.09.2026, HREDU-237) -- см. "ИСТОЧНИК ДАННЫХ HREDU-237" в шапке файла.
* Раньше искала матрицы (cc_learning_matrice) через их АКТИВНЫЕ ЭЛЕМЕНТЫ с mir_code_id
* (FK на cc_mir_code). Теперь мир-код -- СВОБОДНЫЙ ТЕКСТ (wildcard) прямо на самой
* compound_program (custom_elem f_mir_code) -- один SQL читает ВСЕ модульные программы
* сразу (is_active проверяется тут же, без отдельного FilterActiveMatrixIds() -- УБРАНА
* как более ненужная, тот же принцип, что в отчёте).
*
* РЕШЕНИЕ ПО ПУСТОМУ f_mir_code: матрица БЕЗ заданного мир-кода (f_mir_code == "") НЕ
* считается "подходящей" ни под один мир-код руководителя -- то есть НЕ попадёт в этот
* конкретный сужающий список. ЭТО НЕ ТО ЖЕ САМОЕ, что "пустая ось = без ограничения" в
* аудитории отчёта (HREDU-182_procent_obuchennyh.js) -- там смысл другой ("кто входит в
* аудиторию"), здесь -- "какие матрицы релевантны ИМЕННО этому мир-коду" (та же логика,
* что была у старой модели: элемент БЕЗ mir_code_id тоже не делал матрицу "подходящей"
* через этот механизм).
*
* ОТКАЧЕНО (01.10.2026, реальный тест): пробовали версию, где эта функция возвращала
* {ids, quotedHexIds} и query_qual строился на ЗАКВОЧЕННЫХ hex-строках
* (MatchSome($elem/id,("0x..."))) -- результат ХУЖЕ: пикер теперь падает с ошибкой прямо
* при клике (XAML-исключение в диалоге выбора), а не просто показывает пустой список, как
* раньше. Это означает, что $elem/id на самом деле ЧИСЛО (не строка) -- сравнение с
* заквоченным строковым литералом ломает XQuery-выражение целиком. ОТКАЧЕНО обратно к
* голым десятичным id без кавычек -- см. функцию ниже и её использование в Run(). Причина
* исходной проблемы (пикер пуст у руководителей) ПОКА НЕ НАЙДЕНА -- нужен другой подход,
* не угадывание формата id.
* @param {string[]} aTargetMirCodes
* @returns {number[]}
*/
function GetMatrixIdsByMirCodes(aTargetMirCodes)
{
    var allRows, i, j, row, matrixIdSet;

    allRows = GetCompoundProgramRowsForFilter();
    alert("GetMatrixIdsByMirCodes(). Всего модульных программ (compound_program): " + ArrayCount(allRows));

    matrixIdSet = [];
    for (i = 0; i < ArrayCount(allRows); i++)
    {
        row = allRows[i];
        if (!IsActiveText(row.f_matrix_active)) { continue; }
        if (row.f_mir_code == undefined || String(row.f_mir_code) == "") { continue; }
        for (j = 0; j < ArrayCount(aTargetMirCodes); j++)
        {
            if (MatchAnySemicolonPattern(row.f_mir_code, aTargetMirCodes[j], true))
            {
                matrixIdSet.push(Int(row.id));
                break;
            }
        }
    }

    matrixIdSet = ArraySelectDistinct(matrixIdSet, "This");
    alert("GetMatrixIdsByMirCodes(). Подходящих активных модульных программ: " + ArrayCount(matrixIdSet));
    return matrixIdSet;
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
    oForm.title = "Фильтры отчёта (Процент обученных)";
    oForm.message = null;

    DebugAlert("3b. Читаем текущий URL страницы для восстановления фильтров (сначала параметр cur_page_url, потом Request.Url)");
    sModalPageUrl = GetCurPageUrlSafe();
    DebugAlert("3b2. Итоговый URL, который используем: [" + sModalPageUrl + "]");
    sDefaultMatrixID = SanitizeIdFieldValue(GetQueryParam(sModalPageUrl, "matrix_id"));
    sDefaultMacroregion = GetQueryParam(sModalPageUrl, "macroregion");
    sDefaultMirCodeID = SanitizeIdFieldValue(GetQueryParam(sModalPageUrl, "mir_code_id"));
    sDefaultPositionCommonID = SanitizeIdFieldValue(GetQueryParam(sModalPageUrl, "position_common_id"));
    sDefaultProgramID = SanitizeIdFieldValue(GetQueryParam(sModalPageUrl, "program_id"));
    DebugAlert("3c. Значения по умолчанию из URL: matrix_id=[" + sDefaultMatrixID + "] macroregion=[" + sDefaultMacroregion
        + "] mir_code_id=[" + sDefaultMirCodeID + "] position_common_id=[" + sDefaultPositionCommonID
        + "] program_id=[" + sDefaultProgramID + "]");

    // ДОБАВЛЕНО (24.09.2026) -- см. "ФИЛЬТРАЦИЯ ПИКЕРА..." в шапке файла. По умолчанию --
    // fail-safe: показываем ПУСТОЙ список матриц (sentinel id=0, никогда не совпадёт ни с
    // одной реальной матрицей), пока явно не докажем, что пользователю можно показать
    // больше (УОРиАП -- вообще без ограничений, руководитель -- список по мир-кодам).
    // ИСПРАВЛЕНО (24.09.2026, реальный тест): раньше тут была явная проверка
    // "if (iCurUserId > 0)" перед вызовом IsUorApMember()/BFS -- на реальном тесте
    // руководитель увидел ВСЕ матрицы без ограничений, то есть sMatrixQueryQual остался
    // пустым "" (ветка УОРиАП), а не sentinel -- при том что curUserId в логе печатался
    // корректно. Сравнение "iCurUserId > 0" на таком огромном id (19 цифр) в этом движке
    // ненадёжно -- та же категория проблем, что уже задокументирована в отчёте про
    // Int()/OptInt() на LPE-параметрах. УБРАНО полностью, по образцу уже проверенного гейта
    // в отчёте (HREDU-182_procent_obuchennyh.js): там тоже НЕТ явной проверки "> 0" --
    // IsUorApMember(0) корректно вернёт false, а BFS от несуществующего id=0 корректно даст
    // 0 подчинённых, и sMatrixQueryQual естественным образом останется sentinel-заглушкой
    // "0" -- то есть fail-safe работает сам по себе, без отдельной ветки на невалидный id.
    iCurUserId = GetCurUserIdSafe();

    // ИСПРАВЛЕНО (30.09.2026, реальный тест -- руководитель без ни одной подходящей
    // программы под свои мир-коды получал "sMatrixQueryQual not defined" и модалка не
    // открывалась ВООБЩЕ, ещё на шаге step_0, до всякого клика). ПРИЧИНА: ниже переменная
    // sMatrixQueryQual присваивалась ТОЛЬКО в двух случаях -- УОРиАП (пустая строка) и
    // "руководитель, matrixIds не пуст" (MatchSome(...)) -- а в третьем случае
    // (руководитель, но matrixIds ПУСТ) её не присваивали вообще, ни в одной ветке.
    // Комментарий ниже ОБЕЩАЛ sentinel по умолчанию, но самой инициализации не было --
    // судя по всему, потеряна при правке 24.09.2026 (см. комментарий выше про удаление
    // проверки "iCurUserId > 0"). На этой платформе обращение к НИКОГДА не присвоенной
    // переменной кидает ошибку "not defined" (та же категория, что и находка про typeof),
    // а не даёт undefined как в обычном JS -- отсюда и ошибка. ФИКС: sentinel по умолчанию
    // ставится ЯВНО, ДО if/else -- id=0 никогда не совпадёт ни с одной реальной модульной
    // программой, то есть пикер matrix_id останется пустым, как и было задумано, но без
    // падения.
    sMatrixQueryQual = "MatchSome($elem/id,(0))";

    if (IsUorApMember(iCurUserId))
    {
        DebugAlert("3d. Пользователь id=" + iCurUserId + " -- УОРиАП, пикер матриц БЕЗ ограничений.");
        sMatrixQueryQual = "";
    }
    else
    {
        managerRows = GetManagerIdRows();
        sortedByManagerRows = SortRowsByManagerId(managerRows);
        subordinateIds = GetManagerHierarchySubordinateIds(sortedByManagerRows, iCurUserId);
        DebugAlert("3d. Пользователь id=" + iCurUserId + " -- НЕ УОРиАП, подчинённых по всей цепочке вниз: " + ArrayCount(subordinateIds));

        relevantMirCodes = GetRelevantMirCodes(iCurUserId, subordinateIds);
        DebugAlert("3e. Мир-коды (свой + подчинённых), всего: " + ArrayCount(relevantMirCodes) + " -- [" + ArrayMerge(relevantMirCodes, "This", ", ") + "]");

        matrixIds = GetMatrixIdsByMirCodes(relevantMirCodes);
        DebugAlert("3f. Матриц, подходящих под эти мир-коды: " + ArrayCount(matrixIds) + " -- [" + ArrayMerge(matrixIds, "This", ", ") + "]");

        // ОТКАЧЕНО (01.10.2026) -- см. ОТКАЧЕНО у GetMatrixIdsByMirCodes() выше. Версия с
        // заквоченными hex-строками реально ЛОМАЛА пикер (падал при клике), а не просто не
        // фильтровала. Возвращено к голым десятичным id без кавычек -- это ПРЕЖНЕЕ состояние
        // (пикер пуст у руководителей, но хотя бы не падает) -- причина пустого пикера ещё не
        // найдена, нужен другой подход.
        if (ArrayCount(matrixIds) > 0)
        {
            sMatrixQueryQual = "MatchSome($elem/id,(" + ArrayMerge(matrixIds, "This", ",") + "))";
        }
        else
        {
            // ДОБАВЛЕНО (30.09.2026, по прямой просьбе пользователя) -- вместо того, чтобы
            // молча показать пустой пикер без единого пояснения, честно говорим человеку,
            // почему он пуст. sMatrixQueryQual остаётся sentinel-заглушкой (см. выше).
            DebugAlert("3f2. У руководителя/его подчинённых НЕТ подходящих активных модульных программ -- показываем поясняющее сообщение вместо пустого пикера.");
            oForm.message = "У вас и ваших подчинённых нет ни одной подходящей активной учебной программы -- список матриц обучения пуст.";
        }
    }
    DebugAlert("3g. Итоговый query_qual для matrix_id: [" + sMatrixQueryQual + "]");

    oForm.form_fields = [
        {
            name: "matrix_id",
            label: "Матрица обучения *",
            title: "Выберите матрицу обучения",
            type: "foreign_elem",
            value: sDefaultMatrixID,
            mandatory: true,
            multiple: false,
            // ИЗМЕНЕНО (28.09.2026, HREDU-237) -- было "cc_learning_matrice". Название
            // "compound_program" НЕ ПОДТВЕРЖДЕНО реальным тестом -- см. "ИСТОЧНИК ДАННЫХ
            // HREDU-237" в шапке файла, проверить первым делом.
            catalog: "compound_program",
            // ИТОГ ДИАГНОСТИКИ (24.09.2026, реальный тест, каталог cc_learning_matrice):
            // query_qual в этом удалённом действии РАБОТАЕТ КОРРЕКТНО в общем случае -- на
            // диагностическом тесте с catalog: "collaborator" и query_qual, ограниченным
            // одним curUserId, пикер показал РОВНО одного человека. Значит наш код (сбор
            // subordinateIds, relevantMirCodes, matrixIds, sMatrixQueryQual) полностью
            // верен, а не фильтрует именно каталог cc_learning_matrice -- то есть проблема
            // на стороне конфигурации ЭТОГО каталога в админке платформы (вероятно, свой
            // кастомный проводник выбора вместо стандартного, который query_qual не
            // учитывает). НЕ ПРОВЕРЕНО (28.09.2026), повторяется ли та же проблема для
            // catalog: "compound_program" -- см. "ИСТОЧНИК ДАННЫХ HREDU-237" в шапке файла.
            query_qual: sMatrixQueryQual
        },
        {
            // ИЗМЕНЕНО (16.09.2026, по прямой просьбе пользователя): было select со
            // списком значений из GetMacroregionEntries() (SQL DISTINCT, убрана) -- стало
            // обычное текстовое поле, значение вводится вручную и сравнивается "как есть".
            // Убрано и visibility: false -- поле было скрыто, для текстового поля ввода
            // это не нужно. См. тот же комментарий про непроверенность типа "string" в
            // HREDU-183_filtry_modal_shag1.js -- относится и сюда.
            name: "macroregion",
           label: "Макрорегион",
            type: "string",
            value: sDefaultMacroregion,
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

            // Переносимость: redirect на ТУ ЖЕ страницу, откуда открыли модалку.
            sCleanBaseUrl = sModalPageUrl;
            sCleanBaseUrl = RemoveQueryParam(sCleanBaseUrl, "matrix_id");
            sCleanBaseUrl = RemoveQueryParam(sCleanBaseUrl, "macroregion");
            sCleanBaseUrl = RemoveQueryParam(sCleanBaseUrl, "mir_code_id");
            sCleanBaseUrl = RemoveQueryParam(sCleanBaseUrl, "mir_code");
            sCleanBaseUrl = RemoveQueryParam(sCleanBaseUrl, "position_common_id");
            sCleanBaseUrl = RemoveQueryParam(sCleanBaseUrl, "program_id");
            DebugAlert("7c2. Текущая страница без старых фильтров: [" + sCleanBaseUrl + "]");

            oQueryParams = {
                matrix_id: String(iMatrixID),
                macroregion: sMacroregion,
                mir_code: sMirCodeText,
                mir_code_id: String(iMirCodeID),
                position_common_id: String(iPositionCommonID),
                program_id: String(iProgramID)
            };
            sQueryString = UrlEncodeQuery(oQueryParams);

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
    RESULT = {
        command: "alert",
        msg: ("Ошибка в модалке фильтров (HREDU_182_filtry_percent.js):<br/><pre>" + ExtractUserError(_exMain) + "</pre>"),
        title: "ОШИБКА"
    };
}
