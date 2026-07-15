SELECT [events].[id] [event_id]
       ,[event_results].[id] [eventt_result_id]
       ,[event_results].[person_id]
       ,[collaborators].[fullname] [person_fullname]
       ,coalesce([collaborator].[data].value('(collaborator/custom_elems/custom_elem[name=''f_personal_id'']/value)[1]', 'varchar(max)'), [collaborator].[data].value('(collaborator/custom_elems/custom_elem[name=''TabNum'']/value)[1]', 'varchar(max)')) [f_tab_num]
       ,[collaborator].[data].value('(collaborator/custom_elems/custom_elem[name=''f_assignment_id'']/value)[1]', 'varchar(max)') [f_assignment_id]
       ,format(dateadd(MM, 0 - 1, getdate()), 'yyyyMM') [period]
       ,'Webtutor' [source]
       ,[event].[data].value('(event/custom_elems/custom_elem[name=''f_development_goal'']/value)[1]', 'varchar(max)') [f_development_goal]
       ,[event].[data].value('(event/custom_elems/custom_elem[name=''f_qualification'']/value)[1]', 'varchar(max)') [f_qualification]
       ,year([events].[finish_date]) [budget_period]
       ,format([events].[finish_date], 'yyyy-MM-dd') [end_date]
       ,[events].[name]
       ,CASE [event].[data].value('(event/custom_elems/custom_elem[name=''f_training_category'']/value)[1]', 'varchar(max)')
             WHEN 'Лидерство и саморазвитие'
                    THEN 1
             WHEN 'Профессиональные навыки'
                    THEN 2
             WHEN 'Управленческие навыки'
                    THEN 3
             WHEN 'Коучинг'
                    THEN 4
             WHEN 'Сертификация'
                    THEN 5
             WHEN 'Языковое обучение'
                    THEN 6
             WHEN 'Конференция'
                    THEN 7
             WHEN 'Групповая или индивидуальная оценка'
                    THEN 8
             WHEN 'Соответствие регуляторным требованиям'
                    THEN 9
             WHEN 'Адаптация'
                    THEN 10
             END [f_training_category]
       ,[competences].[code] [f_competence]
       ,CASE
            WHEN [event].[data].value('(event/custom_elems/custom_elem[name=''f_education_format'']/value)[1]', 'varchar(max)') = 'Очное обучение внешнее' THEN 1
            WHEN [event].[data].value('(event/custom_elems/custom_elem[name=''f_education_format'']/value)[1]', 'varchar(max)') = 'Дистанционное обучение внешнее' THEN 1
             ELSE 2
             END [duration_type]
       ,COALESCE([event].[data].value('(event/custom_elems/custom_elem[name=''f_time_for_people'']/value)[1]','int'), [events].[duration_fact], 0) [duration_fact]
       ,CASE
            WHEN [event].[data].value('(event/custom_elems/custom_elem[name=''f_education_format'']/value)[1]', 'varchar(max)') = 'Очное обучение внутреннее' THEN 1
            WHEN [event].[data].value('(event/custom_elems/custom_elem[name=''f_education_format'']/value)[1]', 'varchar(max)') = 'Очное обучение внешнее' THEN 2
            WHEN [event].[data].value('(event/custom_elems/custom_elem[name=''f_education_format'']/value)[1]', 'varchar(max)') = 'Дистанционное обучение внутренее' THEN 3
            WHEN [event].[data].value('(event/custom_elems/custom_elem[name=''f_education_format'']/value)[1]', 'varchar(max)') = 'Дистанционное обучение внешнее' THEN 4
             ELSE 1
             END [event_type]
       ,CASE
             WHEN [event].[data].value('(event/custom_elems/custom_elem[name=''f_education_format'']/value)[1]', 'varchar(max)') = 'Очное обучение внешнее'
                    THEN round([events].[cost] / [events].[duration_fact], 2)
             WHEN [event].[data].value('(event/custom_elems/custom_elem[name=''f_education_format'']/value)[1]', 'varchar(max)') = 'Дистанционное обучение внешнее'
                    THEN round([events].[cost] / [events].[duration_fact], 2)
             ELSE 0
             END [cost]
       ,CASE [events].[currency]
             WHEN 'RUR'
                    THEN 'RUB'
             ELSE [events].[currency]
             END [currency]
       ,coalesce([places].[code], '1') [place]
       ,[events].[education_org_name]
       ,[orgs].[name] [org_name]
FROM [event_results]
INNER JOIN [events]
       ON [events].[id] = [event_results].[event_id]
             AND format([events].[finish_date], 'MMyyyy') = format(dateadd(MM, 0 - 1, getdate()), 'MMyyyy')
             AND (
                    cast([events].[role_id] AS VARCHAR(max)) NOT LIKE '%6922422469347395006%'
                    OR [events].[role_id] IS NULL
                    )
LEFT JOIN [event]
       ON [event].[id] = [events].[id]
LEFT JOIN [competences]
       ON [competences].[id] = TRY_CONVERT(bigint, [event].[data].value('(event/custom_elems/custom_elem[name=''f_competention'']/value)[1]', 'varchar(max)'))
LEFT JOIN [places]
       ON [places].[id] = [events].[place_id]
LEFT JOIN [collaborators]
       ON [collaborators].[id] = [event_results].[person_id]
LEFT JOIN [collaborator]
       ON [collaborator].[id] = [event_results].[person_id]
LEFT JOIN [orgs]
       ON [orgs].[id] = [collaborators].[org_id]
WHERE [event_results].[is_assist] = 1
       AND [event_results].[status_id] != 'missed'
       AND [event_results].[status_id] != 'cancel'
UNION
SELECT [courses].[id] [event_id]
       ,[learnings].[id] [eventt_result_id]
       ,[learnings].[person_id]
       ,[collaborators].[fullname] [person_fullname]
       ,coalesce([collaborator].[data].value('(collaborator/custom_elems/custom_elem[name=''f_personal_id'']/value)[1]', 'varchar(max)'), [collaborator].[data].value('(collaborator/custom_elems/custom_elem[name=''TabNum'']/value)[1]', 'varchar(max)')) [f_tab_num]
       ,[collaborator].[data].value('(collaborator/custom_elems/custom_elem[name=''f_assignment_id'']/value)[1]', 'varchar(max)') [f_assignment_id]
       ,format(dateadd(MM, 0 - 1, getdate()), 'yyyyMM') [period]
       ,'Webtutor' [source]
       ,[course].[data].value('(course/custom_elems/custom_elem[name=''f_development_goal'']/value)[1]', 'varchar(max)') [f_development_goal]
       ,'' [f_qualification]
       ,year([learnings].[last_usage_date]) [budget_period]
       ,format([learnings].[last_usage_date], 'yyyy-MM-dd') [end_date]
       ,[courses].[name]
       ,CASE [course].[data].value('(course/custom_elems/custom_elem[name=''f_training_category'']/value)[1]', 'varchar(max)')
             WHEN 'Лидерство и саморазвитие'
                    THEN 1
             WHEN 'Профессиональные навыки'
                    THEN 2
             WHEN 'Управленческие навыки'
                    THEN 3
             WHEN 'Коучинг'
                    THEN 4
             WHEN 'Сертификация'
                    THEN 5
             WHEN 'Языковое обучение'
                    THEN 6
             WHEN 'Конференция'
                    THEN 7
             WHEN 'Групповая или индивидуальная оценка'
                    THEN 8
             WHEN 'Соответствие регуляторным требованиям'
                    THEN 9
             WHEN 'Адаптация'
                    THEN 10
             END [f_training_category]
       ,[competences].[code] [f_competence]
       ,2 [duration_type]
       ,round(cast(replace([course].[data].value('(course/custom_elems/custom_elem[name=''f_plan_duration'']/value)[1]', 'varchar(max)'), ',', '.') as real), 0) [duration_fact]
       ,3 [event_type]
       ,0 [cost]
       ,'RUB' [currency]
       ,'1' [place]
       ,'Управление обучения, развития и адаптации персонала' [education_org_name]
       ,[orgs].[name] [org_name]
FROM [learnings]
INNER JOIN [courses]
       ON [courses].[id] = [learnings].[course_id]
             AND (
                    cast([courses].[role_id] AS VARCHAR(max)) NOT LIKE '%6921828509128268777%'
                    OR [courses].[role_id] IS NULL
                    )
             AND format([learnings].[last_usage_date], 'MMyyyy') = format(dateadd(MM, 0 - 1, getdate()), 'MMyyyy')
LEFT JOIN [course]
       ON [course].[id] = [courses].[id]
LEFT JOIN [competences]
       ON [competences].[id] = TRY_CONVERT(bigint, [course].[data].value('(course/custom_elems/custom_elem[name=''f_competention'']/value)[1]', 'varchar(max)'))
LEFT JOIN [collaborators]
       ON [collaborators].[id] = [learnings].[person_id]
LEFT JOIN [collaborator]
       ON [collaborator].[id] = [learnings].[person_id]
LEFT JOIN [orgs]
       ON [orgs].[id] = [collaborators].[org_id]
WHERE [learnings].[state_id] = 4
