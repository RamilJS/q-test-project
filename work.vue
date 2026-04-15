var history = annals.au.history;

var objects = history.objects;

if (objects == undefined)
    continue;

// нормализация objects
if (ObjectType(objects) != 'array')
    objects = [objects];

for (var o = 0; o < ArrayCount(objects); o++) {

    var sections = objects[o].section;

    if (sections == undefined)
        continue;

    // нормализация sections
    if (ObjectType(sections) != 'array')
        sections = [sections];

    for (var s = 0; s < ArrayCount(sections); s++) {

        var questions = sections[s].question;

        if (questions == undefined)
            continue;

        // нормализация questions
        if (ObjectType(questions) != 'array')
            questions = [questions];

        for (var q = 0; q < ArrayCount(questions); q++) {

            var qItem = questions[q];

            alert("question object = " + qItem);

            var qid = qItem.PrimaryKey; // ВАЖНО!

            alert("qid = " + qid);

            if (ArrayOptFind(aQuestions, "This.id == " + qid) == undefined) {

                aQuestions.push({
                    id: qid,
                    text: HtmlToPlainText(qItem.text)
                });
            }

            var qObj = new Object();

            qObj.quest_type =
                (qItem.qtype != undefined && qItem.qtype.OptForeignElem != undefined)
                ? qItem.qtype.ForeignElem.name
                : "";

            qObj.result = tools_web.is_correct_question(qItem) ? "верно" : "неверно";

            qObj.answer = ArrayMerge(qItem.variant, "HtmlToPlainText(value)", "; ");
            qObj.correct_answer = ArrayMerge(qItem.variant, "HtmlToPlainText(cor_value)", "; ");

            obj.questions[qid] = qObj;
        }
    }
}
