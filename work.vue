// --- Заголовок вопросов ---
html.AppendStr("<tr><td colspan='9'></td>");
for (var qh_idx = 0; qh_idx < ArrayCount(aQuestions); qh_idx++) {
    var qh_obj = aQuestions[qh_idx];
    html.AppendStr("<td colspan='4' bgcolor='#EA9999'><b>" + (qh_obj.text || "") + "</b></td>");
}
html.AppendStr("</tr>");

// --- Шапка ---
html.AppendStr("<tr>");
html.AppendStr("<td bgcolor='#FCE5CD'><b>Пользователь</b></td>");
html.AppendStr("<td bgcolor='#FCE5CD'><b>Название теста</b></td>");
html.AppendStr("<td bgcolor='#FCE5CD'><b>Код</b></td>");
html.AppendStr("<td bgcolor='#FCE5CD'><b>Организация</b></td>");
html.AppendStr("<td bgcolor='#FCE5CD'><b>Подразделение</b></td>");
html.AppendStr("<td bgcolor='#FCE5CD'><b>Должность</b></td>");
html.AppendStr("<td bgcolor='#FCE5CD'><b>Дата</b></td>");
html.AppendStr("<td bgcolor='#FCE5CD'><b>Статус</b></td>");
html.AppendStr("<td bgcolor='#FCE5CD'><b>Баллы</b></td>");

for (var q_idx = 0; q_idx < ArrayCount(aQuestions); q_idx++) {
    html.AppendStr("<td bgcolor='#FCE5CD'><b>Тип</b></td>");
    html.AppendStr("<td bgcolor='#FCE5CD'><b>Результат</b></td>");
    html.AppendStr("<td bgcolor='#FCE5CD'><b>Полученный ответ</b></td>");
    html.AppendStr("<td bgcolor='#FCE5CD'><b>Правильный ответ</b></td>");
}
html.AppendStr("</tr>");
