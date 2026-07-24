# 4. 데이터 변경 이력 설계1

## 핵심 요약

데이터 변경 이력 설계는 "이 데이터가 누가, 언제, 왜, 어디서 바뀌었는가"를 추적하기 위한 설계 문제이다. 이력 관리 시리즈의 첫 번째인 이 문서는 별도의 이력 테이블을 만들기 전에, 원본 테이블에 **변경 추적 컬럼**을 추가하는 가장 기본적인 방법을 다룬다.

이 문서의 핵심은 다음과 같다.

- 실무에서 데이터가 바뀔 때 자주 나오는 질문은 "누가", "언제", "왜", "이전 값은 무엇인가" 네 가지다.
- 기본 추적 컬럼(`created_at`, `created_by`, `updated_at`, `updated_by`)으로 "누가", "언제"를 알 수 있다.
- 변경 사유 컬럼(`change_type`, `change_reason`)으로 "왜"를 알 수 있다.
- 감사(Audit) 컬럼(`source_system`, `client_ip`)으로 "어디서"를 추적해 보안과 감사에 대응할 수 있다.
- `change_type`은 코드로 표준화하고, `change_reason`은 사람이 읽을 자연어 설명으로 나눠 쓰는 것이 좋다.
- 변경 추적 컬럼만으로는 "이전 값", "특정 시점의 값", "전체 변경 이력"은 알 수 없다. 이는 다음 문서인 이력 테이블에서 해결한다.

## 이력 관리 시리즈

이 주제는 두 편으로 나뉜다.

| 시리즈 | 주제 |
| --- | --- |
| 이력 관리 시리즈1 | 변경 추적 컬럼 (이 문서) |
| 이력 관리 시리즈2 | 이력 테이블 |

시리즈1은 원본 테이블에 컬럼 몇 개를 추가해 "가장 최근 변경"의 흔적을 남기는 방법을 다룬다. 시리즈2는 변경이 일어날 때마다 별도 행으로 이력을 쌓아 모든 변경 이력을 보존하는 방법을 다룬다.

## 이력 관리가 필요한 이유

데이터가 바뀔 때, 현재 값만 저장하는 테이블에서는 다음과 같은 질문에 답하기 어렵다.

- "이 상품의 가격이 왜 바뀌었지?"
- "재고가 왜 갑자기 줄었나요?"
- "이 상태를 누가 바꿨나요?"
- "언제부터 회원 등급이 이렇게 되어 있었나요?"

이런 질문에 답하려면 데이터의 변경 과정을 기록해야 한다. 그렇지 않으면 현재 값만 남고, 무엇이 어떻게 바뀌었는지 추적할 수 없다.

### 실습 예제

가장 단순한 상품 테이블로 시작한다. 이력 관리를 전혀 하지 않는 형태다.

```sql
DROP TABLE IF EXISTS product;
CREATE TABLE product (
    product_id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(200) NOT NULL,
    price INT NOT NULL,
    stock_quantity INT NOT NULL DEFAULT 0,
    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE'
);
```

샘플 데이터를 넣는다.

```sql
INSERT INTO product (name, price, stock_quantity, status)
VALUES ('스마트폰 케이스', 15000, 100, 'ACTIVE');
INSERT INTO product (name, price, stock_quantity, status)
VALUES ('무선 이어폰', 89000, 50, 'ACTIVE');
INSERT INTO product (name, price, stock_quantity, status)
VALUES ('노트북 파우치', 35000, 30, 'ACTIVE');
```

조회하면 현재 상태만 나온다.

```sql
SELECT * FROM product;
```

**[실행 결과]**

| product_id | name | price | stock_quantity | status |
| --- | --- | --- | --- | --- |
| 1 | 스마트폰 케이스 | 15000 | 100 | ACTIVE |
| 2 | 무선 이어폰 | 89000 | 50 | ACTIVE |
| 3 | 노트북 파우치 | 35000 | 30 | ACTIVE |

이제 15,000원인 스마트폰 케이스의 가격을 12,000원으로 내린다.

```sql
UPDATE product
SET price = 12000
WHERE product_id = 1;
```

변경 후 조회하면 값이 바뀌어 있다.

```sql
SELECT * FROM product WHERE product_id = 1;
```

**[실행 결과]**

| product_id | name | price | stock_quantity | status |
| --- | --- | --- | --- | --- |
| 1 | 스마트폰 케이스 | 12000 | 100 | ACTIVE |

가격이 15,000에서 12,000으로 바뀌었다. 그러나 여기서 문제가 생긴다.

### 핵심 질문 네 가지

이 변경에 대해 다음 질문에 전혀 답할 수 없다.

- 문제 1. 누가 변경했나요?
- 문제 2. 언제 변경했나요?
- 문제 3. 왜 변경했나요?
- 문제 4. 이전 값은 무엇이었나요?

이력 관리를 하지 않으면 데이터가 바뀐 순간 과거의 모든 정보가 사라진다. 이것은 마치 칠판을 지우고 새로 쓰는 것과 같아서, 지나간 내용은 영원히 알 수 없다.

### 문제 예시: 특정 시점 값 조회

고객이 어제 주문할 때 가격은 15,000원이었는데, 지금 가격이 12,000원이라고 해보자. 고객이 "어제 가격이 얼마였는지 확인해 달라"고 하면 현재 테이블로는 답할 수 없다.

```sql
-- 고객의 문의: 어제 시점의 가격은?
-- 현재 테이블에서 조회하면
SELECT product_id, name, price AS current_price
FROM product
WHERE product_id = 1;
```

**[실행 결과]**

| product_id | name | current_price |
| --- | --- | --- |
| 1 | 스마트폰 케이스 | 12000 |

현재 가격 12,000만 알 수 있고, 고객이 문의한 어제 시점의 가격 15,000원은 알 수 없다. 이 값을 확인하려면 변경 이력을 남겨야 한다.

이 네 가지 질문을 하나씩 해결하기 위해 이 문서에서는 변경 추적 컬럼을 단계적으로 추가한다. 가장 단순한 형태부터 시작해 점점 확장해 나간다.

## 변경 추적 컬럼 - 기본

가장 먼저 적용할 수 있는 것은 테이블에 변경 추적 컬럼을 추가하는 것이다. "누가, 언제" 생성하고 수정했는지를 기록한다.

### 기본 컬럼 추가

기존 컬럼에 더해 변경 추적 컬럼 4개를 추가한다.

```sql
DROP TABLE IF EXISTS product;
CREATE TABLE product (
    product_id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(200) NOT NULL,
    price INT NOT NULL,
    stock_quantity INT NOT NULL DEFAULT 0,
    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    -- 변경 추적 컬럼
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    created_by VARCHAR(100) NOT NULL,
    updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    updated_by VARCHAR(100) NOT NULL
);
```

각 컬럼의 의미는 다음과 같다.

| 컬럼 | 의미 |
| --- | --- |
| `created_at` | 데이터가 처음 생성된 시각 |
| `created_by` | 데이터를 처음 만든 사람(또는 시스템) |
| `updated_at` | 데이터가 마지막으로 수정된 시각 |
| `updated_by` | 데이터를 마지막으로 수정한 사람(또는 시스템) |

`created_at`은 `DEFAULT CURRENT_TIMESTAMP`로 생성 시각을 자동으로 채우고, `updated_at`은 `ON UPDATE CURRENT_TIMESTAMP`로 행이 수정될 때마다 자동으로 갱신된다.

### 데이터 등록

데이터를 등록할 때 생성자와 수정자를 함께 넣는다.

```sql
INSERT INTO product (name, price, stock_quantity, status, created_by, updated_by)
VALUES ('스마트폰 케이스', 15000, 100, 'ACTIVE', 'admin_kim', 'admin_kim');
INSERT INTO product (name, price, stock_quantity, status, created_by, updated_by)
VALUES ('무선 이어폰', 89000, 50, 'ACTIVE', 'admin_lee', 'admin_lee');
INSERT INTO product (name, price, stock_quantity, status, created_by, updated_by)
VALUES ('노트북 파우치', 35000, 30, 'ACTIVE', 'admin_kim', 'admin_kim');
```

등록된 데이터를 조회한다.

```sql
SELECT product_id, name, price, created_at, created_by, updated_at, updated_by
FROM product;
```

**[실행 결과]**

| product_id | name | price | created_at | created_by | updated_at | updated_by |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | 스마트폰 케이스 | 15000 | 2026-01-15 10:00:00 | admin_kim | 2026-01-15 10:00:00 | admin_kim |
| 2 | 무선 이어폰 | 89000 | 2026-01-15 10:05:00 | admin_lee | 2026-01-15 10:05:00 | admin_lee |
| 3 | 노트북 파우치 | 35000 | 2026-01-15 10:10:00 | admin_kim | 2026-01-15 10:10:00 | admin_kim |

이제 누가, 언제 만들었는지 알 수 있다.

생성 직후에는 `created_at`과 `updated_at`이 같은 값을 가진다. 처음에는 생성과 수정이 동시에 일어난 것으로 취급되기 때문이다.

> 참고: 등록 시점에 수정일(`updated_at`)을 `NULL`로 두면 "이 행은 한 번도 수정된 적이 없다"를 구분할 수 있다는 장점이 있다. 다만 수정일이 `NULL`이면 정렬이나 조회 시 매번 `COALESCE(updated_at, created_at)` 같은 처리가 필요하다. 등록 시점을 '0번째 수정'으로 간주해 `created_at`과 같은 값을 채워 두는 것이 조회는 편하다. 어느 쪽을 택할지는 팀의 정책으로 정한다.

### 데이터 수정

가격을 수정해 본다. 수정할 때는 수정자를 함께 갱신해야 한다.

```sql
UPDATE product
SET price = 12000,
    updated_by = 'admin_park'
WHERE product_id = 1;
```

수정된 데이터를 조회한다.

```sql
SELECT product_id, name, price, created_at, created_by, updated_at, updated_by
FROM product
WHERE product_id = 1;
```

**[실행 결과]**

| product_id | name | price | created_at | created_by | updated_at | updated_by |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | 스마트폰 케이스 | 12000 | 2026-01-15 10:00:00 | admin_kim | 2026-01-16 14:30:00 | admin_park |

이제 다음 질문에 답할 수 있다.

- 언제 생성됐나요? → 2026-01-15 10:00:00
- 누가 생성했나요? → admin_kim
- 언제 수정됐나요? → 2026-01-16 14:30:00
- 누가 수정했나요? → admin_park

### 장점과 주의점

기본 추적 컬럼의 장점은 다음과 같다.

1. **구현이 간단하다**: 기존 테이블에 컬럼 4개만 추가하면 된다.
2. **성능 부담이 적다**: 별도 테이블 없이 원본 행에 컬럼만 추가하면 된다.

주의할 점은, `updated_by`는 애플리케이션이 매번 명시적으로 채워야 한다는 것이다. `updated_at`은 `ON UPDATE`로 자동 갱신되지만, 수정자는 자동으로 채워지지 않는다.

### 네 가지 질문 점검

기본 추적 컬럼으로 두 가지 질문이 해결된다.

- 문제 1. 누가 변경했나요? → **해결**
- 문제 2. 언제 변경했나요? → **해결**
- 문제 3. 왜 변경했나요? → 미해결
- 문제 4. 이전 값은 무엇이었나요? → 미해결

이렇게 누가, 언제 변경했는지는 알 수 있지만, 아직 왜 변경했는지와 변경 이전의 값은 알 수 없다.

### 기본 추적 컬럼만으로 충분한 경우

다음과 같은 경우에는 기본 추적 컬럼만으로도 충분하다.

1. **이력이 크게 중요하지 않은 단순 데이터**: 코드성 데이터, 설정성 데이터 등
2. **단순 운영 관리 정보**: 부가 정보성 데이터
3. **비즈니스 규칙상 최신 값만 중요한 경우**: 과거 값이 의미 없는 데이터

반대로 가격 변경, 상태 변경, 재고 조정처럼 변경 과정 자체가 중요한 데이터는 더 상세한 추적이 필요하다. 다음 절에서 "왜" 변경했는지를 남기는 방법을 살펴본다.