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


Проблема:
При выполнении отчёта («start_bank_report», формирование банка данных по обучению из WebTutor) возникала ошибка SQL Server:
Msg 8114, Level 16, State 5, Line 1
Error converting data type nvarchar to bigint.
Причина — в SQL-запросе, формирующем отчёт, при JOIN'е таблицы [competences] значение кастомного поля f_competention извлекалось из XML-данных сущностей event/course сразу с приведением к типу bigint:
sql[event].[data].value('(event/custom_elems/custom_elem[name=''f_competention'']/value)[1]', 'bigint')
Если у какой-либо записи это поле было пустым, отсутствовало или содержало нечисловое значение, SQL Server не мог выполнить неявное приведение и падал с ошибкой, прерывая выполнение всего запроса (соответственно, ломался весь отчёт).
Решение:
Заменил прямое приведение к bigint внутри .value() на извлечение значения как текста (varchar(max)) с последующей безопасной конвертацией через TRY_CONVERT, которая возвращает NULL вместо ошибки, если значение нельзя преобразовать в число:
sqlTRY_CONVERT(bigint, [event].[data].value('(event/custom_elems/custom_elem[name=''f_competention'']/value)[1]', 'varchar(max)'))
Правка внесена в двух местах запроса (по одному на каждую часть UNION — для event и для course), как в самом SQL-запросе, так и в JS-скрипте отчёта, где этот запрос собирается построчно через Sql_String.AppendStr(...).
Результат: запрос больше не падает на записях с некорректным/пустым значением f_competention — для таких строк поле f_competence (код компетенции) просто становится NULL, отчёт формируется полностью.
