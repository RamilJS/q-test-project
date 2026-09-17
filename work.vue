// =====================================================================
// HREDU-182. "Процент обученных" -- выборка для Табличных данных.
//
// ТЗ (из письма Антонова + "ТЗ Отчёт по матрицам", присланы пользователем 10.09
// и 14.09.2026) + РЕАЛЬНЫЙ ПРИМЕР ОТЧЁТА, присланный 14.09.2026 ("Макрорегион ВОСТОК",
// лист с колонками Город/Общее кол-во сотрудников/План/Факт/Процент/Обязательно к
// прохождению, построчно по городам + итоговая строка "Общий итог").
//
// РЕШЕНО С ПОЛЬЗОВАТЕЛЕМ (14.09.2026, через AskUserQuestion): реальный отчёт -- это
// ТАБЛИЦА С РАЗБИВКОЙ ПО ГОРОДАМ (как в примере), а НЕ одна строка показателей на всю
// матрицу (как можно было бы прочитать из ТЗ п.4 буквально -- там про "поле" в
// единственном числе, но пример явно про таблицу).
//
// ИЗМЕНЕНО (16.09.2026, по итогам разговора пользователя с тимлидом): выяснилось, что
// у одной матрицы может быть НЕСКОЛЬКО элементов (cc_learning_matrice_element), каждый
// со своей учебной программой (education_method_id) и своим периодом обучения -- см.
// переписку 15.09.2026 про каталоги "Матрицы обучения"/"Элементы матриц обучения".
// Раньше строка таблицы была на КАЖДЫЙ ГОРОД, а показатели total/plan/fact/mandatory
// СУММИРОВАЛИСЬ по всем программам матрицы в одну строку -- значит из таблицы было
// не видно, по какому именно элементу (программе) идёт расчёт. Теперь строка -- это
// ПАРА (Город, Учебная программа): если в городе есть 2 программы матрицы, у него будет
// 2 строки, по одной на каждую. Явно запрошено пользователем 16.09.2026: "сделаем ещё
// одну колонку после город - Учебная программа... чтобы было понятно по какому именно
// элементу матрицы строка отчета".
//
// Поле "город" -- custom_elem с именем "sity" (ПОДТВЕРЖДЕНО пользователем 14.09.2026,
// прислал реальный XML документа collaborator: <custom_elem><name>sity</name>
// <value>Санкт-Петербург</value></custom_elem>). ВНИМАНИЕ: имя технического поля
// именно "sity" (с опечаткой, не "city") -- это НЕ опечатка в этом файле, так
// называется реальное поле в системе.
//
// ИЗМЕНЕНО (17.09.2026, HREDU-215 "Правки 1", по прямому указанию тим-лида пользователя --
// "ошиблись в архитектуре"): поля position_common_id/mir_code_id УДАЛЕНЫ с типа документа
// "Матрицы обучения" (cc_learning_matrice) и ДОБАВЛЕНЫ на тип документа "Элементы матриц
// обучения" (cc_learning_matrice_element, у которых уже были education_method_id/
// start_study_period/end_study_period). Значит аудитория (должность+мир-код) теперь СВОЯ
// У КАЖДОЙ ПРОГРАММЫ (у каждого элемента, а если у одной программы несколько элементов с
// разными position_common_id -- у неё несколько аудиторий, объединяемых через ИЛИ), а НЕ
// ОДНА НА ВСЮ МАТРИЦУ, как было раньше. См. BuildProgramAudienceIndex()/
// CollaboratorInProgramAudience() ниже -- та же замена, что в HREDU-183_tep_reports.js.
//
// ЛОГИКА ПОДСЧЁТА (полностью повторяет HREDU-183_tep_reports.js -- см. "РЕШЕНИЯ" в его
// шапке -- только теперь всё разбито по городам вместо одного общего числа):
//   Общее (total)      -- аудитория ЭТОЙ ПРОГРАММЫ (по всем её активным элементам) +
//                          ручные фильтры пользователя, сгруппировано по городу.
//   План (plan)         -- = Общее (упрощение, период прохождения ещё не реализован,
//                          см. открытый вопрос в HREDU-183_tep_reports.js).
//   Факт (fact)          -- прошедшие тренинг, БЕЗ ограничения аудиторией (см. ТЗ п.3
//                          "не зависимо от условий матрицы"), но с теми же ручными
//                          фильтрами, сгруппировано по городу.
//   Обязательно (mandatory) -- аудитория программы МИНУС прошедшие (пустая дата).
//   Процент (percent)    -- факт/план (округление до целого %, "-" если план = 0).
//     Уточнено с пользователем 10.09.2026 (см. HREDU-183_tep_reports.js) -- в тексте ТЗ
//     написано "план/факт", реально считаем факт/план, подтверждено сверкой с примером
//     (Новосибирск: 10/11=91%, Красноярск: 9/10=90%, Итог: 70/72=97% -- ВСЕ совпадают
//     ТОЛЬКО с факт/план, не план/факт).
//
// ГРУППИРОВКА ПО (ГОРОД, ПРОГРАММА) -- см. ДОПУЩЕНИЯ ниже. Строка появляется, если для
// этой пары есть хотя бы 1 человек в АУДИТОРИИ МАТРИЦЫ (Общее > 0) ПО ЭТОЙ ПРОГРАММЕ,
// ИЛИ хотя бы 1 человек её прошёл (Факт > 0), даже если он сейчас не входит в аудиторию.
//
// ДОПУЩЕНИЯ (уточнить с пользователем при первом реальном прогоне):
//   1. Строки = пары (город, программа), где есть хотя бы 1 человек в АУДИТОРИИ (для
//      этой программы) ИЛИ хотя бы 1 человек её ПРОШЁЛ (даже без аудитории для неё) --
//      см. пункт 16.09.2026 ниже, "ГРУППИРОВКА ПО (ГОРОД, ПРОГРАММА)". Раньше (до
//      16.09.2026) правило было мягче -- разбивка только по городу, и Факт-only город
//      получал строку с Общее=0 автоматически, т.к. строка уже существовала из
//      аудиторного прохода. Сейчас, при разбивке ещё и по программе, аналог того же
//      правила -- порождать строку из фактового прохода тоже, но ТОЛЬКО когда реально
//      есть хотя бы 1 прошедший (иначе при большой факт-базе получился бы взрыв пустых
//      строк "город x программа x 0 фактов x 0 аудитории", которых никто не хочет
//      видеть).
//   2. Сотрудники БЕЗ заполненного города (custom_elem "sity" пустой) -- попадают в
//      отдельную группу "(без города)", чтобы не терять данные молча.
//   3. Итоговая строка "Общий итог" -- сумма по ВСЕМ парам (город, программа), колонка
//      "Учебная программа" в ней -- "-" (относится ко всем программам сразу).
//   4. Сортировка строк (ИЗМЕНЕНО 16.09.2026) -- сначала по названию программы, затем
//      по городу внутри неё (простое сравнение строк, БЕЗ гарантии точной русской
//      локали в этом движке) -- т.е. все города одной программы идут блоком, а не
//      вперемешку город-за-городом. "Общий итог" всегда последней строкой.
//
// Параметры/фильтры (matrix_id, macroregion, mir_code, position_common_id, program_id)
// читаются ИЗ URL -- точно так же, как в HREDU-183_tep_reports.js. macroregion в этой
// выборке работает как ПРЕДФИЛЬТР (например "Восток" -- сузить список городов до
// одного макрорегиона, как в примере), а группировка идёт уже ПО ГОРОДУ внутри него.
//
// ВАЖНО про RESULT: как и в остальных выборках -- RESULT это ПРЯМО массив строк.
// =====================================================================

DEBUG = true;              // На проде поставить false после тестирования
LOG_NAME = "agent";        // TODO: заполнить после создания документа в админке
CUR_OBJECT_ID = 0;         // TODO: заполнить ID документа выборки после её создания в админке (LogAlert защищена try/catch -- забытый 0 не обрушит Run())

// ИСПРАВЛЕНО (16.09.2026): было "mode=matrix_test" -- адрес тестовой страницы, оставшийся
// как TODO-заглушка. Пользователь сообщил, что переименовал реальную (production) страницу
// ТЭП-отчётов с "matrix_test" на "matrix_report". Пока здесь оставался старый адрес, клик
// по строке в "Процент обученных" вёл на СТАРУЮ (тестовую, возможно неактуальную/по-другому
// настроенную) страницу -- это, судя по всему, и есть причина, почему параметр city "не
// доезжал": на новой странице (matrix_report), куда пользователь при ручной проверке заходил
// сам через фильтры, всё работало, а клик по строке уводил на другую, старую страницу.
TEP_REPORT_PAGE_URL = "/view_doc.html?mode=matrix_report";

//-------------------------------------------------------------------------
//              Область функций
//-------------------------------------------------------------------------

function LogAlert(typeLog, message)
{
    try
    {
        tools.call_code_library_method("vtbl_log_lib", "LogAlert", [LOG_NAME, typeLog, CUR_OBJECT_ID, message, DEBUG]);
    }
    catch (_exLog)
    {
        // ничего -- сбой логирования не должен ронять основной код
    }
}

function GetRequestUrlSafe()
{
    try { return String(Request.Url); }
    catch (_ex) { return ""; }
}

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
        if (iParamPos == undefined) { return ""; }
        iValueStart = iParamPos + StrLen(sQMarkMarker);
    }
    iAmpPos = StrOptSubStrPos(sUrl, "&", false, iValueStart);
    iValueEnd = (iAmpPos != undefined ? iAmpPos : iUrlLen);
    sRawValue = StrRangePos(sUrl, iValueStart, iValueEnd);
    try { return UrlDecode(sRawValue); }
    catch (_exDecode) { return sRawValue; }
}

function GetMatrixRows(matrixName)
{
    return ArraySelectAll(XQuery("for $elem in cc_learning_matrices where $elem/name = " + XQueryLiteral(matrixName) + " return $elem"));
}

function GetMatrixElementRows(matrixIds)
{
    return ArraySelectAll(XQuery("for $elem in cc_learning_matrice_elements where MatchSome($elem/cc_learning_matrice_id, (" + ArrayMerge(matrixIds, "This", ",") + ")) and $elem/is_active=true() return $elem"));
}

/*
 * ИЗМЕНЕНО (17.09.2026, HREDU-215 "Правки 1"): у типа документов "Матрицы обучения"
 * (cc_learning_matrice) поле education_method_id УДАЛЕНО (тим-лид пересмотрел архитектуру --
 * см. шапку файла) -- программы теперь заданы ТОЛЬКО через элементы
 * (cc_learning_matrice_element). Раньше эта функция объединяла education_method_id
 * матрицы И её элементов (см. историю правки 15.09.2026, "Int(), Unknown source") --
 * теперь читаем ТОЛЬКО из elementRows, параметр matrixRows убран.
 */
function GetProgramIds(elementRows)
{
    var elementProgramIds, allProgramIds, i;
    elementProgramIds = ArrayExtract(elementRows, "OptInt(This.education_method_id, 0)");
    allProgramIds = [];
    for (i = 0; i < ArrayCount(elementProgramIds); i++) { if (Int(elementProgramIds[i]) > 0) { allProgramIds.push(elementProgramIds[i]); } }
    return ArraySelectDistinct(allProgramIds, "This");
}

/*
 * ПЕРЕИМЕНОВАНО (17.09.2026, было ResolveProgramIds -- возвращала только programIds):
 * теперь возвращает ещё и elementRows -- нужны отдельно для BuildProgramAudienceIndex()
 * ниже (аудитория HREDU-215 теперь считается ПО ЭЛЕМЕНТАМ, не по самой матрице).
 * @returns {Object}   -   { programIds: number[], elementRows: Object[] }.
 */
function ResolveMatrixContext(matrixId, matrixName)
{
    var matrixRows, matrixIds, elementRows, programIds;
    matrixRows = GetMatrixRows(matrixName);
    matrixIds = ArrayExtract(matrixRows, "Int(This.id)");
    if (ArrayCount(matrixIds) == 0)
    {
        throw ("Не найдено ни одной записи cc_learning_matrice с названием [" + matrixName + "]");
    }
    elementRows = GetMatrixElementRows(matrixIds);
    programIds = GetProgramIds(elementRows);
    if (ArrayCount(programIds) == 0)
    {
        throw ("У матрицы [" + matrixName + "] не найдено ни одной активной программы");
    }
    return { programIds: programIds, elementRows: elementRows };
}

function GetActiveCollaboratorRows()
{
    return ArraySelectAll(XQuery("for $elem in collaborators where $elem/is_dismiss=false() return $elem"));
}

function GetPositionIdsByCommonPosition(iCommonPositionFilter)
{
    var positionRows, positionIds, i;
    positionRows = ArraySelectAll(XQuery("for $elem in positions where $elem/position_common_id = " + iCommonPositionFilter + " return $elem"));
    positionIds = [];
    for (i = 0; i < ArrayCount(positionRows); i++) { positionIds.push(Int(positionRows[i].id)); }
    return positionIds;
}

function IdArrayContains(idArray, value)
{
    var i;
    for (i = 0; i < ArrayCount(idArray); i++)
    {
        if (Int(idArray[i]) == Int(value)) { return true; }
    }
    return false;
}

function GetMacroregionRows()
{
    var sqlText;
    sqlText = "";
    sqlText = sqlText + "select cs.id,\r\n";
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_2ewj'']/value)[1]', 'varchar(max)') as macroregion\r\n";
    sqlText = sqlText + "from collaborators cs\r\n";
    sqlText = sqlText + "inner join collaborator c on c.id = cs.id\r\n";
    sqlText = sqlText + "where cs.is_dismiss != 1";
    return ArraySelectAll(XQuery("sql:" + sqlText));
}

/*
 * НОВОЕ (14.09.2026): город -- custom_elem "sity" (имя поля подтверждено пользователем
 * реальным XML документа collaborator). Та же схема, что GetMacroregionRows()/
 * GetMirCodeRows() -- один SQL на всех активных сотрудников сразу.
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
 * Находит город конкретного сотрудника; "(без города)" если поле пустое/не найдено
 * (см. ДОПУЩЕНИЕ №2 в шапке файла -- чтобы не терять данные молча).
 * @param {Object[]} cityRows
 * @param {number} collaboratorID
 * @returns {string}
 */
function FindCity(cityRows, collaboratorID)
{
    var cityRow, sCity;
    cityRow = ArrayOptFind(cityRows, "Int(This.id) == Int(collaboratorID)");
    sCity = (cityRow != undefined && cityRow.sity != undefined ? String(cityRow.sity) : "");
    return (sCity != "" ? sCity : "(без города)");
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

function ExtractMirCodes(rawValue)
{
    var parts, fields, codes, i;
    codes = [];
    parts = ArrayDirect(ArraySelect(String(rawValue).split("|"), "This != ''"));
    for (i = 0; i < ArrayCount(parts); i++)
    {
        fields = ArrayDirect(ArraySelect(String(parts[i]).split("#"), "This != ''"));
        if (ArrayCount(fields) > 0) { codes.push(String(fields[0])); }
    }
    return codes;
}

function CollaboratorHasMirCode(mirCodeRows, collaboratorID, mirCodeFilter)
{
    var row, codes;
    row = ArrayOptFind(mirCodeRows, "Int(This.id) == Int(collaboratorID)");
    if (row == undefined) { return false; }
    codes = ExtractMirCodes(row.mir_codes);
    return (ArrayOptFind(codes, "String(This) == String(mirCodeFilter)") != undefined);
}

function ResolveMirCodeText(iMirCodeID)
{
    if (OptInt(iMirCodeID, 0) <= 0) { return ""; }
    try { return String(tools.open_doc(Int(iMirCodeID)).TopElem.name); }
    catch (_ex) { return ""; }
}

/*
 * НОВОЕ (16.09.2026): резолвит id учебной программы (education_method) в текст --
 * та же схема, что ResolveMirCodeText() (каталог "education_method" -- см.
 * catalog: "education_method" у поля program_id в HREDU-183_filtry_modal_shag1.js).
 * @param {number} iProgramId
 * @returns {string}
 */
function ResolveProgramText(iProgramId)
{
    if (OptInt(iProgramId, 0) <= 0) { return "(без программы)"; }
    try { return String(tools.open_doc(Int(iProgramId)).TopElem.name); }
    catch (_ex) { return "id=" + iProgramId; }
}

/*
 * Резолвит id программы в текст ЧЕРЕЗ УЖЕ ГОТОВЫЙ КЭШ (см. Run() -- programNames
 * строится ОДИН РАЗ на все уникальные programIds, а не по разу на каждого сотрудника x
 * программу -- иначе tools.open_doc() дёргался бы много тысяч раз).
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

// УБРАНО (17.09.2026, HREDU-215 "Правки 1"): GetMatrixAudienceCollaboratorRows() читала
// аудиторию (должность+мир-код) с полей position_common_id/mir_code_id САМОЙ МАТРИЦЫ --
// этих полей у типа документа больше нет, тим-лид перенёс их на "Элементы матриц
// обучения" (см. шапку файла и HREDU-183_tep_reports.js -- идентичная замена). Аудитория
// теперь СВОЯ У КАЖДОЙ ПРОГРАММЫ (по всем активным элементам с этим education_method_id,
// через ИЛИ между элементами) -- см. BuildProgramAudienceIndex()/CollaboratorInProgramAudience()
// ниже, применяются в Run() ВНУТРИ цикла по программам (а не один раз ко всему пулу
// сотрудников ДО цикла, как раньше).

/*
 * Строит по каждой программе (education_method_id) список "сегментов аудитории" -- один
 * на КАЖДЫЙ активный элемент матрицы с этим education_method_id. Сотрудник входит в
 * аудиторию программы, если подходит ХОТЯ БЫ ПОД ОДИН сегмент. allowedPositionIds
 * считается ОДИН РАЗ НА СЕГМЕНТ здесь (не на каждого сотрудника) -- та же логика и те же
 * функции, что в HREDU-183_tep_reports.js (см. подробный комментарий там).
 * @param {Object[]} elementRows
 * @param {number[]} programIds
 * @returns {Object[]}   -   [{ programId, segments: [{ positionCommonId, allowedPositionIds, mirCodeText }] }].
 */
function BuildProgramAudienceIndex(elementRows, programIds)
{
    var index, i, j, programId, elem, segments, iPos, iMirCode, sMirCodeText, allowedPositionIds;
    index = [];
    for (i = 0; i < ArrayCount(programIds); i++)
    {
        programId = programIds[i];
        segments = [];
        for (j = 0; j < ArrayCount(elementRows); j++)
        {
            elem = elementRows[j];
            if (OptInt(elem.education_method_id, 0) == Int(programId))
            {
                iPos = OptInt(elem.position_common_id, 0);
                iMirCode = OptInt(elem.mir_code_id, 0);
                allowedPositionIds = (iPos > 0 ? GetPositionIdsByCommonPosition(iPos) : []);
                sMirCodeText = ResolveMirCodeText(iMirCode);
                segments.push({ positionCommonId: iPos, allowedPositionIds: allowedPositionIds, mirCodeText: sMirCodeText });
            }
        }
        index.push({ programId: Int(programId), segments: segments });
    }
    return index;
}

function FindProgramAudienceSegments(audienceIndex, programId)
{
    var row;
    row = ArrayOptFind(audienceIndex, "Int(This.programId) == Int(programId)");
    return (row != undefined ? row.segments : []);
}

function CollaboratorInProgramAudience(collaboratorRow, segments, mirCodeRows)
{
    var i, seg, positionOk, mirCodeOk;
    for (i = 0; i < ArrayCount(segments); i++)
    {
        seg = segments[i];
        positionOk = (seg.positionCommonId <= 0 || IdArrayContains(seg.allowedPositionIds, OptInt(collaboratorRow.position_id, 0)));
        mirCodeOk = (seg.mirCodeText == "" || CollaboratorHasMirCode(mirCodeRows, Int(collaboratorRow.id), seg.mirCodeText));
        if (positionOk && mirCodeOk) { return true; }
    }
    return false;
}

/*
 * Применяет 4 ручных фильтра пользователя -- идентично HREDU-183_tep_reports.js.
 */
function ApplyManualFilters(collaboratorRows, iPositionFilter, sMacroregionFilter, sMirCodeFilter, macroRows)
{
    var allowedPositionIds, mirCodeRows, filteredRows, i, macroRow;

    filteredRows = collaboratorRows;

    if (iPositionFilter > 0)
    {
        allowedPositionIds = GetPositionIdsByCommonPosition(iPositionFilter);
        collaboratorRows = filteredRows;
        filteredRows = [];
        for (i = 0; i < ArrayCount(collaboratorRows); i++)
        {
            if (IdArrayContains(allowedPositionIds, OptInt(collaboratorRows[i].position_id, 0))) { filteredRows.push(collaboratorRows[i]); }
        }
    }

    if (sMacroregionFilter != "")
    {
        collaboratorRows = filteredRows;
        filteredRows = [];
        for (i = 0; i < ArrayCount(collaboratorRows); i++)
        {
            macroRow = ArrayOptFind(macroRows, "Int(This.id) == Int(collaboratorRows[i].id)");
            if (macroRow != undefined && String(macroRow.macroregion) == sMacroregionFilter) { filteredRows.push(collaboratorRows[i]); }
        }
    }

    if (sMirCodeFilter != "")
    {
        mirCodeRows = GetMirCodeRows();
        collaboratorRows = filteredRows;
        filteredRows = [];
        for (i = 0; i < ArrayCount(collaboratorRows); i++)
        {
            if (CollaboratorHasMirCode(mirCodeRows, Int(collaboratorRows[i].id), sMirCodeFilter)) { filteredRows.push(collaboratorRows[i]); }
        }
    }

    return filteredRows;
}

function GetCompletionDateRows(programIds)
{
    var sqlText;
    sqlText = "";
    sqlText = sqlText + "select ec.collaborator_id, e.education_method_id, min(ec.start_date) as first_date\r\n";
    sqlText = sqlText + "from event_collaborators ec\r\n";
    sqlText = sqlText + "join events e on e.id = ec.event_id\r\n";
    sqlText = sqlText + "where e.education_method_id in (" + ArrayMerge(programIds, "This", ",") + ")\r\n";
    sqlText = sqlText + "group by ec.collaborator_id, e.education_method_id";
    return ArraySelectAll(XQuery("sql:" + sqlText));
}

function FindCompletionDate(dateRows, collaboratorID, programID)
{
    var dateRow;
    dateRow = ArrayOptFind(dateRows, "Int(This.collaborator_id) == Int(collaboratorID) && Int(This.education_method_id) == Int(programID)");
    return (dateRow != undefined ? StrDate(Date(dateRow.first_date), false) : "");
}

/*
 * ИЗМЕНЕНО (16.09.2026): накопитель теперь по ПАРЕ (город, программа), а не только по
 * городу -- см. "ГРУППИРОВКА ПО (ГОРОД, ПРОГРАММА)" в шапке файла. Ищем циклом (как и
 * раньше) -- ArraySelect по строке-выражению не годится для ИЗМЕНЯЕМОГО накопителя.
 * @param {Object[]} acc
 * @param {string} sCity
 * @param {number} iProgramId
 * @param {string} sProgramName
 * @returns {Object}
 */
function GetOrCreateCityProgramAcc(acc, sCity, iProgramId, sProgramName)
{
    var i;
    for (i = 0; i < ArrayCount(acc); i++)
    {
        if (acc[i].city == sCity && Int(acc[i].programId) == Int(iProgramId)) { return acc[i]; }
    }
    var newAcc;
    newAcc = { city: sCity, programId: Int(iProgramId), programName: sProgramName, total: 0, mandatory: 0, fact: 0 };
    acc.push(newAcc);
    return newAcc;
}

/*
 * Округление факт/план в проценты (до целого, обычное арифметическое округление).
 * "-" если план = 0 (см. ДОПУЩЕНИЕ -- в реальных данных пока не встречалось, но
 * возможно в теории, если у города вся аудитория уже "выпала" -- на деле план всегда
 * = общее в этой версии, так что план=0 означает и общее=0, т.е. города вообще нет
 * в аудитории -- такая строка сюда не попадёт, см. ДОПУЩЕНИЕ №1).
 * @param {number} nFact
 * @param {number} nPlan
 * @returns {string}
 */
function FormatPercent(nFact, nPlan)
{
    if (nPlan <= 0) { return "-"; }
    return String(Int((nFact / nPlan) * 100 + 0.5)) + "%";
}

/*
 * ИСПРАВЛЕНИЕ (14.09.2026, БАГ С "&macroregion="): реальный тест показал, что итоговая
 * ссылка искажается ПРИ ОТОБРАЖЕНИИ виджетом "Табличные данные" -- "&macroregion="
 * превращалось в "%C2%AForegion=" (т.е. "&macr" пропадало, вместо него -- символ "¯",
 * U+00AF). Причина: "macr" -- это ИМЕННО ТАКОЕ имя у "легаси" HTML-сущности безточки
 * с запятой (как &amp, &lt, &nbsp) -- она означает символ "¯" (macron) и НЕ требует ";"
 * на конце. Судя по всему, виджет вставляет значение поля "link"/*_link ПРЯМО в атрибут
 * href как HTML-текст, без экранирования "&" в "&amp;" -- поэтому браузер видит в
 * "&macroregion=" сначала "&macr" (валидная сущность!) и стирает её, заменяя на "¯",
 * а не сам символ "&". Никакого отношения к UrlEncodeQuery()/percent-encoding это не
 * имеет -- проблема на уровне HTML, а не URL. Фикс: экранируем "&" САМИ в "&amp;" перед
 * тем, как класть готовую ссылку в поле RESULT -- тогда браузер сначала раскодирует
 * "&amp;" обратно в "&", и только ПОСЛЕ этого получившийся URL uже не содержит "&macr"
 * как отдельную подстроку для сущности. Без regex -- см. HtmlEscapeAmp() ниже, тот же
 * строковый API (StrOptSubStrPos/StrRangePos/StrLen), что и в GetQueryParam().
 * @param {string} sUrl
 * @returns {string}
 */
function HtmlEscapeAmp(sUrl)
{
    var sResult, iPos, iUrlLen, iSearchStart;
    sResult = "";
    iSearchStart = 0;
    iUrlLen = StrLen(sUrl);
    while (true)
    {
        iPos = StrOptSubStrPos(sUrl, "&", false, iSearchStart);
        if (iPos == undefined)
        {
            sResult = sResult + StrRangePos(sUrl, iSearchStart, iUrlLen);
            break;
        }
        sResult = sResult + StrRangePos(sUrl, iSearchStart, iPos) + "&amp;";
        iSearchStart = iPos + 1;
    }
    return sResult;
}

/*
 * Строит ссылку на страницу ТЭП-отчётов (HREDU-183_tep_reports.js) с нужным
 * набором параметров -- ровно те же параметры, что читает сама ТЭП-выборка
 * (см. GetQueryParam(...) в HREDU-183_tep_reports.js): matrix_id, macroregion,
 * mir_code, position_common_id, program_id, result_type + НОВЫЙ параметр city
 * (см. HREDU-183_tep_reports.js -- добавлен туда для этого дрилл-дауна).
 *
 * sCity = "" (пустая строка) -> ссылка ведёт на ВЕСЬ матрикс без фильтра по городу
 * (используется для строки "Общий итог").
 *
 * ИЗМЕНЕНО (16.09.2026): program_id теперь берётся ИЗ КОНКРЕТНОЙ СТРОКИ (её программа
 * матрицы), а не из ручного фильтра пользователя -- раз строка теперь и так соответствует
 * ровно одной программе (см. "ГРУППИРОВКА ПО (ГОРОД, ПРОГРАММА)"), логично, чтобы клик по
 * ней вёл в ТЭП-отчёт, УЖЕ отфильтрованный именно по этой программе. iProgramId=0 ->
 * без фильтра по программе (используется для строки "Общий итог").
 *
 * @param {number} iMatrixId
 * @param {string} sMacroregionFilter
 * @param {string} sMirCodeFilter
 * @param {number} iPositionFilter
 * @param {number} iProgramId
 * @param {string} sResultType   -   "total"|"plan"|"fact"|"mandatory"
 * @param {string} sCity
 * @returns {string}
 */
function BuildTepLink(iMatrixId, sMacroregionFilter, sMirCodeFilter, iPositionFilter, iProgramId, sResultType, sCity)
{
    var oQueryParams, sQueryString, sSeparator;
    oQueryParams = {
        matrix_id: String(iMatrixId),
        macroregion: sMacroregionFilter,
        mir_code: sMirCodeFilter,
        position_common_id: String(iPositionFilter),
        program_id: String(iProgramId),
        result_type: sResultType,
        city: sCity
    };
    sQueryString = UrlEncodeQuery(oQueryParams);
    sSeparator = (StrOptSubStrPos(TEP_REPORT_PAGE_URL, "?", false) != undefined ? "&" : "?");
    // HtmlEscapeAmp() -- см. комментарий над ней: "&" экранируем в "&amp;", потому что
    // виджет вставляет это значение прямо в HTML (href) без собственного экранирования.
    return HtmlEscapeAmp(TEP_REPORT_PAGE_URL + sSeparator + sQueryString);
}

//-------------------------------------------------------------------------
//              Точка входа
//-------------------------------------------------------------------------

function Run()
{
    LogAlert(2, "Run(). НАЧАЛО (Процент обученных)");
    var sFullUrl, matrixId, matrixDoc, matrixName, iProgramFilter, sMacroregionFilter, sMirCodeFilter, iPositionFilter;
    var matrixContext, elementRows, audienceIndex, segments;
    var programIds, filteredProgramIds, programNames, i, j;
    var activeRows, manualFilteredRows;
    var macroRows, mirCodeRows, cityRows, dateRows;
    var acc, cityProgramAcc, sCity, sProgramName, sDate, row;
    var totalAcc, resultRows, id;

    RESULT = [];

    try
    {
        sFullUrl = GetRequestUrlSafe();
        LogAlert(1, "Run(). Request.Url = [" + sFullUrl + "]");

        matrixId = OptInt(GetQueryParam(sFullUrl, "matrix_id"), 0);
        iProgramFilter = OptInt(GetQueryParam(sFullUrl, "program_id"), 0);
        sMacroregionFilter = GetQueryParam(sFullUrl, "macroregion");
        sMirCodeFilter = GetQueryParam(sFullUrl, "mir_code");
        iPositionFilter = OptInt(GetQueryParam(sFullUrl, "position_common_id"), 0);

        LogAlert(1, "Run(). matrixId=" + matrixId + " programFilter=" + iProgramFilter
            + " macroregionFilter=[" + sMacroregionFilter + "] mirCodeFilter=[" + sMirCodeFilter
            + "] positionCommonIdFilter=" + iPositionFilter);

        if (matrixId == 0)
        {
            throw ("Не передан matrix_id -- выбранная пользователем матрица обучения");
        }

        matrixDoc = tools.open_doc(matrixId).TopElem;
        matrixName = String(matrixDoc.name);

        // ИЗМЕНЕНО (17.09.2026, HREDU-215 "Правки 1"): ResolveProgramIds() переименована в
        // ResolveMatrixContext() -- возвращает ещё и elementRows, нужны для аудитории ПО
        // ЭЛЕМЕНТАМ (BuildProgramAudienceIndex() ниже).
        matrixContext = ResolveMatrixContext(matrixId, matrixName);
        programIds = matrixContext.programIds;
        elementRows = matrixContext.elementRows;

        if (iProgramFilter > 0)
        {
            filteredProgramIds = [];
            for (i = 0; i < ArrayCount(programIds); i++)
            {
                if (Int(programIds[i]) == iProgramFilter) { filteredProgramIds.push(programIds[i]); }
            }
            programIds = filteredProgramIds;
            if (ArrayCount(programIds) == 0)
            {
                throw ("Программа [" + iProgramFilter + "] не найдена среди программ выбранной матрицы");
            }
        }

        activeRows = GetActiveCollaboratorRows();
        macroRows = GetMacroregionRows();
        cityRows = GetCityRows();
        dateRows = GetCompletionDateRows(programIds);
        // ИЗМЕНЕНО (17.09.2026): mirCodeRows раньше грузился ЛЕНИВО (либо внутри
        // GetMatrixAudienceCollaboratorRows(), убрана, либо внутри ApplyManualFilters()
        // при ручном фильтре по мир-коду). Теперь нужен ВСЕГДА -- для аудитории КАЖДОЙ
        // программы (CollaboratorInProgramAudience() в цикле total/mandatory ниже).
        mirCodeRows = GetMirCodeRows();

        // НОВОЕ (16.09.2026): имена программ резолвим ОДИН РАЗ на все уникальные
        // programIds (не по разу на каждого сотрудника x программу -- иначе
        // tools.open_doc() дёргался бы многие тысячи раз, см. FindProgramName()).
        programNames = [];
        for (j = 0; j < ArrayCount(programIds); j++)
        {
            programNames.push({ id: Int(programIds[j]), name: ResolveProgramText(programIds[j]) });
        }

        // ДОБАВЛЕНО (17.09.2026, HREDU-215 "Правки 1"): индекс аудитории ПО ПРОГРАММАМ
        // (была одна аудитория на всю матрицу -- GetMatrixAudienceCollaboratorRows(),
        // убрана; теперь своя у каждого элемента/программы, см. комментарий над
        // BuildProgramAudienceIndex() выше).
        audienceIndex = BuildProgramAudienceIndex(elementRows, programIds);

        // ИЗМЕНЕНО (17.09.2026): раньше здесь было ДВА отдельных пула -- "аудитория
        // матрицы + ручные фильтры" (для total/plan/mandatory) и "без аудитории, только
        // ручные фильтры" (для fact). Теперь аудитория не пул-фильтр, а проверка ПО
        // КАЖДОЙ ПАРЕ (сотрудник x программа) внутри цикла ниже -- значит пул для
        // total/mandatory и пул для fact СОВПАДАЮТ (оба -- "активные + ручные фильтры"),
        // достаточно посчитать один раз.
        manualFilteredRows = ApplyManualFilters(activeRows, iPositionFilter, sMacroregionFilter, sMirCodeFilter, macroRows);
        LogAlert(1, "Run(). Сотрудников после ручных фильтров (база и для total/mandatory, и для fact): " + ArrayCount(manualFilteredRows));

        acc = [];

        // total/mandatory -- по каждому (сотрудник x программа), сгруппировано по ПАРЕ
        // (город, программа) -- см. "ГРУППИРОВКА ПО (ГОРОД, ПРОГРАММА)" в шапке. НО
        // теперь, В ОТЛИЧИЕ от 16.09.2026, засчитываем сотрудника в total/mandatory
        // программы, ТОЛЬКО ЕСЛИ он входит в АУДИТОРИЮ ИМЕННО ЭТОЙ программы (см.
        // "ИЗМЕНЕНО (17.09.2026)" выше -- аудитория теперь своя у каждой программы).
        for (i = 0; i < ArrayCount(manualFilteredRows); i++)
        {
            sCity = FindCity(cityRows, Int(manualFilteredRows[i].id));
            for (j = 0; j < ArrayCount(programIds); j++)
            {
                segments = FindProgramAudienceSegments(audienceIndex, programIds[j]);
                if (CollaboratorInProgramAudience(manualFilteredRows[i], segments, mirCodeRows))
                {
                    sProgramName = FindProgramName(programNames, programIds[j]);
                    cityProgramAcc = GetOrCreateCityProgramAcc(acc, sCity, programIds[j], sProgramName);
                    sDate = FindCompletionDate(dateRows, Int(manualFilteredRows[i].id), programIds[j]);
                    cityProgramAcc.total = cityProgramAcc.total + 1;
                    if (sDate == "") { cityProgramAcc.mandatory = cityProgramAcc.mandatory + 1; }
                }
            }
        }

        // fact -- по каждому (сотрудник x программа), только ПРОЙДЕННЫЕ, БЕЗ ограничения
        // аудиторией (см. ТЗ п.3 "не зависимо от условий матрицы" -- см. шапку файла).
        // ДОПУЩЕНИЕ №1 (см. шапку файла, изменено 16.09.2026): накопитель (город,
        // программа) создаём здесь ТОЛЬКО когда реально есть завершение (sDate != "") --
        // иначе при большой факт-базе (тысячи активных сотрудников x несколько программ)
        // получился бы взрыв пустых строк "город x программа x 0 x 0", которых никто не
        // хочет видеть. Пара (город, программа), где есть только факт без аудитории,
        // всё равно получает свою строку (Общее=0), просто не для КАЖДОЙ комбинации, а
        // только там, где реально кто-то прошёл.
        for (i = 0; i < ArrayCount(manualFilteredRows); i++)
        {
            sCity = FindCity(cityRows, Int(manualFilteredRows[i].id));
            for (j = 0; j < ArrayCount(programIds); j++)
            {
                sDate = FindCompletionDate(dateRows, Int(manualFilteredRows[i].id), programIds[j]);
                if (sDate != "")
                {
                    sProgramName = FindProgramName(programNames, programIds[j]);
                    cityProgramAcc = GetOrCreateCityProgramAcc(acc, sCity, programIds[j], sProgramName);
                    cityProgramAcc.fact = cityProgramAcc.fact + 1;
                }
            }
        }

        // ИЗМЕНЕНО (16.09.2026, по запросу пользователя): сортировка теперь СНАЧАЛА по
        // названию программы, ПОТОМ по городу внутри неё -- то есть все города одной
        // программы идут подряд одним блоком, а следующая программа начинается только
        // после того, как закончился блок предыдущей (а не вперемешку город-за-городом,
        // как было раньше). ОБЫЧНЫМ ЦИКЛОМ (пузырьком), а не через возможную функцию-
        // хелпер вроде ArraySort(): такая функция НИ РАЗУ не встречалась и не
        // подтверждалась в этом тикете (в отличие от ArraySelectDistinct/ArrayExtract/
        // ArrayMerge и т.д.), а гадать с непроверенными функциями платформы уже дорого
        // обходилось (regex, function-as-value, .indexOf/.substring -- см. историю
        // тикета) -- поэтому используем только то, что 100% работает: простые циклы
        // и операторы сравнения.
        var iOuter, iInner, tmpAcc;
        for (iOuter = 0; iOuter < ArrayCount(acc) - 1; iOuter++)
        {
            for (iInner = 0; iInner < ArrayCount(acc) - 1 - iOuter; iInner++)
            {
                if (acc[iInner].programName > acc[iInner + 1].programName
                    || (acc[iInner].programName == acc[iInner + 1].programName && acc[iInner].city > acc[iInner + 1].city))
                {
                    tmpAcc = acc[iInner];
                    acc[iInner] = acc[iInner + 1];
                    acc[iInner + 1] = tmpAcc;
                }
            }
        }

        resultRows = [];
        id = 0;
        totalAcc = { total: 0, mandatory: 0, fact: 0 };
        for (i = 0; i < ArrayCount(acc); i++)
        {
            id = id + 1;
            row = acc[i];
            resultRows.push({
                id: id,
                city: row.city,
                program: row.programName,
                total: row.total,
                plan: row.total, // План = Общее, см. "РЕШЕНИЯ" в шапке
                fact: row.fact,
                percent: FormatPercent(row.fact, row.total),
                mandatory: row.mandatory,
                // ПОДТВЕРЖДЕНО (14.09.2026, реальный тест пользователя): виджет "Табличные
                // данные" различает клик ТОЛЬКО по строке целиком -- один "link" на всю
                // строку. ИЗМЕНЕНО (16.09.2026): program_id в ссылке теперь берётся из
                // КОНКРЕТНОЙ строки (row.programId), а не из общего фильтра -- см.
                // BuildTepLink(). Режим по-прежнему фиксирован на "total"; план/факт/
                // обязательно пользователь смотрит либо прямо в этой таблице, либо
                // переключает "Режим отчёта" вручную на целевой странице.
                link: BuildTepLink(matrixId, sMacroregionFilter, sMirCodeFilter, iPositionFilter, row.programId, "total", row.city)
            });
            totalAcc.total = totalAcc.total + row.total;
            totalAcc.mandatory = totalAcc.mandatory + row.mandatory;
            totalAcc.fact = totalAcc.fact + row.fact;
        }

        id = id + 1;
        // sCity = "" и iProgramId = 0 для "Общий итог" -- ссылка ведёт на ВЕСЬ матрикс
        // (без фильтра по городу и без фильтра по программе).
        resultRows.push({
            id: id,
            city: "Общий итог",
            program: "-",
            total: totalAcc.total,
            plan: totalAcc.total,
            fact: totalAcc.fact,
            percent: FormatPercent(totalAcc.fact, totalAcc.total),
            mandatory: totalAcc.mandatory,
            link: BuildTepLink(matrixId, sMacroregionFilter, sMirCodeFilter, iPositionFilter, 0, "total", "")
        });

        RESULT = resultRows;
        LogAlert(2, "Run(). Готово. Строк (город x программа): " + (ArrayCount(resultRows) - 1) + " + итоговая строка");
    }
    catch (_ex)
    {
        RESULT = [];
        ERROR = 1;
        MESSAGE = ExtractUserError(_ex);
        LogAlert(4, "Run(). ОШИБКА: " + MESSAGE);
    }
    LogAlert(2, "Run(). КОНЕЦ");
}

Run();

// ЗАКРЫТО (14.09.2026, КЛИКАБЕЛЬНОСТЬ): реальный тест пользователя подтвердил, что
// виджет "Табличные данные" различает клик ТОЛЬКО по строке целиком -- отдельной ссылки
// "на конкретную ячейку/число" у него нет (независимо от того, по какой колонке
// кликнули, срабатывает один и тот же "link" всей строки). Поэтому четыре поля
// total_link/plan_link/fact_link/mandatory_link и соответствующие им закомментированные
// варианты колонок -- УБРАНЫ как мёртвый код (см. историю тикета -- раньше они были
// здесь как непроверенная гипотеза). РЕШЕНИЕ: один клик по строке города -> ТЭП-отчёт в
// режиме "total"; план/факт/обязательно пользователь смотрит НЕ переходом по клику, а
// либо прямо в этой таблице (числа уже видны), либо переключает "Режим отчёта" вручную
// в фильтрах на целевой странице (см. HREDU-183_filtry_modal_shag1.js).
// ДОБАВЛЕНО (16.09.2026, по запросу пользователя): колонка "Учебная программа" сразу
// после "Город" -- строка теперь соответствует паре (город, программа), а не только
// городу, см. "ГРУППИРОВКА ПО (ГОРОД, ПРОГРАММА)" в шапке файла.
COLUMNS = [
    { "data": "id", "editable": true, "hidden": true, "sortable": false },
    { "data": "link", "hidden": true, "editable": false, "sortable": false }, // проверенный row-level link
    { "data": "city", "title": "Город", "type": "string", "editable": false, "sortable": true },
    { "data": "program", "title": "Учебная программа", "type": "string", "editable": false, "sortable": true },
    { "data": "total", "title": "Общее кол-во сотрудников", "type": "integer", "editable": false, "sortable": true },
    { "data": "plan", "title": "План", "type": "integer", "editable": false, "sortable": true },
    { "data": "fact", "title": "Факт", "type": "integer", "editable": false, "sortable": true },
    { "data": "percent", "title": "Процент", "type": "string", "editable": false, "sortable": false },
    { "data": "mandatory", "title": "Обязательно к прохождению", "type": "integer", "editable": false, "sortable": true }
];
