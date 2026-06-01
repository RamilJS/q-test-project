for (sName in aTaskStageNames) {

oTask = ArrayOptFind(aTasks, "This.name == sName");

if (oTask == undefined)
continue;

resTasks = ArraySelect(aTasks, "This.parent_task_id == oTask.id");

for (oTask in resTasks) {

```
oTaskOption = ParseJson(oTask.desc);

// изменение статуса задачи помощником
try {
  if (
    oTaskOption.HasProperty("mentor_action_date") &&
    Date(oTaskOption.mentor_action_date) >= dLastDate
  ) {

    oOptionsMentor.send_flag = true;

    oOptionsMentor.changeResultTasks +=
      (oOptionsMentor.changeResultTasks == ""
        ? "<br/><br/>Внесены изменения в следующие задачи на испытательный срок: <br/>"
        : "") +
      "Задача '" + oTask.name + "'<br/>Статус: " +
      (oTaskOption.mentor_status == 'passed' ? '' : 'не ') +
      "выполнено<br/><br/>";
  }
}
catch(_e){}

// комментарии помощника к задаче
try {

  if (oTaskOption.HasProperty("comments")) {

    aComments = ArraySelect(
      oTaskOption.comments,
      "Date(This.date) >= dLastDate && Int(This.person_id) == Int(mentorID)"
    );

    if (ArrayCount(aComments) > 0) {

      oOptionsMentor.send_flag = true;

      oOptionsMentor.addTaskComments +=
        (oOptionsMentor.addTaskComments == ""
          ? "<br/><br/>Внесены комментарии к задачам на испытательный срок: <br/>"
          : "") +
        "Задача '" + oTask.name + "': <br/>" +
        ArrayMerge(
          aComments,
          "This.date + ': ' + This.comment",
          "<br/>"
        );
    }
  }

}
catch(_e){}
```

}
}

// блок "Изучите необходимые документы и материалы"

oTask = ArrayOptFind(
aTasks,
"This.name == 'Изучите необходимые документы и материалы'"
);

if (oTask != undefined) {

resTasks = ArraySelect(
aTasks,
"This.parent_task_id == oTask.id"
);

for (oTask in resTasks) {

```
oTaskOption = ParseJson(oTask.desc);

try {

  if (
    oTaskOption.HasProperty("mentor_action_date") &&
    Date(oTaskOption.mentor_action_date) >= dLastDate
  ) {

    oOptionsMentor.send_flag = true;

    oOptionsMentor.changeInfoTasks +=
      (oOptionsMentor.changeInfoTasks == ""
        ? "<br/><br/>Внесены изменения в следующие задачи по изучению информации: <br/>"
        : "") +
      "Задача '" + oTask.name + "'<br/>Статус: " +
      (oTaskOption.mentor_status == 'passed' ? '' : 'не ') +
      "выполнено<br/><br/>";
  }

}
catch(_e){}
```

}
}

