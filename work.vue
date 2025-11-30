// iRequestID — ID заявки, который приходит в функцию
var reqDoc = tools.open_doc(iRequestID).TopElem;

// Берём person_id из заявки
var personId = reqDoc.person_id;
if (personId == undefined || personId == null || personId == "") {
    alert("В заявке нет person_id");
    return;
}

// Делаем XQuery для поиска руководителя
var xq = "
    for $p in persons
    where $p/id = " + SqlLiteral(personId) + "
    return $p/func_managers/func_manager[is_native=1]
";

// Выполняем запрос
var mgrArr = XQuery(xq);

// Проверяем результат
if (ArrayCount(mgrArr) == 0) {
    alert('У сотрудника нет руководителя (is_native=1 не найден)');
    return;
}

// Берём первого (он единственный)
var mgr = mgrArr[0];

// Теперь у тебя есть:
var managerId = mgr.person_id;
var managerName = mgr.person_fullname;

// Для проверки выводим в лог
alert("Manager ID: " + managerId);
alert("Manager Name: " + managerName);

