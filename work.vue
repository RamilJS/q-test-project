// HREDU-183. ТЭП_общее_кол-во / ТЭП_план / ТЭП_факт / ТЭП_обязательно -- выборка для
// Табличных данных. Один файл, четыре режима через параметр result_type -- по образцу
// education_accept_event_card (там тоже один result_type переключает поведение одной
// выборки). Построено на основе HREDU-181_vostok_polny_spisok_draft.js -- та же матрица/
// программы/сотрудники/даты/фильтры, плюс новое: аудитория матрицы и разбиение по
// результату (см. ниже).
//
// ТЗ (HREDU-183, полный текст прислан пользователем 10.09.2026):
//   Общее кол-во сотрудников -- все, кто подходят под матрицу (должность + мир-код).
//   План -- кол-во сотрудников которые подходят под матрицу и у кого уже наступил
//     период прохождения тренинга.
//   Факт -- кол-во сотрудников, прошедших тренинг (НЕ ЗАВИСИМО от условий матрицы).
//   Обязательно к прохождению -- все, кто подходят под матрицу и при этом ещё не
//     проходили тренинг.
//   % обученных -- факт/план (уточнено с пользователем 10.09.2026 -- в тексте ТЗ
//     написано "план/факт", но по смыслу метрики и по факту подтверждения это факт/план;
//     сам % не входит в эту выборку -- это отдельный лёгкий расчёт для "Процент обученных",
//     не список сотрудников).
//
// РЕШЕНИЯ, ПРИНЯТЫЕ С ПОЛЬЗОВАТЕЛЕМ (10.09.2026):
//   1. Аудитория матрицы (position_common_id + mir_code_id НА САМОЙ cc_learning_matrice,
//      не на выбранных пользователем фильтрах) -- РЕАЛИЗУЕМ сейчас. Подтверждено
//      диагностикой (HREDU-183_diagnostic_matrix_audience_fields.js, реальный прогон):
//      у cc_learning_matrice есть свои поля mir_code_id и position_common_id (см. пример:
//      матрица id=7682761831139375285 имеет mir_code_id=6940453875862016547,
//      position_common_id=6389124187847681444). Это МАНДАТОРНОЕ условие "кому вообще
//      адресована матрица" -- отдельная вещь от РУЧНЫХ фильтров пользователя (те же имена
//      полей, но разный смысл): ручные фильтры дополнительно СУЖАЮТ то, что уже прошло
//      через аудиторию, а не заменяют её.
//   2. "Период прохождения тренинга" (нужен для честного "План") -- НЕ РЕАЛИЗОВАН. На
//      матрице/элементе есть start_study_period=1/end_study_period=4 (числа, не даты --
//      см. диагностику), но неясно: единицы измерения и от какой даты сотрудника
//      отсчитывать. Пользователь решил не тратить на это время сейчас -- УПРОЩЕНИЕ:
//      План = Общее (без доп. фильтра по периоду). Это совпадает с тем, что мы уже видели
//      на тестовых данных пользователя раньше в этом тикете (план и общее количество были
//      равны). ОТКРЫТЫЙ ВОПРОС, вернуться при необходимости -- аналогично открытым
//      вопросам №1-3 в HREDU-181_vostok_polny_spisok_draft.js.
//   3. Факт -- "не зависимо от условий матрицы" ПОНИМАЕТСЯ БУКВАЛЬНО: НЕ применяем
//      фильтр аудитории матрицы для этого режима (сотрудник мог когда-то пройти программу
//      матрицы, даже если сейчас должность/мир-код уже не подходят под матрицу). Базовый
//      пул для факта -- все активные сотрудники (как и раньше в HREDU-181), плюс ручные
//      фильтры пользователя (macroregion/mir_code/position_common_id/program_id) всё
//      равно применяются -- это их выбор, не аудитория матрицы.
//   4. Обязательно -- аудитория матрицы (как Общее) МИНУС те, кто прошёл (т.е. строки
//      с пустой датой прохождения).
//
// ИТОГОВАЯ АРХИТЕКТУРА: сначала строим ряды "сотрудник x программа" ТОЧНО как в
// HREDU-181 (программы матрицы, активные сотрудники, дата прохождения, все 4 ручных
// фильтра из URL). Разница только в ДВУХ местах:
//   а) для result_type != "fakt" -- ПЕРЕД ручными фильтрами дополнительно применяем
//      аудиторию матрицы (GetMatrixAudienceCollaboratorRows());
//   б) ПОСЛЕ того, как готовые строки (с completion_date) собраны -- для "fakt" оставляем
//      только строки с НЕпустой датой, для "obyazatelno" -- только с ПУСТОЙ датой,
//      для "obshee"/"plan" -- оставляем все строки без изменений.
//
// Параметры выборки (настраиваются на вкладке "Параметры" каждого из 4 виджетов
// Табличные данные -- ПО ОДНОМУ ФИКСИРОВАННОМУ ЗНАЧЕНИЮ result_type НА КАЖДЫЙ ВИДЖЕТ,
// а не из URL -- это конфигурация конкретного отчёта-представления, не фильтр
// пользователя):
//   result_type -- один из: "obshee" | "plan" | "fakt" | "obyazatelno".
// Фильтры (matrix_id, macroregion, mir_code, position_common_id, program_id) читаются
// ТАК ЖЕ, как в HREDU-181 -- из Request.Url (см. HREDU-183_diagnostic_get_params.js) --
// это работает надёжно в контексте ВЫБОРКИ (не удалённого действия, там был другой
// механизм через {{curEnv.curEnvUrl}}, здесь он не нужен).
//
// ВАЖНО про RESULT: как и в HREDU-181 -- RESULT это ПРЯМО массив строк, без обёртки.

DEBUG = true;              // На проде поставить false после тестирования
LOG_NAME = "agent";        // TODO: уточнить после создания документа в админке
CUR_OBJECT_ID = 0;         // TODO: заполнить ID документа выборки после её создания в админке

//-------------------------------------------------------------------------
//              Область функций
//-------------------------------------------------------------------------

function LogAlert(typeLog, message)
{
    tools.call_code_library_method("vtbl_log_lib", "LogAlert", [LOG_NAME, typeLog, CUR_OBJECT_ID, message, DEBUG]);
}

/*
 * Достаёт полный URL текущей страницы (см. HREDU-181_vostok_polny_spisok_draft.js --
 * тот же приём, подтверждён диагностикой). В контексте ВЫБОРКИ Request.Url надёжен.
 * @returns {string}
 */
function GetRequestUrlSafe()
{
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
 * строк (их нет в этом движке), через штатный API платформы (StrOptSubStrPos/
 * StrRangePos/StrLen), декодирование через UrlDecode(). Идентична версии в
 * HREDU-181_vostok_polny_spisok_draft.js.
 * @param {string} sUrl
 * @param {string} sParamName
 * @returns {string}
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
 * Находит все записи cc_learning_matrice с указанным названием (см. открытый вопрос №1
 * в HREDU-181_vostok_polny_spisok_draft.js -- матрица может быть "размножена").
 * @param {string} matrixName
 * @returns {Object[]}
 */
function GetMatrixRows(matrixName)
{
    LogAlert(1, "GetMatrixRows(). НАЧАЛО. matrixName=" + matrixName);
    var matrixRows;
    matrixRows = ArraySelectAll(XQuery("for $elem in cc_learning_matrices where $elem/name = " + XQueryLiteral(matrixName) + " return $elem"));
    LogAlert(1, "GetMatrixRows(). Найдено записей: " + ArrayCount(matrixRows));
    LogAlert(1, "GetMatrixRows(). КОНЕЦ");
    return matrixRows;
}

/*
 * Находит активные элементы (программы) для указанных записей матрицы.
 * @param {number[]} matrixIds
 * @returns {Object[]}
 */
function GetMatrixElementRows(matrixIds)
{
    LogAlert(1, "GetMatrixElementRows(). НАЧАЛО");
    var elementRows;
    elementRows = ArraySelectAll(XQuery("for $elem in cc_learning_matrice_elements where MatchSome($elem/cc_learning_matrice_id, (" + ArrayMerge(matrixIds, "This", ",") + ")) and $elem/is_active=true() return $elem"));
    LogAlert(1, "GetMatrixElementRows(). Найдено элементов: " + ArrayCount(elementRows));
    LogAlert(1, "GetMatrixElementRows(). КОНЕЦ");
    return elementRows;
}

/*
 * Собирает уникальный список ID программ (education_method) -- и с самой матрицы,
 * и с её элементов.
 * @param {Object[]} matrixRows
 * @param {Object[]} elementRows
 * @returns {number[]}
 */
function GetProgramIds(matrixRows, elementRows)
{
    LogAlert(1, "GetProgramIds(). НАЧАЛО");
    var matrixProgramIds, elementProgramIds, allProgramIds, programIds, i;
    matrixProgramIds = ArrayExtract(matrixRows, "Int(This.education_method_id)");
    elementProgramIds = ArrayExtract(elementRows, "Int(This.education_method_id)");
    allProgramIds = [];
    for (i = 0; i < ArrayCount(matrixProgramIds); i++)
    {
        allProgramIds.push(matrixProgramIds[i]);
    }
    for (i = 0; i < ArrayCount(elementProgramIds); i++)
    {
        allProgramIds.push(elementProgramIds[i]);
    }
    programIds = ArraySelectDistinct(allProgramIds, "This");
    LogAlert(1, "GetProgramIds(). Уникальных программ: " + ArrayCount(programIds));
    LogAlert(1, "GetProgramIds(). КОНЕЦ");
    return programIds;
}

/*
 * Строит справочник { id, title } по программам обучения.
 * @param {number[]} programIds
 * @returns {Object[]}
 */
function GetProgramTitles(programIds)
{
    LogAlert(1, "GetProgramTitles(). НАЧАЛО");
    var titles, i, programID, educationMethodDoc;
    titles = [];
    for (i = 0; i < ArrayCount(programIds); i++)
    {
        programID = programIds[i];
        educationMethodDoc = tools.open_doc(programID).TopElem;
        titles.push({ id: String(programID), title: String(educationMethodDoc.name) });
    }
    LogAlert(1, "GetProgramTitles(). КОНЕЦ");
    return titles;
}

/*
 * Читает всех действующих сотрудников (базовый пул, ДО аудитории матрицы и ДО ручных
 * фильтров).
 * @returns {Object[]}
 */
function GetActiveCollaboratorRows()
{
    LogAlert(1, "GetActiveCollaboratorRows(). НАЧАЛО");
    var collaboratorRows;
    collaboratorRows = ArraySelectAll(XQuery("for $elem in collaborators where $elem/is_dismiss=false() return $elem"));
    LogAlert(1, "GetActiveCollaboratorRows(). Найдено сотрудников: " + ArrayCount(collaboratorRows));
    LogAlert(1, "GetActiveCollaboratorRows(). КОНЕЦ");
    return collaboratorRows;
}

/*
 * Находит ID документов коллекции "positions", у которых position_common_id совпадает
 * с переданным ID (см. подробное объяснение схемы в HREDU-181_vostok_polny_spisok_draft.js).
 * @param {number} iCommonPositionFilter
 * @returns {number[]}
 */
function GetPositionIdsByCommonPosition(iCommonPositionFilter)
{
    LogAlert(1, "GetPositionIdsByCommonPosition(). НАЧАЛО. iCommonPositionFilter=" + iCommonPositionFilter);
    var positionRows, positionIds, i;
    positionRows = ArraySelectAll(XQuery("for $elem in positions where $elem/position_common_id = " + iCommonPositionFilter + " return $elem"));
    positionIds = [];
    for (i = 0; i < ArrayCount(positionRows); i++)
    {
        positionIds.push(Int(positionRows[i].id));
    }
    LogAlert(1, "GetPositionIdsByCommonPosition(). Найдено конкретных должностей: " + ArrayCount(positionIds));
    LogAlert(1, "GetPositionIdsByCommonPosition(). КОНЕЦ");
    return positionIds;
}

/*
 * Проверяет вхождение числа в массив чисел обычным циклом (без ArraySelect()/строк-
 * выражений со ссылкой на внешние переменные -- см. объяснение в HREDU-181).
 * @param {number[]} idArray
 * @param {number} value
 * @returns {boolean}
 */
function IdArrayContains(idArray, value)
{
    var i;
    for (i = 0; i < ArrayCount(idArray); i++)
    {
        if (Int(idArray[i]) == Int(value))
        {
            return true;
        }
    }
    return false;
}

/*
 * Достаёт макрорегион (custom_elem f_2ewj) по всем действующим сотрудникам одним SQL-запросом.
 * @returns {Object[]}
 */
function GetMacroregionRows()
{
    LogAlert(1, "GetMacroregionRows(). НАЧАЛО");
    var sqlText, macroRows;
    sqlText = "";
    sqlText = sqlText + "select cs.id,\r\n";
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_2ewj'']/value)[1]', 'varchar(max)') as macroregion\r\n";
    sqlText = sqlText + "from collaborators cs\r\n";
    sqlText = sqlText + "inner join collaborator c on c.id = cs.id\r\n";
    sqlText = sqlText + "where cs.is_dismiss != 1";
    macroRows = ArraySelectAll(XQuery("sql:" + sqlText));
    LogAlert(1, "GetMacroregionRows(). Строк: " + ArrayCount(macroRows));
    LogAlert(1, "GetMacroregionRows(). КОНЕЦ");
    return macroRows;
}

/*
 * Достаёт сырое значение мир-кодов (custom_elem f_mir_codes) по всем действующим
 * сотрудникам одним SQL-запросом.
 * @returns {Object[]}
 */
function GetMirCodeRows()
{
    LogAlert(1, "GetMirCodeRows(). НАЧАЛО");
    var sqlText, rows;
    sqlText = "";
    sqlText = sqlText + "select cs.id,\r\n";
    sqlText = sqlText + "       c.data.value('(*/custom_elems/custom_elem[name=''f_mir_codes'']/value)[1]', 'varchar(max)') as mir_codes\r\n";
    sqlText = sqlText + "from collaborators cs\r\n";
    sqlText = sqlText + "inner join collaborator c on c.id = cs.id\r\n";
    sqlText = sqlText + "where cs.is_dismiss != 1";
    rows = ArraySelectAll(XQuery("sql:" + sqlText));
    LogAlert(1, "GetMirCodeRows(). Строк: " + ArrayCount(rows));
    LogAlert(1, "GetMirCodeRows(). КОНЕЦ");
    return rows;
}

/*
 * Разбирает сырое значение f_mir_codes ("#LASK#17#|#LASM#17#...") в массив кодов без процентов.
 * @param {string} rawValue
 * @returns {string[]}
 */
function ExtractMirCodes(rawValue)
{
    var parts, fields, codes, i;
    codes = [];
    parts = ArrayDirect(ArraySelect(String(rawValue).split("|"), "This != ''"));
    for (i = 0; i < ArrayCount(parts); i++)
    {
        fields = ArrayDirect(ArraySelect(String(parts[i]).split("#"), "This != ''"));
        if (ArrayCount(fields) > 0)
        {
            codes.push(String(fields[0]));
        }
    }
    return codes;
}

/*
 * Проверяет, есть ли у сотрудника указанный мир-код среди любых его мир-кодов.
 * @param {Object[]} mirCodeRows
 * @param {number} collaboratorID
 * @param {string} mirCodeFilter
 * @returns {boolean}
 */
function CollaboratorHasMirCode(mirCodeRows, collaboratorID, mirCodeFilter)
{
    var row, codes;
    row = ArrayOptFind(mirCodeRows, "Int(This.id) == Int(collaboratorID)");
    if (row == undefined)
    {
        return false;
    }
    codes = ExtractMirCodes(row.mir_codes);
    return (ArrayOptFind(codes, "String(This) == String(mirCodeFilter)") != undefined);
}

/*
 * Резолвит ID объекта cc_mir_codes в его текстовый код -- нужно, чтобы превратить
 * mir_code_id САМОЙ МАТРИЦЫ (аудитория) в текст для сравнения с f_mir_codes сотрудника
 * (та же функция, что в HREDU-183_filtry_modal_shag1.js).
 * @param {number} iMirCodeID
 * @returns {string}
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
 * НОВОЕ (10.09.2026): фильтрует сотрудников по АУДИТОРИИ МАТРИЦЫ -- собственным полям
 * position_common_id/mir_code_id записи cc_learning_matrice (ТЗ: "все, кто подходят под
 * матрицу (должность + мир-код)"). Если у матрицы поле не заполнено (0/пусто) -- по этой
 * оси ограничения нет (аналогично тому, как в ручных фильтрах 0 означает "без фильтра").
 * @param {Object} matrixDoc            -   TopElem документа cc_learning_matrice.
 * @param {Object[]} collaboratorRows   -   Кандидаты (обычно результат GetActiveCollaboratorRows()).
 * @returns {Object[]}
 */
function GetMatrixAudienceCollaboratorRows(matrixDoc, collaboratorRows)
{
    LogAlert(1, "GetMatrixAudienceCollaboratorRows(). НАЧАЛО");
    var iAudiencePositionCommonId, iAudienceMirCodeId, sAudienceMirCodeText;
    var allowedPositionIds, mirCodeRows, filteredRows, i;

    iAudiencePositionCommonId = OptInt(matrixDoc.position_common_id, 0);
    iAudienceMirCodeId = OptInt(matrixDoc.mir_code_id, 0);
    sAudienceMirCodeText = ResolveMirCodeText(iAudienceMirCodeId);
    LogAlert(1, "GetMatrixAudienceCollaboratorRows(). Аудитория матрицы: position_common_id=" + iAudiencePositionCommonId + " mir_code_id=" + iAudienceMirCodeId + " (текст=[" + sAudienceMirCodeText + "])");

    filteredRows = collaboratorRows;

    if (iAudiencePositionCommonId > 0)
    {
        allowedPositionIds = GetPositionIdsByCommonPosition(iAudiencePositionCommonId);
        collaboratorRows = filteredRows;
        filteredRows = [];
        for (i = 0; i < ArrayCount(collaboratorRows); i++)
        {
            if (IdArrayContains(allowedPositionIds, OptInt(collaboratorRows[i].position_id, 0)))
            {
                filteredRows.push(collaboratorRows[i]);
            }
        }
        LogAlert(1, "GetMatrixAudienceCollaboratorRows(). После фильтра по должности аудитории осталось: " + ArrayCount(filteredRows));
    }

    if (sAudienceMirCodeText != "")
    {
        mirCodeRows = GetMirCodeRows();
        collaboratorRows = filteredRows;
        filteredRows = [];
        for (i = 0; i < ArrayCount(collaboratorRows); i++)
        {
            if (CollaboratorHasMirCode(mirCodeRows, Int(collaboratorRows[i].id), sAudienceMirCodeText))
            {
                filteredRows.push(collaboratorRows[i]);
            }
        }
        LogAlert(1, "GetMatrixAudienceCollaboratorRows(). После фильтра по мир-коду аудитории осталось: " + ArrayCount(filteredRows));
    }

    LogAlert(1, "GetMatrixAudienceCollaboratorRows(). КОНЕЦ. Итого в аудитории: " + ArrayCount(filteredRows));
    return filteredRows;
}

/*
 * Находит минимальную дату прохождения по каждому сотруднику и программе.
 * @param {number[]} programIds
 * @returns {Object[]}
 */
function GetCompletionDateRows(programIds)
{
    LogAlert(1, "GetCompletionDateRows(). НАЧАЛО");
    var sqlText, dateRows;
    sqlText = "";
    sqlText = sqlText + "select ec.collaborator_id, e.education_method_id, min(ec.start_date) as first_date\r\n";
    sqlText = sqlText + "from event_collaborators ec\r\n";
    sqlText = sqlText + "join events e on e.id = ec.event_id\r\n";
    sqlText = sqlText + "where e.education_method_id in (" + ArrayMerge(programIds, "This", ",") + ")\r\n";
    sqlText = sqlText + "group by ec.collaborator_id, e.education_method_id";
    dateRows = ArraySelectAll(XQuery("sql:" + sqlText));
    LogAlert(1, "GetCompletionDateRows(). Строк: " + ArrayCount(dateRows));
    LogAlert(1, "GetCompletionDateRows(). КОНЕЦ");
    return dateRows;
}

/*
 * Ищет дату прохождения конкретного сотрудника по конкретной программе.
 * @param {Object[]} dateRows
 * @param {number} collaboratorID
 * @param {number} programID
 * @returns {string}
 */
function FindCompletionDate(dateRows, collaboratorID, programID)
{
    var dateRow;
    dateRow = ArrayOptFind(dateRows, "Int(This.collaborator_id) == Int(collaboratorID) && Int(This.education_method_id) == Int(programID)");
    return (dateRow != undefined ? StrDate(Date(dateRow.first_date), false) : "");
}

/*
 * Ищет макрорегион конкретного сотрудника.
 * @param {Object[]} macroRows
 * @param {number} collaboratorID
 * @returns {string}
 */
function FindMacroregion(macroRows, collaboratorID)
{
    var macroRow;
    macroRow = ArrayOptFind(macroRows, "Int(This.id) == Int(collaboratorID)");
    return (macroRow != undefined && macroRow.macroregion != undefined ? String(macroRow.macroregion) : "");
}

/*
 * Ищет название программы по её ID.
 * @param {Object[]} programTitles
 * @param {number} programID
 * @returns {string}
 */
function FindProgramTitle(programTitles, programID)
{
    var titleRow;
    titleRow = ArrayOptFind(programTitles, "String(This.id) == String(programID)");
    return (titleRow != undefined ? String(titleRow.title) : "");
}

/*
 * Собирает строки отчёта для ОДНОГО сотрудника -- по одной строке на каждую программу
 * матрицы (те же 6 полей, что в HREDU-181).
 * @param {Object} collaborator
 * @param {Object[]} macroRows
 * @param {Object[]} dateRows
 * @param {Object[]} programTitles
 * @param {number[]} programIds
 * @returns {Object[]}
 */
function BuildReportRows(collaborator, macroRows, dateRows, programTitles, programIds)
{
    var rows, row, i, programID;
    rows = [];
    for (i = 0; i < ArrayCount(programIds); i++)
    {
        programID = programIds[i];
        row = new Object();
        row.fullname = String(collaborator.fullname);
        row.position_name = String(collaborator.position_name);
        row.subdivision_name = String(collaborator.position_parent_name);
        row.macroregion = FindMacroregion(macroRows, Int(collaborator.id));
        row.program_name = FindProgramTitle(programTitles, programID);
        row.completion_date = FindCompletionDate(dateRows, Int(collaborator.id), programID);
        rows.push(row);
    }
    return rows;
}

/*
 * Резолвит выбранную матрицу (matrix_id) в список ID программ обучения.
 * @param {number} matrixId
 * @param {string} matrixName
 * @returns {number[]}
 */
function ResolveProgramIds(matrixId, matrixName)
{
    LogAlert(1, "ResolveProgramIds(). НАЧАЛО. matrixId=" + matrixId);
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

    LogAlert(1, "ResolveProgramIds(). КОНЕЦ");
    return programIds;
}

/*
 * Точка входа. Собирает данные ОДНОГО из 4 отчётов-представлений ТЭП, в зависимости от
 * result_type -- см. подробности архитектуры и принятых решений в шапке файла.
 * @returns {void}
 */
function Run()
{
    LogAlert(2, "Run(). НАЧАЛО. result_type=" + result_type);
    var sResultType, sFullUrl, matrixId, matrixDoc, matrixName;
    var iProgramFilter, sMacroregionFilter, sMirCodeFilter, iPositionFilter;
    var programIds, programTitles, collaboratorRows, macroRows, mirCodeRows, dateRows;
    var collaboratorReportRows, i, j;
    var filteredProgramIds, filteredCollaboratorRows, allowedPositionIds, filteredResultRows;

    ERROR = 0;
    MESSAGE = "";
    RESULT = [];

    try
    {
        sResultType = String(result_type); // "obshee" | "plan" | "fakt" | "obyazatelno" -- параметр выборки, не из URL
        if (sResultType != "obshee" && sResultType != "plan" && sResultType != "fakt" && sResultType != "obyazatelno")
        {
            throw ("Неизвестный result_type=[" + sResultType + "] -- ожидается одно из: obshee, plan, fakt, obyazatelno");
        }

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

        programIds = ResolveProgramIds(matrixId, matrixName);

        if (iProgramFilter > 0)
        {
            filteredProgramIds = [];
            for (i = 0; i < ArrayCount(programIds); i++)
            {
                if (Int(programIds[i]) == iProgramFilter)
                {
                    filteredProgramIds.push(programIds[i]);
                }
            }
            programIds = filteredProgramIds;
            if (ArrayCount(programIds) == 0)
            {
                throw ("Программа [" + iProgramFilter + "] не найдена среди программ выбранной матрицы");
            }
        }

        programTitles = GetProgramTitles(programIds);
        collaboratorRows = GetActiveCollaboratorRows();

        // НОВОЕ (10.09.2026): аудитория матрицы -- ДЛЯ ВСЕХ РЕЖИМОВ, КРОМЕ "fakt" (см.
        // "РЕШЕНИЯ" в шапке файла -- факт буквально "не зависимо от условий матрицы").
        if (sResultType != "fakt")
        {
            collaboratorRows = GetMatrixAudienceCollaboratorRows(matrixDoc, collaboratorRows);
            LogAlert(1, "Run(). После фильтра аудитории матрицы осталось сотрудников: " + ArrayCount(collaboratorRows));
        }

        // Дальше -- РУЧНЫЕ фильтры пользователя, точно как в HREDU-181 (без изменений).
        if (iPositionFilter > 0)
        {
            allowedPositionIds = GetPositionIdsByCommonPosition(iPositionFilter);
            filteredCollaboratorRows = [];
            for (i = 0; i < ArrayCount(collaboratorRows); i++)
            {
                if (IdArrayContains(allowedPositionIds, OptInt(collaboratorRows[i].position_id, 0)))
                {
                    filteredCollaboratorRows.push(collaboratorRows[i]);
                }
            }
            collaboratorRows = filteredCollaboratorRows;
            LogAlert(1, "Run(). После ручного фильтра по типовой должности осталось сотрудников: " + ArrayCount(collaboratorRows));
        }

        macroRows = GetMacroregionRows();
        if (sMacroregionFilter != "")
        {
            filteredCollaboratorRows = [];
            for (i = 0; i < ArrayCount(collaboratorRows); i++)
            {
                if (FindMacroregion(macroRows, Int(collaboratorRows[i].id)) == sMacroregionFilter)
                {
                    filteredCollaboratorRows.push(collaboratorRows[i]);
                }
            }
            collaboratorRows = filteredCollaboratorRows;
            LogAlert(1, "Run(). После ручного фильтра по макрорегиону осталось сотрудников: " + ArrayCount(collaboratorRows));
        }

        if (sMirCodeFilter != "")
        {
            mirCodeRows = GetMirCodeRows();
            filteredCollaboratorRows = [];
            for (i = 0; i < ArrayCount(collaboratorRows); i++)
            {
                if (CollaboratorHasMirCode(mirCodeRows, Int(collaboratorRows[i].id), sMirCodeFilter))
                {
                    filteredCollaboratorRows.push(collaboratorRows[i]);
                }
            }
            collaboratorRows = filteredCollaboratorRows;
            LogAlert(1, "Run(). После ручного фильтра по мир-коду осталось сотрудников: " + ArrayCount(collaboratorRows));
        }

        dateRows = GetCompletionDateRows(programIds);

        RESULT = [];
        for (i = 0; i < ArrayCount(collaboratorRows); i++)
        {
            collaboratorReportRows = BuildReportRows(collaboratorRows[i], macroRows, dateRows, programTitles, programIds);
            for (j = 0; j < ArrayCount(collaboratorReportRows); j++)
            {
                RESULT.push(collaboratorReportRows[j]);
            }
        }
        LogAlert(1, "Run(). Строк до фильтра по result_type: " + ArrayCount(RESULT));

        // НОВОЕ (10.09.2026): финальное разбиение по result_type -- см. "ИТОГОВАЯ
        // АРХИТЕКТУРА" в шапке файла. "obshee"/"plan" -- без изменений (План = Общее,
        // см. "РЕШЕНИЯ" пункт 2).
        if (sResultType == "fakt")
        {
            filteredResultRows = [];
            for (i = 0; i < ArrayCount(RESULT); i++)
            {
                if (RESULT[i].completion_date != "")
                {
                    filteredResultRows.push(RESULT[i]);
                }
            }
            RESULT = filteredResultRows;
        }
        else if (sResultType == "obyazatelno")
        {
            filteredResultRows = [];
            for (i = 0; i < ArrayCount(RESULT); i++)
            {
                if (RESULT[i].completion_date == "")
                {
                    filteredResultRows.push(RESULT[i]);
                }
            }
            RESULT = filteredResultRows;
        }

        LogAlert(2, "Run(). Готово. result_type=" + sResultType + ", строк отчёта: " + ArrayCount(RESULT));
    }
    catch (_ex)
    {
        ERROR = 1;
        MESSAGE = ExtractUserError(_ex);
        LogAlert(4, "Run(). ОШИБКА: " + MESSAGE);
    }
    LogAlert(2, "Run(). КОНЕЦ");
}

//-------------------------------------------------------------------------
//              Область основного кода
//-------------------------------------------------------------------------

Run();
