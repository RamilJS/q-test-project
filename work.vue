// =====================================================================
// HREDU-181. Диагностика: где реально лежит "типовая должность"
// (position_common_id / common_position) относительно коллекции collaborators.
//
// Зачем: фильтр position_name в HREDU-181_vostok_polny_spisok_draft.js сейчас сравнивает
// String(collaborator.position_name) с filter -- это, похоже, конкретная должность
// (каталог position), а не типовая (common_position), на которую настроен фильтр в LPE
// (см. открытый вопрос №3 в шапке черновика и обсуждение по HREDU-174).
//
// Что делает скрипт:
//   1. Берёт первую строку действующего сотрудника из коллекции collaborators и печатает
//      ВСЕ её поля -- вдруг position_common_id (или похожее имя) уже лежит прямо тут.
//   2. Если на этой строке есть поле-ссылка на документ должности (предположение: называется
//      position_id) -- открывает сам этот документ и печатает ВСЕ его поля тоже, вдруг
//      position_common_id живёт там.
//
// Если имя поля-ссылки на должность не "position_id" (гадаем) -- шаг 2 скажет об этом прямо,
// и нужно будет посмотреть в дампе шага 1, как эта ссылка называется на самом деле, и
// прогнать скрипт ещё раз, подставив её вручную (или просто прислать мне дамп из шага 1 --
// разберём вместе).
//
// Как запустить: как тестовый remote_action/скрипт-агент, по аналогии с
// HREDU-181_diagnostic_learning_matrice_names.js. Пришли мне вывод alert() целиком.
// =====================================================================

/*
 * Печатает все поля объекта через alert().
 * @param {string} label    -   Заголовок для лога.
 * @param {Object} obj      -   Объект, поля которого нужно распечатать.
 * @returns {void}
 */
function DumpFields(label, obj)
{
    var dump, fld;
    dump = "";
    for (fld in obj)
    {
        dump = dump + fld.Name + " = " + String(fld) + "\r\n";
    }
    alert("--- " + label + " ---\r\n" + dump);
}

function Run()
{
    var collaboratorRow, positionDoc;

    collaboratorRow = ArrayOptFirstElem(XQuery("for $elem in collaborators where $elem/is_dismiss=false() return $elem"));
    if (collaboratorRow == undefined)
    {
        alert("Не нашлось ни одного действующего сотрудника -- странно, проверь коллекцию collaborators");
        return;
    }
    DumpFields("collaborators (первая строка, id=" + Int(collaboratorRow.id) + ")", collaboratorRow);

    try
    {
        if (collaboratorRow.position_id != undefined && Int(collaboratorRow.position_id) > 0)
        {
            positionDoc = tools.open_doc(Int(collaboratorRow.position_id)).TopElem;
            DumpFields("position (документ, id=" + Int(collaboratorRow.position_id) + ")", positionDoc);
        }
        else
        {
            alert("На строке collaborators нет поля position_id (или оно 0) -- посмотри в дампе выше, как называется поле-ссылка на должность, и пришли мне -- подставим вручную");
        }
    }
    catch (_ex)
    {
        alert("Не удалось открыть документ должности по предполагаемому полю position_id: " + ExtractUserError(_ex) + "\r\n(это нормально, если поле называется иначе -- см. дамп выше)");
    }
}

Run();
