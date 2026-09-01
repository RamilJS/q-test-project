// =====================================================================
// HREDU-176. Диагностика: почему UpdateCollaboratorMirCodes() падает на
// XQuery() с "Неизвестная ошибка".
//
// Дублирует по одному все три обращения к XQuery() из UpdateCollaboratorMirCodes(),
// каждое в своём try/catch, чтобы понять, какое именно из трёх падает --
// без прогона всего агента целиком.
//
// ВАЖНО: я не имею доступа к самой платформе (WTV) из этой сессии и не могу
// сам проверить, сохранился ли object_name у типа документа collaborator_mircode
// (без cc_) -- эта правка делалась вручную в админке платформы, вне файлов,
// которые я редактирую здесь. Ниже -- ровно тот тест, который позволит
// увидеть это со стороны платформы.
// =====================================================================

function SafeXQuery(label, queryArg)
{
    alert("--- " + label + " ---");
    try
    {
        var rows = XQuery(queryArg);
        var count = 0;
        try { count = ArrayCount(ArraySelectAll(rows)); }
        catch (eCount) { alert("OK: XQuery() выполнился, но ArrayCount/ArraySelectAll упал: " + eCount); }
        alert("OK: " + label + " -- count=" + count);
        return true;
    }
    catch (eq)
    {
        alert("FAIL: " + label + " -- " + eq);
        return false;
    }
}

function Run()
{
    // 1) Известная рабочая коллекция -- контроль, что XQuery вообще жив.
    SafeXQuery("XQuery('cc_mir_codes')", "cc_mir_codes");

    // 2) Подозреваемая новая коллекция.
    SafeXQuery("XQuery('collaborator_mircodes')", "collaborator_mircodes");

    // 3) Проверка самого типа документа отдельно от коллекции: существует ли
    // он вообще под именем collaborator_mircode и что реально в нём лежит.
    // Если тип не создаётся/падает здесь -- дело в object_name/типе документа,
    // а не в регистрации коллекции для XQuery.
    alert("--- tools.new_doc_by_name('collaborator_mircode') ---");
    try
    {
        var oNewDoc = tools.new_doc_by_name("collaborator_mircode");
        alert("OK: new_doc_by_name -- TopElem доступен, id (до Save) = " + oNewDoc.TopElem.id);
    }
    catch (eDoc)
    {
        alert("FAIL: new_doc_by_name('collaborator_mircode') -- " + eDoc);
    }
}

Run();
