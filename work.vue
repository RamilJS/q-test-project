-- Шаг 1: сколько строк event_collaborators вообще привязано к этим 16 мероприятиям (без фильтра is_collaborator)
select count(*) as ec_rows_total
from event_collaborators ec
join events e on e.id = ec.event_id
where e.education_method_id = 6499763079148161049;

-- Шаг 2: какие вообще значения встречаются в is_collaborator у этих строк
select ec.is_collaborator, count(*) as cnt
from event_collaborators ec
join events e on e.id = ec.event_id
where e.education_method_id = 6499763079148161049
group by ec.is_collaborator;
