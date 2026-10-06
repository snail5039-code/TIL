# 26.10.06 52일차(SQL JOIN · dvdrental 실습)

## [TIL] PostgreSQL JOIN으로 여러 테이블 연결해서 조회하기

어제까지는 테이블 하나에서 `SELECT`, `WHERE`, `GROUP BY`를 썼다면, 오늘은 **JOIN**으로 두 개 이상의 테이블을 연결해서 조회하는 방법을 배웠다. `world`, `dvdrental` 데이터베이스로 1:N, M:N, 1:1 관계를 직접 이어 붙여 봤다.

이론과 실습은 다음 흐름으로 이어졌다.

```plain text
JOIN 기본 문법
→ INNER JOIN (일치하는 행만)
→ LEFT JOIN (왼쪽 테이블은 전부 유지)
→ FROM에 어떤 테이블을 둘까?
→ ON과 WHERE의 차이
→ RIGHT / FULL / Self JOIN
→ world · dvdrental 예제
→ JOIN + GROUP BY + HAVING
→ dvdrental 실습 16문제
```

---

## 1. JOIN이란

JOIN은 두 개 이상의 테이블을 **조인 조건**으로 연결해서 데이터를 조회하는 SQL 구문이다. 보통 한쪽 테이블의 FK와 다른 쪽 테이블의 PK를 같다고 놓고 연결한다.

```sql
SELECT 컬럼명
FROM 테이블1
JOIN유형 테이블2 ON 조인조건;
```

| 종류 | 결과 | 사용 빈도 |
| --- | --- | --- |
| INNER JOIN | 양쪽에서 조건이 일치하는 행만 | 가장 많이 씀 |
| LEFT JOIN | 왼쪽 테이블 전부 + 오른쪽 일치 행 (없으면 NULL) | 많이 씀 |
| RIGHT JOIN | 오른쪽 테이블 전부 + 왼쪽 일치 행 | LEFT로 순서만 바꾸면 돼서 잘 안 씀 |
| FULL JOIN | 일치하는 행 + 양쪽의 일치하지 않는 행 전부 | 가끔 |
| Self JOIN | 같은 테이블을 자기 자신과 조인 | JOIN 유형이 아니라 방식 |

---

## 2. INNER JOIN

두 테이블에서 조인 조건이 **일치하는 행만** 반환한다. `INNER`를 빼고 `JOIN`만 써도 INNER JOIN으로 동작한다.

### 1:N — 도시와 국가 (`world`)

```sql
SELECT city.name AS cityname,
       country.name AS countryname,
       country.continent
FROM city
JOIN country ON city.countrycode = country.code;   -- INNER 생략 가능
```

### M:N — 배우와 출연 영화 (`dvdrental`)

배우와 영화는 M:N이라 중계 테이블 `film_actor`를 거쳐서 두 번 JOIN한다.

```sql
SELECT actor.first_name,
       actor.last_name,
       film.title
FROM actor
INNER JOIN film_actor ON actor.actor_id = film_actor.actor_id
INNER JOIN film ON film_actor.film_id = film.film_id;
```

```plain text
actor 1 ── N film_actor N ── 1 film
```

---

## 3. LEFT JOIN

왼쪽(`FROM`) 테이블의 **모든 행**을 유지하고, 오른쪽 테이블은 일치하는 행만 붙인다. 일치하는 행이 없으면 오른쪽 컬럼은 **NULL**이 된다.

### 1:1 — 모든 국가와 수도 (`world`)

```sql
SELECT country.name AS country_name,
       city.name AS capital_city
FROM country
LEFT JOIN city ON country.capital = city.id;

-- 수도 데이터가 없는 국가만
SELECT country.name AS country_name,
       city.name AS capital_city
FROM country
LEFT JOIN city ON country.capital = city.id
WHERE city.id IS NULL;
```

같은 쿼리를 INNER JOIN으로 쓰면 수도 데이터가 없는 국가(예: `Antarctica`)는 **아예 조회되지 않는다**.

```sql
SELECT country.name AS country_name,
       city.name AS capital_city
FROM country
INNER JOIN city ON country.capital = city.id
WHERE country.name = 'Antarctica';   -- 0건
```

### 1:N — 모든 영화의 재고 (`dvdrental`)

영화 하나에 재고가 여러 개면, **재고마다 영화 정보가 반복**해서 나온다.

```sql
SELECT f.film_id, f.title, i.inventory_id
FROM film f
LEFT JOIN inventory i ON f.film_id = i.film_id;

-- 재고가 없는 영화만
SELECT f.film_id, f.title, i.inventory_id
FROM film f
LEFT JOIN inventory i ON f.film_id = i.film_id
WHERE i.inventory_id IS NULL;
```

### M:N — 모든 영화와 대여 고객 (`dvdrental`)

```sql
SELECT f.title,
       c.customer_id,
       c.first_name,
       c.last_name,
       r.rental_date
FROM film f
LEFT JOIN inventory i ON f.film_id = i.film_id
LEFT JOIN rental r ON i.inventory_id = r.inventory_id
LEFT JOIN customer c ON r.customer_id = c.customer_id;
```

- 같은 고객이 같은 영화를 여러 번 대여하면 대여 기록마다 조회된다.

- 재고나 대여 기록이 없으면 고객 정보와 대여일은 NULL로 표시된다.

- 체인 중간에 LEFT JOIN을 썼으면 **뒤쪽도 LEFT JOIN**으로 이어야 앞에서 살린 행이 유지된다.

---

## 4. FROM에 어떤 테이블을 둘까

```plain text
INNER JOIN → 조회하려는 대상의 "기준" 테이블을 FROM에 두면 읽기 쉽다 (결과는 순서와 무관)
LEFT JOIN  → 연결 데이터가 없어도 "모든 행을 유지할" 테이블을 FROM에 둔다 (순서가 결과를 바꾼다)
```

---

## 5. ON과 WHERE의 차이

- `ON` : 두 테이블의 행을 **연결하는** 조건

- `WHERE` : 조인 **결과에서 남길 행**을 고르는 조건

- INNER JOIN에서는 어디에 써도 결과가 같지만, **LEFT JOIN에서 오른쪽 테이블 조건**은 위치에 따라 결과가 달라진다.

**모든 영화를 유지하고, 1번 매장 재고만 연결**

```sql
SELECT f.film_id, f.title, i.inventory_id, i.store_id
FROM film f
LEFT JOIN inventory i ON f.film_id = i.film_id
                     AND i.store_id = 1;   -- 2,511행
```

1번 매장에 재고가 없는 영화도 나오고, 그 영화의 재고 정보는 NULL이다.

**1번 매장에 재고가 있는 영화만**

```sql
SELECT f.film_id, f.title, i.inventory_id, i.store_id
FROM film f
LEFT JOIN inventory i ON f.film_id = i.film_id
WHERE i.store_id = 1;                      -- 2,270행
```

조인 후에 `store_id = 1`만 남기므로 NULL 행도 조건을 만족하지 못해 빠진다. 사실상 INNER JOIN과 같은 결과가 된다.

```plain text
LEFT JOIN 전체        → 4,623행
ON에 store_id = 1    → 2,511행 (모든 영화 유지)
WHERE에 store_id = 1 → 2,270행 (1번 매장 재고 있는 영화만)
```

---

## 6. RIGHT JOIN · FULL JOIN · Self JOIN

- **RIGHT JOIN** : 오른쪽 테이블의 모든 행 + 왼쪽 일치 행. LEFT JOIN에서 테이블 순서만 바꾸면 같아서 잘 쓰지 않는다.

- **FULL JOIN** : 일치하는 행은 연결하고, 양쪽의 일치하지 않는 행도 모두 포함한다. 짝이 없는 쪽 컬럼은 NULL.

- **Self JOIN** : JOIN 유형이 아니라 같은 테이블을 자기 자신과 조인하는 **방식**이다. 별칭을 다르게 주고 INNER / LEFT 등 무엇이든 쓸 수 있다.

---

## 7. world 예제

**특정 대륙(Asia)에서 인구 500만 이상인 도시**

```sql
SELECT co.continent,
       co.name AS country,
       ci.name AS city,
       ci.population
FROM country co
JOIN city ci ON co.code = ci.countrycode
WHERE co.continent = 'Asia'
  AND ci.population >= 5000000
ORDER BY ci.population DESC;
```

**국가와 수도, 공식 언어**

```sql
SELECT co.name AS country_name,
       ci.name AS capital_city,
       cl."Language"
FROM country co
JOIN city ci ON co.capital = ci.id
JOIN countrylanguage cl ON co.code = cl.countrycode
WHERE cl.isofficial = 'T';
```

`Language` 컬럼은 대문자로 만들어져 있어서 큰따옴표로 감싸야 한다. (따옴표 없으면 소문자 `language`로 찾는다.)

---

## 8. dvdrental 예제: 고객 정보

JOIN을 하나씩 늘려 가며 필요한 정보를 붙였다.

```plain text
customer → address → city → country
```

```sql
SELECT c.first_name,
       c.last_name,
       c.email,
       a.address,
       ci.city,
       co.country
FROM customer c
JOIN address a ON c.address_id = a.address_id
JOIN city ci ON a.city_id = ci.city_id
JOIN country co ON ci.country_id = co.country_id;
```

**도시별 고객 수**

```sql
SELECT ci.city, COUNT(*) AS customer_count
FROM customer c
JOIN address a ON c.address_id = a.address_id
JOIN city ci ON a.city_id = ci.city_id
GROUP BY ci.city_id, ci.city
ORDER BY COUNT(*) DESC;
```

도시 이름이 같은 다른 도시가 있을 수 있어서 `city_id`까지 같이 GROUP BY 했다.

---

## 9. dvdrental 예제: 배우 · 영화 · 카테고리

**배우별 출연 영화 수**

```sql
SELECT a.first_name,
       a.last_name,
       COUNT(fa.film_id) AS num_of_films
FROM actor a
JOIN film_actor fa ON a.actor_id = fa.actor_id
GROUP BY a.actor_id, a.first_name, a.last_name
ORDER BY num_of_films;
```

**카테고리별 영화 수**

```sql
SELECT c.name AS category,
       COUNT(f.film_id) AS film_count
FROM category c
JOIN film_category fc ON c.category_id = fc.category_id
JOIN film f ON fc.film_id = f.film_id
GROUP BY c.name
ORDER BY film_count DESC;
```

**배우가 출연한 영화를 카테고리까지 포함해서**

```sql
SELECT a.first_name,
       a.last_name,
       f.title,
       c.name AS category
FROM actor a
JOIN film_actor fa ON a.actor_id = fa.actor_id
JOIN film f ON fa.film_id = f.film_id
JOIN film_category fc ON f.film_id = fc.film_id
JOIN category c ON fc.category_id = c.category_id;
```

**출연 영화가 30편 이상인 배우 (JOIN + HAVING)**

```sql
SELECT a.first_name,
       a.last_name,
       COUNT(fa.film_id) AS film_count
FROM actor a
JOIN film_actor fa ON a.actor_id = fa.actor_id
GROUP BY a.actor_id, a.first_name, a.last_name
HAVING COUNT(fa.film_id) >= 30
ORDER BY film_count DESC;
```

---

## 10. JOIN 실습 (dvdrental)

먼저 dvdrental에서 자주 쓰는 연결 경로를 정리해 두고 풀었다.

```plain text
customer ─ rental ─ inventory ─ film ─ film_actor ─ actor
                 └ payment          └ film_category ─ category
customer ─ address ─ city ─ country
staff ─ store ─ address
```

**1. 고객의 이름과 대여일**

```sql
SELECT c.first_name, c.last_name, r.rental_date
FROM customer c
JOIN rental r ON c.customer_id = r.customer_id;
```

**2. 배우의 이름과 출연한 영화의 제목**

```sql
SELECT a.first_name, a.last_name, f.title
FROM actor a
JOIN film_actor fa ON a.actor_id = fa.actor_id
JOIN film f ON fa.film_id = f.film_id;
```

**3. first_name이 'Penelope'인 배우가 출연한 영화의 제목**

```sql
SELECT a.first_name, a.last_name, f.title
FROM actor a
JOIN film_actor fa ON a.actor_id = fa.actor_id
JOIN film f ON fa.film_id = f.film_id
WHERE a.first_name = 'Penelope';
```

**4. 배우들이 출연한 영화의 등급 (중복 없이)**

```sql
SELECT DISTINCT f.rating
FROM actor a
JOIN film_actor fa ON a.actor_id = fa.actor_id
JOIN film f ON fa.film_id = f.film_id;   -- G, PG, PG-13, R, NC-17
```

**5. 배우별 출연 영화 등급 (중복 없이)**

```sql
SELECT DISTINCT a.actor_id, a.first_name, a.last_name, f.rating
FROM actor a
JOIN film_actor fa ON a.actor_id = fa.actor_id
JOIN film f ON fa.film_id = f.film_id
ORDER BY a.actor_id, f.rating;
```

DISTINCT는 SELECT한 컬럼의 **조합** 기준이라 "배우 + 등급" 쌍이 중복 없이 나온다.

**6. 고객의 이름과 대여한 영화의 제목**

```sql
SELECT c.first_name, c.last_name, f.title
FROM customer c
JOIN rental r ON c.customer_id = r.customer_id
JOIN inventory i ON r.inventory_id = i.inventory_id
JOIN film f ON i.film_id = f.film_id;
```

rental에는 film_id가 없고 inventory_id만 있어서 **inventory를 거쳐야** film에 닿는다.

**7. 'Yentl Idaho'를 대여한 고객 정보 (중복 없이)**

```sql
SELECT DISTINCT c.customer_id, c.first_name, c.last_name, c.email
FROM customer c
JOIN rental r ON c.customer_id = r.customer_id
JOIN inventory i ON r.inventory_id = i.inventory_id
JOIN film f ON i.film_id = f.film_id
WHERE f.title = 'Yentl Idaho';
```

**8. 직원의 이름과 근무 매장의 주소, 도시**

```sql
SELECT s.first_name, s.last_name, a.address, ci.city
FROM staff s
JOIN store st ON s.store_id = st.store_id
JOIN address a ON st.address_id = a.address_id
JOIN city ci ON a.city_id = ci.city_id;
```

staff에도 address_id가 있지만, 문제는 **매장 주소**라서 store의 address_id로 연결해야 한다.

**9. 고객별 대여 영화 제목, 지불 금액, 대여일**

```sql
SELECT c.customer_id, c.first_name, c.last_name,
       f.title, p.amount, r.rental_date
FROM customer c
JOIN rental r ON c.customer_id = r.customer_id
JOIN payment p ON r.rental_id = p.rental_id
JOIN inventory i ON r.inventory_id = i.inventory_id
JOIN film f ON i.film_id = f.film_id
ORDER BY c.customer_id, r.rental_date;
```

**10. 'Action' 카테고리 영화에 출연한 배우 (중복 없이)**

```sql
SELECT DISTINCT a.actor_id, a.first_name, a.last_name
FROM actor a
JOIN film_actor fa ON a.actor_id = fa.actor_id
JOIN film_category fc ON fa.film_id = fc.film_id
JOIN category cat ON fc.category_id = cat.category_id
WHERE cat.name = 'Action'
ORDER BY a.actor_id;
```

영화 제목은 필요 없어서 film 테이블을 건너뛰고 `film_actor.film_id = film_category.film_id`로 바로 연결했다.

**11. 출연 배우가 있는 영화별 배우 수**

```sql
SELECT f.film_id, f.title, COUNT(fa.actor_id) AS actor_count
FROM film f
JOIN film_actor fa ON f.film_id = fa.film_id
GROUP BY f.film_id, f.title;
```

"출연 배우가 **있는**" 영화라서 INNER JOIN이 맞다. 결과는 997편으로, 배우 정보가 없는 영화 3편은 빠진다.

**12. 출연 배우 5명 초과 영화, 배우 수 내림차순**

```sql
SELECT f.film_id, f.title, COUNT(fa.actor_id) AS actor_count
FROM film f
JOIN film_actor fa ON f.film_id = fa.film_id
GROUP BY f.film_id, f.title
HAVING COUNT(fa.actor_id) > 5
ORDER BY actor_count DESC;
```

**13. 카테고리별 평균 대여료**

```sql
SELECT cat.name AS category, AVG(f.rental_rate) AS avg_rental_rate
FROM category cat
JOIN film_category fc ON cat.category_id = fc.category_id
JOIN film f ON fc.film_id = f.film_id
GROUP BY cat.category_id, cat.name
ORDER BY avg_rental_rate DESC;   -- 1위 Games (약 3.25)
```

**14. 고객이 있는 국가별 고객 수**

```sql
SELECT co.country, COUNT(c.customer_id) AS customer_count
FROM customer c
JOIN address a ON c.address_id = a.address_id
JOIN city ci ON a.city_id = ci.city_id
JOIN country co ON ci.country_id = co.country_id
GROUP BY co.country_id, co.country
ORDER BY customer_count DESC;
```

**15. 고객 ID 1이 가장 많이 대여한 영화의 카테고리**

```sql
SELECT cat.name AS category, COUNT(*) AS rental_count
FROM rental r
JOIN inventory i ON r.inventory_id = i.inventory_id
JOIN film_category fc ON i.film_id = fc.film_id
JOIN category cat ON fc.category_id = cat.category_id
WHERE r.customer_id = 1
GROUP BY cat.category_id, cat.name
ORDER BY rental_count DESC
LIMIT 1;                          -- Classics, 6회
```

고객 이름은 필요 없어서 customer 테이블 없이 `rental.customer_id`로 바로 걸렀다.

**16. 재고(inventory)에 등록되지 않은 영화**

```sql
SELECT f.film_id, f.title
FROM film f
LEFT JOIN inventory i ON f.film_id = i.film_id
WHERE i.inventory_id IS NULL;     -- 42편
```

"없는 것"을 찾을 때는 **LEFT JOIN + IS NULL** 패턴을 쓴다.

---

## 11. 헷갈린 점

처음엔 INNER JOIN과 LEFT JOIN 결과가 비슷해 보여서 아무거나 써도 되는 줄 알았다. 그런데 Antarctica처럼 짝이 없는 행이 있으면 INNER JOIN은 그 행을 **조용히 빼 버린다**. 오류도 안 나서 더 위험하다. "모든 ~"이라는 말이 문제에 있으면 LEFT JOIN을 먼저 떠올려야 한다.

LEFT JOIN을 썼는데 결과가 INNER JOIN처럼 나온 적도 있었다. 오른쪽 테이블 조건(`i.store_id = 1`)을 WHERE에 넣었기 때문이다. NULL로 채워진 행은 `NULL = 1`이 참이 아니라서 WHERE에서 걸러진다. **오른쪽 테이블 조건으로 연결만 제한하고 왼쪽 행은 살리고 싶으면 ON**에 써야 한다.

```plain text
ON    → 연결할지 말지 결정 (왼쪽 행은 그대로 남음)
WHERE → 조인 결과에서 남길지 말지 결정 (NULL 행도 빠질 수 있음)
```

9번(지불 금액)을 풀면서 결과가 16,044건이 아니라 14,596건이라 이상했다. 확인해 보니 rental 중 1,452건은 payment 기록이 없어서 **INNER JOIN에서 빠진 것**이었다. JOIN을 하나 추가할 때마다 행 수가 줄거나 늘 수 있으니 결과 건수를 같이 확인하는 습관이 필요하다.

```sql
SELECT COUNT(*)
FROM rental r
LEFT JOIN payment p ON r.rental_id = p.rental_id
WHERE p.payment_id IS NULL;   -- 1452
```

6번에서 rental과 film을 바로 연결하려다 막혔다. rental에는 `film_id`가 없고 `inventory_id`만 있다. 대여는 "영화"가 아니라 "실제 DVD 한 장(재고)"을 빌리는 거라서 **inventory를 반드시 거쳐야** 한다. ERD를 보고 연결 경로부터 정하고 쿼리를 쓰는 게 빨랐다.

8번도 비슷했다. staff 테이블에 `address_id`가 있어서 그걸로 연결했는데, 그건 **직원 본인 주소**였다. 문제는 "근무하는 매장의 주소"라서 store를 거쳐 store의 address_id로 연결해야 한다. 같은 이름의 컬럼이라도 어느 테이블의 것인지가 중요하다.

GROUP BY에 이름만 쓰면 안 되는 이유도 알게 됐다. 동명이인 배우나 같은 이름의 도시가 있으면 서로 다른 대상이 한 그룹으로 합쳐진다. 그래서 `GROUP BY a.actor_id, a.first_name, a.last_name`처럼 **PK를 함께** 묶는다.

마지막으로 컬럼명이 겹치는 문제. 여러 테이블에 `name`, `last_update` 같은 같은 이름의 컬럼이 있으면 `column reference is ambiguous` 오류가 난다. 별칭(`c.`, `f.`)을 항상 붙이는 습관을 들이면 이런 오류도 막고 어느 테이블 컬럼인지 읽기도 쉽다.

---

## 추가!

오늘 배운 것 정리

- JOIN은 FK와 PK 같은 조인 조건으로 두 개 이상의 테이블을 연결해서 조회한다.

- INNER JOIN은 일치하는 행만, LEFT JOIN은 왼쪽 행을 전부 유지하고 없는 쪽은 NULL로 채운다.

- `JOIN`만 쓰면 INNER JOIN이다. RIGHT JOIN은 LEFT JOIN으로 순서만 바꾸면 된다.

- M:N 관계는 중계 테이블(`film_actor`, `film_category`)을 거쳐 두 번 JOIN한다.

- 1:N JOIN에서는 "1"쪽 정보가 "N"쪽 행 수만큼 반복된다.

- LEFT JOIN은 모든 행을 유지할 테이블을 FROM에 둔다.

- LEFT JOIN에서 오른쪽 테이블 조건은 ON이냐 WHERE냐에 따라 결과가 달라진다.

- "없는 것" 찾기는 `LEFT JOIN ... WHERE 오른쪽.PK IS NULL` 패턴.

- JOIN 뒤에도 GROUP BY, HAVING, ORDER BY, LIMIT을 그대로 쓸 수 있다.

- GROUP BY에는 이름만 쓰지 말고 PK를 함께 넣는다.

- 컬럼명이 겹칠 수 있으니 테이블 별칭을 항상 붙인다.

JOIN 쿼리를 작성할 때는 다음 순서로 생각하면 된다.

```plain text
출력해야 할 컬럼은 어느 테이블에 있나?
→ 그 테이블들을 잇는 경로는? (ERD에서 FK 따라가기)
→ 짝이 없는 행도 남겨야 하나? (INNER / LEFT)
→ 조건은 연결 조건(ON)인가, 결과 필터(WHERE)인가?
→ 묶고 거를 게 있나? (GROUP BY → HAVING)
→ 정렬과 개수 (ORDER BY → LIMIT)
```
