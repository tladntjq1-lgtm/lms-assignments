# Chapter 07 확장 실습 답안 템플릿

> **과제:** 실전 프로젝트 1 — 온라인 강의 수강신청 DB 완성하기  
> **사용 방법:** 이 파일을 내려받아 본인의 GitHub 저장소에 `chapter07_answer.md`라는 이름으로 저장한 뒤 실습하면서 바로 작성합니다.  
> **제출 방법:** LMS에는 파일을 직접 업로드하지 않고, **본인 GitHub 저장소의 `chapter07_answer.md` 파일 URL**을 제출합니다.

---

## 제출 전 주의

이 파일과 캡처 화면에는 실제 비밀번호, 전체 DB 접속 URL, API Key, 개인정보를 기록하지 않습니다.

```text
GitHub 계정 또는 별칭:tladntjq1-lgtm
과제 작성일:2026.9.15
사용한 AI 도구:cluade
```

---

# 1. 시작 환경 확인

다음을 실행합니다.

```sql
SELECT current_database();ai_database_book
SELECT current_user;postgres
SELECT current_schema();public
SHOW search_path;"$user", public
SHOW transaction_read_only;off
```

| 확인 항목 | 실제 결과 | 의미 |
| --- | --- | --- |
| `current_database()` |  |  |현재 데이터베이스
| `current_user` |  |  |현재 유저 내 포스트그레스계정
| `current_schema()` |  |  |현재 선택된 스키마
| `search_path` |  |  |접속경로
| `transaction_read_only` |  |  |읽기전용 off

- [x] 현재 DB가 `ai_database_book`이다.
- [x] 쓰기 가능한 연결인지 확인했다.
- [x] 실행할 SQL 범위를 확인했다.
- [x] Auto-commit 상태를 확인했다.

### 프로젝트 SQL을 실행하기 전에 시작 상태를 확인해야 하는 이유

```text

``` 실습할 db에 실행되지않고 다른 db를 보고있으면 결과가 달라지고 출력되는 값등 이 달라질 수 있고 이상한 쪽에 만들어질 수 있어서 

---

# 2. 프로젝트 범위와 요구사항 읽기

## 2-1. 포함 범위

본문을 그대로 복사하지 말고 자신의 말로 정리합니다.

```text
학생

강사

강의

수강신청

신청 상태

강의 기준 가격

신청 당시 기록 금액

```

## 2-2. 제외 범위

```text
실제 결제 승인·실패·환불

강의 정원·대기열

전체 상태 변경 이력

진도·수료·콘텐츠

쿠폰·할인 이력

```

### 범위를 명확하게 정해야 하는 이유

```text
테이블을 만들 때 범위를 명확히 하지않으면 관계없는 테이블이 너무많아지고 테이블구조가 너무 많아져서 범위를 명확하게 구분하여 재현가능하게 만들어야한다
```

## 2-3. 요구사항 / 프로젝트 결정 / 미확정 질문 구분

| ID | 종류 | 내용 요약 | DB 구조/규칙에 미치는 영향 |
| --- | --- | --- | --- |
| P07-R01 | 요구사항 | 학생은 이름, 이메일, 가입일을 가진다 | `students` 테이블에 `name`, `email`, `joined_at` 컬럼 필요 (모두 `NOT NULL`) |
| P07-R05 | 요구사항 | 수강신청은 학생, 강의, 신청일, 상태, 신청 시 기록 금액을 가진다 | `enrollments` 테이블에 `student_id`(FK), `course_id`(FK), `enrolled_at`, `status`, `recorded_amount` 컬럼 필요 |
| P07-R07 | 요구사항 | 학생·강사 이메일은 각 테이블 "안에서" 공백·동일 문자열 중복 불가 | `students.email`, `instructors.email` 각각에 `UNIQUE` + 공백 방지 `CHECK` 적용. 단 두 테이블 사이의 중복은 이 요구사항 범위 밖(→ P07-Q01) |
| P07-D02 | 프로젝트 결정 | 할인 기능이 없는 현재 버전에서는 신청 생성 시 `courses.price`를 `enrollments.recorded_amount`에 복사해 보존 | 신청 INSERT 시 `INSERT ... SELECT`로 그 순간의 가격을 복사. `recorded_amount NUMERIC(12,0) NOT NULL CHECK (>= 0)`. 이후 `courses.price`가 바뀌어도 이미 저장된 `recorded_amount`는 자동으로 따라 바뀌지 않음(CHECK로 강제 동기화하지 않음) |
| P07-D03 | 프로젝트 결정 | 같은 학생-강의 조합에서 "진행 중(신청/수강중)" 상태의 중복 신청 금지 | `(student_id, course_id)`에 `status IN ('신청','수강중')` 조건의 부분 고유 인덱스(`uq_course_enrollments_active`) 생성. 완료·취소된 이력은 여러 건 존재 가능 |
| P07-Q01 | 미확정 질문 | 학생과 강사 사이에서도 이메일을 전역 고유하게 제한해야 하는가? | 아직 결론이 없으므로 `students.email`↔`instructors.email` 간 전역 `UNIQUE` 제약은 만들지 않음. AI가 제안해도 이 단계에서는 확정하지 않고 미확정으로 남겨둠 |

### 미확정 질문을 바로 제약조건으로 만들면 안 되는 이유

```text
제약조건은 "확정된 요구사항"을 강제하는 도구이지, "그럴듯해 보이는 아이디어"를 강제하는 도구가 아니기 때문입니다.
```

---

# 3. 네 테이블의 한 행 의미와 관계

## 3-1. 한 행 의미

```text
course_project.students 한 행 =
students 한행은 학생한명의 데이터
course_project.instructors 한 행 =
instructors 한 행 강사한명의데이터
course_project.courses 한 행 =
courses 한 행 강의한개의 데이터
course_project.enrollments 한 행 =
```enrollments 한 행 특정학생이 특정강의에 신청한 사건 한건'''

## 3-2
| 테이블 | PK | FK | 중요 규칙 |
| --- | --- | --- | --- |
| students | `id` | 없음 | `email` `UNIQUE` + `NOT NULL`(공백 불가 `CHECK`), `name` `NOT NULL` |
| instructors | `id` | 없음 | `email` `UNIQUE` + `NOT NULL`(공백 불가 `CHECK`), `name` `NOT NULL` |
| courses | `id` | `instructor_id` → `instructors.id` (`ON DELETE RESTRICT`) | `price NUMERIC(12,0) NOT NULL CHECK (price >= 0)`, `difficulty`는 허용된 값 중 하나(`CHECK`), `title NOT NULL` |
| enrollments | `id` | `student_id` → `students.id`, `course_id` → `courses.id` (둘 다 `ON DELETE RESTRICT`) | `status`는 신청/수강중/완료/취소 중 하나(`CHECK`), `recorded_amount NUMERIC(12,0) NOT NULL CHECK (>= 0)`, `(student_id, course_id)`에 진행 중 상태만 걸리는 부분 고유 인덱스 `uq_course_enrollments_active` |

## 3-3. 관계를 양방향 문장으로 작성

instructors ↔ courses:
한 강사는 0개 이상의 강의를 담당할 수 있다.
한 강의는 정확히 한 명의 강사를 참조한다.

students ↔ enrollments:
한 학생은 0개 이상의 수강신청을 가질 수 있다.
한 수강신청은 정확히 한 명의 학생을 참조한다.

courses ↔ enrollments:
한 강의는 0개 이상의 수강신청을 가질 수 있다.
한 수강신청은 정확히 한 개의 강의를 참조한다.


### 학생과 강의의 N:M 관계가 `enrollments`를 통해 어떻게 바뀌는지 설명

```text
중간테이블이 생기면서 복잡한관계를 enrollments가 풀어주면서
1:n관계를 풀어주고 있습니다.
```

### `enrollments`가 단순 연결 테이블이 아니라 사건 테이블이라고 볼 수 있는 이유

단순 연결 테이블이라면 student_id와 course_id만 있으면 되지만,
enrollments는 enrolled_at(신청일), status(상태), recorded_amount(신청 당시
기록 금액)처럼 "그 신청이라는 사건 자체에 속하는 속성"을 갖고 있다.
이 값들은 학생 전체나 강의 전체를 설명하는 값이 아니라, 특정 학생이
특정 강의를 신청한 그 순간(사건)에만 해당하는 값이다. 그래서 이 테이블의
한 행은 "학생-강의 쌍"이 아니라 "학생이 강의를 신청한 사건 한 건"을
의미한다.
# 4. `recorded_amount`의 의미 이해

```text
courses.price = 현재 시점 기준 그 강의의 기준 가격 (강의 자체에 속한, 계속 최신값으로 갱신되는 사실)

enrollments.recorded_amount = 그 신청이 만들어지던 순간에 신청 행에 복사해 저장한 금액 (신청이라는 사건에 속한, 그 시점에 고정되는 과거 기록)

### 두 값이 처음에는 같아도 같은 의미가 아닌 이유

```text
두 값이 처음에는 같아도 같은 의미가 아닌 이유

신청을 생성하는 순간에는 courses.price 값을 그대로 recorded_amount에
복사하기 때문에 두 값이 우연히 같다. 하지만 courses.price는 강의의
"현재" 가격이라서 나중에 가격이 바뀌면 그 즉시 새 값으로 바뀐다. 반면
recorded_amount는 이미 지나간 신청 사건에 기록된 값이라서 강의 가격이
바뀌어도 자동으로 따라 바뀌지 않는다. 즉 courses.price는 "지금 얼마인가"
를 답하고, recorded_amount는 "그때 얼마였는가"를 대답한다는 점에서
같은 숫자여도 가리키는 사실 자체가 다르다.


recorded_amount를 실제 결제 성공액이나 회계 매출로 해석하면 안 되는 이유

recorded_amount는 그냥 신청할 때 강의 가격을 복사해서 적어둔 숫자일
뿐이다. 실제로 돈을 냈는지, 결제가 성공했는지, 나중에 환불됐는지는
이 값과 상관이 없다. 이번 프로젝트에는 결제나 환불 관련 테이블 자체가
없기 때문에, recorded_amount를 매출로 계산하면 결제 안 한 신청이나
환불된 신청까지 매출에 포함되는 오류가 생길 수 있다.


```

---

# 5. STEP 01 — 스키마와 테이블 생성

실행 파일:

```text
code/chapter07/01_course_project_schema.sql
```

## 5-1. 실행 전 예상

```text
course_project 스키마 존재 여부: 실행전이니까 없음
예상 테이블 수: 4
예상 데이터 행 수: 0 (스키마만 만들고 아직 데이터는 안 넣으니까)
예상되는 명명 제약조건 수: 15개
예상되는 NOT NULL 열 수: 20개
부분 고유 인덱스 존재 여부: 존재함 (uq_course_enrollments_active)
```

## 5-2. 실행 결과

```text
실제 테이블 수:4
실제 명명 제약조건 수:15
실제 NOT NULL 열 수:20
부분 고유 인덱스:1
네 테이블의 실제 행 수:0
통과 메시지:'Chapter 07 course project schema creation passed'
```

### 예상과 실제 비교

```완벽히 일치

```

### 증거 화면

권장 경로:

```text
assignments/chapter07/images/step05_schema.png
```

`여기에 스키마/테이블 생성 검증 화면을 삽입하세요.`

---

# 6. STEP 02 — Seed 데이터 입력

실행 파일:

```text
code/chapter07/02_course_project_seed.sql
```

## 6-1. 실행 전 예상

```text
students:3
instructors:2
courses:3
enrollments:4
recorded_amount 합계:470000
학생 101 신청 건수:2
강의 301 신청 건수:2
강사 201 담당 강의 수:2
활성 중복 신청:0
```

## 6-2. 실제 결과

```text
students:3
instructors:2
courses:3
enrollments:4
recorded_amount 합계:470000
학생 101 신청 건수:2
강의 301 신청 건수:2
강사 201 담당 강의 수:2
활성 중복 신청:0
1001 상태:수강중
1004 상태:신청
1005 존재 여부:없음
통과 메시지:Chapter 07 course project seed passed
```

### Seed 데이터를 단순 예제가 아니라 검증 데이터라고 볼 수 있는 이유

```text
Seed 데이터는 화면에 뭔가 보이게 하려고 아무 값이나 채운 게 아니라,
관계와 규칙이 실제로 지켜지는지 확인하기 위해 일부러 설계된 데이터다.
예를 들어 학생 101에게 신청을 2건 넣은 건 "한 학생이 여러 신청을 가질
수 있다"는 1:N 관계를 확인하기 위한 것이고, 강의 301에 신청 2건, 강사
201에 강의 2개를 넣은 것도 같은 이유다. 신청마다 상태와 금액을 다르게
넣은 것도 enrollments가 단순 연결이 아니라 사건별로 다른 값을 가진다는
걸 보여주기 위해서다. 그래서 이 데이터는 "예시용"이 아니라 요구사항이
실제로 구조에 반영됐는지 증명하는 검증용 데이터다.
```

---

# 7. STEP 03 — 변경 시나리오 실행

실행 파일:

```text
code/chapter07/03_course_project_changes.sql
```

## 7-1. 실행 전에 상태 변화를 예상

| 신청 ID | 변경 전 예상 상태 | 변경 후 예상 상태 | 예상 
students = 3
instructors = 2
courses = 3
enrollments = 5
1001 = 완료 / recorded_amount 100000
1004 = 취소 / recorded_amount 150000
1005 = 신청 / recorded_amount 120000
활성 중복 = 0
전체 recorded_amount = 590000
취소 제외 recorded_amount(4건) = 440000
```

## 7-2. 실제 결과

```text
1001 상태 / recorded_amount:100,000
1004 상태 / recorded_amount:150,000
1005 상태 / recorded_amount:120,000
최종 enrollments 행 수:5
전체 recorded_amount 합계:590000
취소 제외 건수:4
취소 제외 recorded_amount 합계:440000
활성 중복 신청:0
통과 메시지:Chapter 07 course project changes passed
```

### 조건부 UPDATE에서 예상 이전 상태를 확인해야 하는 이유

```text
조건부 UPDATE에서 예상 이전 상태를 확인해야 하는 이유

그냥 무조건 바꾸면 중간에 다른데서 이미 상태가 바뀌었어도 모르고
덮어쓸 수 있음. 그래서 WHERE에 이전 상태(수강중 등)를 같이 넣어서
내가 생각한 상태랑 실제 상태가 같을 때만 바뀌게 하는거임. 다르면
아예 안바뀌니까 실수로 잘못 덮어쓰는걸 막아줌.
```

### 증거 화면

권장 경로:

```text
assignments/chapter07/images/step07_changes.png
```

`여기에 주요 변경 전/후 결과를 삽입하세요.`

---

# 8. STEP 04 — 최종 완료 게이트 실행

실행 파일:

```text
code/chapter07/04_course_project_validation.sql
```

## 8-1. 최종 검증 결과

```text
students=3, instructors=2, courses=3, enrollments=5
서비스 JOIN 결과 행 수: 5
학생 101 신청 수: 2
강의 301 신청 수: 2
강사 201 강의 수: 2
고아 관계 수: 0
활성 중복 신청 수: 0
전체 recorded_amount: 590000
취소 제외 recorded_amount: 440000
통과 메시지: Chapter 07 course project validation passed
```

### SQL 파일 4개가 모두 실행되었다는 사실과 프로젝트 검증 PASS가 다른 이유

```text
파일이 오류 없이 실행됐다는건 그냥 문법이나 제약조건 어긴게 없었다는
뜻일 뿐임. 근데 실제로 행 수가 맞는지, 금액 계산이 맞는지, 고아
데이터 없는지, 중복 없는지 같은건 따로 확인 안하면 모름. 그래서
validation 파일이 이런것들을 하나하나 다 체크해서 전부 맞아야만
PASS를 띄워주는거고, 이게 진짜 프로젝트가 요구사항대로 완성됐다는
증거임. 그냥 실행만 됐다고 결과가 맞다는 보장은 안됨.

```

### 증거 화면

권장 경로:

```text
assignments/chapter07/images/step08_validation.png
```

`여기에 최종 validation PASS 화면을 삽입하세요.`

---

# 9. 무결성 테스트

실행 파일:

```text
code/chapter07/05_course_project_integrity_tests.sql
```

> 오류 테스트는 파일 전체를 무작정 실행하지 않고 **한 테스트 구간씩** 실행합니다.

## 9-1. 허용되어야 하는 경계값 1개

```text
테스트 내용: 무료 강의(price=0, description=NULL) 개설 + 무료 신청(recorded_amount=0) 등록 후 임시 행 삭제
기대 결과: 성공 (CHECK 제약조건 위반 없이 INSERT 통과)
실제 결과: 성공 — INSERT 2건, DELETE 2건 모두 정상 처리됨 (Updated Rows: 4, 오류 없음)
왜 허용되어야 하는가: price와 recorded_amount는 CHECK (>= 0)만 만족하면 되므로 0은 유효한 값이고, description은 NOT NULL이 아니라서 NULL도 정상적으로 허용되는 값이기 때문
```

## 9-2. 실패해야 하는 테스트 1 — 잘못된 참조 또는 값

```text
테스트 내용: 학생 이름(name)에 NULL 값을 넣어서 students 테이블에 INSERT 시도
기대 결과: 실패
실제 오류 핵심: [23502] "name" 칼럼의 null 값이 not null 제약조건을 위반했습니다
동작한 제약조건/규칙: students.name의 NOT NULL 제약조건
왜 실패해야 하는가: 요구사항(P07-R01)상 학생은 반드시 이름을 가져야 하므로, 이름 없이 학생 데이터를 저장하는 건 허용되면 안 되기 때문
```

## 9-3. 실패해야 하는 테스트 2 — 활성 중복 신청

```text
테스트 내용: 학생 101·강의 302에 이미 활성(수강중) 신청이 있는 상태에서, 같은 학생·강의 조합으로 새 신청(수강중, id=1910)을 또 삽입 시도
기대 결과: 실패
실제 오류 핵심: [23505] 중복된 키 값이 "uq_course_enrollments_active" 고유 제약 조건을 위반함, 세부: (student_id, course_id)=(101, 302) 키가 이미 있음
동작한 인덱스/규칙: uq_course_enrollments_active 부분 고유 인덱스 (student_id, course_id 조합에 대해 status가 '신청' 또는 '수강중'인 행에만 적용되는 UNIQUE)
왜 실패해야 하는가: 같은 학생이 같은 강의를 이미 진행 중(신청/수강중)인데 또 신청하는 건 업무 규칙(P07-D03)상 허용되지 않기 때문

## 9-4. 실패 후 기준 상태 재검증

```text
04 validation 재실행 결과: 5개 행(enrollments), 전체 recorded_amount=590000, 취소 제외 4건/440000 — 모두 기존 기준값과 일치
기준 데이터가 유지되었는가: 예. 실패 테스트(NOT NULL, UNIQUE, CHECK, 활성 중복 신청 등)들이 ROLLBACK으로 정리되어 기존 데이터에 아무 영향을 주지 않았음을 확인함

### 실패 테스트가 프로젝트 품질 검증에 필요한 이유

```text
정상 케이스만 확인하면 규칙이 진짜 막아주는지 모름. 예를 들어 FK나
중복 방지 인덱스를 걸어놨어도 실제로 잘못된 값을 넣어봐야 진짜
막히는지 확인이 됨. 그리고 실패 테스트 하고 나서도 기존 데이터가
그대로 있어야 이 실패가 다른 데이터를 망가뜨리지 않는다는것도
확인할 수 있음.
```

### 증거 화면

권장 경로:

```text
assignments/chapter07/images/step09_integrity.png
```

`여기에 대표 실패 테스트와 기준 상태 유지 결과를 삽입하세요.`

---

# 10. 재현성 실험
### x
> 이 단계는 본인의 실습 환경이며 보존할 데이터가 없을 때만 수행합니다.

실행 순서:

```text
reset_course_project.sql
→ 01_course_project_schema.sql
→ 02_course_project_seed.sql
→ 03_course_project_changes.sql
→ 04_course_project_validation.sql
```

```text
처음 실행의 최종 결과:
재실행의 최종 결과:
두 결과가 일치했는가:
중간에 수동 수정이 필요했는가:
```

### 다른 사람이 같은 순서로 실행해 같은 결과를 얻는 것이 중요한 이유

```text

```

---

# 11. Chapter 01~06 개인 프로젝트를 중간 프로젝트 초안으로 확장

온라인 강의 예제를 이름만 바꾸지 않고 본인의 아이디어를 사용합니다.

## 11-1. 프로젝트 기본 정보

프로젝트 이름: 나의 지출내역 관리
해결하려는 문제: 개인의 일상 지출을 카테고리·결제수단별로 기록하고, 카테고리별 예산과 비교해서 조회하기 위함
주요 사용자: 가계부를 직접 쓰는 개인(본인, 단일 사용자)

## 11-2. 포함 범위 / 제외 범위

[포함]
1. 지출 카테고리 관리
2. 지출 내역(사건) 기록 — 금액/지출일/메모
3. 결제수단 관리 및 지출 내역에 선택적으로 연결
4. 카테고리별 월 예산 설정 및 조회

[제외]
1. 여러 사용자(가족 공유 가계부) 지원
2. 예산 초과 시 자동 알림/경고 기능
3. 카드사·은행 API 연동, 자동 지출 인식

## 11-3. 요구사항

최소 8개를 작성합니다.
| ID | 요구사항 | 관련 테이블/관계 | 검증 방법 후보 |
| --- | --- | --- | --- |
| P07-MR01 | 시스템은 지출 카테고리를 관리한다 | categories | categories에 행이 저장되는지 SELECT로 확인 |
| P07-MR02 | 카테고리는 이름과 설명을 가진다 | categories.name, description | 두 컬럼 값이 정상 입력되는지 확인 |
| P07-MR03 | 사용자는 지출 내역을 여러 건 기록할 수 있다 | expenses | 같은 카테고리로 여러 expenses 행이 쌓이는지 확인 |
| P07-MR04 | 지출 내역은 금액, 지출일, 메모를 가진다 | expenses.amount/spent_at/memo | 세 컬럼 값이 정상 입력되는지 확인 |
| P07-MR05 | 지출 내역은 정확히 하나의 카테고리를 참조한다 | expenses.category_id (FK NOT NULL) | category_id NULL 입력 시 실패하는지 확인 |
| P07-MR06 | 지출 내역은 결제수단을 선택적으로 가질 수 있다 | payment_methods ── expenses (FK, NULL 허용) | payment_method_id 없이도 저장되는지 확인 |
| P07-MR07 | 사용자는 카테고리별로 월 단위 예산을 설정할 수 있다 | budgets (category_id, year_month, limit_amount) | budgets에 행이 저장되는지 확인 |
| P07-MR08 | 사용자는 특정 기간의 지출 합계와 해당 카테고리 예산을 비교 조회할 수 있다 | expenses + budgets (JOIN) | 기간별 SUM(amount)과 budgets.limit_amount를 함께 조회하는 쿼리 확인 |

## 11-4. 프로젝트 결정

| ID | 이번 프로젝트에서 내린 결정 | 이유 | 구현 후보 |
| --- | --- | --- | --- |
| P07-MD01 | 결제수단은 expenses에 문자열로 직접 넣지 않고 별도 payment_methods 테이블로 관리한다 | 같은 결제수단 값이 여러 지출에서 반복되므로(정규화), 오타·표기 불일치를 막기 위함 | payment_methods 테이블 + FK |
| P07-MD02 | 카테고리·월 조합당 예산은 하나만 존재할 수 있다 | 같은 달에 같은 카테고리 예산이 여러 개면 어떤 게 맞는 예산인지 알 수 없음 | UNIQUE(category_id, year_month) |
| P07-MD03 | 지출 내역을 수정·삭제해도 이전 값의 변경 이력은 따로 저장하지 않는다 | 현재 범위에서는 이력 관리까지 다루지 않기로 함(확장 백로그로 남김) | 별도 이력 테이블 없음 |

## 11-5. 미확정 질문

P07-MQ01. 카테고리 이름은 반드시 고유해야 하는가? (Chapter05 P05-Q01에서 계속 미확정)
P07-MQ02. 예산을 초과한 지출을 시스템이 막아야 하는가, 아니면 조회만 가능하면 되는가?
P07-MQ03. 지출·예산 데이터를 나중에 여러 사용자로 확장할 가능성을 지금부터 대비해야 하는가?
```

---

# 12. 개인 프로젝트 ERD와 한 행 의미

## 12-1. 테이블 후보

최소 4개를 권장합니다.

| 테이블 | 한 행의 의미 | PK 후보 | FK 후보 | 주요 규칙 |
| 테이블 | 한 행의 의미 | PK 후보 | FK 후보 | 주요 규칙 |
| --- | --- | --- | --- | --- |
| categories | 지출 카테고리 1종 | id | 없음 | name NOT NULL (고유 여부는 미확정 — P07-MQ01) |
| payment_methods | 결제수단 1종 (현금/카드/계좌이체 등) | id | 없음 | name NOT NULL, UNIQUE |
| budgets | 특정 카테고리의 특정 월 예산 계획 1건 | id | category_id → categories.id | limit_amount >= 0 (CHECK), UNIQUE(category_id, year_month) |
| expenses | 지출이 발생한 사건 1건 | id | category_id → categories.id (NOT NULL), payment_method_id → payment_methods.id (NULL 허용) | amount >= 0 (CHECK), spent_at NOT NULL |

## 12-2. 관계 문장

```text
1. 카테고리 한 종은 여러 지출 내역에서 반복 사용될 수 있다.
   지출 내역 한 건은 정확히 하나의 카테고리를 참조한다.
   → categories 1 ── N expenses

2. 카테고리 한 종은 여러 개의 월별 예산을 가질 수 있다(단 같은 달 예산은 하나뿐).
   예산 한 건은 정확히 하나의 카테고리를 참조한다.
   → categories 1 ── N budgets

3. 결제수단 한 종은 여러 지출 내역에서 반복 사용될 수 있다.
   지출 내역 한 건은 결제수단을 하나 참조하거나, 참조하지 않을 수 있다(선택).
   → payment_methods 1 ── N expenses (FK nullable)
```

## 12-3. ERD

권장 이미지 경로:

```text
assignments/chapter07/images/personal_project_erd.png
```

`여기에 본인의 ERD 이미지를 삽입하세요.`

### Chapter 05~06 ERD에서 이번에 바꾼 점

```text

```

---

# 13. 개인 프로젝트 완료 기준 만들기

“잘 동작한다”처럼 모호하게 쓰지 말고 검증 가능한 기준을 최소 6개 작성합니다.

| 번호 | 완료 기준 | 자동 SQL 검증 가능? | 검증 방법 |
| ---: | --- | --- | --- |
| 1 | Seed 실행 후 categories/payment_methods/budgets/expenses 행 수가 예상값과 같다 | 예 | 각 테이블 COUNT(*)로 확인 |
| 2 | expenses가 존재하지 않는 category_id·payment_method_id를 참조하는 고아 행이 0건이다 | 예 | LEFT JOIN 후 NULL 확인 쿼리 |
| 3 | amount < 0 이거나 limit_amount < 0인 값은 저장이 거부된다 | 예 | 음수 INSERT 시도 → CHECK 위반 오류 확인 |
| 4 | 같은 카테고리·같은 월에 예산이 두 건 이상 저장되지 않는다 | 예 | 중복 INSERT 시도 → UNIQUE 위반 오류 확인 |
| 5 | 특정 기간의 지출 합계와 해당 카테고리 예산을 함께 조회하는 쿼리가 정상적으로 값을 반환한다 | 예 | JOIN + SUM + GROUP BY 쿼리 실행 확인 |
| 6 | 지출 내역이 하나도 없어도 카테고리·결제수단은 독립적으로 등록·조회될 수 있다 | 예 | 지출 없는 카테고리/결제수단 INSERT 후 SELECT로 존재 확인 |

예시 형식:

```text
Seed 실행 후 A/B/C/D 테이블의 행 수가 각각 5/3/8/12다.
존재하지 않는 부모를 참조하는 행은 0건이다.
허용되지 않은 상태 입력은 DB가 거부한다.
검증 SQL이 예상 결과를 반환한다.
```

---

# 14. AI를 프로젝트 리뷰어로 사용


내가 사용한 프롬프트

```text
나는 데이터베이스 입문 수업의 중간 프로젝트 초안을 작성하고 있습니다.
아래 프로젝트를 대신 완성하지 말고 리뷰어로 검토해 주세요.
특히 다음을 찾아 주세요.
1. 요구사항과 ERD가 연결되지 않는 부분
2. 한 행의 의미가 섞인 테이블
3. 빠진 PK/FK 관계 후보
4. 근거 없이 추가한 UNIQUE / NOT NULL / CHECK / CASCADE
5. 미확정 정책을 내가 임의로 확정한 부분
6. Seed 데이터로 검증하기 어려운 요구사항
7. 모호해서 자동 검증할 수 없는 완료 기준
8. 아직 배우지 않은 기능을 꼭 넣지 않아도 되는 부분

정답 ERD나 전체 SQL을 먼저 작성하지 말고,
문제점과 확인 질문을 우선순위 순으로 알려 주세요.
```


# 15. 최종 성찰

아래 문장은 반드시 본인의 말로 작성합니다.

1. 데이터베이스 프로젝트가 완료되었다고 판단하려면
   SQL 파일의 존재보다 다른 사람이 같은 순서로 실행해도 같은 구조와
   데이터가 나오고, 실패 케이스까지 검증됐는지가 중요하다.

2. Seed 데이터의 목적은 단순히 화면을 채우는 것이 아니라
   1:N 관계, 여러 상태값, 금액 검산 같은 요구사항이 실제로 구조에
   반영됐는지 확인하기 위한 것이다.

3. 실패 테스트가 필요한 이유는
   제약조건이 진짜로 잘못된 데이터를 막아주는지 직접 확인해봐야
   믿을 수 있기 때문이다.

4. 요구사항과 프로젝트 결정을 구분해야 하는 이유는
   요구사항은 무조건 지켜야 하는 거고 결정은 이번 버전에서만 임의로
   정한 거라, 나중에 결정이 바뀌어도 요구사항 자체는 안 바뀌기 때문이다.

5. 내가 만든 개인 프로젝트에서 가장 먼저 추가 확인해야 할 정책은
   예산을 초과했을 때 시스템이 막아야 하는지, 아니면 그냥 보여주기만
   하면 되는지(P07-MQ02)이다.

---

# 16. 제출 체크리스트

- [x] `chapter07_answer.md`를 본인 저장소에 만들었다.
- [x] 시작 환경과 현재 DB를 확인했다.
- [x] 프로젝트 포함/제외 범위를 설명했다.
- [x] 요구사항/결정/미확정 질문을 구분했다.
- [x] 네 테이블의 한 행 의미와 관계를 설명했다.
- [x] `01_course_project_schema.sql`을 실행하고 결과를 확인했다.
- [x] `02_course_project_seed.sql`의 기준 상태를 확인했다.
- [x] `03_course_project_changes.sql` 전후 상태를 비교했다.
- [x] `04_course_project_validation.sql` PASS를 확인했다.
- [x] 허용 경계값 1개 이상을 확인했다.
- [x] 실패 테스트 2개 이상을 한 구간씩 실행했다.
- [x] 실패 후 validation을 다시 실행했다.
- [x] 개인 프로젝트 요구사항 8개 이상을 작성했다.
- [x] 프로젝트 결정 3개 이상과 미확정 질문 3개 이상을 작성했다.
- [x] 개인 프로젝트 ERD를 작성했다.
- [x] 검증 가능한 완료 기준 6개 이상을 작성했다.
- [x] AI 제안을 수용/수정/보류/거절로 구분했다.
- [x] 핵심 캡처는 3~4장 정도로 정리했다.
- [x] 캡처에 비밀번호나 개인정보가 없다.
- [x] GitHub 웹에서 Markdown과 이미지가 정상적으로 보인다.
- [x] 최종 파일을 commit/push했다.

---

# 17. LMS 제출 URL

아래 형식의 **본인 GitHub 파일 URL**을 LMS에 제출합니다.

```text
https://github.com/<본인-GitHub-ID>/<본인-저장소>/blob/main/assignments/chapter07/chapter07_answer.md
```

내 제출 URL:

```text
https://github.com/tladntjq1-lgtm/ai-database-seob/blob/main/assignments/chapter07/chapter07_answer.md
```

> 저장소 메인 URL, 교수자 템플릿 URL, Raw URL이 아니라 **작성 완료된 본인 `chapter07_answer.md` 파일 화면 URL**을 제출합니다.