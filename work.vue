// HREDU-181. Восток_полный_список -- удалённое действие для выборки данных отчёта.
// Черновик: БЕЗ фильтра по видимости (подчинённость/HR) -- пока отдаёт всех активных
// сотрудников. Видимость добавим отдельным шагом, когда определимся с userIsHR/аналогом.
//
// Параметр: matrix_id -- id одной из записей cc_learning_matrice
// (или можно передать имя матрицы напрямую -- см. TODO ниже).
//
// Макрорегион (f_2ewj) -- custom_elem на сотруднике, значения вида "СЗФО", "ЮФО", "Поволжье".
// Достаётся тем же SQL-паттерном (c.data.value(...) по XML-колонке collaborator.data), что
// используется в живом настраиваемом отчёте "Отчет проверки незаполненых полей для матриц
// обучения" (custom_report, автор Илья Щапов) -- см. блок "4a) Макрорегион" ниже.
//
// ПОДТВЕРЖДЕНО диагностикой (HREDU-181_diagnostic_learning_matrice_names.js, прогон
// пользователем 04.09.2026): имена коллекций 'cc_learning_matrices' и
// 'cc_learning_matrice_elements' -- верные, поля совпадают с ожиданиями из HREDU-184/180.
//
// ВСЁ ЕЩЁ ОТКРЫТО:
//   1. Логика "матрица размножена на несколько записей с одинаковым name" (одна запись
//      cc_learning_matrice на комбинацию должность/мир-код) -- предположение из HREDU-181/184,
//      тимлидом напрямую не подтверждено. Проверить нечем: реальные матрицы будет заводить
//      администратор системы позже, тестовая запись в системе пока одна ("Матрица тест").
//      НЕ БЛОКИРУЕТ разработку: код ниже одинаково корректно работает в обоих случаях --
//      он ищет ВСЕ записи cc_learning_matrice с данным name и объединяет программы со всех
//      найденных записей и их элементов, так что результат не зависит от того, одна это
//      запись или несколько.
//   2. У самой записи cc_learning_matrice ЕСТЬ собственное поле education_method_id (помимо
//      того, что программы также приходят через её cc_learning_matrice_element). В тестовой
//      записи значение education_method_id матрицы совпало со значением у её единственного
//      элемента -- похоже, элемент "зеркалит" программу матрицы, но на одной записи это не
//      доказать. Поэтому ниже programIds собираются И с самой матрицы, и с её элементов
//      (с дедупликацией) -- это ничего не портит, если предположение верно (дубликат просто
//      схлопнется), и подстраховывает, если неверно.
//   3. НОВЫЙ вопрос -- нужно уточнить у тимлида: у cc_learning_matrice есть поля
//      position_common_id и mir_code_id. Очень похоже, что это не просто атрибуты, а условие
//      "к каким сотрудникам применяется матрица" (по аналогии со старой системой
//      compound_program и её таргетингом f_position_names/f_mir_code). Если это так, то
//      список сотрудников в отчёте (шаг 4 ниже) должен фильтроваться по совпадению
//      position.position_common_id (см. HREDU-174) и мир-кода сотрудника через каталог
//      cc_collaborator_mircode (см. HREDU-176/178/179), а не показывать вообще всех активных,
//      как сейчас. Это ОТДЕЛЬНЫЙ вопрос от видимости по подчинённости/HR (кто видит отчёт) --
//      этот про то, какие строки вообще должны туда попадать. Пока сознательно не реализовано
//      (по указанию "без фильтров"), но важно не потерять при добавлении фильтров.

function getParam(sName, sDefault)
{
    var sValue = PARAMETERS.GetOptProperty(sName);
    if (sDefault != undefined && (sValue == undefined || sValue == ""))
    {
        sValue = sDefault;
    }
    return sValue;
}

var matrixId = OptInt(getParam("matrix_id", "0"), 0);

ERROR = 0;
MESSAGE = "";
RESULT = new Object();
RESULT.columns = [];
RESULT.rows = [];

try
{
    if (matrixId == 0)
    {
        throw ("Не передан matrix_id -- выбранная пользователем матрица обучения");
    }

    var matrixDoc, matrixName, matrixRows, matrixIds, elementRows;
    var matrixProgramIds, elementProgramIds, allProgramIds, programIds;
    var pid, edDoc, collaboratorRows, macroSqlStr, macroRows;
    var sqlStr, dateRows, collab, row, mRow, pid2, dRow, i;

    // 1) Название матрицы -- по нему собираем все записи cc_learning_matrice с тем же
    //    названием (см. открытый вопрос №1 в шапке файла про "размножение" по должностям/мир-кодам).
    matrixDoc = tools.open_doc(matrixId).TopElem;
    matrixName = String(matrixDoc.name);

    matrixRows = ArraySelectAll(XQuery("for $elem in cc_learning_matrices where $elem/name = " + XQueryLiteral(matrixName) + " return $elem"));
    matrixIds = ArrayExtract(matrixRows, "Int(This.id)");

    if (ArrayCount(matrixIds) == 0)
    {
        throw ("Не найдено ни одной записи cc_learning_matrice с названием [" + matrixName + "]");
    }

    // 2) Элементы матрицы (программы обучения) по всем найденным записям сразу
    elementRows = ArraySelectAll(XQuery("for $elem in cc_learning_matrice_elements where MatchSome($elem/cc_learning_matrice_id, (" + ArrayMerge(matrixIds, "This", ",") + ")) and $elem/is_active=true() return $elem"));

    // Программы -- берём и с самой матрицы (education_method_id), и с её элементов,
    // затем дедуплицируем (см. открытый вопрос №2 в шапке файла).
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

    if (ArrayCount(programIds) == 0)
    {
        throw ("У матрицы [" + matrixName + "] не найдено ни одной активной программы (cc_learning_matrice / cc_learning_matrice_element)");
    }

    // 3) Названия программ -- они же станут заголовками динамических колонок
    for (i = 0; i < ArrayCount(programIds); i++)
    {
        pid = programIds[i];
        edDoc = tools.open_doc(pid).TopElem;
        RESULT.columns.push({ id: String(pid), title: String(edDoc.name) });
    }

    // 4) Сотрудники -- ПОКА без фильтра по подчинённости/HR, все действующие
    collaboratorRows = ArraySelectAll(XQuery("for $elem in collaborators where $elem/is_dismiss=false() return $elem"));

    // 4a) Макрорегион -- custom_elem f_2ewj, напрямую в XQuery-выборке collaborators не виден,
    //     поэтому достаём отдельным SQL-запросом по образцу живого настраиваемого отчёта
    //     "Отчет проверки незаполненых полей для матриц обучения" (c.data.value(...) по XML).
    macroSqlStr = new Binary();
    macroSqlStr.AppendStr("select cs.id,\r\n");
    macroSqlStr.AppendStr("       c.data.value('(*/custom_elems/custom_elem[name=''f_2ewj'']/value)[1]', 'varchar(max)') as macroregion\r\n");
    macroSqlStr.AppendStr("from collaborators cs\r\n");
    macroSqlStr.AppendStr("inner join collaborator c on c.id = cs.id\r\n");
    macroSqlStr.AppendStr("where cs.is_dismiss != 1");

    macroRows = ArraySelectAll(XQuery("sql:" + macroSqlStr.GetStr()));

    // 5) Даты прохождения -- по каждому сотруднику и программе минимальная start_date
    //    среди мероприятий (events), где сотрудник есть в участниках (event_collaborators).
    //    Без фильтра по статусу мероприятия -- по указанию тимлида, не усложняем пока.
    sqlStr = new Binary();
    sqlStr.AppendStr("select ec.collaborator_id, e.education_method_id, min(ec.start_date) as first_date\r\n");
    sqlStr.AppendStr("from event_collaborators ec\r\n");
    sqlStr.AppendStr("join events e on e.id = ec.event_id\r\n");
    sqlStr.AppendStr("where ec.is_collaborator = 1\r\n");
    sqlStr.AppendStr("  and e.education_method_id in (" + ArrayMerge(programIds, "This", ",") + ")\r\n");
    sqlStr.AppendStr("group by ec.collaborator_id, e.education_method_id");

    dateRows = ArraySelectAll(XQuery("sql:" + sqlStr.GetStr()));

    // 6) Собираем строки отчёта: 4 обязательных поля + по одной дате на программу
    for (i = 0; i < ArrayCount(collaboratorRows); i++)
    {
        collab = collaboratorRows[i];
        row = new Object();
        row.fullname = String(collab.fullname);
        row.position_name = String(collab.position_name);
        row.subdivision_name = String(collab.position_parent_name);
        mRow = ArrayOptFind(macroRows, "Int(This.id) == Int(collab.id)");
        row.macroregion = (mRow != undefined && mRow.macroregion != undefined ? String(mRow.macroregion) : "");
        row.dates = new Object();
        for (pid2 in programIds)
        {
            dRow = ArrayOptFind(dateRows, "Int(This.collaborator_id) == Int(collab.id) && Int(This.education_method_id) == Int(pid2)");
            row.dates.SetProperty(String(pid2), (dRow != undefined ? StrDate(Date(dRow.first_date), false) : ""));
        }
        RESULT.rows.push(row);
    }
}
catch (_ex)
{
    ERROR = 1;
    MESSAGE = ExtractUserError(_ex);
}
