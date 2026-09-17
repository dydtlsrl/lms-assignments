# Chapter 08 확장 실습 답안 템플릿

> **과제:** JOIN과 집계로 서비스 질문에 답하기  
> **사용 방법:** 이 파일을 내려받아 본인의 GitHub 저장소에 `chapter08_answer.md`라는 이름으로 저장한 뒤 실습하면서 바로 작성합니다.  
> **제출 방법:** LMS에는 파일을 직접 업로드하지 않고, **본인 GitHub 저장소의 `chapter08_answer.md` 파일 URL**을 제출합니다.

---

## 제출 전 주의

이 파일과 캡처 화면에는 실제 비밀번호, 전체 DB 접속 URL, API Key, 개인정보를 기록하지 않습니다.

```text
GitHub 계정 또는 별칭: dydtlsrl@gmail.com
과제 작성일: 2026-09-17
사용한 AI 도구: chat GPT
```

---

# 1. Chapter 07 기준 상태 확인

다음을 실행합니다.

```text
code/chapter08/00_check_course_project.sql
```

## 1-1. 사전 검사 결과

```text
검증 메시지:Chapter 08 prerequisite check passed

students 행 수: 3
instructors 행 수: 2
courses 행 수: 3
enrollments 행 수: 5

전체 신청 건수: 5
전체 recorded_amount: 590000
활성 신청 건수: 3
활성 recorded_amount: 340000
취소 제외 신청 건수: 4
취소 제외 recorded_amount: 440000
```

기준값:

```text
students = 3
instructors = 2
courses = 3
enrollments = 5

전체 = 5 / 590000
활성 = 3 / 340000
취소 제외 = 4 / 440000
```

### 기준값이 다르면 그대로 진행하면 안 되는 이유

```text
이후 JOIN과 집계 쿼리의 결과가 기준 데이터와 달라져,
실습 결과를 올바르게 검증할 수 없기 때문이다.
```

### 증거 화면

권장 경로:

```text
assignments/chapter08/images/step01_prerequisite.png
```

![사전검사통과](images/step01_prerequisite.png)

---

# 2. 업무 질문을 SQL보다 먼저 정의하기

다음 세 질문을 각각 SQL 작성 전에 먼저 정의합니다.

## 질문 A

```text
업무 질문:
각 수강신청은 어떤 학생이 어떤 강좌를 신청했으며,
해당 강좌의 담당 강사는 누구인가?

결과 한 행의 의미:

수강신청 1건의 학생, 강좌, 담당 강사, 신청 상태, 기록 금액
포함 상태:
신청, 수강중, 완료, 취소 전체

제외 상태:
없음

JOIN할 테이블:
enrollments, students, courses, instructors

JOIN 경로:
enrollments → students
enrollments → courses → instructors

INNER JOIN / LEFT JOIN 선택:
INNER JOIN

그 이유:
사전 검사에서 학생·강좌·강사 참조가 모두 정상임을 확인했으므로,
연결 정보가 모두 있는 수강신청만 조회하면 된다.

예상 행 수:
5행
```

## 질문 B

```text
업무 질문:
강좌별 수강신청 건수와 recorded_amount 합계는 얼마이며,
현재 활성 상태(신청·수강중)의 신청 건수와 금액은 얼마인가?

결과 한 행의 의미:
강좌 1개에 대한 전체 신청 현황과 활성 신청 현황

포함 상태:
전체 신청 건수와 금액에는 신청, 수강중, 완료, 취소를 모두 포함
활성 신청 건수와 금액에는 신청, 수강중만 포함

제외 상태:
활성 신청 집계에서는 완료, 취소 제외

JOIN할 테이블:
courses, instructors, enrollments

JOIN 경로:
courses → instructors
courses → enrollments

집계 대상:
강좌별 전체 신청 건수, 전체 recorded_amount,
활성 신청 건수, 활성 recorded_amount

예상 결과:
3행

강좌 301: 전체 2건 / 200000, 활성 1건 / 100000
강좌 302: 전체 2건 / 240000, 활성 2건 / 240000
강좌 303: 전체 1건 / 150000, 활성 0건 / 0
```

## 질문 C

```text
업무 질문:
강사별로 담당 강좌 수와 수강신청 현황은 어떠하며,
취소를 제외한 유효 신청 건수와 recorded_amount는 얼마인가?

결과 한 행의 의미:
강사 1명의 담당 강좌 운영 현황과 수강신청 집계

포함 상태:
전체 신청 건수와 금액에는 신청, 수강중, 완료, 취소를 모두 포함
유효 신청 집계에는 신청, 수강중, 완료를 포함

제외 상태:
유효 신청 집계에서는 취소 제외

JOIN할 테이블:
instructors, courses, enrollments

JOIN 경로:
instructors → courses → enrollments

집계 대상:
강사별 담당 강좌 수,
전체 신청 건수와 recorded_amount,
취소 제외 신청 건수와 recorded_amount

예상 결과:
2행

강사 201: 담당 강좌 2개, 전체 4건 / 440000, 취소 제외 4건 / 440000
강사 202: 담당 강좌 1개, 전체 1건 / 150000, 취소 제외 0건 / 0
```

---

# 3. INNER JOIN과 다중 JOIN

## 3-1. 신청 한 건마다 학생 이름과 강의 제목 조회

실행 전 예상:

```text
결과 한 행 = 수강신청 1건의 학생 이름, 강의 제목, 신청 상태, recorded_amount
예상 행 수 = 5행
JOIN 경로 = enrollments → students, enrollments → courses
```

내가 실행한 SQL:

```sql
SELECT
    e.id AS enrollment_id,
    s.name AS student_name,
    c.title AS course_title,
    e.status
FROM course_project.enrollments AS e
INNER JOIN course_project.students AS s
    ON e.student_id = s.id
INNER JOIN course_project.courses AS c
    ON e.course_id = c.id
ORDER BY e.id;
```

실제 결과:

```text
실제 행 수: 5행
예상과 일치 여부: 일치
```

### 학생 이름이 여러 번 보이는 것이 중복 오류가 아닐 수 있는 이유

```text
결과 한 행이 학생 1명이 아니라 수강신청 1건을 의미하기 때문이다.
한 학생이 여러 강의를 신청할 수 있으므로 김민지와 이준호의 이름이
각각 서로 다른 enrollment_id의 행에 여러 번 나타나는 것은 정상이다.
```

## 3-2. 학생·강의·강사까지 연결

```text
결과 한 행 = 수강신청 1건의 학생 이름, 강의 제목, 담당 강사 이름
강사까지 가는 JOIN 경로 = enrollments → courses → instructors
```

```sql
SELECT
    e.id AS enrollment_id,
    s.name AS student_name,
    c.title AS course_title,
    i.name AS instructor_name,
    e.status,
    e.recorded_amount
FROM course_project.enrollments AS e
INNER JOIN course_project.students AS s
    ON e.student_id = s.id
INNER JOIN course_project.courses AS c
    ON e.course_id = c.id
INNER JOIN course_project.instructors AS i
    ON c.instructor_id = i.id
ORDER BY e.id;
```

실제 행 수:

```text
5행
```

### 증거 화면

권장 경로:

```text
assignments/chapter08/images/step03_inner_join.png
```

`여기에 다중 JOIN 결과 화면을 삽입하세요.`
![다중JOIN결과](images/step03_inner_join.png)
---

# 4. LEFT JOIN과 0건 표현

## 4-1. 강의별 취소 제외 신청 수

신청이 없는 강의도 결과에 남도록 작성합니다.

실행 전:

```text
결과 한 행 = 강의 한 개
강의 303의 예상 실제 신청 수 = 0
강의 303의 예상 고유 학생 수 = 0
강의 303의 예상 recorded_amount = 0
```

내 SQL:

```sql
SELECT
    c.id AS course_id,
    c.title AS course_title,
    COUNT(e.id) AS non_cancelled_count,
    COUNT(DISTINCT e.student_id) AS student_count,
    COALESCE(SUM(e.recorded_amount), 0) AS non_cancelled_recorded_amount
FROM course_project.courses AS c
LEFT JOIN course_project.enrollments AS e
    ON c.id = e.course_id
   AND e.status <> '취소'
GROUP BY c.id, c.title
ORDER BY c.id;

```

실제 결과:
강의 301: 신청수	2, 고유 학생 수 2, recorded_amount	200000
강의 302: 신청수	2, 고유 학생 수 2, recorded_amount	240000
강의 303: 신청수	0, 고유 학생 수 0, recorded_amount	0


```text
강의 301: 신청 수 2, 고유 학생 수 2, recorded_amount 200000
강의 302: 신청 수 2, 고유 학생 수 2, recorded_amount 240000
강의 303: 신청 수 0, 고유 학생 수 0, recorded_amount 0
```

## 4-2. `COUNT(*)`와 `COUNT(e.id)` 비교

강의 303을 기준으로 작성합니다.

```text
COUNT(*) 결과: 1
COUNT(e.id) 결과: 0
COUNT(DISTINCT e.student_id) 결과: 0
```

### 왜 `COUNT(*) = 1`인데 실제 신청 수는 0일 수 있나요?

```text
LEFT JOIN은 오른쪽 테이블에 연결되는 행이 없어도
왼쪽 테이블의 강의 행을 1행으로 남긴다.
강의 303은 취소를 제외하면 연결되는 신청이 없지만,
강의 자체의 행이 남아 있으므로 COUNT(*)는 1이 된다.
```

### 자식 사건 수를 셀 때 `COUNT(child.id)`가 더 적절한 이유

```text
COUNT(child.id)는 실제로 연결된 자식 행의 id만 센다.
LEFT JOIN으로 만들어진 NULL 행은 세지 않으므로,
신청이 0건인 강의를 정확히 0으로 표현할 수 있다.
```

---

# 5. `LEFT JOIN`에서 `ON`과 `WHERE` 조건 비교

취소 제외 신청만 연결한다고 가정합니다.

## 5-1. 조건을 `ON`에 둔 경우

```sql
SELECT
    s.id,
    s.name,
    COUNT(e.id) AS non_cancelled_count
FROM course_project.students AS s
LEFT JOIN course_project.enrollments AS e
    ON s.id = e.student_id
   AND e.status <> '취소'
GROUP BY s.id, s.name
ORDER BY s.id;
```

```text
결과 학생 수: 3명
박서연 포함 여부: 포함 (non_cancelled_count = 0)
```

## 5-2. 조건을 `WHERE`에 둔 경우

```sql
SELECT
    s.id,
    s.name,
    COUNT(e.id) AS non_cancelled_count
FROM course_project.students AS s
LEFT JOIN course_project.enrollments AS e
    ON s.id = e.student_id
WHERE e.status <> '취소'
GROUP BY s.id, s.name
ORDER BY s.id;
```

```text
결과 학생 수: 2명
박서연 포함 여부: 미포함
```

## 5-3. 차이 설명

```text
ON 조건이 LEFT JOIN의 오른쪽 연결 대상을 제한하는 방식:
학생 행은 모두 유지한 채, 취소가 아닌 신청 행만 연결한다.
연결할 신청이 없어도 학생은 NULL 또는 집계값 0으로 남는다.

WHERE 조건이 JOIN 이후 결과 행을 제거하는 방식:
JOIN 결과가 만들어진 뒤 취소가 아닌 신청 행만 남긴다.
연결된 신청이 없던 학생의 NULL 행은 조건을 만족하지 못해 제거된다.

이번 사례에서 ON = 3명, WHERE = 2명이 되는 이유:
박서연은 취소된 신청만 있으므로 ON 조건에서는 0건인 학생으로 남고,
WHERE 조건에서는 취소가 아닌 신청 행이 없어 결과에서 제거된다.
```

---

# 6. 신청이 없는 학생 찾기 — 두 방법 비교

## 방법 1. `LEFT JOIN ... IS NULL`

```sql
SELECT
    s.id,
    s.name,
    s.email
FROM course_project.students AS s
LEFT JOIN course_project.enrollments AS e
    ON s.id = e.student_id
   AND e.status <> '취소'
WHERE e.id IS NULL;
```

## 방법 2. `NOT EXISTS`

```sql
SELECT
    s.id,
    s.name,
    s.email
FROM course_project.students AS s
WHERE NOT EXISTS (
    SELECT 1
    FROM course_project.enrollments AS e
    WHERE e.student_id = s.id
      AND e.status <> '취소'
);
```

```text
방법 1 결과: 1행
방법 2 결과: 1행
두 결과가 같은가: 같다
찾아진 학생: 박서연
```

### 두 방식의 공통 의미를 자신의 말로 설명

```text
두 방식 모두 취소를 제외한 수강신청 이력이 하나도 없는 학생을 찾는다.
박서연은 취소된 신청만 있으므로, 취소 제외 신청이 없는 학생으로 조회된다.
```

---

# 7. 기본 집계 검산

다음 결과를 직접 확인합니다.

| 분석 범위 | 예상 건수 | 실제 건수 | 예상 금액 | 실제 금액 | 일치? |
| --- | ---: | ---: | ---: | ---: | --- |
| 전체 신청 | 5 | 5 | 590000 | 590000 | 일치 |
| 활성 신청 | 3 | 3 | 340000 | 340000 | 일치 |
| 취소 제외 | 4 | 4 | 440000 | 440000 | 일치 |
| 취소 | 1 | 1 | 150000 | 150000 | 일치 |

## 7-1. 전체 평균 `recorded_amount`

```text
예상 평균: 118000.00
실제 평균: 118000.00
```

## 7-2. 취소 제외 평균

```text
예상 평균: 110000.00
실제 평균: 110000.00
```

### `recorded_amount`를 실제 회계 매출이라고 부르면 안 되는 이유

```text
recorded_amount는 수강신청 시점에 기록된 금액이다.
취소된 신청의 금액도 포함될 수 있으며, 환불·결제 완료·매출 인식 여부를
반영하지 않으므로 실제 회계 매출과 동일한 값으로 볼 수 없다.
```

---

# 8. `GROUP BY`, `HAVING`, `FILTER`

## 8-1. 상태별 신청 건수

```sql
SELECT
    status,
    COUNT(*) AS enrollment_count,
    SUM(recorded_amount) AS total_recorded_amount
FROM course_project.enrollments
GROUP BY status
ORDER BY CASE status
    WHEN '신청' THEN 1
    WHEN '수강중' THEN 2
    WHEN '완료' THEN 3
    WHEN '취소' THEN 4
    ELSE 99
END;
```

결과:

```text
신청: 2건 / 240000
수강중: 1건 / 100000
완료: 1건 / 100000
취소: 1건 / 150000
상태별 합계: 5건 / 590000
```

### 상태별 건수 합이 전체 신청 5건과 맞는지 검산

```text
2 + 1 + 1 + 1 = 5이므로 전체 신청 건수 5건과 일치한다.
```

## 8-2. 강의별 취소 제외 신청 수와 금액

```sql
SELECT
    c.id AS course_id,
    c.title AS course_title,
    COUNT(e.id) AS non_cancelled_count,
    COUNT(DISTINCT e.student_id) AS student_count,
    COALESCE(SUM(e.recorded_amount), 0) AS non_cancelled_recorded_amount
FROM course_project.courses AS c
LEFT JOIN course_project.enrollments AS e
    ON c.id = e.course_id
   AND e.status <> '취소'
GROUP BY c.id, c.title
ORDER BY c.id;
```

```text
강의 301: 2건 / 200000
강의 302: 2건 / 240000
강의 303: 0건 / 0
강의별 합계를 다시 더한 값: 4건 / 440000
전체 취소 제외 기준 440000과 일치 여부: 일치
```

## 8-3. `HAVING` 사용

취소 제외 신청이 2건 이상인 강의를 조회합니다.

```sqlSELECT
    c.id,
    c.title,
    COUNT(e.id) AS enrollment_count
FROM course_project.courses AS c
JOIN course_project.enrollments AS e
    ON c.id = e.course_id
WHERE e.status <> '취소'
GROUP BY c.id, c.title
HAVING COUNT(e.id) >= 2
ORDER BY c.id;
```

```text
예상 강의 수: 2개
실제 강의 수: 2개
```

---

# 9. 과대 집계 오류 직접 관찰

강사 201의 강의 가격 합계를 구한다고 가정합니다.

## 9-1. 신청까지 JOIN해서 잘못 집계한 결과

```sql
SELECT
    i.id AS instructor_id,
    i.name AS instructor_name,
    SUM(c.price) AS wrong_course_price_sum
FROM course_project.instructors AS i
JOIN course_project.courses AS c
    ON i.id = c.instructor_id
JOIN course_project.enrollments AS e
    ON c.id = e.course_id
GROUP BY i.id, i.name
ORDER BY i.id;
```

```text
강사 201 잘못된 가격 합계: 440000
```

본문 기준:

```text
440000
```

## 9-2. 강의 수준에서 올바르게 집계

```sql
SELECT
    i.id AS instructor_id,
    i.name AS instructor_name,
    COALESCE(SUM(c.price), 0) AS course_price_sum
FROM course_project.instructors AS i
LEFT JOIN course_project.courses AS c
    ON i.id = c.instructor_id
GROUP BY i.id, i.name
ORDER BY i.id;
```

```text
강사 201 올바른 가격 합계: 220000
```

본문 기준:

```text
220000
```

## 9-3. 왜 두 결과가 달라졌나요?

```text
JOIN 전 강의 행 수: 강사 201의 강의는 2행이다.

JOIN 후 강의가 반복된 이유:
강의와 수강신청은 1대다 관계이므로,
수강신청 테이블을 JOIN하면 각 강의가 신청 건수만큼 반복된다.

SUM이 무엇을 반복해서 더했는가:
강의 301의 가격 100000이 2번, 강의 302의 가격 120000이 2번 더해져
100000 + 100000 + 120000 + 120000 = 440000이 되었다.
```

### `SUM(DISTINCT c.price)`를 일반적인 해결책으로 사용하면 안 되는 이유

```text
서로 다른 강의의 가격이 우연히 같으면 DISTINCT가 한 가격을 중복으로 제거한다.
따라서 강의 수만큼 가격을 합산해야 하는 상황에서 정확한 결과를 보장하지 못한다.
집계 대상이 강의라면 수강신청 테이블을 JOIN하지 않고 강의 수준에서 집계해야 한다.
```

### 증거 화면

권장 경로:

```text
assignments/chapter08/images/step09_over_aggregation.png
```

`여기에 잘못된 합계와 올바른 합계를 비교한 화면을 삽입하세요.`

![잘못된 합계](images/step09_over_aggregation.png)

![올바른 합계](images/step09_over_aggregation02.png)

---

# 10. 상세 결과 ↔ 집계 결과 교차 검산

강의 하나를 선택합니다.

```text
선택한 course_id: 301
강의 제목: 데이터베이스 입문
```

## 10-1. 상세 신청 행 조회

```sql
SELECT
    e.id AS enrollment_id,
    e.student_id,
    e.status,
    e.recorded_amount
FROM course_project.enrollments AS e
WHERE e.course_id = 301
ORDER BY e.id;
```

```text
상세 행 수: 2건
상세 recorded_amount를 직접 더한 값: 200000
```

## 10-2. 집계 SQL

```sql
SELECT
    COUNT(*) AS enrollment_count,
    SUM(recorded_amount) AS total_recorded_amount
FROM course_project.enrollments
WHERE course_id = 301;
```

```text
집계 건수: 2건
집계 금액: 200000
```

## 10-3. 비교

```text
상세 행 수와 COUNT 결과 일치 여부: 일치
상세 금액 합과 SUM 결과 일치 여부: 일치
다르다면 원인: 해당 없음
```

---

# 11. 자동 완료 게이트

다음을 실행합니다.

```text
code/chapter08/03_join_aggregation_validation.sql
```

```text
최종 검증 메시지: Chapter 08 join and aggregation validation passed
```

기대 메시지:

```text
Chapter 08 join and aggregation validation passed
```

### 자동 검증이 통과했어도 사람이 SQL 의미를 설명해야 하는 이유

```text
자동 검증은 기준 데이터에서 결과값이 맞는지 확인할 뿐,
JOIN 조건과 집계 기준이 업무 질문에 적절한지까지 판단하지는 못한다.
사람이 결과 한 행의 의미, LEFT JOIN과 WHERE 조건의 차이,
집계 단위와 과대 집계 가능성을 설명해야 SQL이 왜 올바른지 확인할 수 있다.
```

---

# 12. 개인 프로젝트 업무 질문 3개 만들기

Chapter 07에서 작성한 개인 프로젝트를 사용합니다.

| 질문 ID | 업무 질문 | 결과 한 행 | 포함/제외 범위 | JOIN 경로 | 집계 대상 | 검산 방법 |
| --- | --- | --- | --- | --- | --- | --- |
| P08-Q01 | 사용자별 등록한 할 일 수와 가장 가까운 마감일은 언제인가? | 사용자 1명 | 할 일이 없는 사용자도 포함 | users → todos | 할 일 수, 가장 가까운 마감일 | 특정 사용자의 todos 행 수와 직접 비교 |
| P08-Q02 | 목표별 추천 퀘스트 수와 연결된 퀘스트 실행 기록 수는 얼마인가? | 목표 1개 | 추천·실행 기록이 없는 목표도 포함 | goals → quest_recommendations → quests → quest_completion_records | 추천 수, 퀘스트 수, 실행 기록 수 | 특정 목표의 추천·실행 기록을 직접 조회해 비교 |
| P08-Q03 | 퀘스트별 실행 기록 수와 마지막 완료 시점은 언제인가? | 퀘스트 1개 | 실행 기록이 없는 퀘스트도 포함 | quests → quest_completion_records | 실행 기록 수, 마지막 완료 시점 | 특정 퀘스트의 완료 기록 행 수·최대 완료일과 비교 |

## 12-1. 질문 1 SQL

```sql
SELECT
    u.id AS user_id,
    u.name AS user_name,
    COUNT(t.id) AS todo_count,
    MIN(t.due_at) AS nearest_due_at,
    MAX(t.created_at) AS latest_todo_created_at
FROM users AS u
LEFT JOIN todos AS t
    ON u.id = t.user_id
GROUP BY u.id, u.name
ORDER BY u.id;
```

```text
예상 결과:
사용자별 할 일 수와 가장 가까운 마감일이 1행씩 조회된다.
할 일이 없는 사용자는 todo_count가 0이고 마감일은 NULL이다.

실제 결과:
미실행 — PostgreSQL 테이블 미생성

검산 결과:
특정 사용자 1명의 todos 행을 직접 세어 todo_count와 비교한다.
```

## 12-2. 질문 2 SQL

```sql
SELECT
    g.id AS goal_id,
    g.title AS goal_title,
    COUNT(DISTINCT qr.id) AS recommendation_count,
    COUNT(DISTINCT q.id) AS recommended_quest_count,
    COUNT(DISTINCT qcr.id) AS completion_record_count
FROM goals AS g
LEFT JOIN quest_recommendations AS qr
    ON g.id = qr.goal_id
LEFT JOIN quests AS q
    ON qr.quest_id = q.id
LEFT JOIN quest_completion_records AS qcr
    ON qr.quest_id = qcr.quest_id
   AND qr.user_id = qcr.user_id
GROUP BY g.id, g.title
ORDER BY g.id;
```

```text
예상 결과:
목표별 추천 건수, 서로 다른 추천 퀘스트 수, 연결된 실행 기록 수가 조회된다.
추천이 없는 목표도 0건으로 남는다.

실제 결과:
미실행 — PostgreSQL 테이블 미생성

검산 결과:
특정 목표의 quest_recommendations 행 수와
연결된 quest_completion_records 행을 직접 비교한다.
```

## 12-3. 질문 3 SQL

```sql
SELECT
    q.id AS quest_id,
    q.title AS quest_title,
    q.reward_point,
    COUNT(qcr.id) AS completion_record_count,
    MAX(qcr.completed_at) AS last_completed_at
FROM quests AS q
LEFT JOIN quest_completion_records AS qcr
    ON q.id = qcr.quest_id
GROUP BY q.id, q.title, q.reward_point
ORDER BY q.id;
```

```text
예상 결과:
퀘스트별 실행 기록 수와 가장 최근 완료 시점이 1행씩 조회된다.
실행 기록이 없는 퀘스트는 completion_record_count가 0이고
last_completed_at은 NULL이다.

실제 결과:
미실행 — PostgreSQL 테이블 미생성

검산 결과:
특정 퀘스트의 quest_completion_records 행 수와
completed_at의 최대값을 직접 비교한다.
```

> 아직 개인 프로젝트 테이블을 PostgreSQL로 완성하지 않았다면 SQL 초안과 예상 검산 방법까지만 작성하고 `미실행`이라고 명시합니다.

---

# 13. AI를 JOIN·집계 리뷰어로 활용

## 13-1. 내가 AI에게 전달한 질문

```text
EgoQuest ERD를 기준으로 목표별 추천 퀘스트 수와 연결된 퀘스트 실행 기록 수를 조회하려고 한다.
결과 한 행은 목표 1개이며, 추천이나 실행 기록이 없는 목표도 결과에 남아야 한다.

goals → quest_recommendations → quests → quest_completion_records 경로를 사용하려 한다.
quest_completion_records는 quest_id와 user_id를 가지므로,
추천을 받은 사용자와 실행 기록의 사용자가 같은 경우만 연결하려 한다.

LEFT JOIN 위치, COUNT 대상, 중복 집계 위험을 검토해 달라.
```

## 13-2. 내 SQL과 AI SQL 비교

| 검토 항목 | 내 판단/SQL | AI 제안 | 최종 선택 | 이유 |
| --- | --- | --- | --- | --- |
| 결과 한 행 | 목표 1개 | 목표 1개 | 목표 1개 | 업무 질문의 기준 단위가 목표이기 때문 |
| 상태 범위 | 상태 조건 없이 모든 추천·실행 기록 포함 | 상태를 적용해야 한다면 `quest_completion_records.status`의 실제 값부터 정의 | 상태 조건 없음 | ERD에 상태값의 구체적 의미가 정의되지 않아 임의로 제외하지 않음 |
| JOIN 경로 | goals → quest_recommendations → quests → quest_completion_records | 실행 기록은 `quest_id`와 `user_id`를 모두 연결 | AI 제안 반영 | 같은 퀘스트라도 다른 사용자의 실행 기록이 섞이지 않게 하기 위함 |
| INNER/LEFT 선택 | LEFT JOIN | 모든 연결에서 LEFT JOIN | LEFT JOIN | 추천·실행 기록이 없는 목표도 0건으로 남겨야 함 |
| COUNT 대상 | 추천·퀘스트·실행 기록 수 집계 | `COUNT(DISTINCT qr.id)`, `COUNT(DISTINCT q.id)`, `COUNT(DISTINCT qcr.id)` | AI 제안 반영 | 다중 JOIN으로 같은 ID가 반복되는 과대 집계를 방지 |
| 과대 집계 위험 | 퀘스트와 실행 기록 JOIN 시 행 수 증가 가능 | `COUNT(*)` 대신 실제 PK를 DISTINCT로 집계 | AI 제안 반영 | 추천·퀘스트·실행 기록의 관계가 1:N이기 때문 |
| 상세 검산 방법 | 집계 결과만 확인 | 목표 하나를 골라 추천 행과 실행 기록 행을 상세 조회 | AI 제안 반영 | 집계값이 어떤 원본 행에서 나왔는지 직접 확인 가능 |

### AI가 만든 SQL에서 발견한 위험 또는 확인한 점

```textquest_completion_records에는 recommendation_id가 없으므로,
특정 실행 기록이 어느 추천으로 인해 발생했는지를 직접 증명할 수 없다.

따라서 이 SQL의 실행 기록 수는
'해당 목표에 추천된 퀘스트를 같은 사용자가 실행한 기록 수'로 해석해야 하며,
'해당 추천으로 인해 발생한 실행 수'라고 단정하면 안 된다.

또한 LEFT JOIN 결과에서 COUNT(*)를 사용하면
추천이나 실행 기록이 없는 목표도 1건으로 셀 수 있으므로,
각 테이블의 실제 PK를 COUNT해야 한다.
```

### AI SQL이 실행 성공했다고 바로 정답이라고 할 수 없는 이유

```textSQL 문법이 맞아도 결과 한 행의 기준, 상태 범위, JOIN 조건이
업무 질문의 의미와 맞지 않으면 잘못된 결과가 나올 수 있다.

특히 다중 JOIN에서는 실행은 성공하지만 행이 반복되어
COUNT나 SUM이 과대 집계될 수 있으므로,
상세 행 조회와 집계 결과를 함께 검산해야 한다.
```

---

# 14. 최종 성찰

아래 문장은 본인의 말로 작성합니다.

```text
1. JOIN SQL을 작성하기 전에 가장 먼저 정해야 하는 것은
   결과 한 행이 무엇을 의미하는지와 어떤 업무 질문에 답할 것인지 이다.

2. LEFT JOIN에서 COUNT(*) 대신 COUNT(child.id)를 검토해야 하는 이유는
   연결된 자식 행이 없어도 LEFT JOIN은 부모 행을 남기므로 실제 자식 건수를 정확히 세기 위해서 이다.

3. ON과 WHERE 조건 위치가 중요한 이유는
   ON은 연결할 행을 제한하지만 WHERE는 JOIN 이후 결과 행을 제거하여 LEFT JOIN의 결과가 달라질 수 있기 때문이다.

4. 여러 1:N 관계를 JOIN한 뒤 바로 SUM하면 위험한 이유는
   부모 행이 자식 행 수만큼 반복되어 같은 값이 여러 번 더해지는 과대 집계가 발생할 수 있기 때문이다.

5. 집계 결과를 신뢰하기 전에 가장 좋은 검산 방법 중 하나는
   특정 대상의 상세 행을 직접 조회해 건수와 금액을 손으로 더한 뒤 집계 결과와 비교하는 것이다.
```

---

# 15. 제출 체크리스트

- [x] `chapter08_answer.md`를 본인 저장소에 만들었다.
- [x] `00_check_course_project.sql`이 통과했다.
- [x] 업무 질문마다 결과 한 행을 먼저 정의했다.
- [x] INNER JOIN과 다중 JOIN을 실행했다.
- [x] LEFT JOIN에서 0건 부모를 확인했다.
- [x] `COUNT(*)`와 `COUNT(child.id)` 차이를 설명했다.
- [x] ON과 WHERE 조건 위치 차이를 직접 비교했다.
- [x] `LEFT JOIN ... IS NULL`과 `NOT EXISTS`를 비교했다.
- [x] 전체/활성/취소 제외 기준값을 직접 검산했다.
- [x] `GROUP BY`, `HAVING`을 사용했다.
- [x] 과대 집계 오류와 수정 결과를 비교했다.
- [x] 상세 결과와 집계 결과를 교차 검산했다.
- [x] `03_join_aggregation_validation.sql`이 통과했다.
- [x] 개인 프로젝트 업무 질문 3개를 작성했다.
- [x] AI SQL을 실행 성공 여부가 아니라 의미와 검산 결과로 평가했다.
- [x] 핵심 캡처는 3~4장 정도만 사용했다.
- [x] 비밀번호·개인정보·비밀정보가 없다.
- [x] GitHub 웹에서 Markdown과 이미지가 정상적으로 보인다.
- [x] 최종 답안을 commit/push했다.

---

# 16. LMS 제출 URL

아래 형식의 **본인 GitHub 파일 URL**을 LMS에 제출합니다.

```text
https://github.com/<본인-GitHub-ID>/<본인-저장소>/blob/main/assignments/chapter08/chapter08_answer.md
```

내 제출 URL:

```text
https://github.com/dydtlsrl/ai-data-analysis/blob/main/00_llm-data-analysis-course/assignments/chapter08_SQL/chapter08_answer.md
```

> 저장소 메인 URL, 교수자 템플릿 URL, Raw URL이 아니라 **작성 완료된 본인 `chapter08_answer.md` 파일 화면 URL**을 제출합니다.
