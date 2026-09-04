// HREDU-181. Восток_полный_список -- удалённое действие для выборки данных отчёта.
// Черновик: БЕЗ фильтра по видимости (подчинённость/HR) -- пока отдаёт всех активных
// сотрудников. Видимость добавим отдельным шагом, когда определимся с userIsHR/аналогом.
//
// Параметр: matrix_id -- id одной из размноженных записей cc_learning_matrice
// (или можно передать имя матрицы напрямую -- см. TODO ниже).
//
// Макрорегион (f_2ewj) -- НАЙДЕН и подтверждён примерами реальных документов collaborator:
// это custom_elem на сотруднике, значения вида "СЗФО", "ЮФО", "Поволжье". Достаётся тем же
// SQL-паттерном (c.data.value(...) по XML-колонке collaborator.data), что используется в
// живом настраиваемом отчёте "Отчет проверки незаполненых полей для матриц обучения"
// (custom_report, автор Илья Щапов) -- см. блок "4a) Макрорегион" ниже.
//
// ВАЖНО -- не проверено на реальной системе, нужна диагностика перед боевым использованием:
//   1. Имена коллекций 'cc_learning_matrices' и 'cc_learning_matrice_elements' в XQuery --
//      это ДОГАДКА по аналогии с cc_mir_code -> cc_mir_codes (HREDU-176). Возможно,
//      правильные имена другие -- надо проверить так же, как проверяли cc_mir_codes.
//   2. Логика "матрица размножена на несколько записей с одинаковым name" --
//      предположение из HREDU-181/184, тимлидом напрямую не подтверждено.

function getParam(sName, sDefault) {
    var sValue = PARAMETERS.GetOptProperty(sName);
    if (sDefault != undefined && (sValue == undefined || sValue == "")) {
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

try {
    if (matrixId == 0) {
        throw ("Не передан matrix_id -- выбранная пользователем матрица обучения");
    }

    // 1) Название матрицы -- по нему собираем все "размноженные" записи cc_learning_matrice
    //    (одна и та же матрица может быть представлена несколькими записями --
    //    по одной на комбинацию должность/мир-код, см. обсуждение HREDU-181).
    var matrixDoc = tools.open_doc(matrixId).TopElem;
    var matrixName = String(matrixDoc.name);

    var matrixRows = ArraySelectAll(XQuery("for $elem in cc_learning_matrices where $elem/name = " + XQueryLiteral(matrixName) + " return $elem"));
    var matrixIds = ArrayExtract(matrixRows, "Int(This.id)");

    if (ArrayCount(matrixIds) == 0) {
        throw ("Не найдено ни одной записи cc_learning_matrice с названием [" + matrixName + "]");
    }

    // 2) Элементы матрицы (программы обучения) по всем найденным записям сразу
    var elementRows = ArraySelectAll(XQuery("for $elem in cc_learning_matrice_elements where MatchSome($elem/cc_learning_matrice_id, (" + ArrayMerge(matrixIds, "This", ",") + ")) and $elem/is_active=true() return $elem"));

    // Уникальные программы -- одна и та же программа может повторяться между
    // "размноженными" записями матрицы (см. TODO выше)
    var programIds = ArraySelectDistinct(ArrayExtract(elementRows, "Int(This.education_method_id)"), "This");

    if (ArrayCount(programIds) == 0) {
        throw ("У матрицы [" + matrixName + "] не найдено ни одной активной программы (cc_learning_matrice_element)");
    }

    // 3) Названия программ -- они же станут заголовками динамических колонок
    for (var pid in programIds) {
        var edDoc = tools.open_doc(pid).TopElem;
        RESULT.columns.push({ id: String(pid), title: String(edDoc.name) });
    }

    // 4) Сотрудники -- ПОКА без фильтра по подчинённости/HR, все действующие
    var collaboratorRows = ArraySelectAll(XQuery("for $elem in collaborators where $elem/is_dismiss=false() return $elem"));

    // 4a) Макрорегион -- custom_elem f_2ewj, напрямую в XQuery-выборке collaborators не виден,
    //     поэтому достаём отдельным SQL-запросом по образцу живого настраиваемого отчёта
    //     "Отчет проверки незаполненых полей для матриц обучения" (c.data.value(...) по XML).
    var macroSqlStr = new Binary();
    macroSqlStr.AppendStr("select cs.id,\r\n");
    macroSqlStr.AppendStr("       c.data.value('(*/custom_elems/custom_elem[name=''f_2ewj'']/value)[1]', 'varchar(max)') as macroregion\r\n");
    macroSqlStr.AppendStr("from collaborators cs\r\n");
    macroSqlStr.AppendStr("inner join collaborator c on c.id = cs.id\r\n");
    macroSqlStr.AppendStr("where cs.is_dismiss != 1");

    var macroRows = ArraySelectAll(XQuery("sql:" + macroSqlStr.GetStr()));

    // 5) Даты прохождения -- по каждому сотруднику и программе минимальная start_date
    //    среди мероприятий (events), где сотрудник есть в участниках (event_collaborators).
    //    Без фильтра по статусу мероприятия -- по указанию тимлида, не усложняем пока.
    var sqlStr = new Binary();
    sqlStr.AppendStr("select ec.collaborator_id, e.education_method_id, min(ec.start_date) as first_date\r\n");
    sqlStr.AppendStr("from event_collaborators ec\r\n");
    sqlStr.AppendStr("join events e on e.id = ec.event_id\r\n");
    sqlStr.AppendStr("where ec.is_collaborator = 1\r\n");
    sqlStr.AppendStr("  and e.education_method_id in (" + ArrayMerge(programIds, "This", ",") + ")\r\n");
    sqlStr.AppendStr("group by ec.collaborator_id, e.education_method_id");

    var dateRows = ArraySelectAll(XQuery("sql:" + sqlStr.GetStr()));

    // 6) Собираем строки отчёта: 4 обязательных поля + по одной дате на программу
    for (var collab in collaboratorRows) {
        var row = new Object();
        row.fullname = String(collab.fullname);
        row.position_name = String(collab.position_name);
        row.subdivision_name = String(collab.position_parent_name);
        var mRow = ArrayOptFind(macroRows, "Int(This.id) == Int(collab.id)");
        row.macroregion = (mRow != undefined && mRow.macroregion != undefined ? String(mRow.macroregion) : "");
        row.dates = new Object();
        for (var pid2 in programIds) {
            var dRow = ArrayOptFind(dateRows, "Int(This.collaborator_id) == Int(collab.id) && Int(This.education_method_id) == Int(pid2)");
            row.dates.SetProperty(String(pid2), (dRow != undefined ? StrDate(Date(dRow.first_date), false) : ""));
        }
        RESULT.rows.push(row);
    }
}
catch (_ex) {
    ERROR = 1;
    MESSAGE = ExtractUserError(_ex);
}
