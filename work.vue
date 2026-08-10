Исправление

В файле ued_report_generate нужно заменить:

if (staffrating_task_expert_comment)
    sqlString.AppendStr("                  ,[task].[data].value('(task/custom_fields/custom_field[name=''view_coll_exec_comment'']/value)[1]', 'varchar(max)') [staffrating_task_expert_comment]\r\n");


на:
    if (staffrating_task_expert_comment)
    sqlString.AppendStr("                  ,[task].[data].value('(task/custom_fields/custom_field[name=''custom_expert_exec_comment'']/value)[1]', 'varchar(max)') [staffrating_task_expert_comment]\r\n");
