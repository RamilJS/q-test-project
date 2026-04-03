var raw_person_id = aCollaboratorIDArray[i];

if (raw_person_id == undefined)
    continue;

raw_person_id = Trim(raw_person_id);

if (raw_person_id == "")
    continue;

var person_id = OptInt(raw_person_id);

var raw_ass_id = aAssessmentIDArray[j];

if (raw_ass_id == undefined)
    continue;

raw_ass_id = Trim(raw_ass_id);

if (raw_ass_id == "")
    continue;

var ass_id = OptInt(raw_ass_id);
