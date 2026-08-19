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

## 변경 추적 컬럼 - 변경 사유

기본 추적 컬럼만으로는 "누가, 언제"는 알 수 있지만 "왜"는 알 수 없다. 실무에서는 변경 사유가 중요한 경우가 많다.

### 문제 상황

가격이 바뀌었다고 하자. 담당자에게 물어보면 "할인 이벤트 때문에 내렸다"고 한다. 하지만 시간이 지나거나 담당자가 바뀌면 왜 내렸는지 알 수 없게 된다.

- "이 가격 변경은 할인 이벤트인가요, 원가 조정인가요?"
- "언제까지 유효한 가격인가요?"
- "특정 고객사 전용 가격을 적용한 건가요?"

이런 질문에 답하려면 변경 사유를 남겨야 한다.

### 변경 사유 컬럼 추가

기본 추적 컬럼에 변경 사유를 담는 컬럼을 추가한다.

```sql
DROP TABLE IF EXISTS product;
CREATE TABLE product (
    product_id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(200) NOT NULL,
    price INT NOT NULL,
    stock_quantity INT NOT NULL DEFAULT 0,
    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    -- 기본 변경 추적 컬럼
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    created_by VARCHAR(100) NOT NULL,
    updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    updated_by VARCHAR(100) NOT NULL,
    -- 변경 사유 컬럼
    change_type VARCHAR(50),
    change_reason VARCHAR(500)
);
```

추가된 컬럼의 의미는 다음과 같다.

| 컬럼 | 의미 |
| --- | --- |
| `change_type` | 변경 유형 (예: PRICE_CHANGE, STATUS_CHANGE, STOCK_ADJUST 등) |
| `change_reason` | 변경 사유 (자유 형식의 텍스트) |

### 데이터 등록

데이터를 등록할 때는 변경 유형을 `'CREATE'`로 한다.

```sql
INSERT INTO product (name, price, stock_quantity, status, created_by, updated_by, change_type, change_reason)
VALUES ('스마트폰 케이스', 15000, 100, 'ACTIVE', 'admin_kim', 'admin_kim', 'CREATE', '신규 상품 등록');
INSERT INTO product (name, price, stock_quantity, status, created_by, updated_by, change_type, change_reason)
VALUES ('무선 이어폰', 89000, 50, 'ACTIVE', 'admin_lee', 'admin_lee', 'CREATE', '신규 상품 등록');
```

```sql
SELECT product_id, name, price, change_type, change_reason, updated_by
FROM product;
```

**[실행 결과]**

| product_id | name | price | change_type | change_reason | updated_by |
| --- | --- | --- | --- | --- | --- |
| 1 | 스마트폰 케이스 | 15000 | CREATE | 신규 상품 등록 | admin_kim |
| 2 | 무선 이어폰 | 89000 | CREATE | 신규 상품 등록 | admin_lee |

### 가격 변경 - 할인 이벤트

할인 이벤트로 가격을 내린다.

```sql
UPDATE product
SET price = 12000,
    updated_by = 'admin_park',
    change_type = 'PRICE_CHANGE',
    change_reason = '봄 시즌 할인 이벤트 (2026-03-01 ~ 2026-03-31)'
WHERE product_id = 1;
```

```sql
SELECT product_id, name, price, change_type, change_reason
FROM product
WHERE product_id = 1;
```

**[실행 결과]**

| product_id | name | price | change_type | change_reason |
| --- | --- | --- | --- | --- |
| 1 | 스마트폰 케이스 | 12000 | PRICE_CHANGE | 봄 시즌 할인 이벤트 (2026-03-01 ~ 2026-03-31) |

이제 왜 가격이 바뀌었는지 알 수 있다.

### 재고 조정

재고를 실사한 후 수량을 조정한다.

```sql
UPDATE product
SET stock_quantity = 95,
    updated_by = 'warehouse_kim',
    change_type = 'STOCK_ADJUST',
    change_reason = '실물 재고 실사 결과 5개 손실 확인'
WHERE product_id = 1;
```

```sql
SELECT product_id, name, stock_quantity, change_type, change_reason
FROM product
WHERE product_id = 1;
```

**[실행 결과]**

| product_id | name | stock_quantity | change_type | change_reason |
| --- | --- | --- | --- | --- |
| 1 | 스마트폰 케이스 | 95 | STOCK_ADJUST | 실물 재고 실사 결과 5개 손실 확인 |

### 상태 변경 - 판매 중지

상품을 판매 중지 상태로 바꾼다.

```sql
UPDATE product
SET status = 'INACTIVE',
    updated_by = 'admin_lee',
    change_type = 'STATUS_CHANGE',
    change_reason = '판매사 계약 종료로 판매 중지'
WHERE product_id = 2;
```

```sql
SELECT product_id, name, status, change_type, change_reason
FROM product;
```

**[실행 결과]**

| product_id | name | status | change_type | change_reason |
| --- | --- | --- | --- | --- |
| 1 | 스마트폰 케이스 | ACTIVE | STOCK_ADJUST | 실물 재고 실사 결과 5개 손실 확인 |
| 2 | 무선 이어폰 | INACTIVE | STATUS_CHANGE | 판매사 계약 종료로 판매 중지 |

### 변경 유형 표준화

`change_type`은 자유롭게 쓰기보다 미리 정한 코드값으로 표준화하는 것이 좋다. 예시는 다음과 같다.

| change_type | 설명 |
| --- | --- |
| CREATE | 신규 등록 |
| PRICE_CHANGE | 가격 변경 |
| STOCK_ADJUST | 재고 조정 |
| STATUS_CHANGE | 상태 변경 |
| INFO_UPDATE | 정보 수정 (이름, 설명 등) |
| PROMOTION | 프로모션 적용 |
| CORRECTION | 오류 정정 |

### change_type과 change_reason을 나누는 이유

변경 사유를 `change_reason` 하나에 자유 텍스트로만 담을 수도 있다. 그런데 별도로 `change_type`을 두는 이유는 **데이터 활용과 검색의 효율성** 때문이다.

**1. 컴퓨터가 이해하기 쉬운 데이터 (표준화된 코드)**

`change_reason`은 사람이 읽기 위한 것이라 표현이 제각각이다. "가격 변경", "할인 적용", "금액 조정" 등 사람마다 다르게 적을 수 있다. 이런 경우 자유 텍스트만으로는 같은 '가격 변경' 작업인지 판별하기 어렵다. 반면 `change_type`은 `PRICE_CHANGE`처럼 코드로 통일되므로 시스템이 정확하게 분류할 수 있다.

**2. 통계와 집계에 유용**

예를 들어 "올해 가격 변경이 총 몇 번 있었나"를 집계할 때, `change_type`이 없으면 자유 텍스트를 뒤져야 한다.

`change_type`이 없을 때(비효율적):

```sql
SELECT * FROM product
WHERE change_reason LIKE '%가격%'
   OR change_reason LIKE '%할인%'
   OR change_reason LIKE '%조정%';
```

이 방식은 검색 조건이 장황하고, 오타가 있거나 다른 표현을 쓴 경우 누락될 위험이 있다.

`change_type`이 있을 때(효율적):

```sql
SELECT * FROM product
WHERE change_type = 'PRICE_CHANGE';
```

한 가지 조건으로 명확하게 조회할 수 있고 결과도 정확하다.

정리하면 `change_reason`은 **사람을 위한 상세 설명**으로, `change_type`은 **시스템을 위한 분류 코드**로 나눠 쓰는 것이 실무적으로 좋은 설계다.

### 장점과 네 가지 질문 점검

변경 사유 컬럼의 장점은 다음과 같다.

1. **변경 사유를 알 수 있다**: 왜 바뀌었는지 알 수 있다.
2. **분류와 통계가 쉽다**: `change_type`으로 유형별 집계가 가능하다.
3. **문제 해결에 도움이 된다**: "이 가격은 할인 이벤트 때문"이라는 근거를 제시할 수 있다.

네 가지 질문 중 세 개가 해결된다.

- 문제 1. 누가 변경했나요? → 해결
- 문제 2. 언제 변경했나요? → 해결
- 문제 3. 왜 변경했나요? → **해결**
- 문제 4. 이전 값은 무엇이었나요? → 미해결

이제 문제 4만 남는다. 문제 4를 다루기 전에, 실무에서 자주 필요한 감사(Audit) 정보를 먼저 살펴본다.

## 변경 추적 컬럼 - 감사(Audit) 컬럼

규모가 큰 시스템은 "누가, 언제, 왜"뿐 아니라 더 상세한 감사 정보가 필요한 경우가 많다. 특히 다음과 같은 질문에 답해야 할 때가 있다.

- "이 변경이 어느 경로에서 일어났나요?" (관리자 화면? API? 배치?)
- "어떤 IP에서 접속해 변경했나요?"

### 감사 정보가 필요한 이유

같은 데이터라도 여러 경로를 통해 변경될 수 있다.

1. **웹 관리자 화면**: 운영자가 직접 수정
2. **API 연동**: 외부 파트너사를 통한 수정
3. **배치 시스템**: 정기 배치 작업에 의한 수정
4. **모바일 앱**: 앱을 통한 수정

문제가 생겼을 때 "어디서 발생했는지"를 알아야 원인을 좁히고 빠르게 대응할 수 있다.

### 감사 컬럼 추가

기존 컬럼에 감사 컬럼을 추가한다.

```sql
DROP TABLE IF EXISTS product;
CREATE TABLE product (
    product_id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(200) NOT NULL,
    price INT NOT NULL,
    stock_quantity INT NOT NULL DEFAULT 0,
    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    -- 기본 변경 추적 컬럼
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    created_by VARCHAR(100) NOT NULL,
    updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    updated_by VARCHAR(100) NOT NULL,
    -- 변경 사유 컬럼
    change_type VARCHAR(50),
    change_reason VARCHAR(500),
    -- 감사(Audit) 컬럼
    source_system VARCHAR(50),
    client_ip VARCHAR(50)
);
```

추가된 감사 컬럼의 의미는 다음과 같다.

| 컬럼 | 의미 |
| --- | --- |
| `source_system` | 변경이 발생한 시스템 (예: WEB_ADMIN, API, BATCH, MOBILE 등) |
| `client_ip` | 요청이 발생한 클라이언트 IP 주소 |

### 데이터 등록 - 웹 관리자 화면

관리자가 웹 관리자 화면에서 상품을 등록하는 경우다.

```sql
INSERT INTO product (
    name, price, stock_quantity, status,
    created_by, updated_by,
    change_type, change_reason,
    source_system, client_ip
) VALUES (
    '스마트폰 케이스', 15000, 100, 'ACTIVE',
    'admin_kim', 'admin_kim',
    'CREATE', '신규 상품 등록',
    'WEB_ADMIN', '192.168.1.3'
);
```

```sql
SELECT product_id, name, price, source_system, client_ip
FROM product;
```

**[실행 결과]**

| product_id | name | price | source_system | client_ip |
| --- | --- | --- | --- | --- |
| 1 | 스마트폰 케이스 | 15000 | WEB_ADMIN | 192.168.1.3 |

`source_system`이 `WEB_ADMIN`으로, 웹 관리자 화면에서 등록됐음을 확인할 수 있다. `client_ip`를 통해 어떤 IP에서 접속한 요청인지도 확인할 수 있다.

### 데이터 수정 - API 연동

외부 파트너사가 API를 통해 가격을 수정하는 경우다.

```sql
UPDATE product
SET price = 13000,
    updated_by = 'partner_api',
    change_type = 'PRICE_CHANGE',
    change_reason = '파트너 프로모션 가격 기획',
    source_system = 'API',
    client_ip = '123.123.123.123'
WHERE product_id = 1;
```

```sql
SELECT product_id, name, price, source_system, client_ip, change_reason
FROM product
WHERE product_id = 1;
```

**[실행 결과]**

| product_id | name | price | source_system | client_ip | change_reason |
| --- | --- | --- | --- | --- | --- |
| 1 | 스마트폰 케이스 | 13000 | API | 123.123.123.123 | 파트너 프로모션 가격 기획 |

`source_system`이 `API`로, API 연동을 통해 수정됐음을 확인할 수 있다. `client_ip`로 어느 곳에서 온 요청인지도 확인할 수 있다.

### 데이터 수정 - 배치 작업

야간 배치가 재고를 조정하는 경우다.

```sql
UPDATE product
SET stock_quantity = 85,
    updated_by = 'batch_system',
    change_type = 'STOCK_ADJUST',
    change_reason = '야간 재고 동기화 배치',
    source_system = 'BATCH',
    client_ip = '10.0.0.5'
WHERE product_id = 1;
```

```sql
SELECT product_id, name, stock_quantity, source_system, client_ip
FROM product
WHERE product_id = 1;
```

**[실행 결과]**

| product_id | name | stock_quantity | source_system | client_ip |
| --- | --- | --- | --- | --- |
| 1 | 스마트폰 케이스 | 85 | BATCH | 10.0.0.5 |

### source_system 표준화

`source_system`도 시스템 경로를 코드값으로 표준화하는 것이 좋다. 예시는 다음과 같다.

| source_system | 설명 |
| --- | --- |
| WEB_ADMIN | 웹 관리자 화면 |
| WEB_USER | 웹 사용자 화면 |
| MOBILE_APP | 모바일 앱 |
| API | 외부 API 연동 |
| BATCH | 배치 시스템 |
| MIGRATION | 데이터 마이그레이션 |
| MANUAL | 수동 DB 작업 (DBA 직접 수정) |

### 장점

감사 컬럼의 장점은 다음과 같다.

1. **문제 발생 시스템 특정**: "이 변경은 어느 경로에서 일어났다"를 확실히 알 수 있다.
2. **보안과 감사 대응**: 어떤 IP에서 접속했는지 추적할 수 있다.

### 네 가지 질문 점검

감사 컬럼까지 더하면 질문 1, 2, 3이 더 확실하게 강화된다.

- 문제 1. 누가 변경했나요? → **해결**
- 문제 2. 언제 변경했나요? → **해결**
- 문제 3. 왜 변경했나요? → **해결 (강화)**
- 문제 4. 이전 값은 무엇이었나요? → 미해결

여기까지 학습한 감사 컬럼은 문제 1, 2, 3을 더 튼튼하게 뒷받침한다. 하지만 여전히 문제 4는 남는다.

### 과거 이력과 문제 4

이제 남은 것은 문제 4, "이전 값은 무엇이었나"이다. 변경 추적 컬럼만으로는 다음 세 가지를 해결할 수 없다.

1. **마지막 변경 흔적만 남는다**: 이전 변경 이력은 덮어써진다.
2. **이전 값을 알 수 없다**: 변경 전 가격, 변경 전 재고 등을 알 수 없다.
3. **변경 횟수를 알 수 없다**: 몇 번 바뀌었는지 파악할 수 없다.

지금까지의 방식은 모두 "현재 행에 최신 변경 흔적을 남기는 방식"이다. 이 한계 때문에 과거 값을 온전히 보존하려면 별도의 이력 테이블이 필요하다. 이는 다음 문서(이력 관리 시리즈2)에서 다룬다.

### 추적용 요청 ID (Correlation ID)

한 가지 더, 실무에서 유용한 추적 컬럼으로 요청 ID(`request_id`, Correlation ID)가 있다.

하나의 사용자 요청이 여러 시스템에 걸쳐 처리될 때, 동일한 `request_id`를 부여하면 전체 흐름을 하나로 묶어 추적할 수 있다.

예를 들어 고객이 "주문한 재고가 안 맞는다"는 문의를 하면, 각 시스템에 남은 `request_id`로 주문 시스템, 결제 시스템, 재고 시스템의 변경을 하나의 요청으로 추적할 수 있다.

## 정리

### 변경 추적 컬럼 리뷰

지금까지 살펴본 변경 추적 컬럼을 정리하면 다음과 같다.

#### 1. 기본 변경 추적 컬럼

가장 기본이 되는 4개 컬럼이다.

| 컬럼 | 타입 | 설명 |
| --- | --- | --- |
| `created_at` | DATETIME | 최초 등록 시각 |
| `created_by` | VARCHAR(100) | 최초 등록자 |
| `updated_at` | DATETIME | 마지막 수정 시각 |
| `updated_by` | VARCHAR(100) | 마지막 수정자 |

**용도**: 거의 모든 테이블에 기본으로 두는 것이 좋다. "최소한의 추적"이 필요한 모든 곳에 적용할 수 있다.

#### 2. 변경 사유 컬럼

변경의 이유를 남기는 컬럼이다.

| 컬럼 | 타입 | 설명 |
| --- | --- | --- |
| `change_type` | VARCHAR(50) | 변경 유형 (PRICE_CHANGE, STATUS_CHANGE 등) |
| `change_reason` | VARCHAR(500) | 변경 사유 (자유 텍스트) |

**용도**: 변경 이유가 중요한 데이터에 사용한다. 가격, 상태, 등급 등 왜 바뀌었는지가 중요한 경우에 적합하다.

#### 3. 감사(Audit) 컬럼

상세한 감사 추적을 위한 컬럼이다.

| 컬럼 | 타입 | 설명 |
| --- | --- | --- |
| `source_system` | VARCHAR(50) | 변경이 발생한 시스템 |
| `client_ip` | VARCHAR(50) | 클라이언트 IP |

**용도**: 여러 경로에서 데이터가 변경되는 시스템, 보안과 감사 추적이 필요한 경우에 사용한다.

### 전체 컬럼 예시

세 종류의 변경 추적 컬럼을 모두 적용한 전체 예시는 다음과 같다.

```sql
CREATE TABLE product (
    -- 기본 데이터 컬럼
    product_id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(200) NOT NULL,
    price INT NOT NULL,
    stock_quantity INT NOT NULL DEFAULT 0,
    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',

    -- 기본 변경 추적 컬럼
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    created_by VARCHAR(100) NOT NULL,
    updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    updated_by VARCHAR(100) NOT NULL,

    -- 변경 사유 컬럼
    change_type VARCHAR(50),
    change_reason VARCHAR(500),

    -- 감사(Audit) 컬럼
    source_system VARCHAR(50),
    client_ip VARCHAR(50)
);
```

### 한계점 정리

변경 추적 컬럼으로 해결할 수 있는 것과 없는 것을 정리하면 다음과 같다.

| 질문 | 해결 가능 여부 |
| --- | --- |
| 누가 변경했나요? | O |
| 언제 변경했나요? | O |
| 왜 변경했나요? | O (`change_reason`이 있는 경우) |
| 어디서 변경했나요? | O (`source_system`이 있는 경우) |
| 이전 값은 무엇이었나요? | X |
| 특정 시점의 값은 무엇이었나요? | X |
| 총 몇 번 변경됐나요? | X |
| 모든 변경 이력을 볼 수 있나요? | X |

결론적으로 "이전 값"과 "전체 변경 이력"을 알기 위해서는 다른 방식이 필요하다. 다음 문서에서 이력 테이블을 활용한 방식을 살펴본다.

### 내용 요약

**이력 관리가 필요한 이유**

데이터는 계속 변한다. '데이터 변경 이력'은 그 과정을 기록하는 것이다. 서비스 운영 중 "과거 가격 확인", "상태 변경 시점", "정정 주체" 같은 질문에 답할 수 있어야 한다. 이력이 없으면 고객 응대, 감사 대응, 데이터 오류 원인 추적이 어렵다. 가격 인상, 상태 변경, 사용자 감사 등에서 이력 관리로 풀어야 하는 실무 문제가 흔히 발생한다.

**기본 개념: `UPDATE`의 함정**

기본 테이블에서 `UPDATE`를 하면 이전 값이 덮어써져 과거 데이터가 사라진다. "누가, 언제, 왜, 이전 값은 무엇인지"라는 네 가지 질문에 대비해야 한다. 특정 시점의 값(예: 어제 시점 가격)을 조회하려 할 때 이 한계가 드러난다.

**변경 추적 컬럼 - 기본**

`created_at`(생성일), `created_by`(생성자), `updated_at`(수정일), `updated_by`(수정자) 컬럼을 둔다. 등록과 수정 시점에 함께 기록해 "누가, 언제"를 알 수 있다. `updated_at`은 `ON UPDATE`로 자동 갱신되지만 `updated_by`는 애플리케이션이 채워야 한다. 이 방식은 이력이 크게 중요하지 않은 단순 데이터에 적합하다.

**변경 추적 컬럼 - 변경 사유**

"누가, 언제"를 넘어 "왜" 변경했는지를 알기 위해 `change_type`(변경 유형)과 `change_reason`(변경 사유) 컬럼을 둔다. `change_type`은 시스템 분류와 통계 집계를 위해 코드로 표준화(예: `PRICE_CHANGE`, `STOCK_ADJUST`)하고, `change_reason`은 사람이 읽을 자연어 설명으로 쓴다.

**변경 추적 컬럼 - 감사(Audit) 컬럼**

변경이 일어난 경로와 환경을 추적하기 위해 `source_system`(발생 시스템)과 `client_ip`(요청 IP)를 둔다. `source_system`은 웹 관리자, API, 배치, 앱 등 변경 요청의 출처를 특정하고, `client_ip`는 보안과 감사 추적에 쓴다. 여러 시스템에 걸친 요청을 묶기 위해 `request_id`(Correlation ID)를 함께 두면 유용하다.

**한계와 다음 단계**

변경 추적 컬럼은 "누가, 언제, 왜, 어디서"에는 답할 수 있지만, "이전 값", "특정 시점의 값", "전체 변경 이력"에는 답할 수 없다. 이 문제는 다음 문서(이력 관리 시리즈2)의 이력 테이블에서 해결한다.