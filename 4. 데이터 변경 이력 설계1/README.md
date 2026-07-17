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