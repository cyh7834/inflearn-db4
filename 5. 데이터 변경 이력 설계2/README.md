# 5. 데이터 변경 이력 설계2

## 핵심 요약

이력 관리 시리즈의 두 번째인 이 문서는 앞 문서에서 해결하지 못한 "이전 값", "특정 시점의 값", "전체 변경 이력"을 다룬다. 핵심은 **변경된 행 자체를 보관하는 것**이며, 그 보관 위치와 형태에 따라 여러 설계 방식이 나뉜다.

이 문서의 핵심은 다음과 같다.

- 이전 값을 별도 컬럼(`previous_price`)에 두는 방식은 구현이 쉽지만 직전 값 하나만 남는다(SCD Type 3).
- 현재 테이블에 `INSERT`로 이력을 쌓고 `is_current`로 최신 행을 구분하면 전체 이력은 보존되지만, 시점 조회와 통계 쿼리가 복잡하고 느리다.
- `valid_from`, `valid_to` 유효 기간 컬럼을 두면 시점 조회가 단순한 범위 조건 하나로 해결된다(SCD Type 2).
- 실무에서 가장 많이 쓰는 패턴은 현재 테이블과 이력 테이블을 **분리**한 전체 행 스냅샷 방식이다.
- 이력 테이블에는 **최초 `INSERT` 시점부터** 반드시 기록해야 한다. 그렇지 않으면 조회할 때마다 `UNION`이 필요하다.
- 컬럼 단위 변경 로그는 용량은 절약되지만 시점 복원과 통계가 사실상 불가능하다.
- 공통 이력 테이블(`audit_log`)은 시스템 전체 감사에 유용하지만 보조 수단이다. 중요한 데이터는 전용 이력 테이블을 둔다.
- 특별한 이유가 없다면 **전체 행 스냅샷을 기본**으로 선택한다. 저장 공간보다 개발 생산성과 데이터 정합성이 비싸다.

## 이력 관리 시리즈

| 시리즈 | 주제 |
| --- | --- |
| 이력 관리 시리즈1 | 변경 추적 컬럼 |
| 이력 관리 시리즈2 | 이력 테이블 (이 문서) |

시리즈1에서는 원본 테이블에 `created_at`, `updated_by`, `change_reason` 같은 컬럼을 추가해 "가장 최근 변경"의 흔적을 남겼다. 이 방식으로는 "누가, 언제, 왜, 어디서"까지만 알 수 있었다. 시리즈2에서는 변경이 일어날 때마다 별도의 행을 남겨 모든 변경 이력을 보존하는 방법을 다룬다.

## 컬럼에 이전 값 보관 방식

지금까지는 현재 테이블에 "추적 정보"를 추가하는 방식이었다. 하지만 이 방식으로는 "이전 값"을 알 수 없었다. 먼저 이전 값을 보관하는 가장 단순한 방법부터 알아본다.

### 아이디어

아이디어는 간단하다. 현재 값과 이전 값을 각각 별도의 컬럼에 저장하는 것이다.

- `price` : 현재 가격
- `previous_price` : 이전 가격
- `price_changed_at` : 가격이 변경된 시점

### 테이블 설계

```sql
DROP TABLE IF EXISTS product;
CREATE TABLE product (
    product_id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(200) NOT NULL,
    -- 현재 가격
    price INT NOT NULL,
    -- 이전 가격
    previous_price INT,
    price_changed_at DATETIME,

    stock_quantity INT NOT NULL DEFAULT 0,
    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',

    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    created_by VARCHAR(100) NOT NULL,
    updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    updated_by VARCHAR(100) NOT NULL
);
```

### 데이터 등록

상품을 처음 등록할 때는 아직 이전 가격이 없다.

```sql
INSERT INTO product (name, price, previous_price, price_changed_at, stock_quantity, created_by, updated_by)
VALUES ('스마트폰 케이스', 15000, NULL, NULL, 100, 'admin_kim', 'admin_kim');
INSERT INTO product (name, price, previous_price, price_changed_at, stock_quantity, created_by, updated_by)
VALUES ('무선 이어폰', 89000, NULL, NULL, 50, 'admin_lee', 'admin_lee');
```

```sql
SELECT product_id, name, price, previous_price, price_changed_at
FROM product;
```

**[실행 결과]**

| product_id | name | price | previous_price | price_changed_at |
| --- | --- | --- | --- | --- |
| 1 | 스마트폰 케이스 | 15000 | NULL | NULL |
| 2 | 무선 이어폰 | 89000 | NULL | NULL |

### 가격 변경

가격을 변경할 때는 현재 가격을 이전 가격으로 옮기고, 새 가격을 넣는다.

```sql
UPDATE product
SET previous_price = price,
    price = 12000,
    price_changed_at = NOW(),
    updated_by = 'admin_park'
WHERE product_id = 1;
```

```sql
SELECT product_id, name, price, previous_price, price_changed_at
FROM product
WHERE product_id = 1;
```

**[실행 결과]**

| product_id | name | price | previous_price | price_changed_at |
| --- | --- | --- | --- | --- |
| 1 | 스마트폰 케이스 | 12000 | 15000 | 2026-03-01 10:00:00 |

이제 이전 가격을 알 수 있다. 현재 가격은 12,000원이고, 이전 가격은 15,000원이다. 추가로 `price_changed_at`을 통해서 가격이 변경된 시점도 알 수 있다.

### 한 번 더 가격 변경

가격을 한 번 더 변경해 본다.

```sql
UPDATE product
SET previous_price = price,
    price = 10000,
    price_changed_at = NOW(),
    updated_by = 'admin_kim'
WHERE product_id = 1;
```

```sql
SELECT product_id, name, price, previous_price, price_changed_at
FROM product
WHERE product_id = 1;
```

**[실행 결과]**

| product_id | name | price | previous_price | price_changed_at |
| --- | --- | --- | --- | --- |
| 1 | 스마트폰 케이스 | 10000 | 12000 | 2026-03-15 14:00:00 |

문제가 생겼다. 현재 가격 10,000원과 직전 가격 12,000원은 알 수 있지만, 최초 가격 15,000원은 사라졌다.

### 한계

이 방식은 "바로 직전 값" 하나만 보관할 수 있다. 두 번 이상 변경되면 과거 값이 사라진다.

```sql
-- 가격 변경 이력을 모두 보고 싶다면?
SELECT product_id, name,
       price AS current_price,
       previous_price AS one_before,
       '(최초 가격, 15000): 사라짐' AS two_before
FROM product
WHERE product_id = 1;
```

**[실행 결과]**

| product_id | name | current_price | one_before | two_before |
| --- | --- | --- | --- | --- |
| 1 | 스마트폰 케이스 | 10000 | 12000 | (최초 가격, 15000): 사라짐 |

### 여러 컬럼을 추적해야 한다면?

가격뿐만 아니라 재고, 상태 등도 이전 값을 추적해야 한다면 어떻게 될까?

```sql
-- 이런 식으로 컬럼이 폭발적으로 늘어난다
CREATE TABLE product_example (
    product_id BIGINT PRIMARY KEY,
    name VARCHAR(200),

    -- 가격 추적
    price INT,
    previous_price INT,
    price_changed_at DATETIME,

    -- 재고 추적
    stock_quantity INT,
    previous_stock_quantity INT,
    stock_changed_at DATETIME,

    -- 상태 추적
    status VARCHAR(20),
    previous_status VARCHAR(20),
    status_changed_at DATETIME

    -- 추적할 컬럼이 늘어날 때마다 3개씩 컬럼이 추가된다...
);
```

컬럼이 너무 많아진다. 관리가 어려워지고, 테이블 구조가 복잡해진다.

### 정리

**장점**

1. 구현이 매우 간단하다.
2. 조회가 빠르다 (조인 없이 한 테이블에서 조회).
3. 직전 값만 필요한 경우 유용하다.

**단점**

1. 직전 값 하나만 보관할 수 있다.
2. 두 번 이상 변경되면 과거 이력이 사라진다.
3. 추적할 컬럼이 많아지면 테이블 구조가 복잡해진다.
4. 특정 시점의 값을 조회할 수 없다.

**언제 사용하면 좋을까?**

- 변경이 거의 없는 데이터 (예: 회원 이름)
- 직전 값만 알면 되는 경우
- 빠른 조회가 중요하고, 전체 이력이 필요 없는 경우

실무에서 이전 값 보관만으로 충분한 경우는 드물다. 대부분은 전체 변경 이력이 필요하다. 다음 절에서는 행 자체를 보관하는 방법을 살펴본다.

### 참고: SCD Type 3

SCD는 Slowly Changing Dimension의 약자로, 데이터 웨어하우스 분야에서 사용하는 용어다. 천천히 변경되는 데이터를 어떻게 관리할 것인가에 대한 여러 가지 방법론이 있다. 그중 Type 3는 "이전 값을 별도 컬럼에 저장하는 방식"이다. 이번에 사용한 방식이 SCD Type 3 방식이다.

## 현재 테이블로 이력 관리 - 시작

이전 컬럼 하나만 저장하는 방식은 한계가 명확했다. 전체 변경 이력을 관리하려면 결국 "행(Row) 자체"를 보관해야 한다. 가장 직관적인 방법은 현재 테이블에 모든 이력을 함께 저장하는 것이다.

### 아이디어

데이터를 수정할 때 기존 행을 `UPDATE`하지 않고, 새로운 행을 `INSERT`한다. 이렇게 하면 모든 변경 이력이 행으로 남는다. 그리고 `is_current` 필드를 사용해서 최신 데이터를 구분한다.

### 테이블 설계

```sql
DROP TABLE IF EXISTS product;
CREATE TABLE product (
    history_id BIGINT PRIMARY KEY AUTO_INCREMENT,
    product_id BIGINT NOT NULL,
    name VARCHAR(200) NOT NULL,
    price INT NOT NULL,
    stock_quantity INT NOT NULL DEFAULT 0,
    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    is_current BOOLEAN NOT NULL DEFAULT TRUE,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    created_by VARCHAR(100) NOT NULL,

    INDEX idx_product_id (product_id),
    INDEX idx_is_current (is_current),
    INDEX idx_product_id2 (product_id, created_at)
);
```

주요 변경 사항은 다음과 같다.

| 컬럼 | 의미 |
| --- | --- |
| `history_id` | 각 이력 행의 고유 ID (Primary Key) |
| `product_id` | 실제 상품 ID (같은 상품의 이력들은 같은 `product_id`를 가진다) |
| `is_current` | 현재 유효한 데이터인지 여부 (TRUE면 최신 데이터) |

여기서는 PK가 `history_id`이기 때문에 같은 `product_id`도 여러 번 등록할 수 있다.

### 데이터 등록

상품을 처음 등록한다.

```sql
INSERT INTO product (product_id, name, price, stock_quantity, status, is_current, created_by, created_at)
VALUES (1, '스마트폰 케이스', 15000, 100, 'ACTIVE', TRUE, 'admin_kim', '2026-01-15 10:00:00');
INSERT INTO product (product_id, name, price, stock_quantity, status, is_current, created_by, created_at)
VALUES (2, '무선 이어폰', 89000, 50, 'ACTIVE', TRUE, 'admin_lee', '2026-01-15 10:05:00');
```

> 참고: `created_at`은 이후에 사용할 예제를 위해서 시간 정보를 직접 입력했다.

```sql
SELECT history_id, product_id, name, price, is_current, created_at
FROM product;
```

**[실행 결과]**

| history_id | product_id | name | price | is_current | created_at |
| --- | --- | --- | --- | --- | --- |
| 1 | 1 | 스마트폰 케이스 | 15000 | 1 | 2026-01-15 10:00:00 |
| 2 | 2 | 무선 이어폰 | 89000 | 1 | 2026-01-15 10:05:00 |

- `history_id`가 PK이다. `history_id`는 자동증가하는 값이다.
- `product_id`는 상품 ID를 직접 입력했다.

### 데이터 변경

스마트폰 케이스의 가격을 15000에서 12000으로 변경해 본다. 이 방식에서 가격을 변경할 때는 두 가지 작업이 필요하다.

1. 기존 행의 `is_current`를 `FALSE`로 변경
2. 새로운 행을 `INSERT`

```sql
-- 1. 기존 행을 과거 데이터로 변경
UPDATE product
SET is_current = FALSE
WHERE product_id = 1 AND is_current = TRUE;

-- 2. 새로운 행 추가
INSERT INTO product (product_id, name, price, stock_quantity, status, is_current, created_by, created_at)
VALUES (1, '스마트폰 케이스', 12000, 100, 'ACTIVE', TRUE, 'admin_park', '2026-03-01 10:00:00');
```

```sql
SELECT history_id, product_id, name, price, is_current, created_at, created_by
FROM product
WHERE product_id = 1
ORDER BY history_id;
```

**[실행 결과]**

| history_id | product_id | name | price | is_current | created_at | created_by |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | 1 | 스마트폰 케이스 | 15000 | 0 | 2026-01-15 10:00:00 | admin_kim |
| 3 | 1 | 스마트폰 케이스 | 12000 | 1 | 2026-03-01 10:00:00 | admin_park |

두 개의 행이 있다. `is_current = 0`인 행은 과거 데이터이고, `is_current = 1`인 행이 현재 데이터다.

### 현재 데이터만 조회

`is_current=TRUE`를 통해 현재 유효한 데이터만 조회할 수 있다.

```sql
SELECT history_id, product_id, name, price, stock_quantity, status, is_current
FROM product
WHERE is_current = TRUE
  AND product_id = 1;
```

**[실행 결과]**

| history_id | product_id | name | price | stock_quantity | status | is_current |
| --- | --- | --- | --- | --- | --- | --- |
| 3 | 1 | 스마트폰 케이스 | 12000 | 100 | ACTIVE | 1 |

### 한 번 더 변경

이번에는 스마트폰 케이스의 가격을 12000에서 10000으로 변경하고, 재고도 100에서 95로 변경한다.

```sql
-- 1. 기존 행을 과거 데이터로 변경
UPDATE product
SET is_current = FALSE
WHERE product_id = 1 AND is_current = TRUE;

-- 2. 새로운 행 추가
INSERT INTO product (product_id, name, price, stock_quantity, status, is_current, created_by, created_at)
VALUES (1, '스마트폰 케이스', 10000, 95, 'ACTIVE', TRUE, 'admin_kim', '2026-03-15 14:00:00');
```

```sql
SELECT history_id, product_id, name, price, stock_quantity, is_current, created_at
FROM product
WHERE product_id = 1
ORDER BY history_id;
```

**[실행 결과]**

| history_id | product_id | name | price | stock_quantity | is_current | created_at |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | 1 | 스마트폰 케이스 | 15000 | 100 | 0 | 2026-01-15 10:00:00 |
| 3 | 1 | 스마트폰 케이스 | 12000 | 100 | 0 | 2026-03-01 10:00:00 |
| 4 | 1 | 스마트폰 케이스 | 10000 | 95 | 1 | 2026-03-15 14:00:00 |

이제 모든 변경 이력이 남는다. 가격이 15,000 → 12,000 → 10,000으로 변경된 것을 모두 확인할 수 있다. 재고도 마지막에 100에서 95로 변경된 것을 확인할 수 있다.

### 과거 데이터만 조회

`is_current=FALSE` 조건을 통해 과거 데이터만 조회할 수 있다.

```sql
SELECT history_id, product_id, name, price, stock_quantity, status, is_current
FROM product
WHERE is_current = FALSE;
```

**[실행 결과]**

| history_id | product_id | name | price | stock_quantity | status | is_current |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | 1 | 스마트폰 케이스 | 15000 | 100 | ACTIVE | 0 |
| 3 | 1 | 스마트폰 케이스 | 12000 | 100 | ACTIVE | 0 |

### 특정 상품의 전체 이력 조회

```sql
SELECT history_id, product_id, name, price, stock_quantity, created_at, created_by,
       CASE WHEN is_current THEN '현재' ELSE '과거' END AS data_status
FROM product
WHERE product_id = 1
ORDER BY created_at;
```

**[실행 결과]**

| history_id | product_id | name | price | stock_quantity | created_at | created_by | data_status |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | 1 | 스마트폰 케이스 | 15000 | 100 | 2026-01-15 10:00:00 | admin_kim | 과거 |
| 3 | 1 | 스마트폰 케이스 | 12000 | 100 | 2026-03-01 10:00:00 | admin_park | 과거 |
| 4 | 1 | 스마트폰 케이스 | 10000 | 95 | 2026-03-15 14:00:00 | admin_kim | 현재 |

### 장점

1. **전체 이력 보관**: 모든 변경 이력이 행으로 남는다.
2. **이전 값 확인 가능**: 언제든 과거 값을 조회할 수 있다.
3. **구현이 단순하다**: 별도의 이력 테이블 없이 하나의 테이블로 관리한다.

## 현재 테이블로 이력 관리 - 단점 1

### 시점 조회의 어려움

이 방식은 특정 시점의 데이터를 조회하기 어렵다는 문제가 있다. 먼저 단일 상품을 기준으로 "2026년 3월 10일 기준으로 가격이 얼마였나요?"라는 질문에 답해 본다. 여기서는 스마트폰 케이스(`product_id=1`)를 조회한다.

```sql
SELECT history_id, product_id, name, price, created_at, is_current
FROM product
WHERE product_id = 1
ORDER BY created_at DESC;
```

**[실행 결과]**

| history_id | product_id | name | price | created_at | is_current |
| --- | --- | --- | --- | --- | --- |
| 4 | 1 | 스마트폰 케이스 | 10000 | 2026-03-15 14:00:00 | 1 |
| 3 | 1 | 스마트폰 케이스 | 12000 | 2026-03-01 10:00:00 | 0 |
| 1 | 1 | 스마트폰 케이스 | 15000 | 2026-01-15 10:00:00 | 0 |

`created_at` 값을 기준으로 각 데이터가 사용된 기간을 정리하면 다음과 같다. (시간은 제외했다)

| 행 | 생성일 | 사용된 기간 |
| --- | --- | --- |
| `history_id=4` | 2026-03-15 | `[2026-03-15 ~ 현재]` |
| `history_id=3` | 2026-03-01 | `[2026-03-01 ~ 2026-03-15 이전]` |
| `history_id=1` | 2026-01-15 | `[2026-01-15 ~ 2026-03-01 이전]` |

따라서 판단은 다음과 같다.

- `history_id=4`는 `[2026-03-15 ~ 현재]` 데이터이므로 2026년 3월 10일 기준을 만족하지 않는다.
- `history_id=3`은 `[2026-03-01 ~ 2026-03-15 이전]` 데이터이므로 2026년 3월 10일 기준을 만족한다.
- `history_id=1`은 `[2026-01-15 ~ 2026-03-01 이전]` 데이터이므로 2026년 3월 10일 기준을 만족하지 않는다.

정리하면 `history_id=3`이 2026년 3월 10일에 사용된 데이터이므로 이 데이터를 선택하면 된다. 쿼리로는 다음과 같이 나타낼 수 있다.

```sql
-- 2026-03-10 기준 가격 조회
SELECT history_id, product_id, name, price, created_at, is_current
FROM product
WHERE product_id = 1
  AND created_at <= '2026-03-10 23:59:59'
ORDER BY created_at DESC;
```

날짜 내림차순으로 정렬한 다음 생성일이 `2026-03-10`보다 같거나 작은 날짜를 찾는다.

> 참고: 예시를 단순화하기 위해 예시들에 밀리초는 없다고 가정한다.

**[실행 결과]**

| history_id | product_id | name | price | created_at | is_current |
| --- | --- | --- | --- | --- | --- |
| 3 | 1 | 스마트폰 케이스 | 12000 | 2026-03-01 10:00:00 | 0 |
| 1 | 1 | 스마트폰 케이스 | 15000 | 2026-01-15 10:00:00 | 0 |

> 참고: 조회 결과 `history_id=3`의 현재 `is_current`는 0(False)으로 나온다. 2026년 3월 10일 당시에는 이 값이 1(True)이었을 것이다.

생성일이 2026-03-10보다 같거나 작은 날짜를 찾으니 2건의 데이터가 나온다. 내림차순에서 가장 상단에 있는 2026-03-01 데이터가 우리가 찾는 데이터다. 그 이전 데이터들은 모두 제거해야 한다.

날짜 내림차순이기 때문에 `LIMIT 1`을 사용하면 딱 하나의 데이터만 찾을 수 있다.

```sql
-- 2026-03-10 기준 가격 조회
SELECT history_id, product_id, name, price, created_at, is_current
FROM product
WHERE product_id = 1
  AND created_at <= '2026-03-10 23:59:59'
ORDER BY created_at DESC
LIMIT 1;
```

**[실행 결과]**

| history_id | product_id | name | price | created_at | is_current |
| --- | --- | --- | --- | --- | --- |
| 3 | 1 | 스마트폰 케이스 | 12000 | 2026-03-01 10:00:00 | 0 |

3월 10일 기준 가격은 12,000원이었다. 단일 상품의 경우 특정 시점의 조회는 간단하다.

## 현재 테이블로 이력 관리 - 단점 2

### 전체 통계에서의 성능 문제

앞서 본 것처럼 단일 상품의 시점 조회는 괜찮다. 하지만 전체 상품의 특정 시점 통계를 내야 한다면 어떻게 될까?

**"2026년 3월 10일 기준, 모든 상품의 총 재고 수량은?"**

우선 모든 데이터를 확인한다.

```sql
SELECT history_id, product_id, name, created_at, stock_quantity
FROM product
ORDER BY created_at DESC;
```

**[실행 결과]**

| history_id | product_id | name | created_at | stock_quantity |
| --- | --- | --- | --- | --- |
| 4 | 1 | 스마트폰 케이스 | 2026-03-15 14:00:00 | 95 |
| 3 | 1 | 스마트폰 케이스 | 2026-03-01 10:00:00 | 100 |
| 2 | 2 | 무선 이어폰 | 2026-01-15 10:05:00 | 50 |
| 1 | 1 | 스마트폰 케이스 | 2026-01-15 10:00:00 | 100 |

2026년 3월 10일 기준 상품은 다음과 같다.

- **스마트폰 케이스**: `history_id=3`, 재고: 100
- **무선 이어폰**: `history_id=2`, 재고: 50

각 상품별 2026년 3월 10일 기준 재고를 모두 구해서 합하는 쿼리는 다음과 같다.

```sql
-- 각 상품별로 해당 시점의 최신 데이터를 찾아야 한다
SELECT SUM(p.stock_quantity) AS total_stock
FROM product p
INNER JOIN (
    SELECT product_id, MAX(created_at) AS max_created_at
    FROM product
    WHERE created_at <= '2026-03-10 23:59:59'
    GROUP BY product_id
) latest ON p.product_id = latest.product_id
        AND p.created_at = latest.max_created_at;
```

**[실행 결과]**

| total_stock |
| --- |
| 150 |

이 쿼리는 서브쿼리(Subquery)와 조인(Join), 그리고 집계 함수(Aggregate Function)가 섞여 있어 처음 보면 복잡해 보일 수 있다. 우리가 원하는 것은 **"2026년 3월 10일 시점에 각 상품의 가장 마지막 상태"**를 찾아내는 것이다. 이 쿼리가 만들어진 과정을 단계별로 살펴본다.

### 1단계: 특정 시점 이전의 데이터만 필터링하기

가장 먼저 해야 할 일은 타임머신을 타고 과거로 돌아가는 것이다. 기준 날짜인 '2026년 3월 10일' 이후에 생성된 데이터는 미래의 일이므로 모두 무시해야 한다.

```sql
SELECT history_id, product_id, name, created_at, stock_quantity
FROM product
WHERE created_at <= '2026-03-10 23:59:59'
ORDER BY created_at DESC;
```

**[실행 결과]**

| history_id | product_id | name | created_at | stock_quantity |
| --- | --- | --- | --- | --- |
| 3 | 1 | 스마트폰 케이스 | 2026-03-01 10:00:00 | 100 |
| 2 | 2 | 무선 이어폰 | 2026-01-15 10:05:00 | 50 |
| 1 | 1 | 스마트폰 케이스 | 2026-01-15 10:00:00 | 100 |

결과를 보면 3월 15일에 생성된 데이터(`history_id=4`)는 제외되었다.

하지만 `product_id = 1`인 상품의 데이터가 2개(1월 15일, 3월 1일) 존재한다. 왜냐하면 `WHERE created_at <= '2026-03-10 23:59:59'` 검색 조건은 `2026-03-10`을 포함한 이전의 데이터를 모두 조회하기 때문이다.

이전에 하나의 상품을 조회할 때는 단순하게 `LIMIT 1`을 적용해서 가장 상단에 있는 데이터를 조회하면 되었다. 하지만 여러 상품이 섞여 있는 경우에는 이 방법을 사용할 수 없다. (여기서는 무선 이어폰의 재고도 찾아야 한다.)

### 2단계: 상품별로 가장 마지막 시간 구하기

이제 필터링된 데이터 중에서, 각 상품별로 가장 최근의 시간(`MAX(created_at)`)을 찾아야 한다. 이것이 바로 그 시점의 '유효한 데이터'이기 때문이다.

스마트폰 케이스의 경우 다음 두 시간 중에 가장 최근인 2026-03-01을 찾으면 된다.

- 2026-03-01 10:00:00
- 2026-01-15 10:00:00

`GROUP BY`를 사용해서 각 상품별로 묶고 가장 큰 날짜를 구한다.

```sql
SELECT product_id, MAX(created_at) AS max_created_at
FROM product
WHERE created_at <= '2026-03-10 23:59:59'
GROUP BY product_id;
```

**[실행 결과]**

| product_id | max_created_at |
| --- | --- |
| 1 | 2026-03-01 10:00:00 |
| 2 | 2026-01-15 10:05:00 |

이제 각 상품별로 3월 10일 기준, 어떤 시간의 데이터가 '진짜'인지 알게 되었다.

- 상품 1번(스마트폰 케이스)은 3월 1일 10시 데이터가 최신이다.
- 상품 2번(무선 이어폰)은 1월 15일 10시 5분 데이터가 최신이다.

### 3단계: 원본 테이블과 조인하여 재고 정보 가져오기

2단계에서 구한 것은 단순히 `product_id`와 `시간` 정보뿐이다. 실제 `stock_quantity(재고)` 정보를 알기 위해서는 이 정보를 바탕으로 다시 원본 `product` 테이블과 조인(Join)해야 한다.

이때 **'상품 ID가 같고' AND '생성 시간이 같은'** 행을 찾아야 정확한 이력 데이터를 가져올 수 있다.

```sql
SELECT p.product_id, p.name, p.stock_quantity, p.created_at
FROM product p
INNER JOIN (
    SELECT product_id, MAX(created_at) AS max_created_at
    FROM product
    WHERE created_at <= '2026-03-10 23:59:59'
    GROUP BY product_id
) latest ON p.product_id = latest.product_id
        AND p.created_at = latest.max_created_at;
```

**[실행 결과]**

| product_id | name | stock_quantity | created_at |
| --- | --- | --- | --- |
| 1 | 스마트폰 케이스 | 100 | 2026-03-01 10:00:00 |
| 2 | 무선 이어폰 | 50 | 2026-01-15 10:05:00 |

드디어 3월 10일 기준의 각 상품별 유효한 행을 하나씩 뽑아냈다.

### 4단계: 최종 집계

이제 각 상품의 재고 수량(`stock_quantity`)을 모두 더하기만 하면 된다.

```sql
SELECT SUM(p.stock_quantity) AS total_stock
FROM product p
INNER JOIN (
    SELECT product_id, MAX(created_at) AS max_created_at
    FROM product
    WHERE created_at <= '2026-03-10 23:59:59'
    GROUP BY product_id
) latest ON p.product_id = latest.product_id
        AND p.created_at = latest.max_created_at;
```

**[실행 결과]**

| total_stock |
| --- |
| 150 |

상품 1의 재고(100개)와 상품 2의 재고(50개)가 합쳐져서 최종 결과 150개가 나왔다.

### 성능 문제

이처럼 **'현재 테이블에 이력을 함께 쌓는 방식'**은 데이터 입력은 쉽지만, 특정 시점의 데이터를 조회하거나 통계를 낼 때 쿼리가 매우 복잡해지고 성능(Performance) 이슈가 발생할 수 있다. 특히 각 상품별 특정 시점을 찾기 위해 특정 시점 이전의 과거 전체 이력을 찾아야 하는 서브쿼리가 필요하고, 또 추가적인 조인이 필요하다. 이런 쿼리는 성능을 최적화하기 어렵다.

만약 상품이 10만 개이고, 각 상품이 평균 10번씩 변경되었다면 테이블에는 100만 개의 행이 있다. 이 중에서 각 상품의 특정 시점 데이터를 찾으려면 서브쿼리에서 전체 테이블을 스캔해야 한다. 그리고 테이블의 크기가 커질수록 이 `GROUP BY`와 `JOIN` 연산은 데이터베이스에 큰 부하를 주게 된다.

결과적으로 이 방식에는 3가지 문제가 있다.

1. 시점 조회와 통계 쿼리 작성의 복잡함
2. 시점 조회와 통계 쿼리의 성능 문제
3. 한 테이블에 너무 많은 데이터 보유

현재 테이블에 이력을 함께 저장하는 방식은 전체 이력을 보관할 수 있지만, 시점 조회와 통계 쿼리에서 성능 문제가 발생한다. 다음 절에서 이 문제를 해결하는 방법을 살펴본다.

## 현재 테이블로 이력 관리 - 유효 기간

`is_current` 플래그만으로는 시점 조회가 어렵다는 것을 확인했다. 이 문제를 해결하기 위해 "유효 기간"을 추가하는 방법이 있다.

### 유효 기간 아이디어

각 행에 "이 데이터가 유효한 기간"을 명시한다.

- `valid_from` : 이 데이터가 유효해진 시점
- `valid_to` : 이 데이터가 더 이상 유효하지 않게 된 시점 (현재 데이터는 `NULL` 또는 먼 미래 날짜)