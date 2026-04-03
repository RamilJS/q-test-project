🔥 ШАГ 1. Понять, что реально приходит

Прямо ПЕРЕД вызовом createReportString вставь:

alert("collab RAW = " + oSelect_collaborator_ids);
alert("assessment RAW = " + oSelect_assessment_ids);

И СРАЗУ после split:

alert("collab AFTER SPLIT = " + ArrayMerge(oSelect_collaborator_ids, "This", "|"));
alert("assessment AFTER SPLIT = " + ArrayMerge(oSelect_assessment_ids, "This", "|"));

👉 Нам нужно увидеть:

строка это или нет
что внутри массива

🔥 ШАГ 2. УБРАТЬ опасный цикл

Вот это УДАЛЯЕШЬ:

for (i in aCollaboratorIDArray)

👉 и заменяешь НА ВСЕГДА:

for (var i = 0; i < ArrayCount(aCollaboratorIDArray); i++)

То же самое для j:

for (var j = 0; j < ArrayCount(aAssessmentIDArray); j++)

ШАГ 3. УБРАТЬ ВСЕ OptInt пока

Сейчас тебе НЕ нужно конвертировать в int.
Нужно понять, что приходит.

Заменяешь:

var person_id = OptInt(raw_person_id);

на:

var person_id = raw_person_id;
alert("person_id = " + person_id);

ШАГ 4. САМЫЙ ВАЖНЫЙ ТЕСТ

Перед XQuery вставь:

if (!IsNumeric(person_id)) {
    alert("НЕ ЧИСЛО person_id = " + person_id);
    continue;
}

if (!IsNumeric(ass_id)) {
    alert("НЕ ЧИСЛО ass_id = " + ass_id);
    continue;
}

ШАГ 5. ЖЁСТКАЯ нормализация (без умных функций)

Заменяешь ВСЮ логику получения ID на это:

var collab_raw = getFormField('select_collaborator_ids', "");
if (collab_raw == undefined || collab_raw == "") {
    oSelect_collaborator_ids = [];
} else {
    collab_raw = String(collab_raw);
    oSelect_collaborator_ids = collab_raw.split(";");
}

var ass_raw = getFormField('select_assessment_ids', "");
if (ass_raw == undefined || ass_raw == "") {
    oSelect_assessment_ids = [];
} else {
    ass_raw = String(ass_raw);
    oSelect_assessment_ids = ass_raw.split(";");
}



  
