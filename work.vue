try
{
    alert("Открываем документ l.id = " + l.id);

    doc = OpenDoc(UrlFromDocID(l.id)).TopElem;

    alert("doc открыт");

    var annals = tools.annals_decrypt(doc);

    alert("annals = " + annals);

    if (annals != null)
    {
        alert("annals НЕ null");

        var history = annals.au.history;

        alert("ObjectType(history) = " + ObjectType(history));
        alert("ArrayCount(history.objects) = " + ArrayCount(history.objects));

        if (ArrayCount(history.objects) == 0)
            continue;

        var sections = ArrayFirstElem(history.objects).section;

        alert("ObjectType(sections) = " + ObjectType(sections));
        alert("ArrayCount(sections) = " + ArrayCount(sections));

        for (var si = 0; si < ArrayCount(sections); si++)
        {
            var sec = sections[si];
            alert("sec = " + sec);

            for (var qi = 0; qi < ArrayCount(sec.question); qi++)
            {
                var q = sec.question[qi];
                alert("question = " + q);

                var qid = q.id;
                alert("qid = " + qid);

                if (ArrayOptFind(aQuestions, "This.id == " + qid) == undefined)
                {
                    alert("Добавляем вопрос в aQuestions");

                    aQuestions.push({
                        id: qid,
                        text: HtmlToPlainText(q.text)
                    });
                }

                var qObj = new Object();

                qObj.quest_type = (q.qtype.OptForeignElem != undefined ? q.qtype.ForeignElem.name : "");
                qObj.result = tools_web.is_correct_question(q) ? "верно" : "неверно";

                alert("variants = " + q.variant);

                qObj.answer = ArrayMerge(q.variant, "HtmlToPlainText(value)", "; ");
                qObj.correct_answer = ArrayMerge(q.variant, "HtmlToPlainText(cor_value)", "; ");

                obj.questions[qid] = qObj;
            }
        }
    }
    else
    {
        alert("annals == NULL !!!");
    }
}
catch(e)
{
    alert("ERROR annals block: " + e);
}
