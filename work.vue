// --- ID заявки ---
var iEventID = OptInt(7578479455174350432);

// --- Открываем заявку ---
var reqDoc = tools.open_doc(iEventID).TopElem;

// --- Берём ID сотрудника из заявки ---
var personId = reqDoc.person_id;

if (!personId) {
    alert("В заявке нет person_id");
    return;
}

// --- Ищем сотрудника в каталоге collaborators ---
var xq =
    "for $c in collaborators " +
    "where $c/id = " + SqlLiteral(personId) + " " +
    "return $c";

var collArr = XQuery(xq);

if (ArrayCount(collArr) == 0) {
    alert("Сотрудник с id " + personId + " не найден в collaborators");
    return;
}

var coll = collArr[0];

// --- Ищем руководителя (func_manager с is_native = 1) ---
var mgr = null;

for (var fm in coll.func_managers.func_manager) {
    if (String(fm.is_native) == "1") {
        mgr = fm;
        break;
    }
}

if (mgr == null) {
    alert("У сотрудника нет руководителя (нет func_manager с is_native=1)");
    return;
}

// --- Результат ---
var managerId = mgr.person_id;
var managerName = mgr.person_fullname;

// --- Проверка / вывод ---
alert("Manager ID: " + managerId);
alert("Manager Name: " + managerName);
