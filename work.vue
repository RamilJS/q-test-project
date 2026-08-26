select count(*) as still_null
from positions p
where isnull(p.is_position_finished, 0) = 0
  and p.position_common_id is null;

select column_name
from information_schema.columns
where table_name = 'positions'
order by ordinal_position;
