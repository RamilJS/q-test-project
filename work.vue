// ДИАГНОСТИКА 20.11.2026: HR (роль user) получает "Недостаточно прав для выполнения операции",
// админ видит выборку. Проверяем: запускается ли выборка вообще и проходит ли SQL-запрос.
sDiagUser = "нет";
try { sDiagUser = String(curUserID); } catch (_eU) {}
alert("cc_tree_subdivision: СТАРТ, curUserID=" + sDiagUser);
