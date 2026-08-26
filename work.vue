select p.id as position_id, p.name as position_name,
       p.basic_collaborator_id,
       c.is_dismiss,
       case when p.basic_collaborator_id is null then 'вакантна'
            when c.is_dismiss = 1 then 'занята уволенным сотрудником'
            else 'прочее' end as reason
from positions p
left join collaborators c on p.basic_collaborator_id = c.id
where isnull(p.is_position_finished, 0) = 0
  and p.position_common_id is null
order by reason, p.name;
