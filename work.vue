// =====================================================================
// HREDU-183/181. Выборка-каталог для picker'а (dlg_select) поля foreign_elem с
// catalog: "cc_learning_matrice" -- по образцу присланного пользователем рабочего
// примера uni_catalog_list_cc_tree_subdivision (выборка дерева подразделений для
// процесса подбора).
//
// ПОЧЕМУ ЭТО БЫЛО НУЖНО (08-09.09.2026): picker для поля foreign_elem с
// catalog: "cc_learning_matrice" падал с "invalid XAML format" -- при этом само имя
// catalog было ПОДТВЕРЖДЕНО верным (диагностика .Name реального документа матрицы).
// Причина оказалась не в правах и не в имени catalog, а в том, что для picker'а нужна
// ОТДЕЛЬНАЯ зарегистрированная выборка (remote_collection) с именем по конвенции
// "uni_catalog_list_<имя_каталога>" -- она строит сам визуал списка выбора (со своими
// колонками, поиском, сортировкой). Для cc_tree_subdivision такая выборка существует
// (код uni_catalog_list_cc_tree_subdivision), для cc_learning_matrice -- нет, поэтому
// picker не мог ничего построить. Этот файл -- аналогичная выборка для матриц.
//
// В ОТЛИЧИЕ от примера с подразделениями: там иерархическое дерево (рекурсивный SQL
// CTE, parent_object_id указывает на родителя в дереве), у матриц обучения иерархии
// нет -- плоский список, поэтому вместо SQL CTE используется обычный XQuery с поиском
// по названию (contains), a parent_object_id везде null.
//
// КАК ЗАРЕГИСТРИРОВАТЬ (важно -- без этого выборка не заработает как picker):
//   1. Завести новый документ remote_collection в админке (аналогично XML, который ты
//      прислал для uni_catalog_list_cc_tree_subdivision).
//   2. code документа должен быть РОВНО "uni_catalog_list_cc_learning_matrice" --
//      судя по всему, платформа находит нужную выборку для picker'а именно по этому
//      имени (конвенция "uni_catalog_list_" + имя catalog из поля foreign_elem).
//   3. catalog_name = "cc_learning_matrice" (как в примере -- совпадает с code).
//   4. cache_vars/wvars -- те же два параметра, что в примере: "search" (string,
//      position 1, значение по умолчанию "") и "catalog_name" (string, position 0,
//      значение по умолчанию "cc_learning_matrice"). В этом скрипте catalog_name
//      реально не используется (как и в примере с подразделениями -- там тоже не
//      влияет на логику), но лучше оставить параметр для единообразия с рабочим
//      образцом.
//   5. exec_code/url -- путь к этому файлу после того, как он будет загружен в
//      codebase (по аналогии с request_recruitment.js в примере).
// =====================================================================

/*
 * Ищет записи cc_learning_matrices по названию (для живого поиска в picker'е,
 * параметр search) -- по аналогии с 'add_from_list' в примере визарда (contains()
 * по XQuery, а не SQL LIKE, т.к. иерархии/CTE тут не нужно).
 * @param {string} sSearchStr   -   Строка поиска (уже Trim()).
 * @returns {Object[]}          -   Массив документов cc_learning_matrice.
 */
function GetMatrixListRows(sSearchStr)
{
    var sXQueryText;
    if (sSearchStr != "")
    {
        sXQueryText = "for $elem in cc_learning_matrices where contains($elem/name, " + XQueryLiteral(sSearchStr) + ") order by $elem/name return $elem";
    }
    else
    {
        sXQueryText = "for $elem in cc_learning_matrices order by $elem/name return $elem";
    }
    return ArraySelectAll(XQuery(sXQueryText));
}

sSearchStr = Trim(search);
aMatrixRows = GetMatrixListRows(sSearchStr);

RESULT = [];
for (i = 0; i < ArrayCount(aMatrixRows); i++)
{
    RESULT.push({
        "id": String(Int(aMatrixRows[i].id)),
        "disp": String(aMatrixRows[i].name),
        "d0": String(aMatrixRows[i].name),
        // Плоский список -- иерархии у матриц обучения нет, в отличие от дерева
        // подразделений в примере, поэтому parent_object_id всегда null.
        "parent_object_id": null,
        "icon": "ico/course.ico"
    });
}

COLUMNS = [
    {
        "data": "id",
        "editable": true,
        "hidden": true,
        "sortable": false
    },
    {
        "data": "parent_object_id",
        "editable": true,
        "hidden": true,
        "sortable": false
    },
    {
        "data": "disp",
        "editable": true,
        "hidden": true,
        "sortable": false
    },
    {
        "data": "d0",
        "title": "Название",
        "type": "string",
        "editable": false,
        "colorsource": "col",
        "sortable": true,
        "multiline": false,
        "width": "100%",
        "minwidth": "100"
    }
];

PAGING.MANUAL = true;
SORT.FIELD = "d0";
SORT.DIRECTION = "ASC";
