select
    p.id                as position_id,
    p.name              as position_name,
    p.position_common_id,
    pc.name             as linked_common_name
from positions p
inner join position_commons pc on pc.id = p.position_common_id
where p.position_common_id is not null
  and exists (
      select 1
      from collaborators cs
      where cs.position_id = p.id
        and cs.is_dismiss = 0
  );
