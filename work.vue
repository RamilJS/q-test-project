alert("Array = " + arr);

// --- Безопасное приведение к массиву ---
var items = [];
if (arr != undefined && arr != null) {
    try {
        // Пытаемся получить первый элемент (если arr — массив)
        var firstElem = arr[0];
        if (firstElem !== undefined && firstElem.person_id !== undefined) {
            items = arr; // arr уже массив/коллекция
        } else if (arr.person_id !== undefined) {
            items = [arr]; // arr — одиночный XML-узел
        } else {
            items = []; // пустой результат
        }
    } catch(e) {
        // Если доступ по индексу вызвал ошибку, считаем, что arr — одиночный узел
        if (arr.person_id !== undefined) {
            items = [arr];
        } else {
            items = [];
        }
    }
}

for (var k = 0; k < ArrayCount(items); k++)
{
    var l = items[k];
