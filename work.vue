select distinct ps.id as position_id, ps.name as position_name
from positions ps
join collaborators cs on ps.basic_collaborator_id = cs.id
where cs.is_dismiss <> 1
  and ps.is_position_finished <> 1
  and ps.name is not null
  and ltrim(rtrim(ps.name)) != ''


---

select
    sum(case when p.name is null or ltrim(rtrim(p.name)) = '' then 1 else 0 end) as empty_name,
    sum(case when p.basic_collaborator_id is null then 1 else 0 end) as no_collaborator_link,
    sum(case when p.basic_collaborator_id is not null and c.id is null then 1 else 0 end) as collaborator_not_found,
    sum(case when c.id is not null and c.is_dismiss = 1 then 1 else 0 end) as collaborator_dismissed,
    count(*) as total_still_null
from positions p
left join collaborators c on p.basic_collaborator_id = c.id
where isnull(p.is_position_finished, 0) = 0
  and p.position_common_id is null;
