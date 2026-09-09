// HREDU-181. Восток_полный_список -- удалённое действие для выборки данных отчёта.
// Черновик: БЕЗ фильтра по видимости (подчинённость/HR) -- пока отдаёт всех активных
// сотрудников, без фильтра по position_common_id/mir_code_id самой матрицы.
//
// Параметры удалённого действия:
//   matrix_id       -- обязательный, id одной из записей cc_learning_matrice.
//   program_id      -- опциональный, id одной программы (education_method) из числа программ
//                       выбранной матрицы -- сужает отчёт до одной программы вместо всех.
//   macroregion     -- опциональный, точное значение макрорегиона (custom_elem f_2ewj).
//   mir_code        -- опциональный, код мир-кода (например "LASK") -- сотрудник попадает в
//                       отчёт, если этот код есть у него СРЕДИ ЛЮБЫХ его мир-кодов (не только
//                       основного/с наибольшим процентом, см. getMirCodeObject() в примере
//                       education_accept_event_card).
//   position_name   -- опциональный, ID типовой должности (position_common_id). Название
//                       параметра осталось историческим (position_name), но ФАКТИЧЕСКИ это ID,
//                       не текст -- см. "ИСПРАВЛЕНО (фильтр по должности)" ниже.
// Все четыре новых параметра -- это РУЧНЫЕ фильтры для того, кто смотрит отчёт (выводятся
// снаружи как доп. фильтры на странице), они не заменяют собой открытый вопрос №3 ниже
// (автоматическая фильтрация сотрудников по position_common_id/mir_code_id САМОЙ МАТРИЦЫ,
// т.е. "к кому матрица вообще применяется") -- это разные механизмы, вопрос №3 всё ещё открыт.
//
// ПОДТВЕРЖДЕНО диагностикой (HREDU-181_diagnostic_learning_matrice_names.js, прогон
// пользователем 04.09.2026): имена коллекций 'cc_learning_matrices' и
// 'cc_learning_matrice_elements' -- верные, поля совпадают с ожиданиями из HREDU-184/180.
//
// ПОДТВЕРЖДЕНО ПОЛНОСТЬЮ end-to-end тестовым прогоном (HREDU-181_test_run_vostok_polny_spisok.js,
// пользователь, 04.09.2026) на двух реальных матрицах ("Матрица тест" / программа "Ассесcмент";
// "Матрица тест новый" / программа "Основы лизинга", 1824 совпадения дат из 2312 сотрудников,
// даты и ФИО визуально сверены и совпадают с ожиданиями). По пути найден и исправлен реальный
// баг: фильтр ec.is_collaborator = 1 в GetCompletionDateRows() отсекал вообще все строки, т.к.
// это поле в реальных данных всегда NULL -- убран (см. комментарий у самой функции).
//
// ВСЁ ЕЩЁ ОТКРЫТО:
//   1. Логика "матрица размножена на несколько записей с одинаковым name" (одна запись
//      cc_learning_matrice на комбинацию должность/мир-код) -- предположение из HREDU-181/184,
//      тимлидом напрямую не подтверждено, проверить нечем (реальные матрицы заведёт
//      администратор позже). НЕ БЛОКИРУЕТ: GetMatrixRows()/GetProgramIds() ниже работают
//      одинаково корректно в обоих случаях -- берём ВСЕ записи с данным name и объединяем
//      программы со всех найденных записей и их элементов.
//   2. У самой записи cc_learning_matrice есть собственное поле education_method_id (помимо
//      того, что программы также приходят через её cc_learning_matrice_element). В тестовой
//      записи оно совпало со значением у единственного элемента -- похоже на "зеркалирование",
//      но на одной записи не доказать. Поэтому GetProgramIds() берёт программы И с матрицы,
//      И с элементов, с дедупликацией -- не портит, если предположение верно, подстраховывает,
//      если нет.
//   3. Нужно уточнить у тимлида: похоже, что position_common_id и mir_code_id на
//      cc_learning_matrice -- это условие "к каким сотрудникам применяется матрица" (по
//      аналогии со старой compound_program и её f_position_names/f_mir_code). Если так, то
//      GetActiveCollaboratorRows() должна фильтровать по position.position_common_id
//      (см. HREDU-174) и мир-коду через cc_collaborator_mircode (см. HREDU-176/178/179), а не
//      отдавать вообще всех активных, как сейчас. Отдельный вопрос от видимости по
//      подчинённости/HR -- этот про то, какие строки вообще должны попадать в отчёт.
//
// ФОРМАТ ОТЧЁТА (обновление 07.09.2026): решили сначала протестировать выборку на стандартном
// элементе LPE "Табличные данные" -- у него список колонок задаётся один раз, руками, в
// JSON-конфиге виджета (sCollectionConfig), и не может меняться в зависимости от того, сколько
// программ в конкретной матрице (разбирались в переписке по HREDU-181). Поэтому ПОКА
// отказались от "широкого" формата (1 строка на сотрудника, по колонке на каждую программу) в
// пользу "длинного": 1 строка на пару сотрудник+программа, с полями program_name и
// completion_date. Колонок теперь фиксированное количество (6), это совместимо со статическим
// виджетом без доработок. Плата: при матрице с N активными программами каждый сотрудник даёт N
// строк -- в GetActiveCollaboratorRows() сейчас все действующие сотрудники без фильтра, так что
// при большом числе программ строк может быть много; у виджета есть постраничная разбивка
// (iPageSize), но если станет медленно -- вернуться к вопросу №3 выше (фильтр по
// position_common_id/mir_code_id матрицы должен сильно сократить число сотрудников на страницу).
//
// ИСПРАВЛЕНО (07.09.2026, по факту "таблица рисуется, но пустая" после натягивания на
// виджет): сверили с реальной рабочей выборкой education_accept_event_card, которая тоже
// привязана к "Табличным данным" -- у неё RESULT это ПРЯМО массив строк
// (`RESULT = []; RESULT.push({...})`), без какой-либо обёртки. В этом черновике было
// `RESULT = new Object(); RESULT.rows = [...]` -- для "общей коллекции" (sCollectionType:
// "common"), судя по всему, это и есть контракт: RESULT должен быть массивом сам по себе,
// а не объектом с полем со строками внутри. Поправлено ниже. Заодно привели чтение параметра
// matrix_id к тому же паттерну, что в рабочем примере (`event_id` там читается как обычная
// глобальная переменная, без PARAMETERS.GetOptProperty) -- убрали GetParam(), читаем
// matrix_id напрямую.
//
// ПОДТВЕРЖДЕНО end-to-end НА ЖИВОМ ВИДЖЕТЕ (07.09.2026, пользователь): после исправлений выше
// все 6 полей (fullname, position_name, subdivision_name, macroregion, program_name,
// completion_date) корректно выводятся в "Табличных данных". Пустой completion_date на
// "Матрица тест новый" оказался не багом -- у неё просто нет исторических завершённых
// мероприятий по её программам; на другой, реальной матрице даты выводятся нормально.
//
// ИСПРАВЛЕНО (фильтры, 07.09.2026, по факту "без фильтров всё ок, с любым фильтром -- пусто"):
// фильтры program_id/macroregion/mir_code/position_name изначально были написаны через
// ArraySelect(array, "строка-выражение со ссылкой на внешнюю переменную") -- по аналогии с
// ArrayOptFind(), который так делать умеет и уже не раз подтверждён рабочим (FindMacroregion(),
// FindCompletionDate(), FindProgramTitle() и т.д.). ПОДТВЕРЖДЕНО ДОКУМЕНТАЦИЕЙ платформы
// (пользователь прислал 07.09.2026): у ArrayOptFind() аргумент qualExpr типизирован как
// (String), а у ArraySelect() аргумент qulExpr -- как (Bool). Это разные механизмы вычисления:
// у ArrayOptFind строка-выражение видит переменные из внешней области видимости (текущий
// контекст), у ArraySelect -- судя по поведению, только This (как будто компилируется отдельно,
// без замыкания на локальные переменные вызывающей функции, по аналогии с new Function() в JS).
// Итог: для выражений со ссылкой на внешние переменные годится только ArrayOptFind(), не
// ArraySelect(). Фильтры заменены на обычные циклы for/if с ручным push() -- без строк-выражений,
// риска нет. Другие использования ArraySelect() в этом файле (в ExtractMirCodes()) ссылаются
// только на This, их не касается.
//
// ИСПРАВЛЕНО (фильтр по должности, 07.09.2026): пользователь настроил элемент фильтра в LPE
// как ссылку на объект "типовая должность", а не как текстовое поле -- то есть параметр
// position_name на самом деле приходит как ID (position_common_id), а не как название
// должности. Диагностикой (3 прогона пользователем) подтверждена схема:
//   - collaborators.position_id -> ссылка на документ коллекции "positions" (9986 записей);
//   - collaborators.position_name -- это ТЕКСТ названия КОНКРЕТНОЙ должности (например
//     "Эксперт"), поле самой коллекции collaborators, к типовой должности отношения не имеет;
//   - на самой "positions" есть поле position_common_id -- ссылка на типовую должность (может
//     быть пустым, если для этой конкретной должности типовая не задана);
//   - коллекции "position"/"cc_position"/"cc_positions" (в единственном числе или с cc_) не
//     существуют -- рабочее название именно "positions".
// Фильтр переписан: сначала GetPositionIdsByCommonPosition(iPositionFilter) находит все
// "positions", у которых position_common_id совпадает с переданным ID, затем сотрудник
// проходит фильтр, если его collaboratorRows[i].position_id входит в этот список (через
// IdArrayContains() -- обычный цикл, не ArraySelect(), см. предыдущее "ИСПРАВЛЕНО" про
// ArraySelect и внешние переменные). НЕ ПРОВЕРЕНО НА ЖИВОМ ВИДЖЕТЕ -- диагностика подтвердила
// схему данных, но сам фильтр после переписывания ещё не гонялся с реальным значением
// position_common_id через LPE; нужно перепроверить на реальном фильтре.
//
// ИСПРАВЛЕНО (фильтр по должности, второй заход, 07.09.2026): при реальном прогоне с
// DEBUG=true упало на "Int(), Unknown source" сразу после того, как
// GetPositionIdsByCommonPosition() успешно нашла 129 подходящих должностей -- то есть сама
// схема (positions.position_common_id) верна, упало на следующем шаге. Причина: не у всех
// 2312 активных сотрудников заполнено collaborators.position_id (нет назначенной должности),
// а Int("") в этой платформе, в отличие от обычного JS, бросает исключение, а не возвращает
// 0/NaN. В коде для сравнения был обычный Int(), заменено на OptInt(..., 0) -- как уже
// сделано для остальных опциональных параметров в этом файле (см. matrix_id/program_id
// выше). Такие сотрудники (без должности) теперь просто не проходят фильтр -- это верно:
// раз должность не назначена, ни к какой типовой должности они не относятся.
//
// TODO при заведении документа remote_action в админке: заполнить LOG_NAME и CUR_OBJECT_ID
// ниже реальными значениями (Сервис >> Показать в XML / Копировать ID документа).
//
// ИЗМЕНЕНО (09.09.2026, связка с модалкой фильтров HREDU-183_filtry_modal_shag1.js):
// раньше matrix_id/program_id/macroregion/mir_code/position_name читались как голые
// глобальные переменные -- это работает, только если параметры выборки настроены вручную
// на вкладке "Параметры" в LPE (см. selections.md, способ "в"). Модалка фильтров вместо
// этого делает redirect на эту же страницу с фильтрами в query string (способ "б" из
// selections.md) -- через серию диагностик (HREDU-183_diagnostic_get_params.js)
// установили, что в этой версии платформы такие параметры не подставляются в глобальные
// переменные автоматически и не читаются через Request.matrix_id/GetParam()/GetOptProperty()
// -- единственный рабочий способ -- взять ВЕСЬ Request.Url строкой и вырезать нужный
// параметр вручную (GetQueryParam() ниже). По пути нашли ещё несколько особенностей этого
// скриптового движка (пригодится, если понадобится писать похожий код в других файлах):
//   - НЕТ regex-литералов (/паттерн/флаги) -- ломает разбор ВСЕГО файла, не только той
//     строки, где встретился;
//   - НЕТ передачи функций как значений/колбэков (например TryRead(function(){...})) --
//     тоже ломает разбор всего файла;
//   - НЕТ .indexOf()/.substring() как методов строки -- строковые операции здесь это
//     ГЛОБАЛЬНЫЕ ФУНКЦИИ платформы (StrOptSubStrPos/StrRangePos/StrLen и т.д., см.
//     документацию datex.ru, раздел "Работа со строками"), не методы;
//   - decodeURIComponent() не работает (не декодирует, тихо проваливается в catch) --
//     нужна родная UrlDecode(); encodeURIComponent() при этом СРАБОТАЛ, но для единой
//     схемы кодирования/декодирования на обеих сторонах (эта выборка + модалка) решили
//     использовать родную пару UrlEncodeQuery()/UrlDecode() везде;
//   - for-in работает только по МАССИВАМ ("Expression is not an array") -- по объекту
//     Request, например, не работает.
// Если Request.Url почему-то недоступен (например, тестовый прогон не через реальную
// страницу с URL) -- есть запасной путь на старое чтение голых глобальных переменных,
// на случай если параметры всё же настроены через вкладку "Параметры".

//-------------------------------------------------------------------------
//              Область констант
//-------------------------------------------------------------------------

DEBUG = false;              // Включает подробные логи уровня 1 [DEBUG] -- на проде и в репозитории должно быть false
LOG_NAME = "agent";         // TODO: уточнить после создания документа в админке -- пока по аналогии с серверными агентами
CUR_OBJECT_ID = 0;          // TODO: заполнить ID документа remote_action после его создания в админке

//-------------------------------------------------------------------------
//              Область функций
//-------------------------------------------------------------------------

/*
 * Управляет логированием. Уровни: 1-[DEBUG], 2-[INFO], 3-[WARN], 4-[ERROR].
 * @param {number} typeLog     -   Уровень логов.
 * @param {string} message     -   Сообщение для логов.
 * @returns {void}
 */
function LogAlert(typeLog, message)
{
    tools.call_code_library_method("vtbl_log_lib", "LogAlert", [LOG_NAME, typeLog, CUR_OBJECT_ID, message, DEBUG]);
}

/*
 * Находит все записи cc_learning_matrice с указанным названием (см. открытый вопрос №1 в шапке файла).
 * @param {string} matrixName   -   Название матрицы.
 * @returns {Object[]}          -   Массив документов cc_learning_matrice.
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
 * @param {number[]} matrixIds  -   ID записей cc_learning_matrice.
 * @returns {Object[]}          -   Массив документов cc_learning_matrice_element.
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
 * и с её элементов (см. открытый вопрос №2 в шапке файла).
 * @param {Object[]} matrixRows     -   Документы cc_learning_matrice.
 * @param {Object[]} elementRows    -   Документы cc_learning_matrice_element.
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
 * Строит справочник { id, title } по программам обучения (education_method) -- используется
 * как справочник для подстановки человекочитаемого названия программы в поле program_name
 * каждой строки отчёта (длинный формат, см. "ФОРМАТ ОТЧЁТА" в шапке файла). ВАЖНО: это уже не
 * список колонок виджета -- при "длинном" формате колонки статичны и не зависят от программ.
 * @param {number[]} programIds     -   ID программ (education_method).
 * @returns {Object[]}              -   Массив { id, title }.
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
 * Читает всех действующих сотрудников. ПОКА без фильтра по подчинённости/HR и без
 * фильтра по position_common_id/mir_code_id матрицы -- см. открытый вопрос №3 в шапке файла.
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
 * Находит ID документов коллекции "positions" (конкретная должность), у которых поле
 * position_common_id (ссылка на "типовую должность") совпадает с переданным ID -- см.
 * "ИСПРАВЛЕНО (фильтр по должности)" в шапке файла: коллекция collaborators отдаёт только
 * position_id (ссылку на КОНКРЕТНУЮ должность) и position_name (её текстовое название), а не
 * position_common_id -- это поле есть только на самом документе "positions". Поэтому фильтр
 * по должности идёт в два шага: сначала здесь находим id всех "positions", у которых нужная
 * типовая должность, а в Run() оставляем только тех сотрудников, чей position_id входит в
 * этот список.
 * @param {number} iCommonPositionFilter   -   ID типовой должности (position_common_id).
 * @returns {number[]}                     -   Массив ID документов positions.
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
 * Проверяет вхождение числа в массив чисел обычным циклом -- без ArraySelect()/строк-выражений
 * (см. "ИСПРАВЛЕНО (фильтры)" в шапке файла про ArraySelect и внешние переменные).
 * @param {number[]} idArray    -   Массив ID.
 * @param {number} value        -   Искомое значение.
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
 * Пытается прочитать Request.Url -- полный URL текущей страницы, включая query string
 * с фильтрами из модалки (HREDU-183_filtry_modal_shag1.js). Обёрнуто в try/catch: если
 * объект Request в этом контексте недоступен, возвращает "" -- вызывающий код (Run())
 * тогда переключается на запасной путь чтения параметров (см. "ИЗМЕНЕНО" в шапке файла).
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
 * Вырезает значение GET-параметра из полного URL строки (см. "ИЗМЕНЕНО" в шапке файла --
 * подтверждено диагностикой HREDU-183_diagnostic_get_params.js). Без regex и без методов
 * строк (.indexOf/.substring здесь не существуют) -- через штатный строковый API платформы:
 * StrOptSubStrPos/StrRangePos/StrLen, декодирование через UrlDecode() (родная пара к
 * UrlEncodeQuery(), которой кодирует модалка фильтров).
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
 * Достаёт макрорегион (custom_elem f_2ewj) по всем действующим сотрудникам одним SQL-запросом
 * (по образцу живого настраиваемого отчёта "Отчет проверки незаполненых полей для матриц обучения").
 * @returns {Object[]}      -   Массив { id, macroregion }.
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
 * Достаёт сырое значение мир-кодов (custom_elem f_mir_codes) по всем действующим сотрудникам
 * одним SQL-запросом -- тот же паттерн, что и GetMacroregionRows(). Разбор строки -- в
 * ExtractMirCodes(). Вызывается только когда реально пришёл фильтр mir_code (см. Run()) --
 * не нужен для самих строк отчёта, только для фильтрации.
 * @returns {Object[]}      -   Массив { id, mir_codes } (mir_codes -- сырая строка вида "#LASK#17#|#LASM#17#").
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
 * По образцу getMirCodeObject() из education_accept_event_card, но нам не нужны ни проценты,
 * ни руководитель мир-кода -- только сами коды, для фильтра "есть ли у сотрудника такой код".
 * @param {string} rawValue     -   Сырое значение custom_elem f_mir_codes.
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
 * Проверяет, есть ли у сотрудника указанный мир-код -- СРЕДИ ЛЮБЫХ его мир-кодов, не только
 * основного/с наибольшим процентом (так решили для фильтра -- см. параметры в шапке файла).
 * @param {Object[]} mirCodeRows    -   Результат GetMirCodeRows().
 * @param {number} collaboratorID   -   ID сотрудника.
 * @param {string} mirCodeFilter    -   Искомый мир-код.
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
 * Находит минимальную дату прохождения (start_date мероприятия) по каждому сотруднику
 * и программе. Без фильтра по статусу мероприятия -- по указанию тимлида, не усложняем.
 * ВАЖНО: фильтр по ec.is_collaborator НЕ используется -- диагностикой (тестовый прогон
 * 04.09.2026, программа 6499763079148161049) подтверждено, что это поле в реальных данных
 * всегда NULL (27 из 27 строк), из-за чего "= 1" отсекал вообще все строки. Сам факт
 * наличия строки в event_collaborators уже означает участие сотрудника в мероприятии.
 * @param {number[]} programIds     -   ID программ (education_method).
 * @returns {Object[]}              -   Массив { collaborator_id, education_method_id, first_date }.
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
 * @param {Object[]} dateRows       -   Результат GetCompletionDateRows().
 * @param {number} collaboratorID   -   ID сотрудника.
 * @param {number} programID        -   ID программы.
 * @returns {string}                -   Дата в виде строки или "" если не найдена.
 */
function FindCompletionDate(dateRows, collaboratorID, programID)
{
    var dateRow;
    dateRow = ArrayOptFind(dateRows, "Int(This.collaborator_id) == Int(collaboratorID) && Int(This.education_method_id) == Int(programID)");
    return (dateRow != undefined ? StrDate(Date(dateRow.first_date), false) : "");
}

/*
 * Ищет макрорегион конкретного сотрудника.
 * @param {Object[]} macroRows      -   Результат GetMacroregionRows().
 * @param {number} collaboratorID   -   ID сотрудника.
 * @returns {string}
 */
function FindMacroregion(macroRows, collaboratorID)
{
    var macroRow;
    macroRow = ArrayOptFind(macroRows, "Int(This.id) == Int(collaboratorID)");
    return (macroRow != undefined && macroRow.macroregion != undefined ? String(macroRow.macroregion) : "");
}

/*
 * Ищет название программы по её ID в справочнике, построенном GetProgramTitles().
 * @param {Object[]} programTitles  -   Результат GetProgramTitles().
 * @param {number} programID        -   ID программы.
 * @returns {string}
 */
function FindProgramTitle(programTitles, programID)
{
    var titleRow;
    titleRow = ArrayOptFind(programTitles, "String(This.id) == String(programID)");
    return (titleRow != undefined ? String(titleRow.title) : "");
}

/*
 * Собирает строки отчёта для ОДНОГО сотрудника -- по одной строке на каждую программу матрицы
 * (длинный формат, см. "ФОРМАТ ОТЧЁТА" в шапке файла): 4 обязательных поля сотрудника + название
 * программы + дата прохождения. Ровно под статический список колонок стандартного виджета LPE
 * "Табличные данные" (fullname, position_name, subdivision_name, macroregion, program_name,
 * completion_date -- 6 полей, без зависимости от числа программ в матрице).
 * @param {Object} collaborator     -   Документ сотрудника (из GetActiveCollaboratorRows()).
 * @param {Object[]} macroRows      -   Результат GetMacroregionRows().
 * @param {Object[]} dateRows       -   Результат GetCompletionDateRows().
 * @param {Object[]} programTitles  -   Результат GetProgramTitles().
 * @param {number[]} programIds     -   ID программ (education_method).
 * @returns {Object[]}              -   Массив строк отчёта (по числу программ в матрице).
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
 * Резолвит выбранную матрицу (matrix_id) в список ID программ обучения: находит все
 * записи cc_learning_matrice с тем же названием и объединяет программы с них и их
 * элементов (см. открытые вопросы №1-2 в шапке файла). Бросает исключение, если матрица
 * или её программы не найдены.
 * @param {number} matrixId     -   ID записи cc_learning_matrice, выбранной пользователем.
 * @returns {number[]}          -   ID программ (education_method).
 */
function ResolveProgramIds(matrixId)
{
    LogAlert(1, "ResolveProgramIds(). НАЧАЛО. matrixId=" + matrixId);
    var matrixDoc, matrixName, matrixRows, matrixIds, elementRows, programIds;

    matrixDoc = tools.open_doc(matrixId).TopElem;
    matrixName = String(matrixDoc.name);

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
        throw ("У матрицы [" + matrixName + "] не найдено ни одной активной программы (cc_learning_matrice / cc_learning_matrice_element)");
    }

    LogAlert(1, "ResolveProgramIds(). КОНЕЦ");
    return programIds;
}

/*
 * Точка входа удалённого действия. Собирает данные отчёта "Восток_полный_список" по
 * выбранной матрице обучения в ДЛИННОМ формате (1 строка на сотрудника+программу, см.
 * "ФОРМАТ ОТЧЁТА" в шапке файла) -- под стандартный виджет LPE "Табличные данные" со
 * статическим списком колонок. Поддерживает опциональные ручные фильтры: program_id,
 * macroregion, mir_code, position_name (см. описание параметров в шапке файла). ПОКА без
 * автоматического фильтра сотрудников по position_common_id/mir_code_id САМОЙ МАТРИЦЫ --
 * см. открытый вопрос №3 в шапке файла (это отдельный механизм, не путать с ручными фильтрами).
 *
 * ВАЖНО про RESULT: для "общей коллекции" (как у education_accept_event_card) виджет
 * ожидает, что RESULT -- это ПРЯМО массив строк, а не объект-обёртка (см. "ИСПРАВЛЕНО" в
 * шапке файла). Параметры читаются как обычные глобальные переменные (по образцу event_id
 * в education_accept_event_card), а не через PARAMETERS.
 * @returns {void}
 */
function Run()
{
    LogAlert(2, "Run(). НАЧАЛО");
    var matrixId, iProgramFilter, sMacroregionFilter, sMirCodeFilter, iPositionFilter;
    var programIds, programTitles, collaboratorRows, macroRows, mirCodeRows, dateRows, collaboratorReportRows, i, j;
    var filteredProgramIds, filteredCollaboratorRows, allowedPositionIds;
    var sFullUrl;

    ERROR = 0;
    MESSAGE = "";
    RESULT = [];

    try
    {
        // ИЗМЕНЕНО (09.09.2026, см. "ИЗМЕНЕНО" в шапке файла): фильтры теперь приходят из
        // query string URL (модалка HREDU-183_filtry_modal_shag1.js делает redirect на эту
        // же страницу с фильтрами в адресе) -- читаем через Request.Url + GetQueryParam().
        // Запасной путь -- если Request недоступен -- старое чтение голых глобальных
        // переменных (работает, только если параметры настроены вручную на вкладке
        // "Параметры" выборки в LPE).
        sFullUrl = GetRequestUrlSafe();
        LogAlert(1, "Run(). Request.Url = [" + sFullUrl + "]");

        if (sFullUrl != "")
        {
            matrixId = OptInt(GetQueryParam(sFullUrl, "matrix_id"), 0);
            iProgramFilter = OptInt(GetQueryParam(sFullUrl, "program_id"), 0);
            sMacroregionFilter = GetQueryParam(sFullUrl, "macroregion");
            sMirCodeFilter = GetQueryParam(sFullUrl, "mir_code");
            // ИЗМЕНЕНО (см. диагностику position_common_id): параметр называется position_name по
            // историческим причинам (так был заведён изначально), но фактически он теперь
            // приходит как ID position_common_id, а НЕ как текст названия должности -- в URL
            // модалка передаёт его под именем "position_common_id".
            iPositionFilter = OptInt(GetQueryParam(sFullUrl, "position_common_id"), 0);
        }
        else
        {
            LogAlert(1, "Run(). Request.Url недоступен -- пробуем запасной путь: голые глобальные переменные");
            matrixId = OptInt(matrix_id, 0);
            iProgramFilter = OptInt(program_id, 0);
            sMacroregionFilter = String(macroregion);
            sMirCodeFilter = String(mir_code);
            iPositionFilter = OptInt(position_name, 0);
        }

        LogAlert(1, "Run(). matrixId=" + matrixId + " programFilter=" + iProgramFilter
            + " macroregionFilter=[" + sMacroregionFilter + "] mirCodeFilter=[" + sMirCodeFilter
            + "] positionCommonIdFilter=" + iPositionFilter);

        if (matrixId == 0)
        {
            throw ("Не передан matrix_id -- выбранная пользователем матрица обучения");
        }

        programIds = ResolveProgramIds(matrixId);

        if (iProgramFilter > 0)
        {
            // ВАЖНО: тут раньше был ArraySelect(programIds, "Int(This) == iProgramFilter") --
            // не заработало (см. "ИСПРАВЛЕНО (фильтры)" в шапке файла): судя по всему,
            // ArraySelect(), в отличие от ArrayOptFind(), НЕ видит переменные из внешней
            // области видимости внутри строки-выражения, только This. Заменено на ручной цикл.
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
            LogAlert(1, "Run(). После фильтра по типовой должности осталось сотрудников: " + ArrayCount(collaboratorRows));
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
            LogAlert(1, "Run(). После фильтра по макрорегиону осталось сотрудников: " + ArrayCount(collaboratorRows));
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
            LogAlert(1, "Run(). После фильтра по мир-коду осталось сотрудников: " + ArrayCount(collaboratorRows));
        }

        dateRows = GetCompletionDateRows(programIds);
        //alert("programIds=" + tools.object_to_text(programIds, 'json') + "\r\ndateRows.count=" + ArrayCount(dateRows) + "\r\ndateRows=" + tools.object_to_text(dateRows, 'json')); // временно для отладки -- убрать перед сдачей

        for (i = 0; i < ArrayCount(collaboratorRows); i++)
        {
            collaboratorReportRows = BuildReportRows(collaboratorRows[i], macroRows, dateRows, programTitles, programIds);
            for (j = 0; j < ArrayCount(collaboratorReportRows); j++)
            {
                RESULT.push(collaboratorReportRows[j]);
            }
        }

        LogAlert(2, "Run(). Готово. Сотрудников: " + ArrayCount(collaboratorRows) + ", программ: " + ArrayCount(programIds) + ", строк отчёта: " + ArrayCount(RESULT));
        //alert(tools.object_to_text(RESULT, 'json')); // временно для отладки -- посмотреть, что реально вернул скрипт; убрать перед сдачей
    }
    catch (_ex)
    {
        ERROR = 1;
        MESSAGE = ExtractUserError(_ex);
        LogAlert(4, "Run(). ОШИБКА: " + MESSAGE);
        //alert("ОШИБКА: " + MESSAGE); // временно для отладки; убрать перед сдачей
    }
    LogAlert(2, "Run(). КОНЕЦ");
}

//-------------------------------------------------------------------------
//              Область основного кода
//-------------------------------------------------------------------------

Run();
