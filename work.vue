-- =====================================================================
-- HREDU-174. Агент создания и проставления типовых должностей
-- Обычный T-SQL (MS SQL Server) для проверки тим-лидом в SSMS.
--
-- ТОЛЬКО ЧТЕНИЕ. insert/update не будет -- создание типовых должностей и
-- проставление связи сотруднику делает сам агент через объектный API
-- платформы (CreateObject/SaveObject или как они реально называются --
-- ждём пример), а не прямой SQL. SQL/XQuery в агенте используется только
-- для того, чтобы НАЙТИ кандидатов.
--
-- Обновлено по реальной XML-выгрузке сотрудника (root tag <collaborator>):
--   is_dismiss = 0        -- подтверждено: это и есть "действующий сотрудник"
--   position_name          -- подтверждено: лежит прямо на записи сотрудника,
--                              отдельный join на справочник должностей не нужен
--   hire_date               -- подтверждено имя поля даты приёма
--
-- ВСЁ ЕЩЁ предположение (не в XML сотрудника, нужна доп. проверка):
--   имя таблицы            -- collaborator (по тегу) или collaborators
--                              (по аналогии с orgs/subdivisions) -- см. шаг 0
--   position_common_id      -- поле-связь на collaborator для типовой
--                              должности -- ещё не существует, HREDU-174
--                              и HREDU-176/184 его создают
-- =====================================================================


-- ШАГ 0 (опционально). Найти реальное имя таблицы сотрудников,
-- если "collaborator"/"collaborators" не совпадёт.

select t.name as table_name
from sys.tables t
where t.name like '%collaborator%'
   or t.name like '%position%'
order by t.name;


-- ШАГ 1. Уникальные названия должностей у действующих сотрудников,
-- у которых ещё не проставлена типовая должность.
-- Это ровно то, что должна вернуть выборка внутри агента (GetActiveEmployeesWithoutCommonPosition).

select distinct
    ltrim(rtrim(c.position_name)) as position_name
from collaborator c
where c.is_dismiss = 0
  and c.position_common_id is null       -- TODO: появится вместе с HREDU-174/184
  and c.position_name is not null
  and ltrim(rtrim(c.position_name)) != '';


-- ШАГ 2. Из них -- те названия, для которых типовой должности ещё нет.
-- Тоже просто чтение -- список для того, чтобы агент решил, для каких
-- названий создавать новую типовую должность через CreateObject().

select distinct
    ltrim(rtrim(c.position_name)) as position_name
from collaborator c
where c.is_dismiss = 0
  and c.position_common_id is null
  and not exists (
      select 1
      from position_common pc
      where pc.name = ltrim(rtrim(c.position_name))
  );


  select column_name, data_type
from information_schema.columns
where table_name = 'position_common'
order by ordinal_position;


select t.name as table_name, c.name as column_name, ty.name as data_type
from sys.tables t
join sys.columns c on c.object_id = t.object_id
join sys.types ty on ty.user_type_id = c.user_type_id
where t.name in ('collaborator', 'collaborators', 'cc_collaborator', 'cc_collaborators')
order by t.name, c.column_id;

select column_name, data_type
from information_schema.columns
where table_name = 'position_commons'
order by ordinal_position;
