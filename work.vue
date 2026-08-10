


EnableLog("main_ued_report", true);
writeLog = 1;
function alert(_str) {
               if (writeLog) {
                              LogEvent("main_ued_report", _str);
    }
               return _str;
}
//----------------------------------------------------------
var ratingArray = new Object();
               ratingArray['text'] = ['Цель не выполнена', 'Цель выполнена частично', 'Цель выполнена'];
               ratingArray['integral'] = ['', 'E', 'D', 'C', 'B', 'A'];
var competenceMarkArray = new Array();
var mainSubd = new Array();
var expertArray = new Array();
var competenceProfiles = new Array();
var commentArray = new Array();
var uedMainSubdArray = new Array();
function getRatingName(_rating, _rating_type){
               var _return_value = '';
               try {
                              _rating_type = _rating_type;
               }
               catch(_e){
                              _rating_type = 'text';
               }
               _rating = OptInt(_rating, 9999);
               try {
                              _return_value = ratingArray[_rating_type][_rating];
               }
               catch(_e){
                              _return_value = '';
               }
               return _return_value;
}
function getCompetenceValues(_plan_id, include_self, include_manager) {
               _plan_id = OptInt(_plan_id, 0);
               var _comp, _queryRow, _compDoc;
               var competenceValues = ArrayOptFind(competenceMarkArray, ("This.id == " + _plan_id));
               if (competenceValues == undefined) {
                              competenceValues = new Object();
                              competenceValues.id = _plan_id;
                              competenceValues.competences = new Object();
                              competenceValues.competences.self = new Object();
                              competenceValues.competences.self.comp_1 = '';
                              competenceValues.competences.self.comp_2 = '';
                              competenceValues.competences.self.comp_3 = '';
                              competenceValues.competences.self.comp_4 = '';
                              competenceValues.competences.self.comp_5 = '';                        
                              competenceValues.competences.manager = new Object();
                              competenceValues.competences.manager.comp_1 = '';
                              competenceValues.competences.manager.comp_2 = '';
                              competenceValues.competences.manager.comp_3 = '';
                              competenceValues.competences.manager.comp_4 = '';
                              competenceValues.competences.manager.comp_5 = '';
                              if (include_self) {
                                            _queryRow = ArrayOptFirstElem(XQuery("for $elem in pas where $elem/assessment_plan_id=" + respElem.id + " and $elem/assessment_appraise_type='competence_appraisal' and $elem/status='self' return $elem"));
                                            _compDoc = tools.open_doc(_queryRow.id).TopElem;
                                            for (_comp in _compDoc.competences) {
                                                           switch(_comp.competence_id.ForeignElem.name) {
                                                                          case "Дорожим клиентом":
                                                                                         competenceValues.competences.self.comp_1 = _comp.mark_text.Value;
                                                                                         break;
                                                                          case "Работаем в команде":
                                                                                         competenceValues.competences.self.comp_2 = _comp.mark_text.Value;
                                                                                         break;
                                                                          case "Отвечаем за результат":
                                                                                         competenceValues.competences.self.comp_3 = _comp.mark_text.Value;
                                                                                         break;
                                                                          case "Проявляем инициативу":
                                                                                         competenceValues.competences.self.comp_4 = _comp.mark_text.Value;
                                                                                         break;
                                                                          case "Совершенствуемся постоянно":
                                                                                         competenceValues.competences.self.comp_5 = _comp.mark_text.Value;
                                                                                         break;
                                                           }
                                            }
                              }
                              if (include_manager) {
                                            _queryRow = ArrayOptFirstElem(XQuery("for $elem in pas where $elem/assessment_plan_id=" + respElem.id + " and $elem/assessment_appraise_type='competence_appraisal' and $elem/status='manager' return $elem"));
                                            _compDoc = tools.open_doc(_queryRow.id).TopElem;
                                            for (_comp in _compDoc.competences) {
                                                           switch(_comp.competence_id.ForeignElem.name) {
                                                                          case "Дорожим клиентом":
                                                                                         competenceValues.competences.manager.comp_1 = _comp.mark_text.Value;
                                                                                         break;
                                                                          case "Работаем в команде":
                                                                                         competenceValues.competences.manager.comp_2 = _comp.mark_text.Value;
                                                                                         break;
                                                                          case "Отвечаем за результат":
                                                                                         competenceValues.competences.manager.comp_3 = _comp.mark_text.Value;
                                                                                         break;
                                                                          case "Проявляем инициативу":
                                                                                         competenceValues.competences.manager.comp_4 = _comp.mark_text.Value;
                                                                                         break;
                                                                          case "Совершенствуемся постоянно":
                                                                                         competenceValues.competences.manager.comp_5 = _comp.mark_text.Value;
                                                                                         break;
                                                           }
                                            }
                              }
                              competenceMarkArray.push(competenceValues);
               }
               return competenceValues;
}
function getSubsArray(_ass_id) {
    var sqlString = new Binary();
    var returnVal = new Object();
    sqlString.AppendStr("IF OBJECT_ID('tempdb..#tt') IS NOT NULL DROP TABLE [#tt];\r\n")
    sqlString.AppendStr("IF OBJECT_ID('tempdb..#tt1') IS NOT NULL DROP TABLE [#tt1];\r\n")
    sqlString.AppendStr("select distinct ap.data.value('(assessment_plan/custom_elems/custom_elem[name=''f_ued_main_subdivision'']/value)[1]', 'varchar(max)') [f_ued_main_subdivision]\r\n")
    sqlString.AppendStr("into #tt\r\n")
    sqlString.AppendStr("from assessment_plan ap\r\n")
    sqlString.AppendStr("join assessment_plans aps on ap.id = aps.id and aps.assessment_appraise_id = " + _ass_id + "\r\n")
    sqlString.AppendStr("select cs.id, cs.fullname, cs.position_name, cs.email, c.data.value('(collaborator/custom_elems/custom_elem[name=''f_ued_subdivision_boss'']/value)[1]', 'varchar(max)') [f_ued_subdivision_boss]\r\n")
    sqlString.AppendStr("into #tt1\r\n")
    sqlString.AppendStr("from collaborator c\r\n")
    sqlString.AppendStr("join collaborators cs on cs.id = c.id and cs.is_dismiss != 1\r\n")
    sqlString.AppendStr("where c.data.value('(collaborator/custom_elems/custom_elem[name=''ued_boss'']/value)[1]', 'varchar(max)') = 'true'\r\n")
    sqlString.AppendStr("select f_cte.f_ued_main_subdivision name, s_cte.fullname, s_cte.position_name, s_cte.email\r\n")
    sqlString.AppendStr("from #tt f_cte\r\n")
    sqlString.AppendStr("left join #tt1 s_cte on s_cte.f_ued_subdivision_boss like ('%' + f_cte.f_ued_main_subdivision + '%')");
    //alert(sqlString.GetStr())
    for (elem in XQuery("sql:" + sqlString.GetStr())) {
        returnVal.SetProperty(elem.name, elem)
    }
    //returnVal = XQuery("sql:" + sqlString.GetStr());
    //alert(ArrayCount(returnVal));
    //alert(tools.array_to_text(returnVal, 'json'));
    //alert(EncodeJson(returnVal));
    return returnVal;
}
 
function getMainSubd(_plan_id) {
    var subdValue = ArrayOptFind(mainSubd, ("This.id == " + _plan_id));
    var _val, findSub;
    if (subdValue == undefined) {
        _val = String(tools.open_doc(_plan_id).TopElem.custom_elems.ObtainChildByKey('f_ued_main_subdivision').value);
        findSub = uedMainSubdArray.GetOptProperty(_val);
        //findSub = undefined;
        if (findSub != undefined && findSub.fullname != undefined)
            subdValue = {id: Int(_plan_id), name: _val, fullname: findSub.fullname, position: findSub.position_name, email: findSub.email};
        else
            subdValue = {id: Int(_plan_id), name: _val, fullname: '', position: '', email: ''};
        mainSubd.push(subdValue);
    }
    return subdValue;
}
function getCompProfile(_plan_id) {
    var subdValue = ArrayOptFind(competenceProfiles, ("This.id == " + _plan_id));
    if (subdValue == undefined) {
        subdValue = {id: Int(_plan_id), name: ArrayOptFirstElem(XQuery("for $elem in pas where $elem/assessment_appraise_type='competence_appraisal' and $elem/assessment_plan_id=" + _plan_id + " return $elem")).competence_profile_id.ForeignElem.name};
        competenceProfiles.push(subdValue);
    }
    return subdValue.name;
}
function getExpert(_plan_id){
    var subdValue = ArrayOptFind(expertArray, ("This.id == " + _plan_id));
    if (subdValue == undefined) {
        var planDoc = tools.open_doc(_plan_id).TopElem;
        var expert = ArrayOptFirstElem(planDoc.custom_experts);
        if (expert != undefined) {
            try {
                subdValue = {id: Int(_plan_id), expert_id: Int(expert.person_id), expert_fullname: String(expert.person_id.ForeignElem.fullname), expert_email: String(expert.person_id.ForeignElem.email), expert_current_state: String(tools.open_doc(expert.person_id).TopElem.custom_elems.ObtainChildByKey("CurrentState").value)}
            }
            catch(_e) {
                subdValue = {id: Int(_plan_id), expert_id: 0, expert_fullname: "", expert_email: "", expert_current_state: ""}
            }
        }
        else {
            subdValue = {id: Int(_plan_id), expert_id: 0, expert_fullname: "", expert_email: "", expert_current_state: ""}
        }
        expertArray.push(subdValue);
    }
    return subdValue;
}
function getAllComment(_plan_id) {
    var subdValue = new Object();
    subdValue.id = _plan_id;
    subdValue.string = "";
    subdValue.comments = [];
    if (!checkHasComment(_plan_id)) {
        subdValue = ArrayOptFind(commentArray, ("This.id == " + _plan_id));
        var subArray = [];
        var elem, elemDoc, _c_c, _fullname;
        if (subdValue == undefined) {
            subdValue = new Object();
            subdValue.id = _plan_id;
            subdValue.string = "";
            subdValue.comments = [];
            subArray = XQuery("for $elem in pas where $elem/assessment_plan_id=" + _plan_id + " return $elem");
            for (elem in subArray) {
                elemDoc = tools.open_doc(elem.id).TopElem;
                for (_c_c in elemDoc.custom_comments) {
                    _fullname = "";
                    try {
                        _fullname = tools.open_doc(_c_c.person_id).TopElem.fullname;
                    }
                    catch(_e) {
                        _fullname = _c_c.person_id
                    }
                    subdValue.comments.push({fullname: String(_fullname), date: _c_c.comment_date, comment: String(_c_c.comment)});
                }
            }
            /*subArray = XQuery("for $elem in pas where $elem/assessment_plan_id=" + _plan_id + " return $elem");
            for (elem in subArray) {
                elemDoc = tools.open_doc(elem.id).TopElem;
                for (_c_c in elemDoc.custom_comments) {
                    subdValue.comments.push({fullname: String(tools.open_doc(_c_c.person_id).TopElem.fullname), date: _c_c.comment_date, comment: String(_c_c.comment)});
                }
            }*/
            subdValue.comments = ArraySort(subdValue.comments, "date", "+");
            subdValue.string = ArrayMerge(subdValue.comments, "StrDate(This.date, true, false) + ' ' + This.fullname + ': ' + This.comment", "<br style=\"mso-data-placement:same-cell;\"/>")
            commentArray.push(subdValue);
        }
    }
    return subdValue;
}
var commentCheckArray = new Array();
function checkHasComment(_plan_id) {
    if (ArrayOptFind(commentCheckArray, "This == _plan_id") == undefined) {
        commentCheckArray.push(_plan_id);
        return false;
    }
    return true;
}
 
ERROR = 0;
MESSAGE = "";
RESULT = new Object();
RESULT.ResultAction = "HIDE=WaitWindow;";
try {
               CONTEXT = tools.read_object(CONTEXT);
               assessment_appraise_id = Int(CONTEXT.GetOptProperty("assessment_appraise_id", "0"));
 
    competence_profile = CONTEXT.GetOptProperty("competence_profile", "all");
 
               competence_self = tools_web.is_true(CONTEXT.GetOptProperty("competence_self", "false"));
               competence_manager = tools_web.is_true(CONTEXT.GetOptProperty("competence_manager", "false"));
               competence_result = tools_web.is_true(CONTEXT.GetOptProperty("competence_result", "false"));
 
               activity_appraisal_task_name = tools_web.is_true(CONTEXT.GetOptProperty("activity_appraisal_task_name", "false"));
               activity_appraisal_task_plan = tools_web.is_true(CONTEXT.GetOptProperty("activity_appraisal_task_plan", "false"));
               activity_appraisal_task_plan_ext = tools_web.is_true(CONTEXT.GetOptProperty("activity_appraisal_task_plan_ext", "false"));
 
    staffrating_task_name = tools_web.is_true(CONTEXT.GetOptProperty("staffrating_task_name", "false"));
    staffrating_task_weight = tools_web.is_true(CONTEXT.GetOptProperty("staffrating_task_weight", "false"));
    staffrating_task_date_plan = tools_web.is_true(CONTEXT.GetOptProperty("staffrating_task_date_plan", "false"));
    staffrating_task_plan = tools_web.is_true(CONTEXT.GetOptProperty("staffrating_task_plan", "false"));
    staffrating_task_criterion = tools_web.is_true(CONTEXT.GetOptProperty("staffrating_task_criterion", "false"));
    staffrating_task_coll_exec = tools_web.is_true(CONTEXT.GetOptProperty("staffrating_task_coll_exec", "false"));
    staffrating_task_coll_comment = tools_web.is_true(CONTEXT.GetOptProperty("staffrating_task_coll_comment", "false")); 
    staffrating_task_boss_exec = tools_web.is_true(CONTEXT.GetOptProperty("staffrating_task_boss_exec", "false"));
    staffrating_task_boss_comment = tools_web.is_true(CONTEXT.GetOptProperty("staffrating_task_boss_comment", "false"));
    staffrating_task_expert_comment = tools_web.is_true(CONTEXT.GetOptProperty("staffrating_task_expert_comment", "false"));
 
    is_dismiss = CONTEXT.GetOptProperty("is_dismiss", "all");
   
    comment_all = tools_web.is_true(CONTEXT.GetOptProperty("comment_all", "false"));
 
    boss_id_name = tools.read_object(CONTEXT.GetOptProperty("boss_id_name", "[]"));
    boss_id_name_count = ArrayCount(boss_id_name);
    if (boss_id_name_count > 0)
        boss_id_name = ArrayExtract(boss_id_name, "id");
 
    manager_filter_type = CONTEXT.GetOptProperty("manager_filter_type", "or");
 
    expert_id_name = tools.read_object(CONTEXT.GetOptProperty("expert_id_name", "[]"));
    expert_id_name_count = ArrayCount(expert_id_name);
    if (expert_id_name_count > 0)
        expert_id_name = ArrayExtract(expert_id_name, "id");
 
    main_subdivision_name = tools.read_object(CONTEXT.GetOptProperty("main_subdivision_name", "[]"));
    main_subdivision_name_count = ArrayCount(main_subdivision_name);
    if (main_subdivision_name_count > 0)
        main_subdivision_name = ArrayExtract(main_subdivision_name, "id");
              
    colArray = [];
               colArray.push({title: "ФИО", field: "person_fullname"});
               colArray.push({title: "Email оцениваемого", field: "person_email"});
               colArray.push({title: "Организация оцениваемого", field: "person_org_name"});
               colArray.push({title: "Укрупненное подразделение", field: "main_subdivision"});
               colArray.push({title: "Укрупненное подразделение. Руководитель. ФИО", field: "main_subdivision_fullname"});
               colArray.push({title: "Укрупненное подразделение. Руководитель. Должность", field: "main_subdivision_position"});
               colArray.push({title: "Укрупненное подразделение. Руководитель. Email", field: "main_subdivision_email"});
               colArray.push({title: "ОргСтруктура: Уровень 1", field: "org_level_0"});
               colArray.push({title: "ОргСтруктура: Уровень 2", field: "org_level_1"});
               colArray.push({title: "Текущее подразделение", field: "person_current_subdivision_name"});
               colArray.push({title: "Сотрудник уволен", field: "person_is_dismiss"});
               colArray.push({title: "Текущий статус сотрудника", field: "person_current_state"});
               colArray.push({title: "ФИО оценивающего руководителя", field: "boss_fullname"});
               colArray.push({title: "Email оценивающего руководителя", field: "boss_email"});
    colArray.push({title: "Текущий статус оценивающего руководителя", field: "boss_current_state"});
               colArray.push({title: "ФИО согласующего руководителя", field: "expert_fullname"});
               colArray.push({title: "Email согласующего руководителя", field: "expert_email"});
    colArray.push({title: "Текущий статус согласующего руководителя", field: "expert_current_state"});
               colArray.push({title: "Профиль компетенций", field: "competence_profile_name"});
               colArray.push({title: "Этап документооборота", field: "workflow_state_name"});
 
    if ((staffrating_task_name || staffrating_task_weight || staffrating_task_date_plan || staffrating_task_plan || staffrating_task_criterion || staffrating_task_coll_exec || staffrating_task_coll_comment || staffrating_task_boss_exec || staffrating_task_boss_comment || staffrating_task_expert_comment) || (activity_appraisal_task_name || activity_appraisal_task_plan || activity_appraisal_task_plan_ext))
                   colArray.push({title: "Тип цели", field: "task_type"});  
    if (staffrating_task_name)
               colArray.push({title: "Цель деятельности: Номер", field: "staffrating_task_name"});
    if (staffrating_task_weight)
               colArray.push({title: "Цель деятельности: Вес", field: "staffrating_task_weight"});
    if (staffrating_task_date_plan)
              colArray.push({title: "Цель деятельности: планируемая дата", field: "staffrating_task_date_plan"});
    if (staffrating_task_plan)
                   colArray.push({title: "Цель деятельности: план достижения", field: "staffrating_task_plan"});
    if (staffrating_task_criterion)
                   colArray.push({title: "Цель деятельности: критерии достижения", field: "staffrating_task_criterion"});
    if (staffrating_task_coll_exec)
                   colArray.push({title: "Цель деятельности: Самооценка", field: "staffrating_task_coll_exec_status"});
    if (staffrating_task_coll_comment)
                   colArray.push({title: "Цель деятельности: Самооценка комментарий", field: "staffrating_task_coll_exec_status"});
    if (staffrating_task_boss_exec)
                   colArray.push({title: "Цель деятельности: Оценка руководителя", field: "staffrating_task_boss_exec_status"});
    if (staffrating_task_boss_comment)
                   colArray.push({title: "Цель деятельности: Оценка руководителя Комментарий", field: "staffrating_task_boss_exec_comment"});
    if (staffrating_task_expert_comment)
                   colArray.push({title: "Цель деятельности: Комментарий согласующего руководителя", field: "staffrating_task_expert_comment"});
    if (activity_appraisal_task_name)
                   colArray.push({title: "Цель развития: развиваемая компетенция", field: "activity_appraisal_task_name"});
    if (activity_appraisal_task_plan)
                   colArray.push({title: "Цель развития: Способ развития", field: "activity_appraisal_task_plan"});
    if (activity_appraisal_task_plan_ext) {
        colArray.push({title: "Цель развития: План - 10% (программы обучения)", field: "activity_appraisal_task_plan_education"});
        colArray.push({title: "Цель развития: План - 10% (внешния программа обучения. Название)", field: "activity_appraisal_task_plan_ext_education"});
        colArray.push({title: "Цель развития: План - 10% (внешния программа обучения. Провайдер)", field: "activity_appraisal_task_plan_ext_education_provider"});
        colArray.push({title: "Цель развития: План - 10% (внешния программа обучения. Стоимость)", field: "activity_appraisal_task_plan_ext_education_cost"});
        colArray.push({title: "Цель развития: План - 10% (внешния программа обучения. Приоритет)", field: "activity_appraisal_task_plan_ext_education_priority"});
        colArray.push({title: "Цель развития: План - 10%", field: "activity_appraisal_task_plan_10"});
        colArray.push({title: "Цель развития: План - 20%", field: "activity_appraisal_task_plan_20"});
        colArray.push({title: "Цель развития: План - 70%", field: "activity_appraisal_task_plan_70"});
    }
               if (competence_self)
                              colArray.push({title: "Дорожим клиентом: Самооценка", field: "comp_1_self"});
               if (competence_manager)
                              colArray.push({title: "Дорожим клиентом: Оценка Руководителя", field: "comp_1_manager"});
               if (competence_self)
                              colArray.push({title: "Работаем в команде: Самооценка", field: "comp_2_self"});
               if (competence_manager)
                              colArray.push({title: "Работаем в команде: Оценка руководителя", field: "comp_2_manager"});
               if (competence_self)
                              colArray.push({title: "Отвечаем за результат: Самооценка", field: "comp_3_self"});
               if (competence_manager)
                              colArray.push({title: "Отвечаем за результат: Оценка руководителя", field: "comp_3_manager"});
               if (competence_self)
                              colArray.push({title: "Проявляем инициативу: Самооценка", field: "comp_4_self"});
               if (competence_manager)
                              colArray.push({title: "Проявляем инициативу: Оценка руководителя", field: "comp_4_manager"});
               if (competence_self)
                              colArray.push({title: "Совершенствуемся постоянно: Самооценка", field: "comp_5_self"});
               if (competence_manager)
                              colArray.push({title: "Совершенствуемся постоянно: Оценка руководителя", field: "comp_5_manager"});
               if (competence_result)
                              colArray.push({title: "Итоговая оценка", field: "integral_mark"});
    if (comment_all)
        colArray.push({title: "Общие комментарии к форме", field: "comment_all"});
 
    uedMainSubdArray = getSubsArray(assessment_appraise_id);
              
               sqlString = new Binary();
               sqlString.AppendStr("DECLARE @ass_app_id BIGINT =  '"+assessment_appraise_id+"';\r\n");
               sqlString.AppendStr("IF OBJECT_ID('tempdb..#tt') IS NOT NULL DROP TABLE [#tt];\r\n");
 
    sqlString.AppendStr("IF OBJECT_ID('tempdb..#tt1') IS NOT NULL DROP TABLE [#tt1];\r\n")
    sqlString.AppendStr("IF OBJECT_ID('tempdb..#tt2') IS NOT NULL DROP TABLE [#tt2];\r\n")
    sqlString.AppendStr("IF OBJECT_ID('tempdb..#tt3') IS NOT NULL DROP TABLE [#tt3];\r\n")
    sqlString.AppendStr("select distinct ap.data.value('(assessment_plan/custom_elems/custom_elem[name=''f_ued_main_subdivision'']/value)[1]', 'varchar(max)') [f_ued_main_subdivision]\r\n")
    sqlString.AppendStr("into #tt1\r\n")
    sqlString.AppendStr("from assessment_plan ap\r\n")
    sqlString.AppendStr("join assessment_plans aps on ap.id = aps.id and aps.assessment_appraise_id =  @ass_app_id\r\n")
    sqlString.AppendStr("select cs.id, cs.fullname, cs.position_name, cs.email, c.data.value('(collaborator/custom_elems/custom_elem[name=''f_ued_subdivision_boss'']/value)[1]', 'varchar(max)') [f_ued_subdivision_boss]\r\n")
    sqlString.AppendStr("into #tt2\r\n")
    sqlString.AppendStr("from collaborator c\r\n")
    sqlString.AppendStr("join collaborators cs on cs.id = c.id and cs.is_dismiss != 1\r\n")
    sqlString.AppendStr("where c.data.value('(collaborator/custom_elems/custom_elem[name=''ued_boss'']/value)[1]', 'varchar(max)') = 'true'\r\n")
    sqlString.AppendStr("select f_cte.f_ued_main_subdivision m_s_name, s_cte.fullname main_subdivision_fullname, s_cte.position_name main_subdivision_position, s_cte.email main_subdivision_email\r\n")
    sqlString.AppendStr("into #tt3\r\n")
    sqlString.AppendStr("from #tt1 f_cte\r\n")
    sqlString.AppendStr("left join #tt2 s_cte on (s_cte.f_ued_subdivision_boss = f_cte.f_ued_main_subdivision or s_cte.f_ued_subdivision_boss like (f_cte.f_ued_main_subdivision + ';%') or s_cte.f_ued_subdivision_boss like ('%;' + f_cte.f_ued_main_subdivision + ';%') or s_cte.f_ued_subdivision_boss like ('%;' + f_cte.f_ued_main_subdivision))\r\n");
    if (main_subdivision_name_count > 0) {
        sqlString.AppendStr("where f_cte.f_ued_main_subdivision in ('" + ArrayMerge(main_subdivision_name, "This", "','") + "')\r\n")
    }
               sqlString.AppendStr("SELECT *\r\n");
               sqlString.AppendStr("INTO [#tt]\r\n");
               sqlString.AppendStr("FROM (\r\n");
    if ((staffrating_task_name || staffrating_task_weight || staffrating_task_date_plan || staffrating_task_plan || staffrating_task_criterion || staffrating_task_coll_exec || staffrating_task_coll_comment || staffrating_task_boss_exec || staffrating_task_boss_comment || staffrating_task_expert_comment) || (activity_appraisal_task_name || activity_appraisal_task_plan || activity_appraisal_task_plan_ext)) {
        if (staffrating_task_name || staffrating_task_weight || staffrating_task_date_plan || staffrating_task_plan || staffrating_task_criterion || staffrating_task_coll_exec || staffrating_task_boss_exec || staffrating_task_boss_comment || staffrating_task_expert_comment) {
            sqlString.AppendStr("       SELECT [assessment_plans].[id]\r\n");
            if (competence_result)
                sqlString.AppendStr("                  ,[assessment_plans].[integral_mark]\r\n");
            //sqlString.AppendStr("                   ,[assessment_plans].[person_fullname]\r\n");
            sqlString.AppendStr("                      ,[collaborators].[fullname] [person_fullname]\r\n");
            sqlString.AppendStr("                      ,[assessment_plan].[data].value('(assessment_plan/workflow_state_name)[1]', 'varchar(max)') [workflow_state_name]\r\n");
            sqlString.AppendStr("       ,[assessment_plan].[data].value('(assessment_plan/custom_elems/custom_elem[name=''f_ued_main_subdivision'']/value)[1]', 'varchar(max)') [main_subdivision]\r\n");
            sqlString.AppendStr("                      ,[collaborators].[email] [person_email]\r\n");
            sqlString.AppendStr("                      ,[collaborators].[org_name] [person_org_name]\r\n");
            sqlString.AppendStr("                      ,[collaborators].[position_parent_id] [person_subdivision_id]\r\n");
            sqlString.AppendStr("                      ,[collaborators].[position_parent_name] [person_current_subdivision_name]\r\n");
            sqlString.AppendStr("                      ,[collaborators].[is_dismiss] [person_is_dismiss]\r\n");
            sqlString.AppendStr("       ,[collaborator].[data].value('(collaborator/custom_elems/custom_elem[name=''CurrentState'']/value)[1]', 'varchar(max)') [person_current_state]\r\n");
            sqlString.AppendStr("                      ,[assessment_plans].[boss_id]\r\n");
            sqlString.AppendStr("                      ,[boss_collaborators].[fullname] [boss_fullname]\r\n");
            sqlString.AppendStr("                      ,[boss_collaborators].[email] [boss_email]\r\n");
            sqlString.AppendStr("       ,[boss_collaborator].[data].value('(collaborator/custom_elems/custom_elem[name=''CurrentState'']/value)[1]', 'varchar(max)') [boss_current_state]\r\n");
            sqlString.AppendStr("                      ,[staffrating_task].[pas_id]\r\n");
            sqlString.AppendStr("                      ,[staffrating_task].[task_type]\r\n");
            if (staffrating_task_name)
                sqlString.AppendStr("                  ,[staffrating_task].[staffrating_task_name]\r\n");
            if (staffrating_task_weight)
            sqlString.AppendStr("                      ,[staffrating_task].[staffrating_task_weight]\r\n");
            if (staffrating_task_plan)
                sqlString.AppendStr("                  ,[staffrating_task].[staffrating_task_plan]\r\n");
            if (staffrating_task_criterion)
                sqlString.AppendStr("                  ,[staffrating_task].[staffrating_task_criterion]\r\n");
            if (staffrating_task_date_plan)
                sqlString.AppendStr("                  ,[staffrating_task].[staffrating_task_date_plan]\r\n");
            if (activity_appraisal_task_name)
                sqlString.AppendStr("                  ,NULL [activity_appraisal_task_id]\r\n");
                sqlString.AppendStr("                  ,NULL [activity_appraisal_task_name]\r\n");
            if (activity_appraisal_task_plan)
                sqlString.AppendStr("                  ,NULL [activity_appraisal_task_plan]\r\n");
            if (activity_appraisal_task_plan_ext) {
                sqlString.AppendStr("                  ,NULL [activity_appraisal_task_plan_education]\r\n");
                sqlString.AppendStr("                  ,NULL [activity_appraisal_task_plan_10]\r\n");
                sqlString.AppendStr("                  ,NULL [activity_appraisal_task_plan_20]\r\n");
                sqlString.AppendStr("                  ,NULL [activity_appraisal_task_plan_70]\r\n");
            }
            if (staffrating_task_coll_exec)
                sqlString.AppendStr("                  ,[staffrating_task].[staffrating_task_coll_exec_status]\r\n");
            if (staffrating_task_coll_comment)
                sqlString.AppendStr("                  ,[staffrating_task].[staffrating_task_coll_exec_comment]\r\n");
            if (staffrating_task_boss_exec)
                sqlString.AppendStr("                  ,[staffrating_task].[staffrating_task_boss_exec_status]\r\n");
            if (staffrating_task_boss_comment)
                sqlString.AppendStr("                  ,[staffrating_task].[staffrating_task_boss_exec_comment]\r\n");
            if (staffrating_task_expert_comment)
                sqlString.AppendStr("                  ,[staffrating_task].[staffrating_task_expert_comment]\r\n");
            if (false) {
                sqlString.AppendStr("                  ,NULL [activity_appraisal_task_coll_exec_status]\r\n");
                sqlString.AppendStr("                  ,NULL [activity_appraisal_task_coll_exec_comment]\r\n");
                sqlString.AppendStr("                  ,NULL [activity_appraisal_task_boss_exec_status]\r\n");
                sqlString.AppendStr("                  ,NULL [activity_appraisal_task_boss_exec_comment]\r\n");
                sqlString.AppendStr("                  ,NULL [activity_appraisal_task_view_coll_exec_comment]\r\n");
            }
            sqlString.AppendStr("       FROM [assessment_plans]\r\n");
            sqlString.AppendStr("       INNER JOIN [assessment_plan]\r\n");
            sqlString.AppendStr("                      ON [assessment_plan].[id] = [assessment_plans].[id]\r\n");
            sqlString.AppendStr("                                     AND [assessment_plans].[assessment_appraise_id] = @ass_app_id\r\n");
            sqlString.AppendStr("       LEFT JOIN (\r\n");
            sqlString.AppendStr("                      SELECT [pas].[assessment_plan_id]\r\n");
            sqlString.AppendStr("                                     ,[pas].[id] [pas_id]\r\n");
            sqlString.AppendStr("                                     ,[staffrating_task].[data].value('weight', 'varchar(max)') [staffrating_task_weight]\r\n");
            sqlString.AppendStr("                                     ,'Цель деятельности' [task_type]\r\n");
            if (staffrating_task_name)
                sqlString.AppendStr("                                 ,[task].[data].value('(task/name)[1]', 'varchar(max)') [staffrating_task_name]\r\n");
            if (staffrating_task_plan)
                sqlString.AppendStr("                                 ,[task].[data].value('(task/plan)[1]', 'varchar(max)') [staffrating_task_plan]\r\n");
            if (staffrating_task_criterion)
                sqlString.AppendStr("                                 ,[task].[data].value('(task/custom_fields/custom_field[name=''plan_goal'']/value)[1]', 'varchar(max)') [staffrating_task_criterion]\r\n");
            if (staffrating_task_date_plan)
                sqlString.AppendStr("                                 ,[task].[data].value('(task/date_plan)[1]', 'datetime') [staffrating_task_date_plan]\r\n");
            if (staffrating_task_coll_exec)
                sqlString.AppendStr("                         ,[task].[data].value('(task/custom_fields/custom_field[name=''coll_exec_status'']/value)[1]', 'varchar(max)') [staffrating_task_coll_exec_status]\r\n");
            if (staffrating_task_coll_comment)
                sqlString.AppendStr("                  ,[task].[data].value('(task/custom_fields/custom_field[name=''coll_exec_comment'']/value)[1]', 'varchar(max)') [staffrating_task_coll_exec_comment]\r\n");
            if (staffrating_task_boss_exec)
                sqlString.AppendStr("                       ,[task].[data].value('(task/custom_fields/custom_field[name=''boss_exec_status'']/value)[1]', 'varchar(max)') [staffrating_task_boss_exec_status]\r\n");
            if (staffrating_task_boss_comment)
                sqlString.AppendStr("                  ,[task].[data].value('(task/custom_fields/custom_field[name=''boss_exec_comment'']/value)[1]', 'varchar(max)') [staffrating_task_boss_exec_comment]\r\n");
            if (staffrating_task_expert_comment)
                sqlString.AppendStr("                  ,[task].[data].value('(task/custom_fields/custom_field[name=''view_coll_exec_comment'']/value)[1]', 'varchar(max)') [staffrating_task_expert_comment]\r\n");
            sqlString.AppendStr("                      FROM [pas]\r\n");
            sqlString.AppendStr("                      INNER JOIN [pa]\r\n");
            sqlString.AppendStr("                                     ON [pa].[id] = [pas].[id]\r\n");
            sqlString.AppendStr("                                                    AND [pas].[status] = 'manager'\r\n");
            sqlString.AppendStr("                                                    AND [pas].[assessment_appraise_type] = 'staffrating'\r\n");
            sqlString.AppendStr("                                                    AND [pas].[assessment_appraise_id] = @ass_app_id\r\n");
            sqlString.AppendStr("                      OUTER APPLY [pa].[data].nodes('pa/tasks/task') [staffrating_task](data)\r\n");
            sqlString.AppendStr("                      LEFT JOIN [task]\r\n");
            sqlString.AppendStr("                                     ON [task].[id] = [staffrating_task].[data].value('task_id', 'bigint')\r\n");
            sqlString.AppendStr("                      WHERE [task].[data].value('(task/custom_fields/custom_field[name=''not_actual'']/value)[1]', 'varchar(max)') is null\r\n");
            sqlString.AppendStr("                      ) [staffrating_task]\r\n");
            sqlString.AppendStr("                      ON [staffrating_task].[assessment_plan_id] = [assessment_plans].[id]\r\n");
            sqlString.AppendStr("       LEFT JOIN [collaborators]\r\n");
            sqlString.AppendStr("                      ON [collaborators].[id] = [assessment_plans].[person_id]\r\n");
            sqlString.AppendStr("       LEFT JOIN [collaborator]\r\n");
            sqlString.AppendStr("                      ON [collaborator].[id] = [assessment_plans].[person_id]\r\n");
            sqlString.AppendStr("       LEFT JOIN [collaborators] [boss_collaborators]\r\n");
            sqlString.AppendStr("                      ON [boss_collaborators].[id] = [assessment_plans].[boss_id]\r\n");
            sqlString.AppendStr("       LEFT JOIN [collaborator] [boss_collaborator]\r\n");
            sqlString.AppendStr("                      ON [boss_collaborator].[id] = [assessment_plans].[boss_id]\r\n");
            sqlString.AppendStr("       WHERE [assessment_plans].[assessment_appraise_id] = @ass_app_id\r\n");
            if (is_dismiss != "all")
                sqlString.AppendStr("   AND [collaborators].[is_dismiss] = " + OptInt(is_dismiss, 0)  + "\r\n");
        }
        if ((staffrating_task_name || staffrating_task_weight || staffrating_task_date_plan || staffrating_task_plan || staffrating_task_criterion || staffrating_task_coll_exec || staffrating_task_boss_exec || staffrating_task_boss_comment || staffrating_task_expert_comment) && (activity_appraisal_task_name || activity_appraisal_task_plan || activity_appraisal_task_plan_ext)) {
            sqlString.AppendStr("       UNION\r\n");
        }
        if (activity_appraisal_task_name || activity_appraisal_task_plan || activity_appraisal_task_plan_ext) {
            sqlString.AppendStr("       SELECT [assessment_plans].[id]\r\n");
            if (competence_result)
                sqlString.AppendStr("                  ,[assessment_plans].[integral_mark]\r\n");
            //sqlString.AppendStr("                   ,[assessment_plans].[person_fullname]\r\n");
            sqlString.AppendStr("                      ,[collaborators].[fullname] [person_fullname]\r\n");
            sqlString.AppendStr("                      ,[assessment_plan].[data].value('(assessment_plan/workflow_state_name)[1]', 'varchar(max)') [workflow_state_name]\r\n");
            sqlString.AppendStr("       ,[assessment_plan].[data].value('(assessment_plan/custom_elems/custom_elem[name=''f_ued_main_subdivision'']/value)[1]', 'varchar(max)') [main_subdivision]\r\n");
            sqlString.AppendStr("                      ,[collaborators].[email] [person_email]\r\n");
            sqlString.AppendStr("                      ,[collaborators].[org_name] [person_org_name]\r\n");
            sqlString.AppendStr("                      ,[collaborators].[position_parent_id] [person_subdivision_id]\r\n");
            sqlString.AppendStr("                      ,[collaborators].[position_parent_name] [person_current_subdivision_name]\r\n");
            sqlString.AppendStr("                      ,[collaborators].[is_dismiss] [person_is_dismiss]\r\n");
            sqlString.AppendStr("       ,[collaborator].[data].value('(collaborator/custom_elems/custom_elem[name=''CurrentState'']/value)[1]', 'varchar(max)') [person_current_state]\r\n");
            sqlString.AppendStr("                      ,[assessment_plans].[boss_id]\r\n");
            sqlString.AppendStr("                      ,[boss_collaborators].[fullname] [boss_fullname]\r\n");
            sqlString.AppendStr("                      ,[boss_collaborators].[email] [boss_email]\r\n");
            sqlString.AppendStr("       ,[boss_collaborator].[data].value('(collaborator/custom_elems/custom_elem[name=''CurrentState'']/value)[1]', 'varchar(max)') [boss_current_state]\r\n");
           sqlString.AppendStr("                      ,[activity_appraisal_task].[pas_id]\r\n");
            sqlString.AppendStr("                      ,[activity_appraisal_task].[task_type]\r\n");
            if (staffrating_task_name)
                sqlString.AppendStr("                  ,NULL [staffrating_task_name]\r\n");
            if (staffrating_task_weight)
                sqlString.AppendStr("                  ,NULL [staffrating_task_weight]\r\n");
            if (staffrating_task_plan)
                sqlString.AppendStr("                  ,NULL [staffrating_task_plan]\r\n");
            if (staffrating_task_criterion)
                sqlString.AppendStr("                  ,NULL [staffrating_task_criterion]\r\n");
            if (staffrating_task_date_plan)
                sqlString.AppendStr("                  ,NULL [staffrating_task_date_plan]\r\n");
            if (activity_appraisal_task_name)
                sqlString.AppendStr("                  ,[activity_appraisal_task].[activity_appraisal_task_id]\r\n");
                sqlString.AppendStr("                  ,[activity_appraisal_task].[activity_appraisal_task_name]\r\n");
            if (activity_appraisal_task_plan)
                sqlString.AppendStr("                  ,[activity_appraisal_task].[activity_appraisal_task_plan]\r\n");
            if (activity_appraisal_task_plan_ext) {
                sqlString.AppendStr("                  ,[activity_appraisal_task].[activity_appraisal_task_plan_education]\r\n");
                sqlString.AppendStr("                  ,[activity_appraisal_task].[activity_appraisal_task_plan_10]\r\n");
                sqlString.AppendStr("                  ,[activity_appraisal_task].[activity_appraisal_task_plan_20]\r\n");
                sqlString.AppendStr("                  ,[activity_appraisal_task].[activity_appraisal_task_plan_70]\r\n");
            }
            if (staffrating_task_coll_exec)
                sqlString.AppendStr("                  ,NULL [staffrating_task_coll_exec_status]\r\n");
            if (staffrating_task_coll_comment)
                sqlString.AppendStr("                  ,NULL [staffrating_task_coll_exec_comment]\r\n");
            if (staffrating_task_boss_exec)
                sqlString.AppendStr("                  ,NULL [staffrating_task_boss_exec_status]\r\n");
            if (staffrating_task_boss_comment)
                sqlString.AppendStr("                  ,NULL [staffrating_task_boss_exec_comment]\r\n");
            if (staffrating_task_expert_comment)
                sqlString.AppendStr("                  ,NULL [staffrating_task_expert_comment]\r\n");
            if (false) {
                sqlString.AppendStr("                  ,[activity_appraisal_task].[activity_appraisal_task_coll_exec_status]\r\n");
                sqlString.AppendStr("                  ,[activity_appraisal_task].[activity_appraisal_task_coll_exec_comment]\r\n");
                sqlString.AppendStr("                  ,[activity_appraisal_task].[activity_appraisal_task_boss_exec_status]\r\n");
                sqlString.AppendStr("                  ,[activity_appraisal_task].[activity_appraisal_task_boss_exec_comment]\r\n");
                sqlString.AppendStr("                  ,[activity_appraisal_task].[activity_appraisal_task_view_coll_exec_comment]\r\n");
            }
            sqlString.AppendStr("       FROM [assessment_plans]\r\n");
            sqlString.AppendStr("       INNER JOIN [assessment_plan]\r\n");
            sqlString.AppendStr("                      ON [assessment_plan].[id] = [assessment_plans].[id]\r\n");
            sqlString.AppendStr("                                     AND [assessment_plans].[assessment_appraise_id] = @ass_app_id\r\n");
            sqlString.AppendStr("       LEFT JOIN (\r\n");
            sqlString.AppendStr("                      SELECT [pas].[assessment_plan_id]\r\n");
            sqlString.AppendStr("                                     ,[pas].[id] [pas_id]\r\n");
            sqlString.AppendStr("                                     ,'Цель развития' [task_type]\r\n");
            if (activity_appraisal_task_name)
                sqlString.AppendStr("                                 ,[task].[id] [activity_appraisal_task_id]\r\n");
                sqlString.AppendStr("                                 ,[task].[data].value('(task/name)[1]', 'varchar(max)') [activity_appraisal_task_name]\r\n");
            if (activity_appraisal_task_plan)
                sqlString.AppendStr("                                 ,[task].[data].value('(task/plan)[1]', 'varchar(max)') [activity_appraisal_task_plan]\r\n");
            if (activity_appraisal_task_plan_ext) {
                sqlString.AppendStr("                          ,[task].[data].value('(task/custom_fields/custom_field[name=''plan_education'']/value)[1]', 'varchar(max)') [activity_appraisal_task_plan_education]\r\n");
                sqlString.AppendStr("                                 ,[task].[data].value('(task/custom_fields/custom_field[name=''plan2'']/value)[1]', 'varchar(max)') [activity_appraisal_task_plan_10]\r\n");
                sqlString.AppendStr("                                 ,[task].[data].value('(task/custom_fields/custom_field[name=''plan2_20'']/value)[1]', 'varchar(max)') [activity_appraisal_task_plan_20]\r\n");
                sqlString.AppendStr("                                 ,[task].[data].value('(task/custom_fields/custom_field[name=''plan2_70'']/value)[1]', 'varchar(max)') [activity_appraisal_task_plan_70]\r\n");
            }
            if (false) {
                sqlString.AppendStr("                         ,[task].[data].value('(task/custom_fields/custom_field[name=''coll_exec_status'']/value)[1]', 'varchar(max)') [activity_appraisal_task_coll_exec_status]\r\n");
                sqlString.AppendStr("                  ,[task].[data].value('(task/custom_fields/custom_field[name=''coll_exec_comment'']/value)[1]', 'varchar(max)') [activity_appraisal_task_coll_exec_comment]\r\n");
                sqlString.AppendStr("                       ,[task].[data].value('(task/custom_fields/custom_field[name=''boss_exec_status'']/value)[1]', 'varchar(max)') [activity_appraisal_task_boss_exec_status]\r\n");
                sqlString.AppendStr("                  ,[task].[data].value('(task/custom_fields/custom_field[name=''boss_exec_comment'']/value)[1]', 'varchar(max)') [activity_appraisal_task_boss_exec_comment]\r\n");
                sqlString.AppendStr("                  ,[task].[data].value('(task/custom_fields/custom_field[name=''view_coll_exec_comment'']/value)[1]', 'varchar(max)') [activity_appraisal_task_view_coll_exec_comment]\r\n");
            }
            sqlString.AppendStr("                      FROM [pas]\r\n");
            sqlString.AppendStr("                      INNER JOIN [pa]\r\n");
            sqlString.AppendStr("                                     ON [pa].[id] = [pas].[id]\r\n");
            sqlString.AppendStr("                                                    AND [pas].[status] = 'manager'\r\n");
            sqlString.AppendStr("                                                    AND [pas].[assessment_appraise_type] = 'activity_appraisal'\r\n");
            sqlString.AppendStr("                                                    AND [pas].[assessment_appraise_id] = @ass_app_id\r\n");
            sqlString.AppendStr("                      OUTER APPLY [pa].[data].nodes('pa/tasks/task') [activity_appraisal_task](data)\r\n");
            sqlString.AppendStr("                      LEFT JOIN [task]\r\n");
            sqlString.AppendStr("                                     ON [task].[id] = [activity_appraisal_task].[data].value('task_id', 'bigint')\r\n");
            sqlString.AppendStr("                      WHERE [activity_appraisal_task].[data].value('parent_task_id', 'bigint') is null\r\n");
            sqlString.AppendStr("                      --WHERE [task].[data].value('(task/custom_fields/custom_field[name=''not_actual'']/value)[1]', 'varchar(max)') != 'true'\r\n");
            sqlString.AppendStr("                      ) [activity_appraisal_task]\r\n");
            sqlString.AppendStr("                      ON [activity_appraisal_task].[assessment_plan_id] = [assessment_plans].[id]\r\n");
            sqlString.AppendStr("       LEFT JOIN [collaborators]\r\n");
            sqlString.AppendStr("                      ON [collaborators].[id] = [assessment_plans].[person_id]\r\n");
            sqlString.AppendStr("       LEFT JOIN [collaborator]\r\n");
            sqlString.AppendStr("                      ON [collaborator].[id] = [assessment_plans].[person_id]\r\n");
            sqlString.AppendStr("       LEFT JOIN [collaborators] [boss_collaborators]\r\n");
            sqlString.AppendStr("                      ON [boss_collaborators].[id] = [assessment_plans].[boss_id]\r\n");
            sqlString.AppendStr("       LEFT JOIN [collaborator] [boss_collaborator]\r\n");
            sqlString.AppendStr("                      ON [boss_collaborator].[id] = [assessment_plans].[boss_id]\r\n");
            sqlString.AppendStr("       WHERE [assessment_plans].[assessment_appraise_id] = @ass_app_id\r\n");
            if (is_dismiss != "all")
                sqlString.AppendStr("   AND [collaborators].[is_dismiss] = " + OptInt(is_dismiss, 0)  + "\r\n");
        }
    }
    else {
        sqlString.AppendStr("            SELECT [assessment_plans].[id]\r\n");
        if (competence_result)
            sqlString.AppendStr("                      ,[assessment_plans].[integral_mark]\r\n");
        sqlString.AppendStr("                          ,[collaborators].[fullname] [person_fullname]\r\n");
        sqlString.AppendStr("                           ,[assessment_plan].[data].value('(assessment_plan/workflow_state_name)[1]', 'varchar(max)') [workflow_state_name]\r\n");
        sqlString.AppendStr("            ,[assessment_plan].[data].value('(assessment_plan/custom_elems/custom_elem[name=''f_ued_main_subdivision'']/value)[1]', 'varchar(max)') [main_subdivision]\r\n");
        sqlString.AppendStr("                          ,[collaborators].[email] [person_email]\r\n");
        sqlString.AppendStr("                          ,[collaborators].[org_name] [person_org_name]\r\n");
        sqlString.AppendStr("                          ,[collaborators].[position_parent_id] [person_subdivision_id]\r\n");
        sqlString.AppendStr("                          ,[collaborators].[position_parent_name] [person_current_subdivision_name]\r\n");
        sqlString.AppendStr("                          ,[collaborators].[is_dismiss] [person_is_dismiss]\r\n");
        sqlString.AppendStr("            ,[collaborator].[data].value('(collaborator/custom_elems/custom_elem[name=''CurrentState'']/value)[1]', 'varchar(max)') [person_current_state]\r\n");
        sqlString.AppendStr("                          ,[assessment_plans].[boss_id]\r\n");
        sqlString.AppendStr("                          ,[boss_collaborators].[fullname] [boss_fullname]\r\n");
        sqlString.AppendStr("                          ,[boss_collaborators].[email] [boss_email]\r\n");
        sqlString.AppendStr("            ,[boss_collaborator].[data].value('(collaborator/custom_elems/custom_elem[name=''CurrentState'']/value)[1]', 'varchar(max)') [boss_current_state]\r\n");
        sqlString.AppendStr("                          ,'' [task_type]\r\n");
        sqlString.AppendStr("            FROM [assessment_plans]\r\n");
        sqlString.AppendStr("            INNER JOIN [assessment_plan]\r\n");
        sqlString.AppendStr("                          ON [assessment_plan].[id] = [assessment_plans].[id]\r\n");
        sqlString.AppendStr("                                         AND [assessment_plans].[assessment_appraise_id] = @ass_app_id\r\n");
        sqlString.AppendStr("            LEFT JOIN [collaborators]\r\n");
        sqlString.AppendStr("                          ON [collaborators].[id] = [assessment_plans].[person_id]\r\n");
        sqlString.AppendStr("            LEFT JOIN [collaborator]\r\n");
        sqlString.AppendStr("                          ON [collaborator].[id] = [assessment_plans].[person_id]\r\n");
        sqlString.AppendStr("            LEFT JOIN [collaborators] [boss_collaborators]\r\n");
        sqlString.AppendStr("                          ON [boss_collaborators].[id] = [assessment_plans].[boss_id]\r\n");
        sqlString.AppendStr("            LEFT JOIN [collaborator] [boss_collaborator]\r\n");
        sqlString.AppendStr("                          ON [boss_collaborator].[id] = [assessment_plans].[boss_id]\r\n");
        sqlString.AppendStr("            WHERE [assessment_plans].[assessment_appraise_id] = @ass_app_id\r\n");
        if (is_dismiss != "all")
            sqlString.AppendStr("       AND [collaborators].[is_dismiss] = " + OptInt(is_dismiss, 0)  + "\r\n");
    }
    sqlString.AppendStr("                ) [t_1];\r\n");
    sqlString.AppendStr("WITH firstCTE\r\n");
    sqlString.AppendStr("AS (\r\n");
    sqlString.AppendStr("                SELECT [subdivisions].[id]\r\n");
    sqlString.AppendStr("                               ,[subdivisions].[parent_object_id]\r\n");
    sqlString.AppendStr("                               ,cast([subdivisions].[name] AS VARCHAR(max)) [name]\r\n");
    sqlString.AppendStr("                               ,0 AS [parent_level]\r\n");
    sqlString.AppendStr("                FROM [subdivisions]\r\n");
    sqlString.AppendStr("                INNER JOIN [#tt]\r\n");
    sqlString.AppendStr("                               ON [#tt].[person_subdivision_id] = [subdivisions].[id]\r\n");
    sqlString.AppendStr("                \r\n");
    sqlString.AppendStr("                UNION ALL\r\n");
    sqlString.AppendStr("                \r\n");
    sqlString.AppendStr("                SELECT [t_0].[id]\r\n");
    sqlString.AppendStr("                               ,[t_1].[parent_object_id]\r\n");
    sqlString.AppendStr("                               ,cast(CONCAT (\r\n");
    sqlString.AppendStr("                                                            [t_1].[name]\r\n");
    sqlString.AppendStr("                                                            ,'|#|'\r\n");
    sqlString.AppendStr("                                                            ,[t_0].[name]\r\n");
    sqlString.AppendStr("                                                            ) AS VARCHAR(max)) [name]\r\n");
    sqlString.AppendStr("                               ,[parent_level] + 1\r\n");
    sqlString.AppendStr("                FROM [subdivisions] AS [t_1]\r\n");
    sqlString.AppendStr("                INNER JOIN firstCTE AS [t_0]\r\n");
    sqlString.AppendStr("                               ON [t_0].[parent_object_id] = [t_1].[id]\r\n");
    sqlString.AppendStr("                )\r\n");
   
               sqlString.AppendStr("SELECT DISTINCT *\r\n");
               sqlString.AppendStr("FROM [#tt]\r\n");
               sqlString.AppendStr("LEFT JOIN (\r\n");
               sqlString.AppendStr("     SELECT DISTINCT [firstCTE].[id] [subdivision_id]\r\n");
               sqlString.AppendStr("                    ,[firstCTE].[name] [person_subdivision_name]\r\n");
               sqlString.AppendStr("     FROM [firstCTE]\r\n");
               sqlString.AppendStr("     INNER JOIN (\r\n");
               sqlString.AppendStr("                    SELECT [id]\r\n");
               sqlString.AppendStr("                                   ,max([parent_level]) [max_level]\r\n");
               sqlString.AppendStr("                    FROM [firstCTE]\r\n");
               sqlString.AppendStr("                    GROUP BY [id]\r\n");
               sqlString.AppendStr("                    ) [t1]\r\n");
               sqlString.AppendStr("                    ON [t1].[id] = [firstCTE].[id]\r\n");
               sqlString.AppendStr("                                   AND [t1].[max_level] = [firstCTE].[parent_level]\r\n");
               sqlString.AppendStr("     ) [t2]\r\n");
               sqlString.AppendStr("     ON [t2].[subdivision_id] = [#tt].[person_subdivision_id]\r\n");
    sqlString.AppendStr("INNER JOIN [#tt3] on [#tt].[main_subdivision] = [#tt3].[m_s_name]\r\n");
               sqlString.AppendStr("ORDER BY [person_fullname]\r\n");
               sqlString.AppendStr("     ,[task_type]\r\n");
    if (staffrating_task_name)
                   sqlString.AppendStr("                ,[staffrating_task_name]\r\n");
    if (activity_appraisal_task_name)
                   sqlString.AppendStr("                ,[activity_appraisal_task_name]\r\n");
               //EnableLog('shchapov', true);
               alert(sqlString.GetStr())
               var rawResultArray = XQuery("sql:" + sqlString.GetStr());
               var returnArray = new Array();
    /*alert("boss_id_name_count:" + boss_id_name_count);
    if (boss_id_name_count > 0)
        alert("boss_id_name: " + tools.object_to_text(boss_id_name, 'json'));
    alert("expert_id_name_count:" + expert_id_name_count);
    if (expert_id_name_count > 0)
        alert("expert_id_name: " + tools.object_to_text(expert_id_name, 'json'));*/
               for (respElem in rawResultArray) {
        expertElem = getExpert(Int(respElem.id));
        if (manager_filter_type == "and") {
            /*if (boss_id_name_count > 0)
                alert("ArrayOptFind(boss_id_name, \"Int(This) == Int(respElem.boss_id)\") == undefined:" + (ArrayOptFind(boss_id_name, "Int(This) == Int(respElem.boss_id)") == undefined));
            if (expert_id_name_count > 0) {
                alert("expertElem: " + tools.object_to_text(expertElem, 'json'))
                alert("ArrayOptFind(expert_id_name, \"Int(This) == Int(expertElem.id)\") == undefined:" + (ArrayOptFind(expert_id_name, "Int(This) == Int(expertElem.id)") == undefined));
                if (ArrayOptFind(expert_id_name, "Int(This) == Int(expertElem.id)") != undefined)
                    alert('not_continue')
            }*/
            if (boss_id_name_count > 0 && ArrayOptFind(boss_id_name, "Int(This) == Int(respElem.boss_id)") == undefined)
                continue;
            if (expert_id_name_count > 0 && ArrayOptFind(expert_id_name, "Int(This) == Int(expertElem.expert_id)") == undefined)
                continue;
        }
        else {
            if (boss_id_name_count > 0) {
                if (expert_id_name_count > 0) {
                    if (ArrayOptFind(boss_id_name, "Int(This) == Int(respElem.boss_id)") == undefined && ArrayOptFind(expert_id_name, "Int(This) == Int(expertElem.expert_id)") == undefined)
                        continue;
                }
                else {
                    if (ArrayOptFind(boss_id_name, "Int(This) == Int(respElem.boss_id)") == undefined)
                        continue;
                }
            }
            else {
                if (expert_id_name_count > 0 && ArrayOptFind(expert_id_name, "Int(This) == Int(expertElem.expert_id)") == undefined)
                    continue;
            }
        }
        /*if (manager_filter_type == "and") {
            if (boss_id_name_count > 0) {
                if (expert_id_name_count > 0) {
 
                }
                else{
 
                }
            }
            else {
                if
            }
           
        }
        else {
            if ((boss_id_name_count > 0 || expert_id_name_count > 0) && (ArrayOptFind(boss_id_name, "Int(This) == Int(respElem.boss_id)") == undefined && ArrayOptFind(expert_id_name, "Int(This) == Int(expertElem.id)") == undefined)) {
                continue;
            }
        }*/
                              r = new Object();
                              r.SetProperty("PrimaryKey", respElem.id);
                              for (fldRespElem in respElem) {
                                            fldRespElemString = String(fldRespElem);
                                            fldRespElemString = StrReplace(StrReplace(StrReplace(fldRespElemString, '<br', '<br style="mso-data-placement:same-cell;"'), '<div>', '<span>'), '</div>', '</span><br style="mso-data-placement:same-cell;"/>');
                                            switch(fldRespElem.Name) {
                                                           case "staffrating_task_coll_exec_status":
                                                           case "staffrating_task_boss_exec_status":
                                                           case "activity_appraisal_task_coll_exec_status":
                                                            case "activity_appraisal_task_boss_exec_status":
                                                                          r.SetProperty(fldRespElem.Name, getRatingName(fldRespElemString, "text"));
                                                                          break;
                                                           case "integral_mark":
                                                                          r.SetProperty(fldRespElem.Name, getRatingName(fldRespElemString, "integral"));
                                                                          break;
                case "workflow_state_name":
                    r.SetProperty(fldRespElem.Name, (StrLowerCase(fldRespElemString) == "корректировка завершена" || StrLowerCase(fldRespElemString) == "постановка завершена" ? "ЗАВЕРШЕНО" : fldRespElemString));
                    break;
                case "person_is_dismiss":
                    r.SetProperty(fldRespElem.Name, (tools_web.is_true(fldRespElemString) ? "Да" : "Нет"));
                    break;
                case "staffrating_task_date_plan":
                    try {
                        r.SetProperty(fldRespElem.Name, (fldRespElemString != "" ? StrDate(Date(fldRespElemString),false) : ""));
                    }
                    catch(_e) {
                        throw ("error: " + _e + "; value:" + fldRespElemString);
                    }
                    break;
                                                           default:
                                                                          r.SetProperty(fldRespElem.Name, fldRespElemString);
                                                                          break;
                                            }
                              }
                              rh = String(respElem.person_subdivision_name).split('|#|');
                              for (i = 0; i < ArrayCount(rh); i++)
                                            r.SetProperty(("org_level_" + i), rh[i]);
        /*_mainS = getMainSubd(Int(respElem.id));
                              r.SetProperty("main_subdivision", _mainS.name);
                              r.SetProperty("main_subdivision_fullname", _mainS.fullname);
                              r.SetProperty("main_subdivision_position", _mainS.position);
                              r.SetProperty("main_subdivision_email", _mainS.email);*/
                              r.SetProperty("competence_profile_name", getCompProfile(Int(respElem.id)));
                              r.SetProperty("expert_fullname", expertElem.expert_fullname);
                              r.SetProperty("expert_email", expertElem.expert_email);
                              r.SetProperty("expert_current_state", expertElem.expert_current_state);
       
        if (competence_self || competence_manager) {
            respElemCompArray = getCompetenceValues(respElem.id, competence_self, competence_manager);
            if (competence_self){
                r.SetProperty('comp_1_self', respElemCompArray.competences.self.comp_1);
                r.SetProperty('comp_2_self', respElemCompArray.competences.self.comp_2);
                r.SetProperty('comp_3_self', respElemCompArray.competences.self.comp_3);
                r.SetProperty('comp_4_self', respElemCompArray.competences.self.comp_4);
                r.SetProperty('comp_5_self', respElemCompArray.competences.self.comp_5);
            }
            if (competence_manager){
                r.SetProperty('comp_1_manager', respElemCompArray.competences.manager.comp_1);
                r.SetProperty('comp_2_manager', respElemCompArray.competences.manager.comp_2);
                r.SetProperty('comp_3_manager', respElemCompArray.competences.manager.comp_3);
                r.SetProperty('comp_4_manager', respElemCompArray.competences.manager.comp_4);
                r.SetProperty('comp_5_manager', respElemCompArray.competences.manager.comp_5);
            }
        }
 
                              if (activity_appraisal_task_plan_ext) {
            ed_meths = String(respElem.activity_appraisal_task_plan_education);
            if(ed_meths != null && ed_meths != "") {
                ed_meths = StrReplace(ed_meths, ";",",");
                ed_meths = XQuery("for $elem in education_methods where MatchSome($elem/id, (" + ed_meths + ")) return $elem");
                if(ArrayCount(ed_meths) > 0) {
                    r.activity_appraisal_task_plan_education = ArrayMerge(ed_meths, "This.name", ",");
                }
            }
        }
        if (comment_all) {
            r.SetProperty("comment_all", getAllComment(Int(respElem.id)).string);
        }
        if (activity_appraisal_task_plan_ext && String(respElem.activity_appraisal_task_name) != '') {
            pasDocTE = tools.open_doc(Int(respElem.pas_id)).TopElem;
            aChildTasks = ArraySelect(pasDocTE.tasks, "This.parent_task_id == Int(respElem.activity_appraisal_task_id)");
            if (ArrayCount(aChildTasks) > 0) {
                for (oTask in aChildTasks) {
                    rCopy =  tools.read_object(tools.object_to_text(r, 'json'));
                    taskDocTE = tools.open_doc(oTask.task_id).TopElem;
                    rCopy.SetProperty('activity_appraisal_task_plan_ext_education', taskDocTE.name);
                    rCopy.SetProperty('activity_appraisal_task_plan_ext_education_provider', taskDocTE.desc);
                    rCopy.SetProperty('activity_appraisal_task_plan_ext_education_cost', taskDocTE.plan);
                    rCopy.SetProperty('activity_appraisal_task_plan_ext_education_priority', taskDocTE.custom_fields.ObtainChildByKey("plan_education").value);
 
                    returnArray.push(rCopy);
                }
            }
            else {
                returnArray.push(r);
            }
        }
        else {
            returnArray.push(r);
        }
       
               }
    if (competence_profile != "all")
        returnArray = ArraySelect(returnArray, "This.competence_profile_name == '" + competence_profile + "'")
    _iColWidth = Int(100.0 / ArrayCount(colArray)) + "%";
    reportStr = new Binary();
    reportStr.AppendStr("<html xmlns:o=\"urn:schemas-microsoft-com:office:office\" xmlns:x=\"urn:schemas-microsoft-com:office:excel\" xmlns=\"http://www.w3.org/TR/REC-html40\"><head><meta http-equiv=\"Content-Type\" content=\"text/html; charset=utf-8\"/><meta name=\"ProgId\" content=\"Excel.Sheet\"/><meta name=\"Generator\" content=\"Microsoft Excel 11\"/></head><body><table border=\"1\" cellpadding=\"2\" cellspacing=\"0\">");
    reportStr.AppendStr("<tr valign=\"top\">");
    for (elem in colArray) {
        reportStr.AppendStr("<td width=\"" + _iColWidth + "\" bgcolor=\"#FFCC99\">" + elem.title + "</td>");
    }
    reportStr.AppendStr("</tr>");
    for (row in returnArray) {
        reportStr.AppendStr("<tr valign=\"top\">");
        for (elem in colArray) {
            reportStr.AppendStr("<td>" + row.GetOptProperty(elem.field, "") + "</td>");
        }
        reportStr.AppendStr("</tr>");
    }
    reportStr.AppendStr("</table></body></html>");
               //LogEvent('shchapov', tools.object_to_text(returnArray, 'json'));
               /*return returnArray;*/
    /*var sTempFilename =  ObtainTempFile(".xls");;
    try
    {
        PutUrlText(sTempFilename, reportStr.GetStr());
    }
    catch(_x_)
    {
        throw("Ошибка построения настраиваемого отчета; Error: " + _x_);
    }
   
    var docActiveNotification = OpenNewDoc( 'x-local://wtv/wtv_active_notification.xmd' );
   
    //docActiveNotification.TopElem.notification_id = iNotificationID;
    docActiveNotification.TopElem.object_id = curUserID;
    docActiveNotification.TopElem.text = "";
    docActiveNotification.TopElem.send_date = Date();
    docActiveNotification.TopElem.subject = "УЭД. Гибкий отчет для администратора";
    docActiveNotification.TopElem.body = "";
 
    docActiveNotification.TopElem.body_type = "html";
    docActiveNotification.TopElem.sender.address = global_settings.settings.own_org.email;
   
    var fldRecipientChild = docActiveNotification.TopElem.recipients.AddChild();
    fldRecipientChild.address = tools.open_doc(curUserID).TopElem.email;
    fldRecipientChild.collaborator_id = curUserID;
    fldRecipientChild.name = tools.open_doc(curUserID).TopElem.fullname;
   
    if (sTempFilename != null)
    {
        var fldAttach = docActiveNotification.TopElem.attachments.AddChild();
        fldAttach.name = "Гибкий отчет для администратора.xls";
        try
        {
            fldAttach.data.LoadFromFile(sTempFilename);
           
            try
            {
                DeleteUrl(sTempFilename);
            }
            catch(_X_)
            { }
        }
        catch(_x_)
        {
            throw("Ошибка: не могу прикрепить сформированный отчет; Error: " + _x_);
        }
       
    }
    docActiveNotification.TopElem.status = 'active';
    docActiveNotification.TopElem.date = Date();
    docActiveNotification.TopElem.is_custom = true;
    docActiveNotification.BindToDb();
    docActiveNotification.Save();
    MESSAGE = "Данные отправлены вам на почту";
               RESULT.ResultAction = "HIDE=WaitWindow;ALERT={messageText};";*/
    tools_web.set_user_data("excel_html_" + curUserID, ({
        "html" : reportStr.GetStr()
    }), 3600);
    //alertLog("end create report");          
    RESULT.ResultAction = "HIDE=WaitWindow;CONFIRM=Отчет готов. Выгрузить?;OPENWINDOW=assessment_excel_export.html";
} catch (_ex) {
               ERROR = 1;
               MESSAGE = ExtractUserError(_ex);
               RESULT.ResultAction = "HIDE=WaitWindow;ALERT={messageText};";
}
