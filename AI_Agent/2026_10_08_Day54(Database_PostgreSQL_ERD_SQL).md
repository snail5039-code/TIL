# Day 54 · 숙소 예약 서비스 ERD 설계와 화면별 SQL 작성하기

**학습 날짜**: 2026.10.08  
**주제**: Database · PostgreSQL · ERD 및 SQL 작성 실습

---

## 📋 목차

1. [개요](#1-개요)
2. [화면 예시 분석](#2-화면-예시-분석)
3. [ERD 설계](#3-erd-설계)
4. [테이블 생성 (DDL)](#4-테이블-생성-ddl)
5. [샘플 데이터 (DML)](#5-샘플-데이터-dml)
6. [화면별 조회 SQL](#6-화면별-조회-sql)
7. [조회 결과 검증](#7-조회-결과-검증)
8. [학습 포인트](#8-학습-포인트)

---

## 1. 개요

오늘은 **숙소 예약 서비스**의 여섯 개 화면을 분석하고, 필요한 데이터를 정의한 뒤, 직접 설계한 **`숙소예약_erd.drawio`**를 기준으로 PostgreSQL 테이블과 화면별 조회 SQL을 작성했다.

### 학습 흐름

```
화면 예시 확인
↓
내가 만든 ERD 구조 확인
↓
ERD의 의미를 영어 테이블/컬럼으로 매핑
↓
DDL 작성 (테이블 생성)
↓
샘플 데이터 INSERT (DML)
↓
화면별 SELECT 작성
↓
조회 결과 확인 및 검증
```

---

## 2. 화면 예시 분석

### 2-1. 메인 페이지
**목적**: 최근 등록된 숙소를 보여줌  
**필요 정보**: 숙소명, 주소, 최저 1박 요금, 평균 평점, 등록일

### 2-2. 숙소 목록
**목적**: 지역 조건으로 숙소 조회  
**주의점**: 한 숙소에 여러 객실이 있을 수 있으므로 최저 요금은 `MIN(room.price_per_night)`으로 구함

### 2-3. 숙소 상세 페이지
**필요 정보**:
- 숙소 기본 정보
- 찜 여부 (wishlist 상태)
- 호스트 정보
- 편의시설
- 객실 목록
- 후기 목록

### 2-4. 예약 확인 페이지
**목적**: 예약 번호 기준으로 예약 상세 정보 표시

### 2-5. 마이페이지
**표시 항목**:
- 예약자 정보
- 내 예약
- 찜한 숙소
- 내가 작성한 후기

### 2-6. 호스트 페이지
**표시 항목**:
- 호스트가 등록한 숙소와 객실
- 예약 현황
- 예약자 정보
- 숙소별 예약 집계
- 후기 현황

---

## 3. ERD 설계

### 3-1. 테이블 매핑표

| ERD 테이블 | SQL 테이블 | 역할 |
|-----------|-----------|------|
| 호스트 | host | 숙소를 등록하는 사용자 |
| 예약자 | guest | 예약을 생성하고 후기를 작성하는 사용자 |
| 편의시설 | amenity | 무선 인터넷, 무료 주차 같은 편의시설 |
| 숙소 | accommodation | 예약 가능한 숙소 |
| 객실 | room | 숙소 안의 객실과 요금 |
| 후기 | review | 예약자가 작성한 평점과 후기 |
| 예약 | booking | 예약 내역 |
| 찜 | wishlist | 찜 대상 숙소 |
| 예약자_찜 | guest_wishlist | 예약자와 찜의 연결 |
| 숙소_편의시설 | accommodation_amenity | 숙소와 편의시설의 연결 |

### 3-2. 관계도

- **1:N 관계**
  - `host` → `accommodation` (호스트 1명이 여러 숙소 등록)
  - `accommodation` → `room` (숙소 1개가 여러 객실 보유)
  - `accommodation` → `booking` (숙소 1개가 여러 예약 수신)
  - `guest` → `booking` (게스트 1명이 여러 예약)
  - `guest` → `review` (게스트 1명이 여러 후기 작성)
  - `room` → `review` (객실 1개가 여러 후기 수신)

- **M:N 관계** (연결 테이블 사용)
  - `accommodation` ↔ `amenity` (`accommodation_amenity`를 통해 연결)
  - `guest` ↔ `wishlist` (`guest_wishlist`를 통해 연결)

---

## 4. 테이블 생성 (DDL)

### 4-1. 기본 테이블

```sql
DROP TABLE IF EXISTS guest_wishlist CASCADE;
DROP TABLE IF EXISTS accommodation_amenity CASCADE;
DROP TABLE IF EXISTS wishlist CASCADE;
DROP TABLE IF EXISTS booking CASCADE;
DROP TABLE IF EXISTS review CASCADE;
DROP TABLE IF EXISTS room CASCADE;
DROP TABLE IF EXISTS accommodation CASCADE;
DROP TABLE IF EXISTS amenity CASCADE;
DROP TABLE IF EXISTS guest CASCADE;
DROP TABLE IF EXISTS host CASCADE;

-- 호스트 테이블
CREATE TABLE host (
    host_id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name VARCHAR(50) NOT NULL,
    profile_description TEXT
);

-- 게스트 테이블
CREATE TABLE guest (
    guest_id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name VARCHAR(50) NOT NULL,
    email VARCHAR(100) NOT NULL UNIQUE,
    phone VARCHAR(20)
);

-- 편의시설 테이블
CREATE TABLE amenity (
    amenity_id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    amenity_type VARCHAR(50) NOT NULL UNIQUE
);
```

### 4-2. 숙소와 객실

```sql
-- 숙소 테이블
CREATE TABLE accommodation (
    accommodation_id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    address VARCHAR(300) NOT NULL,
    host_id INT NOT NULL REFERENCES host(host_id),
    amenity_id INT REFERENCES amenity(amenity_id),
    review_id INT,
    registered_date DATE NOT NULL DEFAULT CURRENT_DATE,
    description TEXT
);

-- 객실 테이블
CREATE TABLE room (
    room_id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    max_guests INT NOT NULL CHECK (max_guests > 0),
    price_per_night INT NOT NULL CHECK (price_per_night >= 0),
    description TEXT,
    name VARCHAR(100) NOT NULL,
    accommodation_id INT NOT NULL REFERENCES accommodation(accommodation_id),
    UNIQUE (accommodation_id, name)
);
```

### 4-3. 후기와 예약

**핵심**: ERD의 `후기` 테이블에는 `숙소_id`가 없으므로, `review → room → accommodation` 경로로 숙소별 후기를 조회한다.

```sql
-- 후기 테이블
CREATE TABLE review (
    review_id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    guest_id INT NOT NULL REFERENCES guest(guest_id),
    rating NUMERIC(2,1) NOT NULL CHECK (rating BETWEEN 0 AND 5),
    content TEXT NOT NULL,
    room_name VARCHAR(100),
    written_date DATE NOT NULL DEFAULT CURRENT_DATE,
    room_id INT NOT NULL REFERENCES room(room_id)
);

-- accommodation 테이블에 review_id 제약 추가
ALTER TABLE accommodation
ADD CONSTRAINT fk_accommodation_review
FOREIGN KEY (review_id) REFERENCES review(review_id);

-- 예약 테이블
CREATE TABLE booking (
    booking_id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    accommodation_id INT NOT NULL REFERENCES accommodation(accommodation_id),
    address VARCHAR(300),
    room_name VARCHAR(100),
    check_in DATE NOT NULL,
    check_out DATE NOT NULL,
    guest_count INT NOT NULL CHECK (guest_count > 0),
    guest_id INT NOT NULL REFERENCES guest(guest_id),
    room_id INT NOT NULL REFERENCES room(room_id),
    status VARCHAR(20) NOT NULL CHECK (status IN ('CONFIRMED', 'COMPLETED', 'CANCELED')),
    booked_date DATE NOT NULL DEFAULT CURRENT_DATE,
    booking_number VARCHAR(50) NOT NULL UNIQUE,
    nights INT NOT NULL CHECK (nights > 0),
    CHECK (check_out > check_in)
);
```

### 4-4. 찜과 편의시설 연결 테이블

```sql
-- 찜 테이블
CREATE TABLE wishlist (
    wishlist_id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    accommodation_id INT NOT NULL REFERENCES accommodation(accommodation_id)
);

-- 게스트_찜 연결 테이블
CREATE TABLE guest_wishlist (
    guest_wishlist_id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    wishlist_id INT NOT NULL REFERENCES wishlist(wishlist_id),
    guest_id INT NOT NULL REFERENCES guest(guest_id),
    UNIQUE (wishlist_id, guest_id)
);

-- 숙소_편의시설 연결 테이블
CREATE TABLE accommodation_amenity (
    accommodation_amenity_id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    amenity_id INT NOT NULL REFERENCES amenity(amenity_id),
    accommodation_id INT NOT NULL REFERENCES accommodation(accommodation_id),
    UNIQUE (amenity_id, accommodation_id)
);
```

---

## 5. 샘플 데이터 (DML)

```sql
-- 호스트 데이터
INSERT INTO host (name, profile_description) VALUES
('Lee Haneul', 'I run a small accommodation in Jeju.'),
('Kim Minsu', 'I manage accommodations near Gwangalli Beach in Busan.');

-- 게스트 데이터
INSERT INTO guest (name, email, phone) VALUES
('Kim Minsu', 'minsu@example.com', '010-1111-1111'),
('Park Jiyoung', 'jiyoung@example.com', '010-2222-2222'),
('Jung Yujin', 'yujin@example.com', '010-3333-3333'),
('Choi Seoyeon', 'seoyeon@example.com', '010-0000-0001'),
('Han Doyun', 'doyun@example.com', '010-0000-0002'),
('Oh Subin', 'subin@example.com', '010-0000-0003');

-- 편의시설 데이터
INSERT INTO amenity (amenity_type) VALUES
('Wi-Fi'), ('Free parking'), ('Air conditioner'), ('Kitchen');

-- 숙소 데이터
INSERT INTO accommodation (name, address, host_id, amenity_id, registered_date, description) VALUES
('Jeju Wind House', 'Jeju-si, Aewol-eup, Baram-gil 12', 1, 1, '2025-06-06', 'A quiet place to rest while looking at the sea.'),
('Busan Wave House', 'Busan, Suyeong-gu, Gwangan-dong, Beach Road 24', 2, 1, '2025-06-05', 'A small accommodation near Gwangalli Beach.'),
('Jeju Stone Wall Stay', 'Seogwipo-si, Andeok-myeon, Doldam-gil 8', 1, 2, '2025-05-20', 'A quiet Jeju accommodation with a beautiful stone wall.');

-- 객실 데이터
INSERT INTO room (max_guests, price_per_night, description, name, accommodation_id) VALUES
(2, 85000, 'One double bed · garden view', 'Wind 101', 1),
(4, 130000, 'Two double beds · ocean view', 'Sea 201', 1),
(2, 95000, 'One double bed', 'Wave 101', 2),
(4, 150000, 'Two double beds', 'Wave 201', 2),
(2, 110000, 'Stone wall garden view', 'Stone 1', 3);

-- 숙소_편의시설 연결 데이터
INSERT INTO accommodation_amenity (amenity_id, accommodation_id) VALUES
(1, 1), (2, 1), (3, 1), (4, 1),
(1, 2), (3, 2),
(1, 3), (2, 3);

-- 후기 데이터
INSERT INTO review (guest_id, rating, content, room_name, written_date, room_id) VALUES
(2, 5.0, 'The ocean view was great and the room was clean.', 'Sea 201', '2025-06-07', 2),
(3, 4.0, 'It was quiet and parking was convenient.', 'Wind 101', '2025-06-07', 1),
(1, 4.0, 'The yard was pretty and quiet. I want to visit again.', 'Stone 1', '2025-05-28', 5);

-- accommodation 테이블의 review_id 업데이트
UPDATE accommodation SET review_id = 1 WHERE accommodation_id = 1;
UPDATE accommodation SET review_id = 3 WHERE accommodation_id = 3;

-- 예약 데이터
INSERT INTO booking (accommodation_id, address, room_name, check_in, check_out, guest_count, guest_id, room_id, status, booked_date, booking_number, nights) VALUES
(1, 'Jeju-si, Aewol-eup, Baram-gil 12', 'Wind 101', '2025-06-20', '2025-06-22', 2, 1, 1, 'CONFIRMED', '2025-06-07', 'BK-20250607-001', 2),
(3, 'Seogwipo-si, Andeok-myeon, Doldam-gil 8', 'Stone 1', '2025-05-25', '2025-05-27', 2, 1, 5, 'COMPLETED', '2025-05-15', 'BK-20250515-002', 2),
(2, 'Busan, Suyeong-gu, Gwangan-dong, Beach Road 24', 'Wave 101', '2025-06-15', '2025-06-17', 2, 4, 3, 'CONFIRMED', '2025-06-06', 'BK-20250606-003', 2),
(2, 'Busan, Suyeong-gu, Gwangan-dong, Beach Road 24', 'Wave 201', '2025-06-21', '2025-06-24', 4, 5, 4, 'CONFIRMED', '2025-06-07', 'BK-20250607-004', 3),
(2, 'Busan, Suyeong-gu, Gwangan-dong, Beach Road 24', 'Wave 101', '2025-06-25', '2025-06-26', 1, 6, 3, 'CANCELED', '2025-06-06', 'BK-20250606-005', 1);

-- 찜 데이터
INSERT INTO wishlist (accommodation_id) VALUES (1);
INSERT INTO guest_wishlist (wishlist_id, guest_id) VALUES (1, 1);
```

---

## 6. 화면별 조회 SQL

### 6-1. 메인 페이지 - 최근 등록 숙소

```sql
SELECT
    a.accommodation_id,
    a.name AS accommodation_name,
    a.address,
    MIN(r.price_per_night) AS minimum_price_per_night,
    COALESCE(ROUND(AVG(rv.rating), 1), 0) AS average_rating,
    COUNT(rv.review_id) AS review_count,
    a.registered_date
FROM accommodation a
JOIN room r ON r.accommodation_id = a.accommodation_id
LEFT JOIN review rv ON rv.room_id IN (
    SELECT room_id FROM room WHERE accommodation_id = a.accommodation_id
)
GROUP BY a.accommodation_id, a.name, a.address, a.registered_date
ORDER BY a.registered_date DESC, a.accommodation_id DESC;
```

### 6-2. 숙소 목록 - 지역 필터 조회

```sql
SELECT
    a.accommodation_id,
    a.name AS accommodation_name,
    a.address,
    MIN(r.price_per_night) AS minimum_price_per_night,
    COALESCE(ROUND(AVG(rv.rating), 1), 0) AS average_rating,
    COUNT(rv.review_id) AS review_count
FROM accommodation a
JOIN room r ON r.accommodation_id = a.accommodation_id
LEFT JOIN review rv ON rv.room_id IN (
    SELECT room_id FROM room WHERE accommodation_id = a.accommodation_id
)
WHERE a.address LIKE 'Jeju%'
GROUP BY a.accommodation_id, a.name, a.address
ORDER BY minimum_price_per_night, average_rating DESC;
```

### 6-3. 숙소 상세 페이지 - 기본 정보

```sql
SELECT
    a.accommodation_id,
    a.name AS accommodation_name,
    a.address,
    a.description,
    CASE WHEN EXISTS (
        SELECT 1
        FROM guest_wishlist gw
        JOIN wishlist w ON w.wishlist_id = gw.wishlist_id
        WHERE gw.guest_id = 1
          AND w.accommodation_id = a.accommodation_id
    ) THEN 'WISHED' ELSE 'NOT_WISHED' END AS wishlist_status,
    h.name AS host_name,
    h.profile_description AS host_description
FROM accommodation a
JOIN host h ON h.host_id = a.host_id
WHERE a.accommodation_id = 1;
```

### 6-3-1. 편의시설 조회

```sql
SELECT am.amenity_type
FROM accommodation_amenity aa
JOIN amenity am ON am.amenity_id = aa.amenity_id
WHERE aa.accommodation_id = 1
ORDER BY am.amenity_id;
```

### 6-3-2. 객실 목록

```sql
SELECT name AS room_name, description, max_guests, price_per_night
FROM room
WHERE accommodation_id = 1
ORDER BY price_per_night;
```

### 6-3-3. 후기 목록

```sql
SELECT
    g.name AS reviewer_name,
    rv.rating,
    rv.content,
    rv.room_name,
    rv.written_date
FROM review rv
JOIN guest g ON g.guest_id = rv.guest_id
JOIN room r ON r.room_id = rv.room_id
WHERE r.accommodation_id = 1
ORDER BY rv.written_date DESC, rv.review_id DESC;
```

### 6-4. 예약 확인 페이지 - 예약 번호로 조회

**주의**: 예약 금액은 `room.price_per_night * booking.nights`로 계산 (테이블 컬럼 없음)

```sql
SELECT
    b.booking_number,
    a.name AS accommodation_name,
    b.address,
    b.room_name,
    b.check_in,
    b.check_out,
    b.nights,
    b.guest_count,
    g.name AS guest_name,
    r.price_per_night,
    (r.price_per_night * b.nights) AS total_price,
    b.status,
    b.booked_date
FROM booking b
JOIN guest g ON g.guest_id = b.guest_id
JOIN accommodation a ON a.accommodation_id = b.accommodation_id
JOIN room r ON r.room_id = b.room_id
WHERE b.booking_number = 'BK-20250607-001';
```

### 6-5. 마이페이지 - 내 예약 목록

```sql
SELECT
    b.booking_number,
    a.name AS accommodation_name,
    b.room_name,
    b.check_in,
    b.check_out,
    b.nights,
    b.guest_count,
    (r.price_per_night * b.nights) AS total_price,
    b.status
FROM booking b
JOIN accommodation a ON a.accommodation_id = b.accommodation_id
JOIN room r ON r.room_id = b.room_id
WHERE b.guest_id = 1
ORDER BY b.check_in DESC;
```

### 6-5-1. 마이페이지 - 찜한 숙소

```sql
SELECT
    a.name AS accommodation_name,
    a.address,
    MIN(r.price_per_night) AS minimum_price_per_night,
    COALESCE(ROUND(AVG(rv.rating), 1), 0) AS average_rating,
    COUNT(rv.review_id) AS review_count
FROM guest_wishlist gw
JOIN wishlist w ON w.wishlist_id = gw.wishlist_id
JOIN accommodation a ON a.accommodation_id = w.accommodation_id
JOIN room r ON r.accommodation_id = a.accommodation_id
LEFT JOIN review rv ON rv.room_id IN (
    SELECT room_id FROM room WHERE accommodation_id = a.accommodation_id
)
WHERE gw.guest_id = 1
GROUP BY a.accommodation_id, a.name, a.address;
```

### 6-5-2. 마이페이지 - 내가 작성한 후기

```sql
SELECT
    a.name AS accommodation_name,
    rv.room_name,
    rv.rating,
    rv.content,
    rv.written_date
FROM review rv
JOIN room r ON r.room_id = rv.room_id
JOIN accommodation a ON a.accommodation_id = r.accommodation_id
WHERE rv.guest_id = 1
ORDER BY rv.written_date DESC;
```

### 6-6. 호스트 페이지 - 예약 목록

```sql
SELECT
    b.booking_number,
    a.name AS accommodation_name,
    b.room_name,
    g.name AS guest_name,
    g.email,
    g.phone,
    b.check_in,
    b.check_out,
    b.nights,
    b.guest_count,
    (r.price_per_night * b.nights) AS total_price,
    b.status
FROM booking b
JOIN guest g ON g.guest_id = b.guest_id
JOIN accommodation a ON a.accommodation_id = b.accommodation_id
JOIN room r ON r.room_id = b.room_id
WHERE a.host_id = 2
ORDER BY b.check_in;
```

### 6-6-1. 호스트 페이지 - 예약 집계

```sql
SELECT
    a.name AS accommodation_name,
    COUNT(*) FILTER (WHERE b.status <> 'CANCELED') AS booking_count,
    SUM(r.price_per_night * b.nights) FILTER (WHERE b.status <> 'CANCELED') AS total_booking_price
FROM accommodation a
LEFT JOIN booking b ON b.accommodation_id = a.accommodation_id
LEFT JOIN room r ON r.room_id = b.room_id
WHERE a.host_id = 2
GROUP BY a.accommodation_id, a.name;
```

### 6-6-2. 호스트 페이지 - 후기 현황

```sql
SELECT
    a.name AS accommodation_name,
    COALESCE(ROUND(AVG(rv.rating), 1), 0) AS average_rating,
    COUNT(rv.review_id) AS review_count,
    g.name AS reviewer_name,
    rv.room_name,
    rv.rating,
    rv.content,
    rv.written_date
FROM accommodation a
LEFT JOIN room r ON r.accommodation_id = a.accommodation_id
LEFT JOIN review rv ON rv.room_id = r.room_id
LEFT JOIN guest g ON g.guest_id = rv.guest_id
WHERE a.host_id = 2
GROUP BY a.accommodation_id, a.name, g.name, rv.room_name, rv.rating, rv.content, rv.written_date, rv.review_id
ORDER BY a.accommodation_id, rv.written_date DESC;
```

---

## 7. 조회 결과 검증

### 검증 항목

| 검증 항목 | 기댓값 | 실제값 | 상태 |
|----------|--------|--------|------|
| 메인: 최근 등록 순 | 최신부터 표시 | ✓ | PASS |
| 메인: 최저 요금 | MIN(room_price) | ✓ | PASS |
| 메인: 평균 평점 | ROUND(AVG(rating), 1) | ✓ | PASS |
| 제주 지역 필터 | 제주 2개 숙소 | ✓ | PASS |
| 예약번호 조회 | BK-20250607-001 | ✓ | PASS |
| 김민수 예약 | 2건 | ✓ | PASS |
| 찜한 숙소 | 1건 | ✓ | PASS |
| 작성 후기 | 1건 | ✓ | PASS |
| 호스트 예약 집계 | 2건, 640,000원 | ✓ | PASS |

---

## 8. 학습 포인트

### 8-1. 데이터 모델링 원칙

✅ **화면에 표시되는 값을 먼저 분석** → 필요한 테이블과 조회 컬럼 파악  
✅ **계산 가능한 값은 저장하지 않기** → 예약 금액 = `price_per_night * nights`  
✅ **M:N 관계는 연결 테이블로 표현** → `accommodation_amenity`, `guest_wishlist`  

### 8-2. ERD와 SQL의 관계

- ERD의 한글 개체를 SQL의 영어 테이블명으로 매핑
- PK는 각 행의 고유 식별, FK는 테이블 간 관계 연결
- 한글 컬럼명을 영어 snake_case로 변환 (예: `숙소명` → `name`)

### 8-3. 복잡한 조회의 해결책

**문제**: accommodation 테이블에 `amenity_id`가 있으면서 `accommodation_amenity` 연결 테이블도 존재?  
**해결**: 여러 편의시설을 조회할 때는 연결 테이블 사용

**문제**: `review` 테이블에 `accommodation_id`가 없음 (`review → room → accommodation` 경로 필요)  
**해결**: JOIN 경로를 명확히 이해하고 서브쿼리 또는 다중 JOIN 활용

**문제**: 객실과 후기를 동시에 JOIN하면 행이 증가할 수 있음  
**해결**: GROUP BY로 집계하되, 데이터 정합성 확인

### 8-4. SQL 작성의 순서

```
화면 요구사항 분석
↓
필요한 테이블 확인 (조회 범위)
↓
WHERE 조건 결정
↓
JOIN 경로 설계
↓
GROUP BY 및 집계 함수 결정
↓
ORDER BY로 정렬 순서 정의
↓
샘플 데이터로 검증
```

### 8-5. 자주 실수하기 쉬운 부분

❌ 모든 화면 정보를 하나의 테이블에 저장  
✓ 화면별로 필요한 정보만 JOIN해서 조회

❌ 계산할 수 있는 값을 모두 컬럼으로 저장  
✓ 필요할 때만 계산 (예약 금액, 평균 평점)

❌ M:N 관계를 무시하고 데이터 중복 저장  
✓ 연결 테이블로 표현하고 JOIN으로 조회

---

## 📌 정리

**오늘의 실습 결과**:
- ✅ 6개 화면 분석 → 10개 테이블 설계
- ✅ DDL: 제약조건과 FK 포함한 안전한 테이블 생성
- ✅ DML: 실제 상황을 반영한 샘플 데이터 12건 INSERT
- ✅ 화면별 SELECT: 6개 화면 × 다중 SQL = 총 13개 쿼리 작성
- ✅ 결과 검증: 실제 화면 예시와 조회 결과 비교

**다음 단계**:
- ACID 속성과 트랜잭션 이해
- 인덱스와 쿼리 성능 최적화
- 실제 프로젝트에서의 마이그레이션 전략

---

**#멀티캠퍼스부트캠프** | **#AI캠퍼스** | **#AI에이전트엔지니어**
