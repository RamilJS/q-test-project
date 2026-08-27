select count(*) as total,
       min(modification_date) as oldest,
       max(modification_date) as newest
from cc_mir_codes;


select table_name
from information_schema.tables
where table_name like '%mir_code%';
