MySQL query to combine responses of the Moodle quiz tool and H5P via xAPI, using the most recent attempts per user.

SET @course := [INSERT COURSE ID];
(SELECT 
    course, quiz, userid, attempt, fraction, qtype, questionid, maxmark,  
    questionsummary, rightanswer, responsesummary, correct, score, 
    time_started, time_completed, duration, type  
FROM (
  SELECT 
    quiz.course,
    quiza.quiz,
    quiza.userid,
    quiza.attempt,
    qas.fraction,
    q.qtype,
    qa.questionid,
    qa.maxmark,
    qa.questionsummary,
    qa.rightanswer,
    qa.responsesummary,
    qa.rightanswer=qa.responsesummary AS correct,
    qas.fraction / qa.maxmark AS score,
    @time_completed:=(
        SELECT timecreated FROM question_attempt_steps AS qasin 
        WHERE qasin.questionattemptid = qas.questionattemptid 
        AND qas.userid = qasin.userid AND qasin.state=‘complete’) 
        AS time_completed,
    @time_started:=(
        SELECT timecreated FROM question_attempt_steps AS qasin 
        WHERE qas.userid = qasin.userid 
        AND qasin.timecreated < time_completed 
        ORDER BY timecreated DESC LIMIT 0,1) 
        AS time_started,
    @time_completed - @time_started AS duration,
    ‘inhouse’ AS type
FROM (
    SELECT qa.*
    FROM quiz_attempts AS qa                    
    LEFT JOIN quiz_attempts AS qa2            
         ON qa.userid = qa2.userid AND qa.attempt < qa2.attempt
    WHERE qa2.attempt is NULL
) AS quiza  
JOIN question_usages AS qu ON qu.id = quiza.uniqueid
JOIN question_attempts AS qa ON qa.questionusageid = qu.id
JOIN quiz ON quiz.id = quiza.quiz
JOIN question_attempt_steps AS qas ON qas.questionattemptid = qa.id
JOIN question AS q ON q.id = qa.questionid

WHERE quiz.course=@course AND (qas.state = ‘gradedpartial’ 
OR qas.state = ‘gradedright’ OR qas.state = ‘gradedwrong’)
ORDER BY qa.questionid) AS Q1)

UNION ALL

(SELECT
    hvp.course,
    xapi.content_id AS quiz,
    xapi.user_id AS userid,
    (SELECT COUNT(*) FROM logstore_standard_log AS lo 
     WHERE lo.objectid = xapi.content_id AND lo.userid = xapi.user_id) 
     AS attempt,
    xapi.raw_score AS fraction,
    xapi.interaction_type AS qtype,
    xapi.content_id AS questionid,
    xapi.max_score AS maxmark,
    description AS questionsummary,
    REPLACE(
         REPLACE(
              correct_responses_pattern, 
              ‘["‘,’’), ‘"]’,’’) AS rightanswer,
    xapi.response AS responsesummary,
    correct_responses_pattern = Concat(‘["‘, response, ‘"]’) AS correct,
    xapi.raw_score / xapi.max_score AS score,    
    @time_started:=(
        SELECT MAX(timecreated) FROM logstore_standard_log AS lo 
        WHERE lo.objectid = xapi.content_id AND lo.userid = xapi.user_id) 
        AS time_started,
    @time_completed:=(
        SELECT timecreated FROM logstore_standard_log AS lo 
        WHERE id > (SELECT id FROM logstore_standard_log AS loin  
                    WHERE loin.objectid = xapi.content_id 
                    AND loin.userid = xapi.user_id 
                    AND loin.timecreated = time_started )
        AND lo.userid = xapi.user_id ORDER BY id LIMIT 0,1)
        AS time_completed,
          @time_completed - @time_started AS duration,
    ‘hvp’ AS type

FROM hvp_xapi_results AS xapi 
JOIN hvp ON hvp.id = xapi.content_id
WHERE course=@course AND interaction_type != ‘compound’)
