select count(*) as total,
       min(modification_date) as oldest,
       max(modification_date) as newest
from cc_mir_code;

select id, name, translate, owner_id, modification_date
from cc_mir_code
order by modification_date desc;
