tArr = XQuery("for $elem in career_reserves where $elem/code ='2025' and $elem/status ='active' return $elem");

for (elem in tArr) {

  if (WeekDay(Date()) == 2) {

    elemDoc = tools.open_doc(elem.id).TopElem;

    managerDoc = ArrayOptFind(elemDoc.tutors, "This.is_native");
    mentorDoc = ArrayOptFind(elemDoc.tutors, "!This.is_native");

    mentorID = undefined;

    if (mentorDoc != undefined)
      mentorID = mentorDoc.person_id;

    dLastDate = DateOffset(DateNewTime(Date()), (0 - (86400 * 7)));

    aTasks = elemDoc.tasks;

    oOptionsMentor = new Object();
    oOptionsMentor.send_flag = false;
    oOptionsMentor.addComments = "<br/><br/>Комментарии к форме: <br/>";
    oOptionsMentor.addTaskComments = [];
    oOptionsMentor.changeResultTasks = "<br/><br/>Внесены изменения в следующие задачи на испытательный срок: <br/>";
    oOptionsMentor.changeInfoTasks = [];
    oOptionsMentor.educResult = [];

    if (mentorID != undefined) {

      oStage = ArrayOptFind(
        aTasks,
        "This.type == 'stage' && (This.parent_task_id == '' || This.parent_task_id == undefined) && This.name == 'Комментарии'"
      );

      aComments = ArraySelect(
        aTasks,
        "This.parent_task_id == oStage.id && OptInt(This.name, 0) == Int(mentorID) && This.start_date >= dLastDate"
      );

      if (ArrayCount(aComments) > 0) {

        oOptionsMentor.send_flag = true;

        oOptionsMentor.addComments =
          "<br/><br/>Комментарии к форме: <br/>" +
          ArrayMerge(
            aComments,
            "StrDate(This.start_date) + ': ' + This.desc",
            "<br/>"
          );
      }
    }

    if (oOptionsMentor.send_flag) {

      sString = "";

      sString += oOptionsMentor.changeInfoTasks;
      sString += oOptionsMentor.changeResultTasks;
      sString += oOptionsMentor.addTaskComments;
      sString += oOptionsMentor.addComments;

      tools.create_notification(
        'vtbl_adaptation_weekly_mentor_digest',
        managerDoc.person_id,
        sString,
        elem.id
      );
    }
  }
}
