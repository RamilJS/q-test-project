



/*
// Данные для тестового подключения
var optionObject = new Object();
optionObject.server = "http://vls-1c-develop/doc_vtbl_dev_maksimova2/ws/DM.1cws";
optionObject.headers = ["Content-Type : text/xml; charset=utf-8", "SOAPAction : http://www.1c.ru/dm#DMService:execute", "Ignore-Errors: 1"];
optionObject.dataBaseID = "ae030af6-4e4c-11e9-8471-1cc1de6c0042"; //поправить при переносе на боевую базу
optionObject.org_id = "6c2dcc21-b5b3-11e8-810b-94049cc88460"; //поправить при переносе на боевую базу
optionObject.document = new Object();
optionObject.document.name = "Ведомость обучения";
optionObject.document.id = "6e268fea-d814-11ec-8134-94049cc8846c"; //поправить при переносе на боевую базу (ID шаблона)
optionObject.document.file_template_id = "84ff530c-d816-11ec-8134-94049cc8846c"; //поправить при переносе на боевую базу (ID шаблона файла)
optionObject.document.process_template_id = "7b23aa28-ba74-4a8e-8a93-191d3f41721e"; //поправить при переносе на боевую базу (ID шаблона)
 */
 
/*
// Данные для предпродуктивного подключения
var optionObject = new Object();
optionObject.server = "http://vls-1c-tapp/doc_vtbl_predprod/ws/DM.1cws";
optionObject.headers = ["Content-Type : text/xml; charset=utf-8", "SOAPAction : http://www.1c.ru/dm#DMService:execute", "Ignore-Errors: 1"];
optionObject.dataBaseID = "ae030af6-4e4c-11e9-8471-1cc1de6c0042"; //поправить при переносе на боевую базу
optionObject.org_id = ""; //поправить при переносе на боевую базу
//Список организаций
optionObject.orgs = [{
    name : "ООО «ФинансБизнесГрупп»",
    id : "88fb579c-cc90-11e8-810b-94049cc88460"
  }, {
    name : "ООО УКА",
    id : "49a9695a-ff83-11e8-846d-1cc1de6c0040"
  }, {
    name : "АО ВТБ Лизинг",
    id : "6c2dcc21-b5b3-11e8-810b-94049cc88460"
  }
];
 
optionObject.document = new Object();
optionObject.document.name = "Ведомость обучения";
optionObject.document.id = "7354514f-851d-11ed-b832-005056ab3b40"; //поправить при переносе на боевую базу (ID шаблона)
optionObject.document.file_template_id = "ac3f5d8a-851d-11ed-b832-005056ab3b40"; //поправить при переносе на боевую базу (ID шаблона файла)
optionObject.document.process_template_id = "e13f21ec-4947-40a4-9acd-68d84973d2f3"; //поправить при переносе на боевую базу (ID шаблона)
 */
 
// Данные для продуктивного подключения
var optionObject = new Object();
//точка подключения
optionObject.server = "http://vls-1c-app-n1/doc_vtbl_work_integration/ws/dm.1cws";
//optionObject.server = "http://vls-1c-web-d/doc_vtbl_dev_ukrazhenko_mchd/ws/dmservice"
//заголовки
//optionObject.headers = ["Content-Type : text/xml; charset=utf-8", "SOAPAction : http://www.1c.ru/dm#DMService:execute", "Ignore-Errors : 1", "WWW-Authorization : NTLM"];
optionObject.headers = ["Content-Type: text/xml; charset=utf-8", "SOAPAction: http://www.1c.ru/dm#DMService:execute", "Ignore-Errors: 1"];
//ID базы данных
optionObject.dataBaseID = "703e79a9-851c-11ed-b832-005056ab3b40";
//ID актуальной организации
optionObject.org_id = "";
optionObject.org_name = "";
//Список организаций
optionObject.orgs = [{
    name : "ООО «ФинансБизнесГрупп»",
    id : "88fb579c-cc90-11e8-810b-94049cc88460"
  }, {
    name : "ООО УКА",
    id : "49a9695a-ff83-11e8-846d-1cc1de6c0040"
  }, {
    name : "АО ВТБ Лизинг",
    id : "6c2dcc21-b5b3-11e8-810b-94049cc88460"
  }
];
//Документ для заполнения
optionObject.document = new Object();
optionObject.document.name = "Ведомость обучения";
optionObject.document.id = "7354514f-851d-11ed-b832-005056ab3b40"; //поправить при переносе на боевую базу (ID шаблона)
optionObject.document.file_template_id = "ac3f5d8a-851d-11ed-b832-005056ab3b40"; //поправить при переносе на боевую базу (ID шаблона файла)
optionObject.document.process_template_id = "e13f21ec-4947-40a4-9acd-68d84973d2f3"; //поправить при переносе на боевую базу (ID шаблона)
 
EnableLog('1c_integration', true);
function alert(_s) {
  LogEvent('1c_integration', _s);
  return _s;
}
 
function getGUID(_int, _empty_num) {
  _int = StrHexInt(_int);
  _empty_num = String(_empty_num);
  var _one_symbol_length = Real(StrLen(_int)) / Real(StrCharCount(_int));
  var _first_pos = 0;
  var _second_pos = _first_pos + _one_symbol_length;
  var _return_int = "";
  var guidMatrix = [8, 4, 4, 4, 12];
  var _elem,
  _i;
  for (_elem in guidMatrix) {
    _return_int += (_return_int == "" ? "" : "-");
    for (_i = 0; _i < _elem; _i++) {
      if (_first_pos < StrLen(_int)) {
        _return_int += StrRangePos(_int, _first_pos, _second_pos);
        _first_pos = _second_pos;
        _second_pos = _first_pos + _one_symbol_length;
      } else {
        _return_int += _empty_num;
      }
    }
  }
  return StrLowerCase(_return_int);
}
 
//Запрос на получение шаблона документа
function getFirstRequestBody() {
  var reqBody = new Binary();
  reqBody.AppendStr("<soap:Envelope xmlns:soap=\"http://schemas.xmlsoap.org/soap/envelope/\">\r\n");
  reqBody.AppendStr(" <soap:Body>\r\n");
  reqBody.AppendStr("   <m:execute xmlns:m=\"http://www.1c.ru/dm\">\r\n");
  reqBody.AppendStr("     <m:request xmlns:xs=\"http://www.w3.org/2001/XMLSchema\"\r\n");
  reqBody.AppendStr("         xmlns:xsi=\"http://www.w3.org/2001/XMLSchema-instance\"\r\n");
  reqBody.AppendStr("         xsi:type=\"m:DMGetNewObjectRequest\">\r\n");
  reqBody.AppendStr("       <m:dataBaseID>" + XmlAttrEncode(optionObject.dataBaseID) + "</m:dataBaseID>\r\n");
  reqBody.AppendStr("       <m:type>DMInternalDocument</m:type>\r\n");
  reqBody.AppendStr("       <m:dataSource xsi:type=\"m:DMInternalDocumentTemplate\">\r\n");
  reqBody.AppendStr("         <m:name>" + XmlAttrEncode(optionObject.document.name) + "</m:name>\r\n");
  reqBody.AppendStr("         <m:objectID>\r\n");
  reqBody.AppendStr("           <m:id>" + XmlAttrEncode(optionObject.document.id) + "</m:id>\r\n");
  reqBody.AppendStr("           <m:type>DMInternalDocumentTemplate</m:type>\r\n");
  reqBody.AppendStr("         </m:objectID>\r\n");
  reqBody.AppendStr("       </m:dataSource>\r\n");
  reqBody.AppendStr("     </m:request>\r\n");
  reqBody.AppendStr("   </m:execute>\r\n");
  reqBody.AppendStr(" </soap:Body>\r\n");
  reqBody.AppendStr("</soap:Envelope>");
  return reqBody.GetStr();
}
 
//Запрос на создание документа
function getSecondRequestBody(learningDocID, learningDocTopElem, prevRequest) {
  var newDocID = XmlAttrEncode(prevRequest.Child('m:documentType').Child('m:objectID').Child('m:id').Value);
  var reqBody = new Binary();
  reqBody.AppendStr("<soap:Envelope xmlns:soap=\"http://schemas.xmlsoap.org/soap/envelope/\">\r\n");
  reqBody.AppendStr(" <soap:Body>\r\n");
  reqBody.AppendStr("   <m:execute xmlns:m=\"http://www.1c.ru/dm\">\r\n");
  reqBody.AppendStr("     <m:request xmlns:xs=\"http://www.w3.org/2001/XMLSchema\" xmlns:xsi=\"http://www.w3.org/2001/XMLSchema-instance\" xsi:type=\"m:DMCreateRequest\">\r\n");
  reqBody.AppendStr("       <m:dataBaseID>" + XmlAttrEncode(optionObject.dataBaseID) + "</m:dataBaseID>\r\n");
  reqBody.AppendStr("       <m:object xsi:type=\"m:DMInternalDocument\">\r\n");
  reqBody.AppendStr("         <m:name>" + XmlAttrEncode(optionObject.document.name) + "</m:name>\r\n");
 
  reqBody.AppendStr("         <m:objectID>\r\n");
  //reqBody.AppendStr("           <m:id>ae030af6-4e4c-11e9-8471-1cc1de6c0100</m:id>\r\n");
  reqBody.AppendStr("           <m:id>" + XmlAttrEncode(getGUID(learningDocID, 0)) + "</m:id>\r\n"); //ID нашего завершенного курса
  reqBody.AppendStr("           <m:type>DMInternalDocument</m:type>\r\n");
  reqBody.AppendStr("         </m:objectID>\r\n");
 
  reqBody.AppendStr("         <m:externalObject>\r\n");
  reqBody.AppendStr("           <m:name>" + XmlAttrEncode(optionObject.document.name) + "</m:name>\r\n");
  reqBody.AppendStr("           <m:id>" + XmlAttrEncode(getGUID(learningDocID, 0)) + "</m:id>\r\n"); //ID нашего завершенного курса
  reqBody.AppendStr("           <m:type>" + prevRequest.Child('m:title').Value + "</m:type>\r\n");
  reqBody.AppendStr("         </m:externalObject>\r\n");
  //Пока непонятно, что с организацией
  /*
  reqBody.AppendStr("         <m:organization>\r\n");
  reqBody.AppendStr("           <m:name>" + XmlAttrEncode(learningDocTopElem.person_id.ForeignElem.org_id.ForeignElem.name) + "</m:name>\r\n");
  //reqBody.AppendStr("           <m:objectID>\r\n");
  //reqBody.AppendStr("             <m:id>6fead453-db17-11e5-a288-ac220bc1d7ca</m:id>\r\n");
  //reqBody.AppendStr("             <m:type>DMOrganization</m:type>\r\n");
  //reqBody.AppendStr("           </m:objectID>\r\n");
  reqBody.AppendStr("         </m:organization>\r\n");
   */
  reqBody.AppendStr("         <m:organization>\r\n");
  reqBody.AppendStr("           <m:name>" + XmlAttrEncode(optionObject.org_name) + "</m:name>\r\n");
  reqBody.AppendStr("           <m:objectID>\r\n");
  reqBody.AppendStr("             <m:id>" + XmlAttrEncode(optionObject.org_id) + "</m:id>\r\n"); //поменять при переносе на бой
  reqBody.AppendStr("             <m:type>DMOrganization</m:type>\r\n");
  reqBody.AppendStr("           </m:objectID>\r\n");
  reqBody.AppendStr("         </m:organization>\r\n");
  reqBody.AppendStr("         <m:responsible>\r\n");
  reqBody.AppendStr("           <m:name/>\r\n");
  reqBody.AppendStr("           <m:objectID>\r\n");
  reqBody.AppendStr("             <m:id/>\r\n");
  reqBody.AppendStr("             <m:type/>\r\n");
  reqBody.AppendStr("           </m:objectID>\r\n");
  reqBody.AppendStr("           <m:externalObject>\r\n");
  reqBody.AppendStr("             <m:name>\\\\" + XmlAttrEncode(learningDocTopElem.person_id.ForeignElem.login) + "</m:name>\r\n");
  reqBody.AppendStr("             <m:id>11111111-1111-1111-1111-111111111111</m:id>\r\n");
  reqBody.AppendStr("             <m:type>DMUser</m:type>\r\n");
  reqBody.AppendStr("           </m:externalObject>\r\n");
  reqBody.AppendStr("         </m:responsible>\r\n");
 
  reqBody.AppendStr("         <m:summary>" + XmlAttrEncode(tools.open_doc(learningDocTopElem.course_id).TopElem.custom_elems.ObtainChildByKey("f_1c_edo_summary_text").value) + "</m:summary>\r\n");
  reqBody.AppendStr("         <m:title>" + XmlAttrEncode(prevRequest.Child('m:title').Value) + "</m:title>\r\n");
 
  additionalProperties = ArraySelect(_firstRequestBody, "This.Name == 'm:additionalProperties'");
  for (additionalProperty in additionalProperties) {
    reqBody.AppendStr("         <m:additionalProperties>\r\n");
    reqBody.AppendStr("           <m:name>" + XmlAttrEncode(additionalProperty.Child("m:name").Value) + "</m:name>\r\n");
    reqBody.AppendStr("           <m:objectID>\r\n");
    reqBody.AppendStr("             <m:id>" + XmlAttrEncode(additionalProperty.Child("m:objectID").Child("m:id").Value) + "</m:id>\r\n");
    reqBody.AppendStr("             <m:type>DMAdditionalProperty</m:type>\r\n");
    reqBody.AppendStr("           </m:objectID>\r\n");
    switch (StrReplace(UnifySpaces(StrLowerCase(additionalProperty.Child("m:name").Value)), ' ', '')) {
    case "наименованиекурса":
    case "названиекурса": {
        reqBody.AppendStr("           <m:propertySimpleValue xsi:type=\"xs:string\">" + XmlAttrEncode(learningDocTopElem.course_id.ForeignElem.name) + "</m:propertySimpleValue>\r\n");
        break;
      }
    case "датаобучения": {
        //reqBody.AppendStr("           <m:propertySimpleValue xsi:type=\"xs:dateTime\">" + StrXmlDate(learningDocTopElem.last_usage_date) + "</m:propertySimpleValue>\r\n");
        reqBody.AppendStr("           <m:propertySimpleValue xsi:type=\"xs:dateTime\">" + StrXmlDate(learningDocTopElem.start_usage_date) + "</m:propertySimpleValue>\r\n");
        break;
      }
    }
    reqBody.AppendStr("         </m:additionalProperties>\r\n");
  }
  _author = prevRequest.Child('m:author');
  reqBody.AppendStr("         <m:author>\r\n");
  reqBody.AppendStr("           <m:name>" + XmlAttrEncode(_author.Child('m:name').Value) + "</m:name>\r\n");
  reqBody.AppendStr("           <m:objectID>\r\n");
  reqBody.AppendStr("             <m:id>" + XmlAttrEncode(_author.Child('m:objectID').Child('m:id').Value) + "</m:id>\r\n");
  reqBody.AppendStr("             <m:presentation>" + XmlAttrEncode(_author.Child('m:objectID').Child('m:presentation').Value) + "</m:presentation>\r\n");
  reqBody.AppendStr("             <m:type>" + XmlAttrEncode(_author.Child('m:objectID').Child('m:type').Value) + "</m:type>\r\n");
  reqBody.AppendStr("             <m:navigationRef>" + XmlAttrEncode(_author.Child('m:objectID').Child('m:navigationRef').Value) + "</m:navigationRef>\r\n");
  reqBody.AppendStr("           </m:objectID>\r\n");
  reqBody.AppendStr("         </m:author>\r\n");
 
  _documentType = prevRequest.Child('m:documentType');
  reqBody.AppendStr("         <m:documentType>\r\n");
  reqBody.AppendStr("           <m:name>" + XmlAttrEncode(_documentType.Child('m:name').Value) + "</m:name>\r\n");
  reqBody.AppendStr("           <m:objectID>\r\n");
  reqBody.AppendStr("             <m:id>" + XmlAttrEncode(_documentType.Child('m:objectID').Child('m:id').Value) + "</m:id>\r\n");
  reqBody.AppendStr("             <m:type>DMInternalDocumentType</m:type>\r\n");
  reqBody.AppendStr("           </m:objectID>\r\n");
  reqBody.AppendStr("         </m:documentType>\r\n");
 
  _folder = prevRequest.Child('m:folder');
  reqBody.AppendStr("         <m:folder>\r\n");
  reqBody.AppendStr("           <m:name>" + XmlAttrEncode(_folder.Child('m:name').Value) + "</m:name>\r\n");
  reqBody.AppendStr("           <m:objectID>\r\n");
  reqBody.AppendStr("             <m:id>" + XmlAttrEncode(_folder.Child('m:objectID').Child('m:id').Value) + "</m:id>\r\n");
  reqBody.AppendStr("             <m:presentation>" + XmlAttrEncode(_folder.Child('m:objectID').Child('m:presentation').Value) + "</m:presentation>\r\n");
  reqBody.AppendStr("             <m:type>DMInternalDocumentFolder</m:type>\r\n");
  reqBody.AppendStr("             <m:navigationRef>e1cib/data/Справочник.ПапкиВнутреннихДокументов?ref=a288ac220bc1d7ca11e5db9b287351ca</m:navigationRef>\r\n");
  reqBody.AppendStr("           </m:objectID>\r\n");
  reqBody.AppendStr("         </m:folder>\r\n");
 
  _template = prevRequest.Child('m:template');
  reqBody.AppendStr("         <m:template>\r\n");
  reqBody.AppendStr("           <m:name>" + XmlAttrEncode(_template.Child('m:name').Value) + "</m:name>\r\n");
  reqBody.AppendStr("           <m:objectID>\r\n");
  reqBody.AppendStr("             <m:id>" + XmlAttrEncode(_template.Child('m:objectID').Child('m:id').Value) + "</m:id>\r\n");
  reqBody.AppendStr("             <m:presentation>" + XmlAttrEncode(_template.Child('m:objectID').Child('m:presentation').Value) + "</m:presentation>\r\n");
  reqBody.AppendStr("             <m:type>DMInternalDocumentTemplate</m:type>\r\n");
  reqBody.AppendStr("             <m:navigationRef>" + XmlAttrEncode(_template.Child('m:objectID').Child('m:navigationRef').Value) + "</m:navigationRef>\r\n");
  reqBody.AppendStr("           </m:objectID>\r\n");
  reqBody.AppendStr("           <m:performanceTerm>3</m:performanceTerm>\r\n");
 
  /*
  reqBody.AppendStr("           <m:activityMatter>\r\n");
  reqBody.AppendStr("             <m:name>Building design</m:name>\r\n");
  reqBody.AppendStr("             <m:objectID>\r\n");
  reqBody.AppendStr("               <m:id>6fead45d-db17-11e5-a288-ac220bc1d7ca</m:id>\r\n");
  reqBody.AppendStr("               <m:presentation>Building design</m:presentation>\r\n");
  reqBody.AppendStr("               <m:type>DMActivityMatter</m:type>\r\n");
  reqBody.AppendStr("               <m:navigationRef>e1cib/data/Справочник.ВопросыДеятельности?ref=a288ac220bc1d7ca11e5db176fead45d</m:navigationRef>\r\n");
  reqBody.AppendStr("             </m:objectID>\r\n");
  reqBody.AppendStr("           </m:activityMatter>\r\n");
   */
 
  /*
  reqBody.AppendStr("           <m:responsible>\r\n");
  reqBody.AppendStr("             <m:name>Administrator</m:name>\r\n");
  reqBody.AppendStr("             <m:objectID>\r\n");
  reqBody.AppendStr("               <m:id>6fead43f-db17-11e5-a288-ac220bc1d7ca</m:id>\r\n");
  reqBody.AppendStr("               <m:presentation>Administrator</m:presentation>\r\n");
  reqBody.AppendStr("               <m:type>DMUser</m:type>\r\n");
  reqBody.AppendStr("               <m:navigationRef>e1cib/data/Справочник.Пользователи?ref=a288ac220bc1d7ca11e5db176fead43f</m:navigationRef>\r\n");
  reqBody.AppendStr("             </m:objectID>\r\n");
  reqBody.AppendStr("           </m:responsible>\r\n");
   */
 
  reqBody.AppendStr("           <m:title>" + XmlAttrEncode(_template.Child('m:title').Value) + "</m:title>\r\n");
  reqBody.AppendStr("           <m:blockDerivedDocuments>false</m:blockDerivedDocuments>\r\n");
  reqBody.AppendStr("           <m:documentType>\r\n");
  reqBody.AppendStr("             <m:name>" + XmlAttrEncode(_template.Child('m:documentType').Child('m:name').Value) + "</m:name>\r\n");
  reqBody.AppendStr("             <m:objectID>\r\n");
  reqBody.AppendStr("               <m:id>" + XmlAttrEncode(_template.Child('m:documentType').Child('m:objectID').Child('m:id').Value) + "</m:id>\r\n");
  reqBody.AppendStr("               <m:presentation>Business memo</m:presentation>\r\n");
  reqBody.AppendStr("               <m:type>DMInternalDocumentType</m:type>\r\n");
  reqBody.AppendStr("               <m:navigationRef>" + XmlAttrEncode(_template.Child('m:documentType').Child('m:objectID').Child('m:presentation').Value) + "</m:navigationRef>\r\n");
  reqBody.AppendStr("             </m:objectID>\r\n");
  reqBody.AppendStr("           </m:documentType>\r\n");
  reqBody.AppendStr("           <m:folder>\r\n");
  reqBody.AppendStr("             <m:name>" + XmlAttrEncode(_template.Child('m:folder').Child('m:name').Value) + "</m:name>\r\n");
  reqBody.AppendStr("             <m:objectID>\r\n");
  reqBody.AppendStr("               <m:id>" + XmlAttrEncode(_template.Child('m:folder').Child('m:objectID').Child('m:id').Value) + "</m:id>\r\n");
  reqBody.AppendStr("               <m:presentation>" + XmlAttrEncode(_template.Child('m:folder').Child('m:objectID').Child('m:presentation').Value) + "</m:presentation>\r\n");
  reqBody.AppendStr("               <m:type>DMInternalDocumentFolder</m:type>\r\n");
  reqBody.AppendStr("               <m:navigationRef>" + XmlAttrEncode(_template.Child('m:folder').Child('m:objectID').Child('m:navigationRef').Value) + "</m:navigationRef>\r\n");
  reqBody.AppendStr("             </m:objectID>\r\n");
  reqBody.AppendStr("           </m:folder>\r\n");
  reqBody.AppendStr("         </m:template>\r\n");
 
  reqBody.AppendStr("       </m:object>\r\n");
  reqBody.AppendStr("     </m:request>\r\n");
  reqBody.AppendStr("   </m:execute>\r\n");
  reqBody.AppendStr(" </soap:Body>\r\n");
  reqBody.AppendStr("</soap:Envelope>\r\n");
  return reqBody.GetStr();
}
 
//Запрос на добавление файла в документ
function getThirdRequestBody(learningDocID, learningDocTopElem, prevRequest) {
  //var newDocID = XmlAttrEncode(prevRequest.Child('m:documentType').Child('m:objectID').Child('m:id').Value);
  var reqBody = new Binary();
  reqBody.AppendStr("<soap:Envelope xmlns:soap=\"http://schemas.xmlsoap.org/soap/envelope/\">\r\n");
  reqBody.AppendStr(" <soap:Body>\r\n");
  reqBody.AppendStr("   <m:execute xmlns:m=\"http://www.1c.ru/dm\">\r\n");
  reqBody.AppendStr("     <m:request xmlns:xs=\"http://www.w3.org/2001/XMLSchema\" xmlns:xsi=\"http://www.w3.org/2001/XMLSchema-instance\" xsi:type=\"m:DMAddFileRequest\">\r\n");
  reqBody.AppendStr("       <m:dataBaseID>" + XmlAttrEncode(optionObject.dataBaseID) + "</m:dataBaseID>\r\n");
 
  reqBody.AppendStr("       <m:file>\r\n");
  reqBody.AppendStr("         <m:name>" + XmlAttrEncode(prevRequest.Child('m:object').Child('m:name').Value) + "</m:name>\r\n");
  reqBody.AppendStr("         <m:objectID>\r\n");
  reqBody.AppendStr("           <m:id>" + XmlAttrEncode(getGUID(learningDocID, 1)) + "</m:id>\r\n"); //ID попытки, добитое единицами
  reqBody.AppendStr("           <m:type>DMFile</m:type>\r\n");
  reqBody.AppendStr("         </m:objectID>\r\n");
  reqBody.AppendStr("         <m:creationDate>" + StrXmlDate(Date()) + "</m:creationDate>\r\n");
  reqBody.AppendStr("         <m:extension>docx</m:extension>\r\n");
  reqBody.AppendStr("         <m:modificationDate>" + StrXmlDate(Date()) + "</m:modificationDate>\r\n");
  reqBody.AppendStr("         <m:modificationDateUniversal>" + StrXmlDate(Date()) + "</m:modificationDateUniversal>\r\n");
  reqBody.AppendStr("         <m:size>565723</m:size>\r\n");
  reqBody.AppendStr("         <m:template>\r\n");
  reqBody.AppendStr("           <m:name>ВедомостьОбученияШаблон</m:name>\r\n");
  reqBody.AppendStr("           <m:objectID>\r\n");
  //reqBody.AppendStr("             <m:id>971f7e51-346b-11e6-b3ae-ac220bc1d7ca</m:id>\r\n"); //поменять при переносе на бой
  reqBody.AppendStr("             <m:id>" + XmlAttrEncode(optionObject.document.file_template_id) + "</m:id>\r\n"); //поменять при переносе на бой
  reqBody.AppendStr("             <m:type>DMFile</m:type>\r\n");
  reqBody.AppendStr("           </m:objectID>\r\n");
  reqBody.AppendStr("         </m:template>\r\n");
  reqBody.AppendStr("       </m:file>\r\n");
 
  reqBody.AppendStr("       <m:owner xsi:type=\"m:DMInternalDocument\">\r\n");
  reqBody.AppendStr("         <m:name>" + XmlAttrEncode(prevRequest.Child('m:object').Child('m:name').Value) + "</m:name>\r\n");
  reqBody.AppendStr("         <m:objectID>\r\n");
  reqBody.AppendStr("           <m:id>" + XmlAttrEncode(prevRequest.Child('m:object').Child('m:objectID').Child('m:id').Value) + "</m:id>\r\n");
  reqBody.AppendStr("           <m:type>" + XmlAttrEncode(prevRequest.Child('m:object').Child('m:objectID').Child('m:type').Value) + "</m:type>\r\n");
  reqBody.AppendStr("         </m:objectID>\r\n");
  reqBody.AppendStr("         <m:externalObject>\r\n");
  reqBody.AppendStr("           <m:name>" + XmlAttrEncode(prevRequest.Child('m:object').Child('m:externalObject').Child('m:name').Value) + "</m:name>\r\n");
  reqBody.AppendStr("           <m:id>" + XmlAttrEncode(prevRequest.Child('m:object').Child('m:externalObject').Child('m:id').Value) + "</m:id>\r\n");
  reqBody.AppendStr("           <m:type>" + XmlAttrEncode(prevRequest.Child('m:object').Child('m:externalObject').Child('m:type').Value) + "</m:type>\r\n");
  reqBody.AppendStr("         </m:externalObject>\r\n");
 
  _organization = prevRequest.Child('m:object').Child('m:organization');
  reqBody.AppendStr("         <m:organization>\r\n");
  reqBody.AppendStr("           <m:name>" + XmlAttrEncode(_organization.Child('m:name').Value) + "</m:name>\r\n");
  reqBody.AppendStr("           <m:objectID>\r\n");
  reqBody.AppendStr("             <m:id>" + XmlAttrEncode(_organization.Child('m:objectID').Child('m:id').Value) + "</m:id>\r\n");
  reqBody.AppendStr("             <m:type>" + XmlAttrEncode(_organization.Child('m:objectID').Child('m:type').Value) + "</m:type>\r\n");
  reqBody.AppendStr("           </m:objectID>\r\n");
  reqBody.AppendStr("         </m:organization>\r\n");
 
  _responsible = prevRequest.Child('m:object').Child('m:responsible');
  reqBody.AppendStr("         <m:responsible>\r\n");
  reqBody.AppendStr("           <m:name>" + XmlAttrEncode(_responsible.Child('m:name').Value) + "</m:name>\r\n");
  reqBody.AppendStr("           <m:objectID>\r\n");
  reqBody.AppendStr("             <m:id>" + XmlAttrEncode(_responsible.Child('m:objectID').Child('m:id').Value) + "</m:id>\r\n");
  reqBody.AppendStr("             <m:presentation>" + XmlAttrEncode(_responsible.Child('m:objectID').Child('m:presentation').Value) + "</m:presentation>\r\n");
  reqBody.AppendStr("             <m:type>" + XmlAttrEncode(_responsible.Child('m:objectID').Child('m:type').Value) + "</m:type>\r\n");
  reqBody.AppendStr("             <m:navigationRef>" + XmlAttrEncode(_responsible.Child('m:objectID').Child('m:navigationRef').Value) + "</m:navigationRef>\r\n");
  reqBody.AppendStr("           </m:objectID>\r\n");
  reqBody.AppendStr("           <m:externalObject>\r\n");
  reqBody.AppendStr("             <m:name>\\\\" + XmlAttrEncode(learningDocTopElem.person_id.ForeignElem.login) + "</m:name>\r\n");
  reqBody.AppendStr("             <m:id>11111111-1111-1111-1111-111111111111</m:id>\r\n");
  reqBody.AppendStr("             <m:type>DMUser</m:type>\r\n");
  reqBody.AppendStr("           </m:externalObject>\r\n");
  reqBody.AppendStr("         </m:responsible>\r\n");
 
  reqBody.AppendStr("         <m:summary>" + XmlAttrEncode(prevRequest.Child('m:object').Child('m:summary').Value) + "</m:summary>\r\n");
  reqBody.AppendStr("         <m:title>" + XmlAttrEncode(prevRequest.Child('m:object').Child('m:title').Value) + "</m:title>\r\n");
 
  additionalProperties = ArraySelect(_firstRequestBody, "This.Name == 'm:additionalProperties'");
  for (additionalProperty in additionalProperties) {
    reqBody.AppendStr("         <m:additionalProperties>\r\n");
    reqBody.AppendStr("           <m:name>" + XmlAttrEncode(additionalProperty.Child("m:name").Value) + "</m:name>\r\n");
    reqBody.AppendStr("           <m:objectID>\r\n");
    reqBody.AppendStr("             <m:id>" + XmlAttrEncode(additionalProperty.Child("m:objectID").Child("m:id").Value) + "</m:id>\r\n");
    reqBody.AppendStr("             <m:type>DMAdditionalProperty</m:type>\r\n");
    reqBody.AppendStr("           </m:objectID>\r\n");
    //switch(StrLowerCase(additionalProperty.Child("m:name").Value)){
    switch (StrReplace(UnifySpaces(StrLowerCase(additionalProperty.Child("m:name").Value)), ' ', '')) {
    case "наименованиекурса":
    case "названиекурса": {
        reqBody.AppendStr("           <m:propertySimpleValue xsi:type=\"xs:string\">" + XmlAttrEncode(learningDocTopElem.course_id.ForeignElem.name) + "</m:propertySimpleValue>\r\n");
        break;
      }
    case "датаобучения": {
        //reqBody.AppendStr("           <m:propertySimpleValue xsi:type=\"xs:dateTime\">" + StrXmlDate(learningDocTopElem.last_usage_date) + "</m:propertySimpleValue>\r\n");
        reqBody.AppendStr("           <m:propertySimpleValue xsi:type=\"xs:dateTime\">" + StrXmlDate(learningDocTopElem.start_usage_date) + "</m:propertySimpleValue>\r\n");
        break;
      }
    }
    reqBody.AppendStr("         </m:additionalProperties>\r\n");
  }
 
  _documentType = prevRequest.Child('m:object').Child('m:documentType');
  reqBody.AppendStr("         <m:documentType>\r\n");
  reqBody.AppendStr("           <m:name>" + XmlAttrEncode(_documentType.Child('m:name').Value) + "</m:name>\r\n");
  reqBody.AppendStr("           <m:objectID>\r\n");
  reqBody.AppendStr("             <m:id>" + XmlAttrEncode(_documentType.Child('m:objectID').Child('m:id').Value) + "</m:id>\r\n");
  reqBody.AppendStr("             <m:type>DMInternalDocumentType</m:type>\r\n");
  reqBody.AppendStr("           </m:objectID>\r\n");
  reqBody.AppendStr("         </m:documentType>\r\n");
 
  _folder = prevRequest.Child('m:object').Child('m:folder');
  reqBody.AppendStr("         <m:folder>\r\n");
  reqBody.AppendStr("           <m:name>" + XmlAttrEncode(_folder.Child('m:name').Value) + "</m:name>\r\n");
  reqBody.AppendStr("           <m:objectID>\r\n");
  reqBody.AppendStr("             <m:id>" + XmlAttrEncode(_folder.Child('m:objectID').Child('m:id').Value) + "</m:id>\r\n");
  reqBody.AppendStr("             <m:presentation>" + XmlAttrEncode(_folder.Child('m:objectID').Child('m:presentation').Value) + "</m:presentation>\r\n");
  reqBody.AppendStr("             <m:type>DMInternalDocumentFolder</m:type>\r\n");
  reqBody.AppendStr("             <m:navigationRef>e1cib/data/Справочник.ПапкиВнутреннихДокументов?ref=a288ac220bc1d7ca11e5db9b287351ca</m:navigationRef>\r\n");
  reqBody.AppendStr("           </m:objectID>\r\n");
  reqBody.AppendStr("         </m:folder>\r\n");
 
  _template = prevRequest.Child('m:object').Child('m:template');
  reqBody.AppendStr("         <m:template>\r\n");
  reqBody.AppendStr("           <m:name>" + XmlAttrEncode(_template.Child('m:name').Value) + "</m:name>\r\n");
  reqBody.AppendStr("           <m:objectID>\r\n");
  reqBody.AppendStr("             <m:id>" + XmlAttrEncode(_template.Child('m:objectID').Child('m:id').Value) + "</m:id>\r\n");
  reqBody.AppendStr("             <m:presentation>" + XmlAttrEncode(_template.Child('m:objectID').Child('m:presentation').Value) + "</m:presentation>\r\n");
  reqBody.AppendStr("             <m:type>DMInternalDocumentTemplate</m:type>\r\n");
  reqBody.AppendStr("             <m:navigationRef>" + XmlAttrEncode(_template.Child('m:objectID').Child('m:navigationRef').Value) + "</m:navigationRef>\r\n");
  reqBody.AppendStr("           </m:objectID>\r\n");
  reqBody.AppendStr("           <m:performanceTerm>3</m:performanceTerm>\r\n");
  /*
  reqBody.AppendStr("           <m:activityMatter>\r\n");
  reqBody.AppendStr("             <m:name>Building design</m:name>\r\n");
  reqBody.AppendStr("             <m:objectID>\r\n");
  reqBody.AppendStr("               <m:id>6fead45d-db17-11e5-a288-ac220bc1d7ca</m:id>\r\n");
  reqBody.AppendStr("               <m:presentation>Building design</m:presentation>\r\n");
  reqBody.AppendStr("               <m:type>DMActivityMatter</m:type>\r\n");
  reqBody.AppendStr("               <m:navigationRef>e1cib/data/Справочник.ВопросыДеятельности?ref=a288ac220bc1d7ca11e5db176fead45d</m:navigationRef>\r\n");
  reqBody.AppendStr("             </m:objectID>\r\n");
  reqBody.AppendStr("           </m:activityMatter>\r\n");
  reqBody.AppendStr("           <m:responsible>\r\n");
  reqBody.AppendStr("             <m:name>Administrator</m:name>\r\n");
  reqBody.AppendStr("             <m:objectID>\r\n");
  reqBody.AppendStr("               <m:id>6fead43f-db17-11e5-a288-ac220bc1d7ca</m:id>\r\n");
  reqBody.AppendStr("               <m:presentation>Administrator</m:presentation>\r\n");
  reqBody.AppendStr("               <m:type>DMUser</m:type>\r\n");
  reqBody.AppendStr("               <m:navigationRef>e1cib/data/Справочник.Пользователи?ref=a288ac220bc1d7ca11e5db176fead43f</m:navigationRef>\r\n");
  reqBody.AppendStr("             </m:objectID>\r\n");
  reqBody.AppendStr("           </m:responsible>\r\n");
   */
  reqBody.AppendStr("           <m:title>" + XmlAttrEncode(_template.Child('m:title').Value) + "</m:title>\r\n");
  reqBody.AppendStr("           <m:blockDerivedDocuments>false</m:blockDerivedDocuments>\r\n");
  reqBody.AppendStr("           <m:documentType>\r\n");
  reqBody.AppendStr("             <m:name>" + XmlAttrEncode(_template.Child('m:documentType').Child('m:name').Value) + "</m:name>\r\n");
  reqBody.AppendStr("             <m:objectID>\r\n");
  reqBody.AppendStr("               <m:id>" + XmlAttrEncode(_template.Child('m:documentType').Child('m:objectID').Child('m:id').Value) + "</m:id>\r\n");
  reqBody.AppendStr("               <m:presentation>Business memo</m:presentation>\r\n");
  reqBody.AppendStr("               <m:type>DMInternalDocumentType</m:type>\r\n");
  reqBody.AppendStr("               <m:navigationRef>" + XmlAttrEncode(_template.Child('m:documentType').Child('m:objectID').Child('m:presentation').Value) + "</m:navigationRef>\r\n");
  reqBody.AppendStr("             </m:objectID>\r\n");
  reqBody.AppendStr("           </m:documentType>\r\n");
  reqBody.AppendStr("           <m:folder>\r\n");
  reqBody.AppendStr("             <m:name>" + XmlAttrEncode(_template.Child('m:folder').Child('m:name').Value) + "</m:name>\r\n");
  reqBody.AppendStr("             <m:objectID>\r\n");
  reqBody.AppendStr("               <m:id>" + XmlAttrEncode(_template.Child('m:folder').Child('m:objectID').Child('m:id').Value) + "</m:id>\r\n");
  reqBody.AppendStr("               <m:presentation>" + XmlAttrEncode(_template.Child('m:folder').Child('m:objectID').Child('m:presentation').Value) + "</m:presentation>\r\n");
  reqBody.AppendStr("               <m:type>DMInternalDocumentFolder</m:type>\r\n");
  reqBody.AppendStr("               <m:navigationRef>" + XmlAttrEncode(_template.Child('m:folder').Child('m:objectID').Child('m:navigationRef').Value) + "</m:navigationRef>\r\n");
  reqBody.AppendStr("             </m:objectID>\r\n");
  reqBody.AppendStr("           </m:folder>\r\n");
  reqBody.AppendStr("         </m:template>\r\n");
  reqBody.AppendStr("       </m:owner>\r\n");
  reqBody.AppendStr("     </m:request>\r\n");
  reqBody.AppendStr("   </m:execute>\r\n");
  reqBody.AppendStr(" </soap:Body>\r\n");
  reqBody.AppendStr("</soap:Envelope>\r\n");
  return reqBody.GetStr();
}
 
//Запрос на получение шаблона документооборота
function getFourRequestBody(_object_id) {
  var reqBody = new Binary();
  reqBody.AppendStr("<soap:Envelope xmlns:soap=\"http://schemas.xmlsoap.org/soap/envelope/\">\r\n");
  reqBody.AppendStr(" <soap:Body>\r\n");
  reqBody.AppendStr("   <m:execute xmlns:m=\"http://www.1c.ru/dm\">\r\n");
  reqBody.AppendStr("     <m:request xmlns:xs=\"http://www.w3.org/2001/XMLSchema\" xmlns:xsi=\"http://www.w3.org/2001/XMLSchema-instance\" xsi:type=\"m:DMGetBusinessProcessByTemplateRequest\">\r\n");
  reqBody.AppendStr("       <m:type>DMComplexBusinessProcess</m:type>\r\n");
  reqBody.AppendStr("       <m:targetID>\r\n");
  reqBody.AppendStr("         <m:id>" + XmlAttrEncode(_object_id) + "</m:id>\r\n");
  reqBody.AppendStr("         <m:type>DMInternalDocument</m:type>\r\n");
  reqBody.AppendStr("       </m:targetID>\r\n");
  reqBody.AppendStr("       <m:businessProcessTemplateId>\r\n");
  reqBody.AppendStr("         <m:id>" + XmlAttrEncode(optionObject.document.process_template_id) + "</m:id>\r\n"); //поменять при переносе на бой
  reqBody.AppendStr("         <m:type>DMComplexBusinessProcessTemplate</m:type>\r\n");
  reqBody.AppendStr("       </m:businessProcessTemplateId>\r\n");
  reqBody.AppendStr("     </m:request>\r\n");
  reqBody.AppendStr("   </m:execute>\r\n");
  reqBody.AppendStr(" </soap:Body>\r\n");
  reqBody.AppendStr("</soap:Envelope>\r\n");
  return reqBody.GetStr();
}
 
//Запрос на запуск документооборота
function getFiveRequestBody(prevRequest, learningDocID, learningDocTopElem) {
  //alert(learningDocTopElem.Xml)
  var reqBody = new Binary();
  reqBody.AppendStr("<soap:Envelope xmlns:soap=\"http://schemas.xmlsoap.org/soap/envelope/\">\r\n");
  reqBody.AppendStr(" <soap:Body>\r\n");
  reqBody.AppendStr("   <m:execute xmlns:m=\"http://www.1c.ru/dm\">\r\n");
  reqBody.AppendStr("     <m:request xmlns:xs=\"http://www.w3.org/2001/XMLSchema\" xmlns:xsi=\"http://www.w3.org/2001/XMLSchema-instance\" xsi:type=\"m:DMLaunchBusinessProcessRequest\">\r\n");
  reqBody.AppendStr("       <m:dataBaseID>" + XmlAttrEncode(optionObject.dataBaseID) + "</m:dataBaseID>\r\n");
  reqBody.AppendStr("       <m:businessProcess xsi:type=\"m:DMComplexBusinessProcess\">\r\n");
  reqBody.AppendStr("         <m:name>" + XmlAttrEncode(prevRequest.Child('m:object').Child('m:name').Value) + "</m:name>\r\n");
  reqBody.AppendStr("         <m:objectID>\r\n");
  reqBody.AppendStr("           <m:id/>\r\n");
  reqBody.AppendStr("           <m:presentation>" + XmlAttrEncode(prevRequest.Child('m:object').Child('m:objectID').Child('m:presentation').Value) + "</m:presentation>\r\n");
  reqBody.AppendStr("           <m:type>" + XmlAttrEncode(prevRequest.Child('m:object').Child('m:objectID').Child('m:type').Value) + "</m:type>\r\n");
  reqBody.AppendStr("           <m:navigationRef>" + XmlAttrEncode(prevRequest.Child('m:object').Child('m:objectID').Child('m:navigationRef').Value) + "</m:navigationRef>\r\n");
  reqBody.AppendStr("         </m:objectID>\r\n");
  reqBody.AppendStr("         <m:author>\r\n");
  reqBody.AppendStr("           <m:name>" + XmlAttrEncode(prevRequest.Child('m:object').Child('m:author').Child('m:name').Value) + "</m:name>\r\n");
  reqBody.AppendStr("           <m:objectID>\r\n");
  reqBody.AppendStr("             <m:id>" + XmlAttrEncode(prevRequest.Child('m:object').Child('m:author').Child('m:objectID').Child('m:id').Value) + "</m:id>\r\n");
  reqBody.AppendStr("             <m:presentation>" + XmlAttrEncode(prevRequest.Child('m:object').Child('m:author').Child('m:objectID').Child('m:presentation').Value) + "</m:presentation>\r\n");
  reqBody.AppendStr("             <m:type>" + XmlAttrEncode(prevRequest.Child('m:object').Child('m:author').Child('m:objectID').Child('m:type').Value) + "</m:type>\r\n");
  reqBody.AppendStr("             <m:navigationRef>" + XmlAttrEncode(prevRequest.Child('m:object').Child('m:author').Child('m:objectID').Child('m:navigationRef').Value) + "</m:navigationRef>\r\n");
  reqBody.AppendStr("           </m:objectID>\r\n");
  reqBody.AppendStr("         </m:author>\r\n");
  reqBody.AppendStr("         <m:importance>\r\n");
  reqBody.AppendStr("           <m:name>Обычная важность</m:name>\r\n");
  reqBody.AppendStr("           <m:objectID>\r\n");
  reqBody.AppendStr("             <m:id>Обычная</m:id>\r\n");
  reqBody.AppendStr("             <m:type>DMBusinessProcessTaskImportance</m:type>\r\n");
  reqBody.AppendStr("           </m:objectID>\r\n");
  reqBody.AppendStr("         </m:importance>\r\n");
  reqBody.AppendStr("         <m:beginDate xsi:nil=\"true\"/>\r\n");
  reqBody.AppendStr("         <m:endDate xsi:nil=\"true\"/>\r\n");
  reqBody.AppendStr("         <m:target>\r\n");
  reqBody.AppendStr("           <m:name>" + XmlAttrEncode(prevRequest.Child('m:object').Child('m:target').Child('m:name').Value) + "</m:name>\r\n");
  reqBody.AppendStr("           <m:objectID>\r\n");
  reqBody.AppendStr("             <m:id>" + XmlAttrEncode(prevRequest.Child('m:object').Child('m:target').Child('m:objectID').Child('m:id').Value) + "</m:id>\r\n");
  reqBody.AppendStr("             <m:presentation>" + XmlAttrEncode(prevRequest.Child('m:object').Child('m:target').Child('m:objectID').Child('m:presentation').Value) + "</m:presentation>\r\n");
  reqBody.AppendStr("             <m:type>" + XmlAttrEncode(prevRequest.Child('m:object').Child('m:target').Child('m:objectID').Child('m:type').Value) + "</m:type>\r\n");
  reqBody.AppendStr("             <m:navigationRef>" + XmlAttrEncode(prevRequest.Child('m:object').Child('m:target').Child('m:objectID').Child('m:navigationRef').Value) + "</m:navigationRef>\r\n");
  reqBody.AppendStr("           </m:objectID>\r\n");
  reqBody.AppendStr("         </m:target>\r\n");
  reqBody.AppendStr("         <m:started>false</m:started>\r\n");
  reqBody.AppendStr("         <m:completed>false</m:completed>\r\n");
  reqBody.AppendStr("         <m:description>" + XmlAttrEncode(replaceDescription(prevRequest.Child('m:object').Child('m:description').Value, learningDocID, learningDocTopElem)) + "</m:description>\r\n");
  //reqBody.AppendStr("         <m:description>Для описания задачи пользователям в ДО нужно доработать описание БП.\r\nПо идее, в полученном из ДО шаблоне бизнес-процесса нужно заполнить поле description.\r\n\r\nТекст пользователи предоставили такой:\r\n\r\nДобрый день, [ФИО]!\r\n\r\nВам назначен электронный курс [НазваниеКурса].\r\nПрохождение курса является обязательным.\r\n\r\nСрок прохождения и ознакомления эл. курса с [ДатаНачалаКурса] по [ДатаОкончанияКурса]\r\n\r\nДля прохождения электронного курса Вам необходимо пройти по ссылке: [СсылкаНаКурс]\r\nПо всем вопросам, возникшим при изучении курса, Вы можете обращаться в Отдел комплаенс и финансового мониторинга по электронной почте MSK-OFM@vtb-leasing.com\r\n\r\nС уважением,\r\nСпециальное должностное лицо-\r\n[ПодготовилДолжностьПодпись]\r\n</m:description>\r\n");
  reqBody.AppendStr("         <m:dueDate xsi:nil=\"true\"/>\r\n");
  reqBody.AppendStr("         <m:state>\r\n");
  reqBody.AppendStr("           <m:name>" + XmlAttrEncode(prevRequest.Child('m:object').Child('m:state').Child('m:name').Value) + "</m:name>\r\n");
  reqBody.AppendStr("           <m:objectID>\r\n");
  reqBody.AppendStr("             <m:id>" + XmlAttrEncode(prevRequest.Child('m:object').Child('m:state').Child('m:objectID').Child('m:id').Value) + "</m:id>\r\n");
  reqBody.AppendStr("             <m:presentation>" + XmlAttrEncode(prevRequest.Child('m:object').Child('m:state').Child('m:objectID').Child('m:presentation').Value) + "</m:presentation>\r\n");
  reqBody.AppendStr("             <m:type>" + XmlAttrEncode(prevRequest.Child('m:object').Child('m:state').Child('m:objectID').Child('m:type').Value) + "</m:type>\r\n");
  reqBody.AppendStr("             <m:navigationRef>" + XmlAttrEncode(prevRequest.Child('m:object').Child('m:state').Child('m:objectID').Child('m:navigationRef').Value) + "</m:navigationRef>\r\n");
  reqBody.AppendStr("           </m:objectID>\r\n");
  reqBody.AppendStr("         </m:state>\r\n");
  reqBody.AppendStr("         <m:dueTimeEnabled>true</m:dueTimeEnabled>\r\n");
  reqBody.AppendStr("         <m:parentTaskEnabled>true</m:parentTaskEnabled>\r\n");
  reqBody.AppendStr("         <m:stateEnabled>false</m:stateEnabled>\r\n");
  reqBody.AppendStr("         <m:businessProcessTemplate xsi:type=\"m:DMComplexBusinessProcessTemplate\">\r\n");
  reqBody.AppendStr("           <m:name>ВедомостиОбученияБП</m:name>\r\n");
  reqBody.AppendStr("           <m:objectID>\r\n");
  reqBody.AppendStr("             <m:id>" + XmlAttrEncode(optionObject.document.process_template_id) + "</m:id>\r\n"); //поменять при переносе на бой - шаблон БП
  reqBody.AppendStr("             <m:presentation>ВедомостиОбученияБП</m:presentation>\r\n");
  reqBody.AppendStr("             <m:type>DMComplexBusinessProcessTemplate</m:type>\r\n");
  reqBody.AppendStr("             <m:navigationRef>e1cib/data/Справочник.ШаблоныКомплексныхБизнесПроцессов?ref=9c2ab4859b965c3c4bf8d7cecf8d0358</m:navigationRef>\r\n");
  reqBody.AppendStr("           </m:objectID>\r\n");
  reqBody.AppendStr("         </m:businessProcessTemplate>\r\n");
  reqBody.AppendStr("         <m:blockedByTemplate>false</m:blockedByTemplate>\r\n");
  reqBody.AppendStr("         <m:targets>\r\n");
  reqBody.AppendStr("           <m:items>\r\n");
  reqBody.AppendStr("             <m:role>\r\n");
  reqBody.AppendStr("               <m:name>" + XmlAttrEncode(prevRequest.Child('m:object').Child('m:targets').Child('m:items').Child('m:role').Child('m:name').Value) + "</m:name>\r\n");
  reqBody.AppendStr("               <m:objectID>\r\n");
  reqBody.AppendStr("                 <m:id>" + XmlAttrEncode(prevRequest.Child('m:object').Child('m:targets').Child('m:items').Child('m:role').Child('m:objectID').Child('m:id').Value) + "</m:id>\r\n");
  reqBody.AppendStr("                 <m:presentation>" + XmlAttrEncode(prevRequest.Child('m:object').Child('m:targets').Child('m:items').Child('m:role').Child('m:objectID').Child('m:presentation').Value) + "</m:presentation>\r\n");
  reqBody.AppendStr("                 <m:type>" + XmlAttrEncode(prevRequest.Child('m:object').Child('m:targets').Child('m:items').Child('m:role').Child('m:objectID').Child('m:type').Value) + "</m:type>\r\n");
  reqBody.AppendStr("                 <m:navigationRef>" + XmlAttrEncode(prevRequest.Child('m:object').Child('m:targets').Child('m:items').Child('m:role').Child('m:objectID').Child('m:navigationRef').Value) + "</m:navigationRef>\r\n");
  reqBody.AppendStr("               </m:objectID>\r\n");
  reqBody.AppendStr("             </m:role>\r\n");
  reqBody.AppendStr("             <m:name>" + XmlAttrEncode(prevRequest.Child('m:object').Child('m:targets').Child('m:items').Child('m:name').Value) + "</m:name>\r\n");
  reqBody.AppendStr("             <m:target>\r\n");
  reqBody.AppendStr("               <m:name>" + XmlAttrEncode(prevRequest.Child('m:object').Child('m:targets').Child('m:items').Child('m:target').Child('m:name').Value) + "</m:name>\r\n");
  reqBody.AppendStr("               <m:objectID>\r\n");
  reqBody.AppendStr("                 <m:id>" + XmlAttrEncode(prevRequest.Child('m:object').Child('m:targets').Child('m:items').Child('m:target').Child('m:objectID').Child('m:id').Value) + "</m:id>\r\n");
  reqBody.AppendStr("                 <m:presentation>" + XmlAttrEncode(prevRequest.Child('m:object').Child('m:targets').Child('m:items').Child('m:target').Child('m:objectID').Child('m:presentation').Value) + "</m:presentation>\r\n");
  reqBody.AppendStr("                 <m:type>" + XmlAttrEncode(prevRequest.Child('m:object').Child('m:targets').Child('m:items').Child('m:target').Child('m:objectID').Child('m:type').Value) + "</m:type>\r\n");
  reqBody.AppendStr("                 <m:navigationRef>" + XmlAttrEncode(prevRequest.Child('m:object').Child('m:targets').Child('m:items').Child('m:target').Child('m:objectID').Child('m:navigationRef').Value) + "</m:navigationRef>\r\n");
  reqBody.AppendStr("               </m:objectID>\r\n");
  reqBody.AppendStr("             </m:target>\r\n");
  reqBody.AppendStr("           </m:items>\r\n");
  reqBody.AppendStr("         </m:targets>\r\n");
  reqBody.AppendStr("         <m:leadingTaskEnabled>true</m:leadingTaskEnabled>\r\n");
  reqBody.AppendStr("         <m:controller/>\r\n");
  reqBody.AppendStr("         <m:routingType>\r\n");
  reqBody.AppendStr("           <m:name>" + XmlAttrEncode(prevRequest.Child('m:object').Child('m:routingType').Child('m:name').Value) + "</m:name>\r\n");
  reqBody.AppendStr("           <m:objectID>\r\n");
  reqBody.AppendStr("             <m:id>" + XmlAttrEncode(prevRequest.Child('m:object').Child('m:routingType').Child('m:objectID').Child('m:id').Value) + "</m:id>\r\n");
  reqBody.AppendStr("             <m:presentation>" + XmlAttrEncode(prevRequest.Child('m:object').Child('m:routingType').Child('m:objectID').Child('m:presentation').Value) + "</m:presentation>\r\n");
  reqBody.AppendStr("             <m:type>" + XmlAttrEncode(prevRequest.Child('m:object').Child('m:routingType').Child('m:objectID').Child('m:type').Value) + "</m:type>\r\n");
  reqBody.AppendStr("             <m:navigationRef/>\r\n");
  reqBody.AppendStr("           </m:objectID>\r\n");
  reqBody.AppendStr("         </m:routingType>\r\n");
 
  _stages = ArraySelect(prevRequest.Child('m:object'), "This.Name == 'm:stages'");
  for (_stage in _stages) {
    reqBody.AppendStr("         <m:stages>\r\n");
    reqBody.AppendStr("           <m:stageID>" + XmlAttrEncode(_stage.Child('m:stageID').Value) + "</m:stageID>\r\n");
    reqBody.AppendStr("           <m:template>\r\n");
    reqBody.AppendStr("             <m:name>" + XmlAttrEncode(_stage.Child('m:template').Child('m:name').Value) + "</m:name>\r\n");
    reqBody.AppendStr("             <m:objectID>\r\n");
    reqBody.AppendStr("               <m:id>" + XmlAttrEncode(_stage.Child('m:template').Child('m:objectID').Child('m:id').Value) + "</m:id>\r\n");
    reqBody.AppendStr("               <m:presentation>" + XmlAttrEncode(_stage.Child('m:template').Child('m:objectID').Child('m:presentation').Value) + "</m:presentation>\r\n");
    reqBody.AppendStr("               <m:type>" + XmlAttrEncode(_stage.Child('m:template').Child('m:objectID').Child('m:type').Value) + "</m:type>\r\n");
    reqBody.AppendStr("               <m:navigationRef>" + XmlAttrEncode(_stage.Child('m:template').Child('m:objectID').Child('m:navigationRef').Value) + "</m:navigationRef>\r\n");
    reqBody.AppendStr("             </m:objectID>\r\n");
    reqBody.AppendStr("           </m:template>\r\n");
    reqBody.AppendStr("           <m:duration>" + XmlAttrEncode(_stage.Child('m:duration').Value) + "</m:duration>\r\n");
    reqBody.AppendStr("           <m:participants>" + XmlAttrEncode(_stage.Child('m:participants').Value) + "</m:participants>\r\n");
    reqBody.AppendStr("           <m:stagePredecessors>" + XmlAttrEncode(_stage.Child('m:stagePredecessors').Value) + "</m:stagePredecessors>\r\n");
    reqBody.AppendStr("           <m:predecessorsUseOption>" + XmlAttrEncode(_stage.Child('m:predecessorsUseOption').Value) + "</m:predecessorsUseOption>\r\n");
    reqBody.AppendStr("           <m:unconditionalPassageExecuted>" + XmlAttrEncode(_stage.Child('m:unconditionalPassageExecuted').Value) + "</m:unconditionalPassageExecuted>\r\n");
    reqBody.AppendStr("           <m:executed>false</m:executed>\r\n");
    //reqBody.AppendStr("           <m:importance>\r\n");
    //reqBody.AppendStr("             <m:name>Обычная важность</m:name>\r\n");
    //reqBody.AppendStr("             <m:objectID>\r\n");
    //reqBody.AppendStr("               <m:id>Обычная</m:id>\r\n");
    //reqBody.AppendStr("               <m:type>DMBusinessProcessTaskImportance</m:type>\r\n");
    //reqBody.AppendStr("             </m:objectID>\r\n");
    //reqBody.AppendStr("           </m:importance>\r\n");
    reqBody.AppendStr("         </m:stages>\r\n");
 
  }
  /*
  reqBody.AppendStr("         <m:predecessors>\r\n");
  reqBody.AppendStr("           <m:followerID>91e5850c-7e48-4cb2-8b09-0aedad9140c2</m:followerID>\r\n");
  reqBody.AppendStr("           <m:passageExecuted>false</m:passageExecuted>\r\n");
  reqBody.AppendStr("           <m:considerationCondition>\r\n");
  reqBody.AppendStr("             <m:name>После успешного выполнения</m:name>\r\n");
  reqBody.AppendStr("             <m:objectID>\r\n");
  reqBody.AppendStr("               <m:id>ПослеУспешногоВыполнения</m:id>\r\n");
  reqBody.AppendStr("               <m:presentation>После успешного выполнения</m:presentation>\r\n");
  reqBody.AppendStr("               <m:type>DMPredecessorsStageConsiderationCondition</m:type>\r\n");
  reqBody.AppendStr("               <m:navigationRef/>\r\n");
  reqBody.AppendStr("             </m:objectID>\r\n");
  reqBody.AppendStr("           </m:considerationCondition>\r\n");
  reqBody.AppendStr("         </m:predecessors>\r\n");
   */
  reqBody.AppendStr("       </m:businessProcess>\r\n");
  reqBody.AppendStr("     </m:request>\r\n");
  reqBody.AppendStr("   </m:execute>\r\n");
  reqBody.AppendStr(" </soap:Body>\r\n");
  reqBody.AppendStr("</soap:Envelope>\r\n");
  return reqBody.GetStr();
}
 
function replaceDescription(description, learningDocID, learningDocTopElem) {
  description = 'Добрый день, [ФИО]!\r\n';
  description += '\r\n';
  description += 'Вами был пройден электронный курс "[НазваниеКурса]".\r\n';
  description += '\r\n';
  description += 'Просьба ознакомиться и подписать ведомость обучения.\r\n';
  description += 'Подписание ведомости является обязательным.\r\n';
  description += '\r\n';
  description += 'По всем вопросам, возникшим при изучении курса и подписании ведомости, Вы можете обращаться в Отдел комплаенс и финансового мониторинга по электронной почте MSK-OFM@vtb-leasing.com\r\n';
  description += '\r\n';
  description += '\r\n';
  description += 'С уважением,\r\n';
  description += 'Отдел Комплаенс и финансового мониторинга\r\n';
  //description += 'Специальное должностное лицо-\r\n';
  //description += '[ПодготовилДолжностьПодпись]';
 
  _ponomarevaDoc = tools.open_doc(6725645739301406107).TopElem;
  var replaceArray = [
    ['[ФИО]', 'learningDocTopElem.person_id.ForeignElem.fullname'],
    ['[НазваниеКурса]', 'learningDocTopElem.course_id.ForeignElem.name'],
    ['[ДатаНачалаКурса]', 'StrDate(learningDocTopElem.start_usage_date, false)'],
    ['[ДатаОкончанияКурса]', 'StrDate(learningDocTopElem.last_usage_date)'],
    ['[СсылкаНаКурс]', '(String("https://als-wt/_wt/") + learningDocTopElem.course_id)'],
    ['[ПодготовилДолжностьПодпись]', '(_ponomarevaDoc.fullname + "\\\r\\\n" + _ponomarevaDoc.position_name + "\\\r\\\n" + _ponomarevaDoc.position_parent_name)']
  ];
  description = String(description);
  for (elem in replaceArray) {
    description = StrReplace(description, elem[0], eval(elem[1]));
  }
  return description;
}
sqlStr = new Binary();
sqlStr.AppendStr("SELECT learnings.id\r\n");
sqlStr.AppendStr("FROM courses\r\n");
sqlStr.AppendStr("JOIN course\r\n");
sqlStr.AppendStr("  ON course.id = courses.id\r\n");
sqlStr.AppendStr("JOIN learnings\r\n");
sqlStr.AppendStr("  ON learnings.course_id = courses.id\r\n");
sqlStr.AppendStr("JOIN learning\r\n");
sqlStr.AppendStr("  ON learning.id = learnings.id\r\n");
sqlStr.AppendStr("WHERE course.data.value('(course/custom_elems/custom_elem[name=''f_1c_edo_send_state'']/value)[1]', 'varchar(max)') = 'true'\r\n");
sqlStr.AppendStr("  AND (\r\n");
sqlStr.AppendStr("    learning.data.value('(learning/custom_elems/custom_elem[name=''f_1c_edo_integrated'']/value)[1]', 'varchar(max)') = 'false'\r\n");
sqlStr.AppendStr("    OR learning.data.value('(learning/custom_elems/custom_elem[name=''f_1c_edo_integrated'']/value)[1]', 'varchar(max)') IS NULL\r\n");
sqlStr.AppendStr("    )\r\n");
sqlStr.AppendStr("  AND learnings.state_id IN ('2','4')\r\n");
sqlStr.AppendStr("  AND learnings.modification_date >= DATEADD(hh, 0 - 14, GETDATE())\r\n");
//sqlStr.AppendStr("  AND learnings.person_id =  6088496220923652448\r\n");//рукин
//sqlStr.AppendStr("  AND learnings.person_id =  6996085564821038025\r\n");//колемасова
//sqlStr.AppendStr("  AND learnings.course_id =  6374322843214053245\r\n");
 
alert('start agent')
 
learningDocArray = XQuery("sql:" + sqlStr.GetStr());
 
for (_learning in learningDocArray) {
 
  learningDoc = tools.open_doc(Int(_learning.id));
  //learningDoc = Parameters.GetOptProperty('learningDoc', Parameters.learningDoc)
  _tempOrg = ArrayOptFind(optionObject.orgs, "This.name == learningDoc.TopElem.person_id.ForeignElem.org_id.ForeignElem.name");
  if (_tempOrg != undefined) {
    optionObject.org_id = _tempOrg.id;
    optionObject.org_name = _tempOrg.name;
    alert('1. Запрос на получение шаблона документа -----------------------------------------------------------');
    alert(optionObject.server);
    firstRequest = new Object();
    firstRequest.request_count = 0;
    firstRequest.request_object = undefined;
    while (firstRequest.request_count < 5 && (firstRequest.request_object == undefined || OptInt(firstRequest.request_object.RespCode, 0) != 200)) {
      firstRequest.request_object = HttpRequest(optionObject.server, 'post', alert(getFirstRequestBody()), alert(ArrayMerge(optionObject.headers, "This", "\n")));
      try {
        if (OptInt(firstRequest.request_object.RespCode, 0) != 200) {
          firstRequest.request_count++;
          alert('Ответ (ошибка)--------------------------------------------------------------------------------------');
          alert(firstRequest.request_object.Body);
        }
      } catch (_e) {
        firstRequest.request_count++;
      }
    }
    if (firstRequest.request_object == undefined || OptInt(firstRequest.request_object.RespCode, 0) != 200) {
      alert("Can't send request");
      continue;
    }
    alert('Ответ ----------------------------------------------------------------------------------------------');
    alert(firstRequest.request_object.Body);
    _firstRequestBody = OpenDocFromStr(firstRequest.request_object.Body).TopElem.Child(0).Child(0).Child(0);
 
    alert('2. Запрос на создание документа --------------------------------------------------------------------');
    alert(optionObject.server);
    secondRequest = new Object();
    secondRequest.request_count = 0;
    secondRequest.request_object = undefined;
    while (secondRequest.request_count < 5 && (secondRequest.request_object == undefined || OptInt(secondRequest.request_object.RespCode, 0) != 200)) {
      secondRequest.request_object = HttpRequest(optionObject.server, 'post', alert(getSecondRequestBody(Int(_learning.id), learningDoc.TopElem, _firstRequestBody)), ArrayMerge(optionObject.headers, "This", "\n"));
      try {
        if (OptInt(secondRequest.request_object.RespCode, 0) != 200) {
          secondRequest.request_count++;
          alert('Ответ (ошибка)--------------------------------------------------------------------------------------');
          alert(secondRequest.request_object.Body);
        }
      } catch (_e) {
        secondRequest.request_count++;
      }
    }
    if (secondRequest.request_object == undefined || OptInt(secondRequest.request_object.RespCode, 0) != 200) {
      alert("Can't send second request");
      continue;
    }
    alert('Ответ ----------------------------------------------------------------------------------------------');
    alert(secondRequest.request_object.Body);
    _secondRequestBody = OpenDocFromStr(secondRequest.request_object.Body).TopElem.Child(0).Child(0).Child(0);
 
    alert('3. Запрос на добавление файла в документ -----------------------------------------------------------');
    thirdRequest = new Object();
    thirdRequest.request_count = 0;
    thirdRequest.request_object = undefined;
    while (thirdRequest.request_count < 5 && (thirdRequest.request_object == undefined || OptInt(thirdRequest.request_object.RespCode, 0) != 200)) {
      thirdRequest.request_object = HttpRequest(optionObject.server, 'post', alert(getThirdRequestBody(Int(_learning.id), learningDoc.TopElem, _secondRequestBody)), ArrayMerge(optionObject.headers, "This", "\n"));
      try {
        if (OptInt(thirdRequest.request_object.RespCode, 0) != 200) {
          thirdRequest.request_count++;
          alert('Ответ (ошибка)--------------------------------------------------------------------------------------');
          alert(thirdRequest.request_object.Body);
        }
      } catch (_e) {
        thirdRequest.request_count++;
      }
    }
    if (thirdRequest.request_object == undefined || OptInt(thirdRequest.request_object.RespCode, 0) != 200) {
      alert("Can't send four request");
      continue;
    }
    alert('Ответ ----------------------------------------------------------------------------------------------');
    alert(thirdRequest.request_object.Body);
    _thirdRequestBody = OpenDocFromStr(thirdRequest.request_object.Body).TopElem.Child(0).Child(0).Child(0);
 
    alert('4. Запрос на получение шаблона документооборота ----------------------------------------------------');
    fourRequest = new Object();
    fourRequest.request_count = 0;
    fourRequest.request_object = undefined;
    while (fourRequest.request_count < 5 && (fourRequest.request_object == undefined || OptInt(fourRequest.request_object.RespCode, 0) != 200)) {
      fourRequest.request_object = HttpRequest(optionObject.server, 'post', alert(getFourRequestBody(_secondRequestBody.Child('m:object').Child('m:objectID').Child('m:id'))), ArrayMerge(optionObject.headers, "This", "\n"));
      try {
        if (OptInt(fourRequest.request_object.RespCode, 0) != 200) {
          fourRequest.request_count++;
          alert('Ответ (ошибка)--------------------------------------------------------------------------------------');
          alert(fourRequest.request_object.Body);
        }
      } catch (_e) {
        fourRequest.request_count++;
      }
    }
    if (fourRequest.request_object == undefined || OptInt(fourRequest.request_object.RespCode, 0) != 200) {
      alert("Can't send four request");
      continue;
    }
    alert('Ответ ----------------------------------------------------------------------------------------------');
    alert(fourRequest.request_object.Body);
    _fourRequestBody = OpenDocFromStr(fourRequest.request_object.Body).TopElem.Child(0).Child(0).Child(0);
 
    alert('5. Запрос на запуск документооборота ---------------------------------------------------------------');
    fiveRequest = new Object();
    fiveRequest.request_count = 0;
    fiveRequest.request_object = undefined;
    while (fiveRequest.request_count < 5 && (fiveRequest.request_object == undefined || OptInt(fiveRequest.request_object.RespCode, 0) != 200)) {
      fiveRequest.request_object = HttpRequest(optionObject.server, 'post', alert(getFiveRequestBody(_fourRequestBody, Int(_learning.id), learningDoc.TopElem)), ArrayMerge(optionObject.headers, "This", "\n"));
      try {
        if (OptInt(fiveRequest.request_object.RespCode, 0) != 200) {
          fiveRequest.request_count++;
          alert('Ответ (ошибка)--------------------------------------------------------------------------------------');
          alert(fiveRequest.request_object.Body);
        }
      } catch (_e) {
        fiveRequest.request_count++;
      }
    }
    if (fiveRequest.request_object == undefined || OptInt(fiveRequest.request_object.RespCode, 0) != 200) {
      alert("Can't send five request");
      continue;
    }
    alert('Ответ ----------------------------------------------------------------------------------------------');
    alert(fiveRequest.request_object.Body);
    _fiveRequestBody = OpenDocFromStr(fiveRequest.request_object.Body).TopElem.Child(0).Child(0).Child(0);
    if (OptInt(fiveRequest.request_object.RespCode, 0) == 200) {
      learningDoc.TopElem.custom_elems.ObtainChildByKey('f_1c_edo_integrated').value = 'true';
      learningDoc.Save();
    }
  }
}
 
