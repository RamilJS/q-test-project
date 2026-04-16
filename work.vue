try {
    qData = K.questions[qid2];
} catch(e) {
    qData = undefined;
}

for (var qh3 = 0; qh3 < ArrayCount(aQuestions); qh3++) {
    var qid2 = aQuestions[qh3].id;
    var qData;
    try {
        qData = K.questions[qid2];
    } catch(e) {
        qData = undefined;
    }
    if (qData == undefined) {
        html.AppendStr("<td colspan='4'> </td>");
    } else {
        var bg = qData.result == "неверно" ? "#F4CCCC" : "#D9EAD3";
        html.AppendStr("<td width='150'>" + qData.quest_type + "</td>");
        html.AppendStr("<td width='80' align='center' bgcolor='" + bg + "'>" + qData.result + "</td>");
        html.AppendStr("<td width='250'>" + qData.answer + "</td>");
        html.AppendStr("<td width='250'>" + qData.correct_answer + "</td>");
    }
}
