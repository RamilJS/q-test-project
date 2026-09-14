// =====================================================================
// HREDU-182. Диагностика: "сырой" дамп аудитории/фактической базы для отчёта
// "Процент обученных" -- НЕ арифметика (её мы уже сверили в
// HREDU-182_diagnostic_procent_crosscheck.js, все тождества были "OK"), а ГЛАЗАМИ
// проверяемые списки людей, чтобы найти РАСХОЖДЕНИЕ С РЕАЛЬНОСТЬЮ (пользователь
// 14.09.2026: "мне кажется, что данные не совпадают с реальными данными").
//
// ПОЧЕМУ арифметика "OK", а данные всё равно могут быть неправильными: crosscheck
// проверяет, что ЧИСЛА ВНУТРИ СЕБЯ согласованы (сумма по городам = целиком по матрице,
// total = mandatory + пройдено-в-аудитории и т.д.) -- это гарантирует, что КОД считает
// без внутренних противоречий, но НЕ гарантирует, что сама БИЗНЕС-ЛОГИКА (кто входит в
// "аудиторию матрицы") соответствует реальности. Судя по двум присланным прогонам:
//   -- матрица 1 (Матрица ТЭП, фильтры macroregion=Москва + mir_code=LASM):
//      аудитория = 0 человек, а Факт = 82 -- то есть НИКТО формально не входит в
//      аудиторию этой матрицы по её собственным критериям (position_common_id/
//      mir_code_id матрицы), хотя 82 человека программу уже прошли.
//   -- матрица 2 (Матрица тест 3, без ручных фильтров): аудитория = 44 человека, и
//      ВСЕ 44 -- в Москве (0 в любом другом городе), хотя Факт разбросан по ~70 городам.
// Это МОЖЕТ быть правдой (аудитория матрицы правда узкая и московская), а МОЖЕТ быть
// признаком того, что аудитория матрицы вообще не должна определяться ТОЛЬКО через
// position_common_id/mir_code_id САМОЙ матрицы (см. GetMatrixAudienceCollaboratorRows()
// в HREDU-182_procent_obuchennyh.js) -- этот скрипт даёт СПИСКИ, а не числа, чтобы можно
// было сверить с тем, что ты реально знаешь про этих людей/матрицу.
//
// ЧТО ПОКАЗЫВАЕТ:
//   1. Сырые аудиторные критерии САМОЙ матрицы (position_common_id/mir_code_id) --
//      резолвлены в текст, чтобы можно было прочитать глазами, что это за роль/мир-код.
//   2. Размер аудитории БЕЗ ручных фильтров (macroregion/mir_code/position/program) --
//      сколько человек вообще подходит под критерии матрицы, до применения твоих
//      фильтров из URL. Если это тоже 0 или тоже "все в одном городе" -- значит дело в
//      самой матрице/критериях, а не в твоих ручных фильтрах.
//   3. Список (до 200 строк) сотрудников из АУДИТОРИИ (с ручными фильтрами, т.е. та
//      же аудитория, что фактически участвует в отчёте) -- id + город + типовая
//      должность (текст) + мир-коды (текст).
//   4. Список (до 200 строк) сотрудников из ФАКТ-базы (прошли обучение, БЕЗ
//      ограничения аудиторией) -- id + город + дата прохождения -- чтобы сверить,
//      реальны ли эти 82/407 "прошедших".
//
// КАК ЗАПУСТИТЬ: как обычная выборка "Табличные данные", на странице с уже
// применёнными фильтрами (matrix_id обязателен в URL).
// =====================================================================

DEBUG = true;
LOG_NAME = "agent";
CUR_OBJECT_ID = 0; // TODO: заполнить после создания документа в админке

function LogAlert(typeLog, message)
{
    try
    {
        tools.call_code_library_method("vtbl_log_lib", "LogAlert", [LOG_NAME, typeLog, CUR_OBJECT_ID, message, DEBUG]);
    }
    catch (_exLog)
    {
        // ничего
    }
}

//-------------------------------------------------------------------------
//              Продублированные функции (см. HREDU-182_procent_obuchennyh.js)
//-------------------------------------------------------------------------

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

function GetProgramIds(matrixRows, elementRows)
{
    var matrixProgramIds, elementProgramIds, allProgramIds, i;
    matrixProgramIds = ArrayExtract(matrixRows, "Int(This.education_method_id)");
    elementProgramIds = ArrayExtract(elementRows, "Int(This.education_method_id)");
    allProgramIds = [];
    for (i = 0; i < ArrayCount(matrixProgramIds); i++) { allProgramIds.push(matrixProgramIds[i]); }
    for (i = 0; i < ArrayCount(elementProgramIds); i++) { allProgramIds.push(elementProgramIds[i]); }
    return ArraySelectDistinct(allProgramIds, "This");
}

function ResolveProgramIds(matrixId, matrixName)
{
    var matrixRows, matrixIds, elementRows, programIds;
    matrixRows = GetMatrixRows(matrixName);
    matrixIds = ArrayExtract(matrixRows, "Int(This.id)");
    if (ArrayCount(matrixIds) == 0)
    {
        throw ("Не найдено ни одной записи cc_learning_matrice с названием [" + matrixName + "]");
    }
    elementRows = GetMatrixElementRows(matrixIds);
    programIds = GetProgramIds(matrixRows, elementRows);
    if (ArrayCount(programIds) == 0)
    {
        throw ("У матрицы [" + matrixName + "] не найдено ни одной активной программы");
    }
    return programIds;
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

function GetCityRows()
{
    var sqlText;
    sqlText = "";
    sqlText = sqlText + "select cs.id,\r\n";
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''sity'']/value)[1]', 'varchar(max)') as sity\r\n";
    sqlText = sqlText + "from collaborators cs\r\n";
    sqlText = sqlText + "inner join collaborator c on c.id = cs.id\r\n";
    sqlText = sqlText + "where cs.is_dismiss != 1";
    return ArraySelectAll(XQuery("sql:" + sqlText));
}

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

/*
 * Все мир-коды сотрудника КАК ТЕКСТ, через "|" -- для дампа (в отличие от
 * CollaboratorHasMirCode, который проверяет только ОДИН конкретный код).
 */
function GetMirCodesText(mirCodeRows, collaboratorID)
{
    var row, codes;
    row = ArrayOptFind(mirCodeRows, "Int(This.id) == Int(collaboratorID)");
    if (row == undefined) { return ""; }
    codes = ExtractMirCodes(row.mir_codes);
    return ArrayMerge(codes, "This", "|");
}

function ResolveMirCodeText(iMirCodeID)
{
    if (OptInt(iMirCodeID, 0) <= 0) { return ""; }
    try { return String(tools.open_doc(Int(iMirCodeID)).TopElem.name); }
    catch (_ex) { return ""; }
}

/*
 * НОВОЕ (эта диагностика): резолвит ID типовой должности (position_common) в текст --
 * та же схема, что ResolveMirCodeText(), только другой каталог.
 * @param {number} iPositionCommonID
 * @returns {string}
 */
function ResolvePositionCommonText(iPositionCommonID)
{
    if (OptInt(iPositionCommonID, 0) <= 0) { return "(не задано)"; }
    try { return String(tools.open_doc(Int(iPositionCommonID)).TopElem.name) + " (id=" + iPositionCommonID + ")"; }
    catch (_ex) { return "ОШИБКА резолва id=" + iPositionCommonID + ": " + ExtractUserError(_ex); }
}

function GetMatrixAudienceCollaboratorRows(matrixDoc, collaboratorRows)
{
    var iAudiencePositionCommonId, iAudienceMirCodeId, sAudienceMirCodeText;
    var allowedPositionIds, mirCodeRows, filteredRows, i;

    iAudiencePositionCommonId = OptInt(matrixDoc.position_common_id, 0);
    iAudienceMirCodeId = OptInt(matrixDoc.mir_code_id, 0);
    sAudienceMirCodeText = ResolveMirCodeText(iAudienceMirCodeId);

    filteredRows = collaboratorRows;

    if (iAudiencePositionCommonId > 0)
    {
        allowedPositionIds = GetPositionIdsByCommonPosition(iAudiencePositionCommonId);
        collaboratorRows = filteredRows;
        filteredRows = [];
        for (i = 0; i < ArrayCount(collaboratorRows); i++)
        {
            if (IdArrayContains(allowedPositionIds, OptInt(collaboratorRows[i].position_id, 0))) { filteredRows.push(collaboratorRows[i]); }
        }
    }

    if (sAudienceMirCodeText != "")
    {
        mirCodeRows = GetMirCodeRows();
        collaboratorRows = filteredRows;
        filteredRows = [];
        for (i = 0; i < ArrayCount(collaboratorRows); i++)
        {
            if (CollaboratorHasMirCode(mirCodeRows, Int(collaboratorRows[i].id), sAudienceMirCodeText)) { filteredRows.push(collaboratorRows[i]); }
        }
    }

    return filteredRows;
}

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

var MAX_DUMP_ROWS = 200;

//-------------------------------------------------------------------------
//              Точка входа
//-------------------------------------------------------------------------

function Run()
{
    LogAlert(2, "Run(). НАЧАЛО (диагностика -- сырой дамп Процент обученных)");
    var sFullUrl, matrixId, matrixDoc, matrixName, iProgramFilter, sMacroregionFilter, sMirCodeFilter, iPositionFilter;
    var programIds, filteredProgramIds, i, j;
    var activeRows, audienceRowsNoManualFilters, audienceFilteredRows, factBaseFilteredRows;
    var macroRows, cityRows, mirCodeRows, dateRows;
    var iAudiencePositionCommonId, iAudienceMirCodeId, sAudienceMirCodeText, sAudiencePositionText;
    var resultRows, id, row, sCity, sHasCompletedAny, sDate, nProgramsCompleted;

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

        if (matrixId == 0)
        {
            throw ("Не передан matrix_id -- открой эту диагностику на странице, где в URL уже есть фильтры");
        }

        matrixDoc = tools.open_doc(matrixId).TopElem;
        matrixName = String(matrixDoc.name);

        programIds = ResolveProgramIds(matrixId, matrixName);
        if (iProgramFilter > 0)
        {
            filteredProgramIds = [];
            for (i = 0; i < ArrayCount(programIds); i++)
            {
                if (Int(programIds[i]) == iProgramFilter) { filteredProgramIds.push(programIds[i]); }
            }
            programIds = filteredProgramIds;
        }

        activeRows = GetActiveCollaboratorRows();
        macroRows = GetMacroregionRows();
        cityRows = GetCityRows();
        mirCodeRows = GetMirCodeRows();
        dateRows = GetCompletionDateRows(programIds);

        iAudiencePositionCommonId = OptInt(matrixDoc.position_common_id, 0);
        iAudienceMirCodeId = OptInt(matrixDoc.mir_code_id, 0);
        sAudienceMirCodeText = ResolveMirCodeText(iAudienceMirCodeId);
        sAudiencePositionText = ResolvePositionCommonText(iAudiencePositionCommonId);

        // Аудитория БЕЗ ручных фильтров пользователя -- ТОЛЬКО критерии самой матрицы.
        audienceRowsNoManualFilters = GetMatrixAudienceCollaboratorRows(matrixDoc, activeRows);

        // Аудитория С ручными фильтрами -- то, что реально используется в самом отчёте.
        audienceFilteredRows = ApplyManualFilters(audienceRowsNoManualFilters, iPositionFilter, sMacroregionFilter, sMirCodeFilter, macroRows);

        // База для Факт -- без ограничения аудиторией матрицы, только ручные фильтры.
        factBaseFilteredRows = ApplyManualFilters(activeRows, iPositionFilter, sMacroregionFilter, sMirCodeFilter, macroRows);

        resultRows = [];
        id = 0;

        id = id + 1;
        resultRows.push({ id: id, section: "ШАПКА", field: "matrix_id / название матрицы", value: matrixId + " / " + matrixName });
        id = id + 1;
        resultRows.push({ id: id, section: "ШАПКА", field: "Программ в матрице (после фильтра program_id)", value: String(ArrayCount(programIds)) });
        id = id + 1;
        resultRows.push({ id: id, section: "ШАПКА", field: "Ручные фильтры (macroregion/mir_code/position_common_id/program_id)",
            value: "[" + sMacroregionFilter + "] / [" + sMirCodeFilter + "] / " + iPositionFilter + " / " + iProgramFilter });

        id = id + 1;
        resultRows.push({ id: id, section: "1. АУДИТОРНЫЕ КРИТЕРИИ САМОЙ МАТРИЦЫ (сырые поля matrixDoc)", field: "", value: "" });
        id = id + 1;
        resultRows.push({ id: id, section: "1. АУДИТОРНЫЕ КРИТЕРИИ", field: "matrixDoc.position_common_id (типовая должность аудитории)", value: sAudiencePositionText });
        id = id + 1;
        resultRows.push({ id: id, section: "1. АУДИТОРНЫЕ КРИТЕРИИ", field: "matrixDoc.mir_code_id (мир-код аудитории)", value: "id=" + iAudienceMirCodeId + ", текст=[" + sAudienceMirCodeText + "]" });
        id = id + 1;
        resultRows.push({ id: id, section: "1. АУДИТОРНЫЕ КРИТЕРИИ", field: "Аудитория БЕЗ ручных фильтров (только критерии матрицы выше)", value: String(ArrayCount(audienceRowsNoManualFilters)) + " человек" });
        id = id + 1;
        resultRows.push({ id: id, section: "1. АУДИТОРНЫЕ КРИТЕРИИ", field: "Аудитория С ручными фильтрами (то, что реально в отчёте)", value: String(ArrayCount(audienceFilteredRows)) + " человек" });
        id = id + 1;
        resultRows.push({ id: id, section: "1. АУДИТОРНЫЕ КРИТЕРИИ", field: "База для Факт (без аудитории, только ручные фильтры)", value: String(ArrayCount(factBaseFilteredRows)) + " человек" });

        id = id + 1;
        resultRows.push({ id: id, section: "2. СПИСОК АУДИТОРИИ (та, что в отчёте) -- id/город/должность/мир-коды -- до " + MAX_DUMP_ROWS + " строк", field: "", value: "" });
        for (i = 0; i < ArrayCount(audienceFilteredRows) && i < MAX_DUMP_ROWS; i++)
        {
            row = audienceFilteredRows[i];
            sCity = FindCity(cityRows, Int(row.id));
            nProgramsCompleted = 0;
            for (j = 0; j < ArrayCount(programIds); j++)
            {
                sDate = FindCompletionDate(dateRows, Int(row.id), programIds[j]);
                if (sDate != "") { nProgramsCompleted = nProgramsCompleted + 1; }
            }
            id = id + 1;
            resultRows.push({
                id: id,
                section: "2. АУДИТОРИЯ",
                field: "id=" + row.id + ", город=[" + sCity + "]",
                value: "мир-коды=[" + GetMirCodesText(mirCodeRows, Int(row.id)) + "], пройдено программ (из " + ArrayCount(programIds) + "): " + nProgramsCompleted
            });
        }
        if (ArrayCount(audienceFilteredRows) > MAX_DUMP_ROWS)
        {
            id = id + 1;
            resultRows.push({ id: id, section: "2. АУДИТОРИЯ", field: "-- обрезано, показаны первые " + MAX_DUMP_ROWS + " из " + ArrayCount(audienceFilteredRows) + " --", value: "" });
        }

        id = id + 1;
        resultRows.push({ id: id, section: "3. СПИСОК ПРОШЕДШИХ (Факт-база, есть хотя бы 1 завершённая программа) -- до " + MAX_DUMP_ROWS + " строк", field: "", value: "" });
        j = 0;
        for (i = 0; i < ArrayCount(factBaseFilteredRows) && j < MAX_DUMP_ROWS; i++)
        {
            row = factBaseFilteredRows[i];
            nProgramsCompleted = 0;
            var k;
            for (k = 0; k < ArrayCount(programIds); k++)
            {
                sDate = FindCompletionDate(dateRows, Int(row.id), programIds[k]);
                if (sDate != "") { nProgramsCompleted = nProgramsCompleted + 1; }
            }
            if (nProgramsCompleted > 0)
            {
                sCity = FindCity(cityRows, Int(row.id));
                id = id + 1;
                resultRows.push({
                    id: id,
                    section: "3. ФАКТ",
                    field: "id=" + row.id + ", город=[" + sCity + "]",
                    value: "пройдено программ (из " + ArrayCount(programIds) + "): " + nProgramsCompleted
                });
                j = j + 1;
            }
        }

        RESULT = resultRows;
        LogAlert(2, "Run(). Готово, строк в результате: " + ArrayCount(RESULT));
    }
    catch (_ex)
    {
        RESULT = [{ id: 0, section: "ОШИБКА", field: "ОШИБКА ВЕРХНЕГО УРОВНЯ", value: ExtractUserError(_ex) }];
        LogAlert(4, "Run(). ОШИБКА: " + ExtractUserError(_ex));
    }
    LogAlert(2, "Run(). КОНЕЦ");
}

Run();

COLUMNS = [
    { "data": "id", "editable": true, "hidden": true, "sortable": false },
    { "data": "section", "title": "Раздел", "type": "string", "editable": false, "sortable": false, "multiline": true, "width": "35%" },
    { "data": "field", "title": "Поле", "type": "string", "editable": false, "sortable": false, "multiline": true, "width": "30%" },
    { "data": "value", "title": "Значение", "type": "string", "editable": false, "sortable": false, "multiline": true, "width": "35%" }
];
