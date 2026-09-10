Выборка для таблицы ТЭП

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
//   а) для result_type != "fact" -- ПЕРЕД ручными фильтрами дополнительно применяем
//      аудиторию матрицы (GetMatrixAudienceCollaboratorRows());
//   б) ПОСЛЕ того, как готовые строки (с completion_date) собраны -- для "fact" оставляем
//      только строки с НЕпустой датой, для "mandatory" -- только с ПУСТОЙ датой,
//      для "total"/"plan" -- оставляем все строки без изменений.
//
// Параметр result_type -- один из: "total" | "plan" | "fact" | "mandatory".
//   ИЗМЕНЕНО (10.09.2026): раньше читался ТОЛЬКО как фиксированное значение на вкладке
//   "Параметры" отдельного виджета (по виджету на режим). Теперь читается СНАЧАЛА из
//   URL (как остальные фильтры) -- это открывает дорогу к переключению режима самим
//   пользователем на фронтенде (вкладки/ссылки/поле в модалке -- способ ещё
//   обсуждается). Если в URL параметра нет -- запасной путь: старое фиксированное
//   значение LPE (обратная совместимость с уже настроенными виджетами), иначе "total".
//   Подробности -- в начале Run().
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
    // ЗАЩИЩЕНО (10.09.2026, по мотивам реальной поломки): если LOG_NAME/CUR_OBJECT_ID
    // ещё не настроены (например CUR_OBJECT_ID=0 -- заглушка "TODO: заполнить после
    // создания документа в админке"), вызов может упасть с ошибкой -- а LogAlert()
    // вызывается ДО главного try/catch в Run(), поэтому необработанное исключение тут
    // рушило ВЕСЬ Run() целиком: RESULT никогда не устанавливался, таблица оставалась
    // пустой БЕЗ какого-либо сообщения об ошибке (именно это и произошло на реальном
    // тесте). Логирование -- вспомогательная вещь, её сбой не должен ронять сам отчёт.
    try
    {
        tools.call_code_library_method("vtbl_log_lib", "LogAlert", [LOG_NAME, typeLog, CUR_OBJECT_ID, message, DEBUG]);
    }
    catch (_exLog)
    {
        // ничего -- сбой логирования не должен ронять основной код
    }
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
        sFullUrl = GetRequestUrlSafe();
        LogAlert(1, "Run(). Request.Url = [" + sFullUrl + "]");

        // ИЗМЕНЕНО (10.09.2026, вынос режима на фронтенд + английские имена): раньше
        // result_type был ТОЛЬКО фиксированным параметром выборки -- задавался один раз
        // в LPE "Параметры" каждого из 4 виджетов, пользователь не мог его поменять сам.
        // Теперь сначала читаем result_type из URL -- так же, как остальные фильтры --
        // если он там есть, режим может переключать сам пользователь (например через
        // поле в модалке фильтров или ссылки-вкладки на странице). Если в URL параметра
        // нет -- запасной путь: старое фиксированное значение из LPE (обратная
        // совместимость с уже настроенными виджетами), а если и его нет -- дефолт "total".
        // Имена режимов ТЕПЕРЬ НА АНГЛИЙСКОМ (было "obshee"/"fakt"/"obyazatelno" --
        // транслит с русского, неудобно читать): total | plan | fact | mandatory.
        // Если у виджетов в админке result_type ещё настроен старыми именами -- их нужно
        // переименовать (obshee->total, fakt->fact, obyazatelno->mandatory, plan остаётся).
        sResultType = GetQueryParam(sFullUrl, "result_type");
        if (sResultType == "")
        {
            try
            {
                sResultType = String(result_type);
            }
            catch (_exResultTypeParam)
            {
                sResultType = "";
            }
        }
        if (sResultType == "" || sResultType == "undefined")
        {
            sResultType = "total";
        }
        if (sResultType != "total" && sResultType != "plan" && sResultType != "fact" && sResultType != "mandatory")
        {
            throw ("Неизвестный result_type=[" + sResultType + "] -- ожидается одно из: total, plan, fact, mandatory");
        }
        LogAlert(1, "Run(). sResultType=[" + sResultType + "]");

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

        // НОВОЕ (10.09.2026): аудитория матрицы -- ДЛЯ ВСЕХ РЕЖИМОВ, КРОМЕ "fact" (см.
        // "РЕШЕНИЯ" в шапке файла -- факт буквально "не зависимо от условий матрицы").
        if (sResultType != "fact")
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
        // АРХИТЕКТУРА" в шапке файла. "total"/"plan" -- без изменений (План = Общее,
        // см. "РЕШЕНИЯ" пункт 2).
        if (sResultType == "fact")
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
        else if (sResultType == "mandatory")
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


              Удаленное действие
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
// ДОБАВЛЕНО (10.09.2026, режим ТЭП-отчёта -- "result_type"): 6-е поле формы, select
// с 4 пунктами (Общее кол-во/План/Факт/Обязательно к прохождению -- значения
// total/plan/fact/mandatory, см. HREDU-183_tep_reports.js). Это НЕ фильтр сотрудников,
// а переключатель того, какой из 4 отчётов показывать. Раньше это было фиксированное
// значение параметра result_type на вкладке "Параметры" у КАЖДОГО из 4 отдельных
// виджетов "Табличные данные" -- пользователь не мог его менять сам. Теперь режим
// выбирается прямо в этой модалке (вместе с остальными фильтрами, одной кнопкой
// "Применить") и попадает в URL так же, как остальные 5 полей -- а значит НА СТРАНИЦЕ
// ТЭП-ОТЧЁТА ТЕПЕРЬ ДОСТАТОЧНО ОДНОГО ВИДЖЕТА "Табличные данные" (не четырёх) --
// HREDU-183_tep_reports.js сам читает result_type из URL и переключает поведение.
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
    sDefaultMatrixID = SanitizeIdFieldValue(GetQueryParam(sModalPageUrl, "matrix_id"));
    sDefaultMacroregion = GetQueryParam(sModalPageUrl, "macroregion");
    sDefaultMirCodeID = SanitizeIdFieldValue(GetQueryParam(sModalPageUrl, "mir_code_id"));
    sDefaultPositionCommonID = SanitizeIdFieldValue(GetQueryParam(sModalPageUrl, "position_common_id"));
    sDefaultProgramID = SanitizeIdFieldValue(GetQueryParam(sModalPageUrl, "program_id"));

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
        + "] program_id=[" + sDefaultProgramID + "] result_type=[" + sDefaultResultType + "]");

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
        },
        {
            name: "result_type",
            label: "Режим отчёта",
            type: "select",
            value: sDefaultResultType,
            entries: [
                { name: "Общее кол-во", value: "total" },
                { name: "План", value: "plan" },
                { name: "Факт", value: "fact" },
                { name: "Обязательно к прохождению", value: "mandatory" }
            ],
            mandatory: true
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
            sResultType = String(getFormField("result_type", "total"));
            DebugAlert("7b. matrix_id=" + iMatrixID + " macroregion=[" + sMacroregion + "] mir_code_id=" + iMirCodeID + " position_common_id=" + iPositionCommonID + " program_id=" + iProgramID + " result_type=[" + sResultType + "]");

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
                result_type: sResultType
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
