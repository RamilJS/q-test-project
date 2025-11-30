iEventID = OptInt(7578479455174350432);

var reqDoc = tools.open_doc(iEventID).TopElem;
var personId = reqDoc.person_id;

if (!personId) {
    alert("В заявке нет person_id");
    return;
}

// Находим сотрудника
var xq =
    "for $c in collaborators " +
    "where $c/id = " + SqlLiteral(personId) + " " +
    "return $c";

var collArr = XQuery(xq);

if (ArrayCount(collArr) == 0) {
    alert("Сотрудник не найден");
    return;
}

var coll = collArr[0];

// Ищем руководителя внутри func_managers
var mgr = coll.func_managers.func_manager.GetOptChildByKey("is_native", "1");

if (!mgr) {
    alert("У сотрудника нет руководителя (is_native=1)");
    return;
}

alert("Manager ID: " + mgr.person_id);
alert("Manager Name: " + mgr.person_fullname);
