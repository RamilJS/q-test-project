var qid2 = aQuestions[qh3].id;
var qData = K.questions[qid2];
if (qData === undefined && typeof qid2 === 'number') {
    qData = K.questions[String(qid2)];
} else if (qData === undefined && typeof qid2 === 'string') {
    qData = K.questions[Number(qid2)];
}
