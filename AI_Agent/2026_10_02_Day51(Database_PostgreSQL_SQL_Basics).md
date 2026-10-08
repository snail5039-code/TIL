# 26.10.02 51일차(Database · PostgreSQL · SQL 기초)

## [TIL] PostgreSQL로 데이터베이스와 SQL 기초 배우기

오늘은 데이터베이스가 무엇인지부터 시작해 테이블 사이의 관계, 스키마와 ERD를 배우고, **PostgreSQL 17**을 설치해 `world` 데이터베이스로 SQL을 직접 실행해 봤다.

이론과 실습은 다음 흐름으로 이어졌다.

```plain text
데이터베이스·DBMS 개념
→ 관계형 DB의 테이블·행·열, 기본 키·외래 키
→ 1:N / M:N / 1:1 관계와 ERD 읽기
→ NoSQL 종류 비교
→ PostgreSQL 설치 + VSCode 연결
→ DDL로 구조 만들기, DML로 데이터 다루기
→ SELECT: WHERE → ORDER BY → LIMIT/OFFSET → GROUP BY → HAVING
```

실습은 `sql/create_db.sql`로 `world`, `dvdrental` 데이터베이스를 만든 뒤 `world.sql`을 불러와 `sql/select_problem.sql`에서 문제를 풀었다.

---

## 1. 데이터베이스와 DBMS

데이터베이스(Database)는 체계적으로 구조화된 데이터의 집합이다. 여러 사용자가 공유해서 사용할 수 있고, 설계와 제약 조건을 잘 잡으면 중복을 줄이고 일관성을 유지할 수 있다.

DBMS(Database Management System)는 이 데이터를 저장·조회·수정·삭제할 수 있게 도와주는 소프트웨어다.

```plain text
데이터베이스(DB) → 저장된 데이터의 집합
DBMS            → 그 데이터를 관리하는 소프트웨어 (PostgreSQL, MySQL, Oracle, SQLite)
```

---

## 2. 관계형 데이터베이스: 테이블 · 행 · 열

관계형 데이터베이스는 데이터를 **테이블**로 나누어 저장하고, 테이블 사이에 관계를 설정해 함께 조회한다. 데이터를 다룰 때는 SQL을 사용한다.

| 용어 | 다른 이름 | 의미 | 예시 |
| --- | --- | --- | --- |
| 테이블(Table) | - | 데이터를 저장하는 기본 단위 | 고객 테이블, 주문 테이블 |
| 행(Row) | 레코드, 튜플 | 개별 항목 하나의 정보 | ‘김민준’ 고객 한 명의 정보 |
| 열(Column) | 필드, 속성 | 각 행이 가지는 속성 | 고객이름, Email |

```plain text
고객 테이블
고객ID | 고객이름 | Email
1      | 김민준   | minjun@example.com
2      | 이서연   | seoyeon@example.com

주문 테이블
주문ID | 고객ID | 상품명 | 날짜
101    | 1      | 노트북 | 2024-01-08
103    | 1      | 태블릿 | 2024-01-12
```

---

## 3. 기본 키(PK)와 외래 키(FK)

**기본 키(Primary Key)**는 각 행을 고유하게 식별하는 열(또는 열의 조합)이다.

- 유일성: 값이 중복될 수 없다. 여러 열로 구성하면 그 조합이 유일해야 한다.

- NOT NULL: 기본 키를 구성하는 열에는 NULL을 넣을 수 없다.

**외래 키(Foreign Key)**는 다른 테이블(또는 같은 테이블)의 기본 키나 고유 키를 참조하는 열이다.

```plain text
고객.고객ID  → 고객 테이블의 PK
주문.주문ID  → 주문 테이블의 PK
주문.고객ID  → 고객.고객ID를 참조하는 FK
```

---

## 4. 테이블 사이의 관계

| 관계 | 의미 | 예시 |
| --- | --- | --- |
| 1:N | 한 행이 상대 테이블의 여러 행과 연결 | 고객 - 주문, 게시글 - 댓글 |
| M:N | 양쪽 모두 상대의 여러 행과 연결 | 학생 - 강좌, 배우 - 영화 |
| 1:1 | 한 행이 상대 테이블의 한 행과만 연결 | 주문 - 환불(주문당 최대 1회) |

### M:N 관계와 연관(중계) 테이블

관계형 DB는 M:N을 직접 표현할 수 없어서 **연관 테이블**을 두고 두 개의 1:N으로 분해한다.

```plain text
학생 1 ── N 수강 N ── 1 강좌

수강 테이블
학생ID(FK) | 강좌ID(FK) | 수강일
S001       | C001       | 2024-01-08
S001       | C003       | 2024-01-12
S003       | C001       | 2024-01-15
```

- 연관 테이블에는 양쪽 테이블의 PK가 FK로 들어간다.

- `(학생ID, 강좌ID)`를 복합 기본 키로 잡으면 같은 학생이 같은 강좌에 중복 등록되는 것을 막을 수 있다.

- 수강일처럼 관계 자체가 가지는 정보도 연관 테이블에 넣을 수 있다.

### 같은 대상도 관점에 따라 관계가 달라진다

```plain text
담임 교사 관점 → 교사 1명 : 학생 여러 명, 학생 1명 : 담임 1명  → 1:N
과목 교사 관점 → 교사 1명 : 학생 여러 명, 학생 1명 : 교사 여러 명 → M:N
```

어떤 서비스를 만들고 싶은지에 따라 모델이 달라지기 때문에, 기획이 구체적이어야 테이블 설계도 정할 수 있다.

### 1:1 관계를 쓰는 경우

- 보안이 필요한 민감 데이터 분리

- 자주 쓰는 데이터와 그렇지 않은 데이터 분리

- 선택적으로 존재하는 데이터 관리 (예: 환불은 있을 수도, 없을 수도 있음)

- 매우 큰 데이터를 별도로 관리

### 관계 생각해 보기

```plain text
부서 - 직원   → 한 부서에 여러 직원, 직원은 한 부서 소속 → 1:N
동아리 - 회원 → 여러 동아리 가입 가능, 동아리엔 여러 회원 → M:N
직원 - 사물함 → 직원 1명당 사물함 1개                   → 1:1
```

---

## 5. 스키마와 ERD

**스키마(Schema)**는 테이블, 열, 데이터 타입, 키, 제약 조건 같은 **구조의 정의**이고, **데이터**는 그 구조에 실제로 저장된 값이다.

```plain text
데이터 변경 → 고객 추가, 이메일 수정
스키마 변경 → 열 추가, 기본 키 변경
```

**ERD(Entity-Relationship Diagram)**는 데이터의 구성과 관계를 그림으로 표현한 것이다. 실제 행이 아니라 저장 구조를 보여준다.

| 구성 요소 | 의미 | 예시 |
| --- | --- | --- |
| 엔티티(Entity) | 관리하려는 대상, 보통 테이블 | 고객, 주문, 상품 |
| 속성(Attribute) | 대상이 가지는 정보, 테이블의 열 | 고객ID, 고객이름 |
| PK / FK | 식별 키 / 참조 키 | customer_id |
| 관계(Relationship) | 테이블 사이를 잇는 선 | CUSTOMER ── ORDERS |

### 까마귀 발 표기법(Crow's Foot)

관계선 끝의 기호는 **그 기호가 붙은 쪽 테이블의 행이 몇 개 연결될 수 있는지**를 나타낸다.

```plain text
CUSTOMER ||──────o{ ORDERS

||  → 정확히 하나 (주문 한 건의 고객은 반드시 1명)
o{  → 0개 이상    (고객 한 명의 주문은 없거나 여러 건)
o|  → 0 또는 1개  (주문 한 건의 환불은 없거나 1건)
```

고객 한 명이 주문을 몇 건 할 수 있는지 알고 싶으면 **주문 쪽 기호**를 본다.

---

## 6. 비관계형 데이터베이스(NoSQL)

NoSQL(Not only SQL)은 테이블 외에 다양한 데이터 모델을 쓰는 데이터베이스를 통칭한다.

| 종류 | 저장 방식 | 활용 예 | 제품 |
| --- | --- | --- | --- |
| 키-값 | 키와 값의 쌍 | 세션 ID로 로그인 정보 조회, 캐싱 | Redis, DynamoDB |
| 그래프 | 노드·엣지·속성 | 친구의 친구 찾기 | Neo4j |
| 문서 | JSON 같은 문서 | 상품마다 다른 속성 저장 | MongoDB |
| 와이드 컬럼 | 파티션 단위 분산 저장 | 기기별 센서 로그, 시계열 | Cassandra |

관계형 DB는 외래 키와 JOIN으로 데이터를 연결하고, 그래프 DB는 노드와 관계를 직접 저장해서 탐색한다. 결국 데이터 구조와 조회 방식에 맞는 DB를 고르는 것이 핵심이다.

---

## 7. PostgreSQL 설치와 연결

### 설치 방법

1. EnterpriseDB 다운로드 페이지에 접속한다. → [https://www.enterprisedb.com/downloads/postgres-postgresql-downloads](https://www.enterprisedb.com/downloads/postgres-postgresql-downloads)

1. **PostgreSQL 17** 버전의 Windows 설치 파일을 받는다.

1. 설치 파일을 실행하고 기본 설정 그대로 **Next**를 계속 누른다.

1. 비밀번호 설정 화면에서 슈퍼유저(`postgres`)의 비밀번호를 정한다. 수업에서는 `1234`로 통일했다.

1. 포트는 기본값 **5432**를 그대로 두고 설치를 끝낸다.

```plain text
설치 파일 실행 → Next 반복 → 비밀번호 설정(postgres 계정) → 포트 5432 → 설치 완료
```

### VSCode에서 DB 연결

1. VSCode Extensions에서 PostgreSQL 확장을 설치한다.

1. 새 연결을 추가하고 아래 정보로 접속한다.

```plain text
Host     : localhost
Port     : 5432
User     : postgres
Password : 설치할 때 정한 비밀번호
Database : postgres (처음 접속용 기본 DB)
```

1. 연결되면 `.sql` 파일을 열어 쿼리를 작성하고 바로 실행할 수 있다.

### 실습용 데이터베이스 준비

```sql
-- sql/create_db.sql
CREATE DATABASE world;

CREATE DATABASE dvdrental;
```

DB를 만든 뒤 각 DB에 연결해서 `world.sql`, `dvdrental.sql`을 실행하면 실습 데이터가 들어간다. `world` 데이터베이스에는 `city`, `country`, `countrylanguage` 세 테이블이 있다.

---

## 8. SQL 기본 규칙과 명령어 분류

```sql
SELECT first_name, last_name
FROM employees
WHERE department_id = 100;   -- 세미콜론으로 문장 구분
```

- 키워드는 대소문자를 구분하지 않지만, 가독성을 위해 **대문자**로 쓰는 것이 관례다.

- 주석은 `--`(한 줄), `/* */`(여러 줄)

- 공백·줄바꿈은 자유롭게 넣을 수 있다.

- 테이블·컬럼명은 `user_profile`처럼 **snake_case** 권장

- PostgreSQL은 따옴표 없는 이름을 **소문자로 처리**하고, 큰따옴표로 감싼 이름은 대소문자를 구분한다.

- 문자열·날짜는 **작은따옴표(')**, 숫자는 따옴표 없이 쓴다.

| 분류 | 의미 | 명령어 |
| --- | --- | --- |
| DDL | 구조 정의·변경 | CREATE, ALTER, DROP, TRUNCATE |
| DML | 데이터 조작 | INSERT, UPDATE, DELETE, SELECT |
| DCL | 접근 권한 제어 | GRANT, REVOKE |
| TCL | 트랜잭션 제어 | BEGIN, COMMIT, ROLLBACK |

`SELECT`만 따로 DQL(데이터 질의어)로 분류하기도 한다.

### 트랜잭션

트랜잭션은 여러 SQL 작업을 하나로 묶은 논리적 작업 단위다. 송금처럼 **출금과 입금이 둘 다 되거나 둘 다 안 되어야** 하는 작업에 쓴다.

```sql
BEGIN;
UPDATE account SET balance = balance - 50000 WHERE owner = 'A';
UPDATE account SET balance = balance + 50000 WHERE owner = 'B';
COMMIT;      -- 문제가 생기면 ROLLBACK;
```

---

## 9. DDL: CREATE · ALTER · DROP · TRUNCATE

### 주요 데이터 타입

| 분류 | 타입 | 설명 |
| --- | --- | --- |
| 숫자 | INT / BIGINT | 정수 / 더 넓은 범위의 정수 |
| 숫자 | DECIMAL(p, s) | 정확한 십진수. 금액에 사용. NUMERIC과 같음 |
| 숫자 | DOUBLE PRECISION | 근삿값 부동소수점. 오차 가능 |
| 문자 | CHAR(n) / VARCHAR(n) / TEXT | 고정 길이 / 최대 n자 가변 / 길이 제한 없음 |
| 날짜 | DATE / TIME / TIMESTAMP | 날짜 / 시간 / 날짜+시간(기본은 시간대 없음) |
| 논리 | BOOLEAN | TRUE / FALSE |

`DECIMAL(10, 2)`는 전체 10자리 중 소수 2자리라서 정수 부분은 최대 8자리다.

### 제약조건

| 제약조건 | 설명 | 예시 |
| --- | --- | --- |
| PRIMARY KEY | 중복·NULL 불가, 행 식별 | `id VARCHAR(7) PRIMARY KEY` |
| NOT NULL | NULL 불가 | `name VARCHAR(10) NOT NULL` |
| UNIQUE | 중복 불가 (NULL은 여러 개 허용) | `email VARCHAR(100) UNIQUE` |
| CHECK | 조건을 만족하는 값만 | `grade INT CHECK (grade BETWEEN 1 AND 4)` |
| FOREIGN KEY | 참조 대상에 있는 값만 | `student_id VARCHAR(7) REFERENCES student(id)` |

- 외래 키는 **참조 무결성**을 지킨다. 없는 학번으로 출결을 넣으면 오류가 난다.

- 별도 옵션이 없으면 출결이 참조 중인 학생을 삭제·변경할 때도 오류가 난다.

- CHECK와 FK만으로는 NULL을 막지 못하므로 필요하면 `NOT NULL`을 함께 쓴다.

- `DEFAULT`는 제약조건이 아니라 값을 **생략**했을 때의 기본값이다. 명시적으로 NULL을 넣으면 기본값으로 바뀌지 않는다.

- `GENERATED ALWAYS AS IDENTITY`는 정수 값을 자동 생성한다. 중복 방지는 `PRIMARY KEY`가 한다.

```sql
CREATE TABLE student (
    id VARCHAR(7) PRIMARY KEY,
    name VARCHAR(10),
    grade INT,
    major VARCHAR(20)
);

CREATE TABLE attendance (
    attendance_id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    student_id VARCHAR(7) REFERENCES student(id),
    date DATE,
    status VARCHAR(10)
);
```

### ALTER · DROP · TRUNCATE

```sql
ALTER TABLE student ADD phone VARCHAR(20);                       -- 컬럼 추가
ALTER TABLE student RENAME COLUMN phone TO phone_number;          -- 이름 변경
ALTER TABLE student ALTER COLUMN name TYPE VARCHAR(100);          -- 타입 변경
ALTER TABLE student DROP COLUMN phone_number;                     -- 컬럼 삭제

DROP TABLE attendance;        -- 테이블 자체를 삭제
TRUNCATE TABLE student;       -- 구조는 남기고 데이터만 전부 삭제
DROP DATABASE demodb;         -- 데이터베이스 삭제
```

---

## 10. DML: INSERT · SELECT · UPDATE · DELETE

```sql
INSERT INTO student (id, name, grade, major)
VALUES ('2024001', '김철수', 1, '컴퓨터공학');

INSERT INTO student VALUES               -- 모든 컬럼을 순서대로 넣으면 컬럼명 생략 가능
    ('2024002', '이영희', 2, '경영학'),
    ('2024003', '박민수', 3, '물리학');

SELECT id, name FROM student WHERE grade = 2;

UPDATE student
SET grade = 2, major = '경제학'
WHERE id = '2024001';

DELETE FROM student
WHERE id = '2024002';
```

> 💡 `UPDATE`와 `DELETE`에서 `WHERE`를 빼면 **모든 행**이 수정·삭제된다.

---

## 11. SELECT 기본: 컬럼 · 별칭 · DISTINCT

```sql
SELECT [DISTINCT] column1 [, column2 ...]
FROM table_name [AS alias]
[WHERE condition]
[GROUP BY column1]
[HAVING group_condition]
[ORDER BY column1 [ASC | DESC]]
[LIMIT N] [OFFSET M];
```

```sql
SELECT * FROM country;                                 -- 전체 컬럼
SELECT name, continent FROM country;                   -- 특정 컬럼
SELECT c.name 국가, c.population 인구 FROM country c;  -- 테이블·컬럼 별칭 (AS 생략 가능)
SELECT DISTINCT continent FROM country;                -- 중복 제거
SELECT DISTINCT continent, region FROM country;        -- 두 값의 "조합" 기준 중복 제거
```

컬럼 별칭은 조회 결과에 보이는 이름만 바꾸고, 원본 컬럼명은 그대로다.

---

## 12. WHERE 조건

| 종류 | 문법 | 비고 |
| --- | --- | --- |
| 비교 | `=`, `!=` (또는 `<>`), `>`, `>=`, `<`, `<=` |  |
| 논리 | `AND`, `OR`, `NOT` | AND가 OR보다 먼저 계산 → 괄호로 묶기 |
| 범위 | `BETWEEN a AND b` | 시작값·끝값 **포함** |
| 포함 | `IN ('KOR', 'JPN')` | `NOT IN` 가능 |
| NULL | `IS NULL`, `IS NOT NULL` | `= NULL`은 쓰지 않는다 |
| 패턴 | `LIKE 'S%'`, `'_'` | `%`는 0개 이상, `_`는 정확히 1개 문자. `ILIKE`는 대소문자 무시 |

```sql
-- 인구 100만 초과이면서 아시아 또는 유럽
SELECT name, continent, population
FROM country
WHERE population > 1000000
  AND (continent = 'Asia' OR continent = 'Europe');
```

---

## 13. WHERE 실습

**인구가 800만 이상인 도시의 name, population**

```sql
SELECT city.name, city.population
FROM city
WHERE city.population >= 8000000;
```

**한국(KOR)에 있는 도시의 name, countrycode**

```sql
SELECT name, countrycode
FROM city
WHERE countrycode = 'KOR';
```

**유럽 대륙에 속한 나라들의 name, region**

```sql
SELECT country.name, country.region
FROM country
WHERE country.continent = 'Europe';
```

**이름이 'San'으로 시작하는 도시의 name**

```sql
SELECT name
FROM city
WHERE name LIKE 'San%';
```

**독립 연도가 1901년 이상인 나라의 name, indepyear**

```sql
SELECT country.name, country.indepyear
FROM country
WHERE country.indepyear >= 1901;
```

**인구가 100만~200만 사이인 한국 도시의 name**

```sql
SELECT city.name
FROM city
WHERE city.population BETWEEN 1000000 AND 2000000
  AND city.countrycode = 'KOR';
```

**인구 500만 이상인 한국·일본·중국 도시의 name, countrycode, population**

```sql
SELECT city.name, city.countrycode, city.population
FROM city
WHERE city.population >= 5000000
  AND city.countrycode IN ('KOR', 'JPN', 'CHN');
```

**이름이 'A'로 시작하고 'a'로 끝나는 도시의 name**

```sql
SELECT city.name
FROM city
WHERE city.name LIKE 'A%a';
```

**동남아시아가 아닌 아시아 나라의 name, region**

```sql
SELECT country.name, country.region
FROM country
WHERE country.continent = 'Asia'
  AND country.region != 'Southeast Asia';
```

**오세아니아에서 기대수명 데이터가 없는 나라의 name, lifeexpectancy, continent**

```sql
SELECT name, lifeexpectancy, continent
FROM country
WHERE continent = 'Oceania'
  AND lifeexpectancy IS NULL;
```

---

## 14. ORDER BY와 NULL 정렬

```sql
ORDER BY 컬럼명 [ASC | DESC]   -- ASC가 기본값
```

```sql
-- 국가 코드순 → 같은 국가 안에서는 인구 많은 순
SELECT name, countrycode, population
FROM city
ORDER BY countrycode, population DESC;
```

PostgreSQL은 NULL을 **가장 큰 값처럼** 취급한다.

```plain text
ASC  → NULL이 마지막
DESC → NULL이 처음
NULLS FIRST / NULLS LAST 로 위치 직접 지정 가능
```

**실습: 대륙별 정렬 후 같은 대륙에서는 GNP 높은 순**

```sql
SELECT name, continent, gnp
FROM country
ORDER BY continent, gnp DESC;
```

**실습: 기대수명 높은 순, NULL은 마지막**

```sql
SELECT name, lifeexpectancy
FROM country
ORDER BY lifeexpectancy DESC NULLS LAST;
```

---

## 15. LIMIT · OFFSET 페이지네이션

```plain text
LIMIT  → 한 페이지에 보여줄 개수
OFFSET → 앞에서 건너뛸 행 수 = (페이지 번호 - 1) * 페이지당 개수
```

```sql
-- 인구 상위 11~15위 (3페이지, 페이지당 5개)
SELECT name, population
FROM city
ORDER BY population DESC
LIMIT 5 OFFSET 10;
```

LIMIT은 정렬된 결과의 일부를 자르는 것이므로 `ORDER BY`와 같이 써야 의미가 있다.

**실습: 인구가 가장 적은 도시 5개**

```sql
SELECT *
FROM city
ORDER BY population
LIMIT 5;
```

**실습: 면적이 넓은 순 11~20위 국가**

```sql
SELECT *
FROM country
ORDER BY surfacearea DESC
LIMIT 10 OFFSET 10;
```

**실습: 기대수명 높은 순 1~5위 국가**

```sql
SELECT name, lifeexpectancy
FROM country
ORDER BY lifeexpectancy DESC NULLS LAST
LIMIT 5;
```

---

## 16. GROUP BY와 집계 함수

GROUP BY는 지정한 컬럼 값이 같은 행끼리 묶고, 그룹마다 집계 함수를 계산한다. SELECT에는 **GROUP BY에 쓴 컬럼과 집계 함수(또는 그 계산식)**만 올 수 있다.

| 함수 | 의미 |
| --- | --- |
| `COUNT(*)` | 전체 행 개수 |
| `COUNT(컬럼)` | 해당 컬럼이 NULL이 아닌 행 개수 |
| `SUM` / `AVG` | 합계 / 평균 (NULL 제외) |
| `MIN` / `MAX` | 최솟값 / 최댓값 (NULL 제외) |

```sql
-- 대륙별 국가 수
SELECT continent, COUNT(*) AS country_count
FROM country
GROUP BY continent;

-- 여러 컬럼으로 그룹화하면 "값의 조합"별로 묶인다
SELECT continent, governmentform, COUNT(*)
FROM country
GROUP BY continent, governmentform
ORDER BY continent, governmentform;
```

**실습: 대륙별 총 인구수**

```sql
SELECT continent, SUM(population) AS total_pop
FROM country
GROUP BY continent;
```

**실습: 대륙별 평균 GNP와 평균 인구**

```sql
SELECT continent, AVG(gnp) AS avg_gnp, AVG(population) AS avg_pop
FROM country
GROUP BY continent;
```

**실습: 인구 50만~100만 도시의 countrycode, district별 도시 수**

```sql
SELECT countrycode, district, COUNT(*) AS city_count
FROM city
WHERE population BETWEEN 500000 AND 1000000
GROUP BY countrycode, district;
```

**실습: 대륙별 국가 수가 많은 순**

```sql
SELECT continent, COUNT(*) AS country_count
FROM country
GROUP BY continent
ORDER BY country_count DESC;
```

**실습: Region별 총 GNP가 가장 높은 Region**

```sql
SELECT region, SUM(gnp) AS total_gnp
FROM country
GROUP BY region
ORDER BY total_gnp DESC
LIMIT 1;          -- North America
```

---

## 17. HAVING과 SELECT 처리 순서

```plain text
WHERE  → 그룹화 "전" 개별 행을 거른다
HAVING → 그룹화 "후" 집계 결과를 거른다
```

SELECT문의 논리적 처리 순서는 다음과 같다.

```plain text
FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT
```

```sql
-- 인구 1000만 이상 국가가 10개 넘는 대륙
SELECT continent, COUNT(*) AS big_countries
FROM country
WHERE population >= 10000000     -- 행 필터
GROUP BY continent
HAVING COUNT(*) > 10;             -- 그룹 필터

-- HAVING에는 SELECT에 없는 집계 함수도 쓸 수 있다
SELECT continent, COUNT(*) AS country_count
FROM country
GROUP BY continent
HAVING AVG(population) > 10000000;
```

**실습: 도시가 10개 이상인 국가의 countrycode, 도시 수**

```sql
SELECT countrycode, COUNT(*) AS city_count
FROM city
GROUP BY countrycode
HAVING COUNT(*) >= 10;
```

**실습: 평균 인구 100만 이상 + 도시 3개 이상인 countrycode, district 그룹**

```sql
SELECT countrycode, district, COUNT(*) AS city_count, SUM(population) AS total_pop
FROM city
GROUP BY countrycode, district
HAVING AVG(population) >= 1000000
   AND COUNT(*) >= 3;
```

**실습: 아시아에서 Region별 평균 GNP가 1000 이상인 Region**

```sql
SELECT region, AVG(gnp) AS avg_gnp
FROM country
WHERE continent = 'Asia'
GROUP BY region
HAVING AVG(gnp) >= 1000;
```

**실습: 독립년도 1900년 이후 국가 중 대륙별 평균 기대수명 70 이상**

```sql
SELECT continent, AVG(lifeexpectancy) AS avg_life
FROM country
WHERE indepyear > 1900
GROUP BY continent
HAVING AVG(lifeexpectancy) >= 70;
```

**실습: 도시 평균 인구 100만 이상, 최소 인구 50만 이상인 국가**

```sql
SELECT countrycode, COUNT(*) AS city_count, SUM(population) AS total_pop
FROM city
GROUP BY countrycode
HAVING AVG(population) >= 1000000
   AND MIN(population) >= 500000;
```

**실습: 인구 50만 이상 도시만 대상으로, 평균 인구 100만 이상인 국가**

```sql
SELECT countrycode, COUNT(*) AS city_count, SUM(population) AS total_pop
FROM city
WHERE population >= 500000
GROUP BY countrycode
HAVING AVG(population) >= 1000000;
```

마지막 두 문제는 비슷해 보이지만 다르다. 앞 문제는 **모든 도시**로 그룹을 만든 뒤 최소 인구를 검사하고, 뒤 문제는 **50만 미만 도시를 먼저 빼고** 남은 도시로 평균을 낸다.

---

## 18. 헷갈린 점

처음에는 `ORDER BY lifeexpectancy DESC LIMIT 5`만 쓰면 기대수명 상위 5개국이 나올 줄 알았다. 그런데 실제로 실행하니 Antarctica, Bouvet Island처럼 **기대수명이 NULL인 나라가 맨 위**에 나왔다. PostgreSQL은 NULL을 가장 큰 값처럼 다뤄서 DESC 정렬 시 NULL이 먼저 오기 때문이다. 순위를 뽑을 때는 `NULLS LAST`를 붙이거나 `WHERE lifeexpectancy IS NOT NULL`로 먼저 걸러야 한다.

```sql
-- NULL 국가가 상위권을 차지함
SELECT name FROM country ORDER BY lifeexpectancy DESC LIMIT 5;

-- 의도한 결과
SELECT name, lifeexpectancy FROM country
ORDER BY lifeexpectancy DESC NULLS LAST
LIMIT 5;
```

NULL 비교도 헷갈렸다. `WHERE lifeexpectancy = NULL`은 오류가 나지 않지만 **결과가 0건**이다. NULL은 "알 수 없는 값"이라 `=` 비교 결과도 참이 아닌 NULL이 되기 때문이다. 그래서 반드시 `IS NULL` / `IS NOT NULL`을 써야 한다.

HAVING에서 SELECT 별칭을 쓰려다 막혔다. `HAVING c > 20`처럼 쓰면 `column "c" does not exist` 오류가 난다. 처리 순서상 HAVING이 SELECT보다 먼저 실행되어 별칭이 아직 없기 때문이다. 반대로 ORDER BY는 SELECT 다음이라 별칭을 쓸 수 있다.

```sql
HAVING COUNT(*) > 20         -- O
HAVING country_count > 20    -- X (별칭은 아직 없음)
ORDER BY country_count DESC  -- O (SELECT 이후라 사용 가능)
```

WHERE와 HAVING의 차이도 처음엔 둘 다 "조건"이라 같아 보였다. 정리하면 **집계 전에 거를 수 있는 조건은 WHERE, 집계 결과로 거르는 조건은 HAVING**이다. WHERE에서 미리 줄이면 그룹화할 데이터도 줄어든다.

문제에서 요구한 컬럼을 빠뜨린 실수도 있었다. "500만 이상인 한국·일본·중국 도시의 name, countrycode, population"을 풀면서 조건만 신경 쓰다가 `SELECT city.name`만 적었다. 조건(WHERE)과 출력 컬럼(SELECT)은 따로 확인해야 한다.

DROP과 TRUNCATE, DELETE도 헷갈렸다.

```plain text
DELETE FROM t WHERE ...  → 조건에 맞는 행만 삭제 (DML)
TRUNCATE TABLE t         → 구조는 남기고 모든 데이터 삭제 (DDL)
DROP TABLE t             → 테이블 구조까지 통째로 삭제 (DDL)
```

마지막으로 ERD의 까마귀 발 기호를 읽는 방향이 헷갈렸다. "고객 한 명이 주문을 몇 건 할 수 있나?"를 알려면 고객 쪽이 아니라 **주문 쪽 끝의 기호(o{)**를 봐야 한다. 기호는 그 기호가 붙은 테이블의 개수를 뜻한다.

---

## 추가!

오늘 배운 것 정리

- 데이터베이스는 데이터의 집합, DBMS는 그것을 관리하는 소프트웨어다.

- 관계형 DB는 테이블(행·열)로 저장하고, PK로 행을 식별하고 FK로 테이블을 연결한다.

- M:N 관계는 연관 테이블을 두고 두 개의 1:N으로 분해한다.

- 같은 대상이라도 서비스 관점에 따라 관계가 달라지므로 기획이 구체적이어야 한다.

- 스키마는 구조의 정의, 데이터는 실제 값이다. ERD는 그 구조를 그림으로 보여준다.

- NoSQL은 키-값, 문서, 그래프, 와이드 컬럼 등 용도에 맞는 모델을 고른다.

- SQL은 DDL(구조), DML(데이터), DCL(권한), TCL(트랜잭션)으로 나뉜다.

- 트랜잭션은 여러 작업을 전부 반영하거나 전부 취소하는 단위다.

- `UPDATE`·`DELETE`는 WHERE 없이 실행하면 전체 행에 적용된다.

- NULL은 `IS NULL`로 비교하고, 정렬할 때는 `NULLS FIRST / LAST`로 위치를 정한다.

- 페이지네이션은 `LIMIT 개수 OFFSET (페이지-1)*개수`로 구현한다.

- WHERE는 행을, HAVING은 그룹을 거른다. HAVING에서는 SELECT 별칭을 쓸 수 없다.

쿼리를 작성할 때는 다음 순서로 생각하면 된다.

```plain text
어떤 테이블에서? (FROM)
→ 어떤 행만? (WHERE)
→ 무엇으로 묶어서? (GROUP BY)
→ 어떤 그룹만? (HAVING)
→ 어떤 컬럼을 보여줄까? (SELECT)
→ 어떤 순서로, 몇 개만? (ORDER BY → LIMIT / OFFSET)
```
