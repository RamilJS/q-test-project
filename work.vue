Было (для events):
Sql_String.AppendStr("LEFT JOIN [competences]\r\n");
Sql_String.AppendStr("  ON [competences].[id] = [event].[data].value('(event/custom_elems/custom_elem[name=''f_competention'']/value)[1]', 'bigint')\r\n");
Стало:
Sql_String.AppendStr("LEFT JOIN [competences]\r\n");
Sql_String.AppendStr("  ON [competences].[id] = TRY_CONVERT(bigint, [event].[data].value('(event/custom_elems/custom_elem[name=''f_competention'']/value)[1]', 'varchar(max)'))\r\n");
Было (для course):
Sql_String.AppendStr("LEFT JOIN [competences]\r\n");
Sql_String.AppendStr("  ON [competences].[id] = [course].[data].value('(course/custom_elems/custom_elem[name=''f_competention'']/value)[1]', 'bigint')\r\n");
Стало:
Sql_String.AppendStr("LEFT JOIN [competences]\r\n");
Sql_String.AppendStr("  ON [competences].[id] = TRY_CONVERT(bigint, [course].[data].value('(course/custom_elems/custom_elem[name=''f_competention'']/value)[1]', 'varchar(max)'))\r\n");
