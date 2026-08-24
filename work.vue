

-- ШАГ 1. Уникальные названия должностей у действующих сотрудников
-- (края обрезаны, внутренние повторы пробелов схлопнуты).
-- Пока без фильтра "без типовой должности" -- поля-связи ещё не существует.

select distinct
    replace(replace(replace(ltrim(rtrim(c.position_name)), ' ', '<>'), '><', ''), '<>', ' ') as position_name
from collaborators c
where c.is_dismiss = 0
  -- and c.position_common_id is null   -- TODO: раскомментировать, когда поле будет создано
  and c.position_name is not null
  and ltrim(rtrim(c.position_name)) != '';


-- ШАГ 2. Из них -- те названия, для которых типовой должности ещё нет
-- (сравнение тоже по нормализованному имени с обеих сторон).

select distinct
    replace(replace(replace(ltrim(rtrim(c.position_name)), ' ', '<>'), '><', ''), '<>', ' ') as position_name
from collaborators c
where c.is_dismiss = 0
  and ltrim(rtrim(c.position_name)) != ''
  and not exists (
      select 1
      from position_commons pc
      where replace(replace(replace(ltrim(rtrim(pc.name)), ' ', '<>'), '><', ''), '<>', ' ')
          = replace(replace(replace(ltrim(rtrim(c.position_name)), ' ', '<>'), '><', ''), '<>', ' ')
  );
