select count(*) as still_null
from positions p
where p.is_active = 1
  and p.position_common_id is null;

---

  select count(*) as total_common_positions
from position_commons;
