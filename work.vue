// =====================================================================
// HREDU-182. Диагностика: сверка арифметики отчёта "Процент обученных"
// (HREDU-182_procent_obuchennyh.js) для ОДНОЙ И ТОЙ ЖЕ матрицы+фильтров.
//
// ЗАЧЕМ (запрошено пользователем 14.09.2026, "не уверен, что данные которые строит
// отчёт корректны"): по аналогии с уже проверенной техникой для ТЭП-отчётов
// (HREDU-183_diagnostic_tep_crosscheck.js, все 3 тождества дали "OK" на реальных
// данных 44/44/98/8) -- строим ДВА НЕЗАВИСИМЫХ расчёта в одном скрипте и сверяем их,
// вместо того чтобы гадать глазами по таблице:
//
//   Расчёт А (ПО ГОРОДАМ) -- 1:1 копия логики HREDU-182_procent_obuchennyh.js: два
//   цикла (аудитория -> total/mandatory, без-аудитории -> fact), группировка по городу
//   через FindCity()/GetOrCreateCityAcc(), + добавлено ОДНО новое накопление --
//   completed_in_audience ПО ГОРОДУ (сколько из аудитории этого города уже прошли) --
//   этого числа нет в самом отчёте, оно нужно только для тождества здесь.
//
//   Расчёт Б (ЦЕЛИКОМ ПО МАТРИЦЕ, БЕЗ городов) -- та же логика, что уже подтверждена в
//   HREDU-183_diagnostic_tep_crosscheck.js (CountPairs() по всей аудитории/факт-базе
//   сразу, без группировки) -- считает total/mandatory/fact/completed_in_audience
//   ОДНИМ проходом, никак не завязанным на понятие "город".
//
// ПРОВЕРКИ:
//   1) ИТОГОВАЯ СВЕРКА: сумма по городам (расчёт А) должна СОВПАСТЬ с числом по всей
//      матрице (расчёт Б) для каждой из 4 величин (total/mandatory/fact/
//      completed_in_audience). Если совпало -- разбивка по городам не теряет и не
//      дублирует людей относительно уже провalidated расчёта из HREDU-183.
//   2) ПОГОРОДНЫЕ ТОЖДЕСТВА (те же 2 из 3, что в HREDU-183 -- "план==total" тут не
//      нужно повторять для каждого города, это тривиально по построению):
//        а) total(город) == mandatory(город) + completed_in_audience(город)
//        б) fact(город) >= completed_in_audience(город)
//   3) ПРОВЕРКА НА ДУБЛИ: XQuery иногда может вернуть один и тот же collaborator
//      несколько раз (например при неаккуратном join) -- считаем count(distinct id) и
//      сверяем с обычным count(). Несовпадение -- сигнал задвоения исходных данных
//      (не ошибка бизнес-логики, а проблема данных/запроса).
//
// Если ВСЁ "OK" -- это сильное свидетельство, что арифметика разбивки по городам верна
// (сама бизнес-логика total/plan/fact/mandatory/percent уже была отдельно подтверждена
// сверкой с реальным примером таблицы -- см. шапку HREDU-182_procent_obuchennyh.js).
//
// КАК ЗАПУСТИТЬ: как обычная выборка "Табличные данные", НА СТРАНИЦЕ, где в URL уже
// есть matrix_id и остальные фильтры (после "Применить" в HREDU-183_filtry_modal_shag1.js
// или HREDU_182_filtry_percent.js -- любая из двух модалок подойдёт, читаются только
// 5 общих фильтров, result_type не используется).
// =====================================================================

DEBUG = true;
LOG_NAME = "agent";
CUR_OBJECT_ID = 0; // TODO: заполнить после создания документа в админке (LogAlert защищена try/catch)

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

//-------------------------------------------------------------------------
//              Продублированные функции (см. HREDU-182_procent_obuchennyh.js /
//              HREDU-183_diagnostic_tep_crosscheck.js -- без повторной документации)
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

function ResolveMirCodeText(iMirCodeID)
{
    if (OptInt(iMirCodeID, 0) <= 0) { return ""; }
    try { return String(tools.open_doc(Int(iMirCodeID)).TopElem.name); }
    catch (_ex) { return ""; }
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

/*
 * Считает кол-во пар "сотрудник x программа" и, отдельно, кол-во таких пар с уже
 * заполненной датой прохождения -- РАСЧЁТ Б (целиком, без городов). Идентична функции
 * из HREDU-183_diagnostic_tep_crosscheck.js.
 * @returns {Object}   -   { totalPairs, completedPairs }
 */
function CountPairs(collaboratorRows, programIds, dateRows)
{
    var i, j, totalPairs, completedPairs, sDate;
    totalPairs = 0;
    completedPairs = 0;
    for (i = 0; i < ArrayCount(collaboratorRows); i++)
    {
        for (j = 0; j < ArrayCount(programIds); j++)
        {
            totalPairs = totalPairs + 1;
            sDate = FindCompletionDate(dateRows, Int(collaboratorRows[i].id), programIds[j]);
            if (sDate != "") { completedPairs = completedPairs + 1; }
        }
    }
    return { totalPairs: totalPairs, completedPairs: completedPairs };
}

/*
 * РАСЧЁТ А (по городам) -- находит/создаёт накопитель города; добавлено новое поле
 * completedInAudience (нет в самом отчёте HREDU-182_procent_obuchennyh.js -- нужно
 * только здесь, для тождества "total == mandatory + completedInAudience" по городу).
 */
function GetOrCreateCityAcc(acc, sCity)
{
    var i;
    for (i = 0; i < ArrayCount(acc); i++)
    {
        if (acc[i].city == sCity) { return acc[i]; }
    }
    var newAcc;
    newAcc = { city: sCity, total: 0, mandatory: 0, fact: 0, completedInAudience: 0 };
    acc.push(newAcc);
    return newAcc;
}

/*
 * Кол-во РАЗЛИЧНЫХ (distinct) id в массиве строк-сотрудников -- для проверки на дубли
 * (см. "ПРОВЕРКА НА ДУБЛИ" в шапке файла). Без regex/непроверенных функций -- обычный
 * цикл + поиск в уже накопленном списке (тот же приём, что и GetOrCreateCityAcc).
 * @param {Object[]} collaboratorRows
 * @returns {number}
 */
function CountDistinctIds(collaboratorRows)
{
    var seenIds, i, id;
    seenIds = [];
    for (i = 0; i < ArrayCount(collaboratorRows); i++)
    {
        id = Int(collaboratorRows[i].id);
        if (!IdArrayContains(seenIds, id)) { seenIds.push(id); }
    }
    return ArrayCount(seenIds);
}

//-------------------------------------------------------------------------
//              Точка входа
//-------------------------------------------------------------------------

function Run()
{
    LogAlert(2, "Run(). НАЧАЛО (диагностика сверки Процент обученных)");
    var sFullUrl, matrixId, matrixDoc, matrixName, iProgramFilter, sMacroregionFilter, sMirCodeFilter, iPositionFilter;
    var programIds, filteredProgramIds, i, j;
    var activeRows, audienceRows, audienceFilteredRows, factBaseFilteredRows;
    var macroRows, cityRows, dateRows;
    var acc, cityAcc, sCity, sDate, row;
    var sumTotal, sumMandatory, sumFact, sumCompletedInAudience;
    var wholeCounts, factWholeCounts, nWholeTotal, nWholeMandatory, nWholeFact, nWholeCompletedInAudience;
    var checkTotal, checkMandatory, checkFact, checkCompleted;
    var nAudienceDistinct, nAudienceRaw, nFactBaseDistinct, nFactBaseRaw, checkAudienceDup, checkFactBaseDup;
    var resultRows, id;

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
            throw ("Не передан matrix_id -- открой эту диагностику на странице, где в URL уже есть фильтры (после \"Применить\" в любой из двух модалок)");
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
        dateRows = GetCompletionDateRows(programIds);

        // --- Пул "аудитория матрицы" (для total/mandatory/completedInAudience) ---
        audienceRows = GetMatrixAudienceCollaboratorRows(matrixDoc, activeRows);
        audienceFilteredRows = ApplyManualFilters(audienceRows, iPositionFilter, sMacroregionFilter, sMirCodeFilter, macroRows);

        // --- Пул "без аудитории" (для fact) ---
        factBaseFilteredRows = ApplyManualFilters(activeRows, iPositionFilter, sMacroregionFilter, sMirCodeFilter, macroRows);

        // ===================== РАСЧЁТ А: ПО ГОРОДАМ =====================
        acc = [];
        for (i = 0; i < ArrayCount(audienceFilteredRows); i++)
        {
            sCity = FindCity(cityRows, Int(audienceFilteredRows[i].id));
            cityAcc = GetOrCreateCityAcc(acc, sCity);
            for (j = 0; j < ArrayCount(programIds); j++)
            {
                sDate = FindCompletionDate(dateRows, Int(audienceFilteredRows[i].id), programIds[j]);
                cityAcc.total = cityAcc.total + 1;
                if (sDate == "") { cityAcc.mandatory = cityAcc.mandatory + 1; }
                else { cityAcc.completedInAudience = cityAcc.completedInAudience + 1; }
            }
        }
        for (i = 0; i < ArrayCount(factBaseFilteredRows); i++)
        {
            sCity = FindCity(cityRows, Int(factBaseFilteredRows[i].id));
            cityAcc = GetOrCreateCityAcc(acc, sCity);
            for (j = 0; j < ArrayCount(programIds); j++)
            {
                sDate = FindCompletionDate(dateRows, Int(factBaseFilteredRows[i].id), programIds[j]);
                if (sDate != "") { cityAcc.fact = cityAcc.fact + 1; }
            }
        }

        sumTotal = 0; sumMandatory = 0; sumFact = 0; sumCompletedInAudience = 0;
        for (i = 0; i < ArrayCount(acc); i++)
        {
            sumTotal = sumTotal + acc[i].total;
            sumMandatory = sumMandatory + acc[i].mandatory;
            sumFact = sumFact + acc[i].fact;
            sumCompletedInAudience = sumCompletedInAudience + acc[i].completedInAudience;
        }

        // ===================== РАСЧЁТ Б: ЦЕЛИКОМ, БЕЗ ГОРОДОВ =====================
        wholeCounts = CountPairs(audienceFilteredRows, programIds, dateRows);
        nWholeTotal = wholeCounts.totalPairs;
        nWholeMandatory = wholeCounts.totalPairs - wholeCounts.completedPairs;
        nWholeCompletedInAudience = wholeCounts.completedPairs;

        factWholeCounts = CountPairs(factBaseFilteredRows, programIds, dateRows);
        nWholeFact = factWholeCounts.completedPairs;

        LogAlert(1, "Run(). А: sumTotal=" + sumTotal + " sumMandatory=" + sumMandatory + " sumFact=" + sumFact
            + " sumCompletedInAudience=" + sumCompletedInAudience);
        LogAlert(1, "Run(). Б: nWholeTotal=" + nWholeTotal + " nWholeMandatory=" + nWholeMandatory + " nWholeFact=" + nWholeFact
            + " nWholeCompletedInAudience=" + nWholeCompletedInAudience);

        checkTotal = (sumTotal == nWholeTotal);
        checkMandatory = (sumMandatory == nWholeMandatory);
        checkFact = (sumFact == nWholeFact);
        checkCompleted = (sumCompletedInAudience == nWholeCompletedInAudience);

        // ===================== ПРОВЕРКА НА ДУБЛИ =====================
        nAudienceRaw = ArrayCount(audienceFilteredRows);
        nAudienceDistinct = CountDistinctIds(audienceFilteredRows);
        checkAudienceDup = (nAudienceRaw == nAudienceDistinct);

        nFactBaseRaw = ArrayCount(factBaseFilteredRows);
        nFactBaseDistinct = CountDistinctIds(factBaseFilteredRows);
        checkFactBaseDup = (nFactBaseRaw == nFactBaseDistinct);

        resultRows = [];
        id = 0;

        id = id + 1;
        resultRows.push({ id: id, metric: "matrix_id / название матрицы", value: matrixId + " / " + matrixName, check: "" });
        id = id + 1;
        resultRows.push({ id: id, metric: "Фильтры (macroregion/mir_code/position_common_id/program_id)",
            value: "[" + sMacroregionFilter + "] / [" + sMirCodeFilter + "] / " + iPositionFilter + " / " + iProgramFilter, check: "" });
        id = id + 1;
        resultRows.push({ id: id, metric: "Программ в матрице (после фильтра program_id)", value: String(ArrayCount(programIds)), check: "" });
        id = id + 1;
        resultRows.push({ id: id, metric: "Городов в разбивке (расчёт А, включая \"(без города)\", без \"Общий итог\")", value: String(ArrayCount(acc)), check: "" });

        id = id + 1;
        resultRows.push({ id: id, metric: "-- 1) ПРОВЕРКА НА ДУБЛИ (сырые строки XQuery vs distinct id) --", value: "", check: "" });
        id = id + 1;
        resultRows.push({ id: id, metric: "Аудитория матрицы: строк / уникальных id", value: nAudienceRaw + " / " + nAudienceDistinct, check: (checkAudienceDup ? "OK" : "ДУБЛИ!") });
        id = id + 1;
        resultRows.push({ id: id, metric: "База для Факт (без аудитории): строк / уникальных id", value: nFactBaseRaw + " / " + nFactBaseDistinct, check: (checkFactBaseDup ? "OK" : "ДУБЛИ!") });

        id = id + 1;
        resultRows.push({ id: id, metric: "-- 2) ИТОГОВАЯ СВЕРКА: сумма по городам (А) vs целиком по матрице (Б) --", value: "", check: "" });
        id = id + 1;
        resultRows.push({ id: id, metric: "total: сумма по городам vs целиком", value: sumTotal + " vs " + nWholeTotal, check: (checkTotal ? "OK" : "MISMATCH!") });
        id = id + 1;
        resultRows.push({ id: id, metric: "mandatory: сумма по городам vs целиком", value: sumMandatory + " vs " + nWholeMandatory, check: (checkMandatory ? "OK" : "MISMATCH!") });
        id = id + 1;
        resultRows.push({ id: id, metric: "fact: сумма по городам vs целиком", value: sumFact + " vs " + nWholeFact, check: (checkFact ? "OK" : "MISMATCH!") });
        id = id + 1;
        resultRows.push({ id: id, metric: "(справочно) пройдено-в-аудитории: сумма по городам vs целиком", value: sumCompletedInAudience + " vs " + nWholeCompletedInAudience, check: (checkCompleted ? "OK" : "MISMATCH!") });

        id = id + 1;
        resultRows.push({ id: id, metric: "-- 3) ПОГОРОДНЫЕ ТОЖДЕСТВА (для каждого города отдельно) --", value: "", check: "" });
        for (i = 0; i < ArrayCount(acc); i++)
        {
            row = acc[i];
            id = id + 1;
            resultRows.push({
                id: id,
                metric: "[" + row.city + "] total == mandatory + пройдено-в-аудитории",
                value: row.total + " == " + row.mandatory + " + " + row.completedInAudience,
                check: (row.total == (row.mandatory + row.completedInAudience) ? "OK" : "MISMATCH!")
            });
            id = id + 1;
            resultRows.push({
                id: id,
                metric: "[" + row.city + "] fact >= пройдено-в-аудитории",
                value: row.fact + " >= " + row.completedInAudience,
                check: (row.fact >= row.completedInAudience ? "OK" : "MISMATCH!")
            });
        }

        id = id + 1;
        resultRows.push({ id: id, metric: "-- 4) СПРАВОЧНО: сами числа по городам (для визуальной сверки с реальными данными) --", value: "", check: "" });
        for (i = 0; i < ArrayCount(acc); i++)
        {
            row = acc[i];
            id = id + 1;
            resultRows.push({
                id: id,
                metric: row.city,
                value: "total=" + row.total + " mandatory=" + row.mandatory + " fact=" + row.fact + " completedInAudience=" + row.completedInAudience,
                check: ""
            });
        }

        RESULT = resultRows;
        LogAlert(2, "Run(). Готово. checkTotal=" + checkTotal + " checkMandatory=" + checkMandatory
            + " checkFact=" + checkFact + " checkCompleted=" + checkCompleted
            + " checkAudienceDup=" + checkAudienceDup + " checkFactBaseDup=" + checkFactBaseDup);
    }
    catch (_ex)
    {
        RESULT = [{ id: 0, metric: "ОШИБКА", value: ExtractUserError(_ex), check: "" }];
        LogAlert(4, "Run(). ОШИБКА: " + ExtractUserError(_ex));
    }
    LogAlert(2, "Run(). КОНЕЦ");
}

Run();

COLUMNS = [
    { "data": "id", "editable": true, "hidden": true, "sortable": false },
    { "data": "metric", "title": "Показатель", "type": "string", "editable": false, "sortable": false, "multiline": true, "width": "50%" },
    { "data": "value", "title": "Значение", "type": "string", "editable": false, "sortable": false, "multiline": true, "width": "35%" },
    { "data": "check", "title": "Проверка", "type": "string", "editable": false, "sortable": false, "width": "15%" }
];
