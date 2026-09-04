select cs.id, cs.fullname, cs.is_dismiss
from event_collaborators ec
join events e on e.id = ec.event_id
join collaborators cs on cs.id = ec.collaborator_id
where e.education_method_id = 6499763079148161049
group by cs.id, cs.fullname, cs.is_dismiss
