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