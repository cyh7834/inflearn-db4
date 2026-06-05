# 2. 공통 코드 설계

## 핵심 요약

공통 코드(Common Code)는 주문 상태, 회원 등급, 결제 수단처럼 여러 곳에서 반복적으로 쓰이는 코드값과 표시 이름을 중앙에서 관리하는 설계 방식이다. 데이터베이스에는 안정적인 코드값을 저장하고, 화면에 보여줄 이름이나 운영 속성은 공통 코드 테이블에서 관리한다.

이 설계의 핵심은 다음과 같다.

- 문자열 상태값을 직접 저장하면 표기 차이, 오타, 변경 비용, 다국어 처리 문제가 생긴다.
- 코드값과 표시 이름을 분리하면 데이터 일관성과 운영 유연성을 얻을 수 있다.
- 실무에서는 단순 코드 테이블보다 `공통 코드 그룹 + 상세 코드` 구조를 많이 쓴다.
- 공통 코드 조인이 많아지면 SQL이 복잡해지므로 애플리케이션 매핑과 캐싱 전략을 함께 고려해야 한다.
- 비즈니스 로직에서 분기 기준으로 쓰는 값은 애플리케이션 `ENUM`을 적극 고려하고, 표시 이름이나 속성 변경이 필요한 경우에는 공통 코드와 결합하는 하이브리드 전략을 사용한다.

## 공통 코드가 필요한 이유

### 하드코딩된 상태값의 문제

주문 상태를 `주문완료`, `배송중`, `주문취소` 같은 문자열로 직접 저장하면 처음에는 단순해 보이지만 다음 문제가 생긴다.

- 데이터 불일치: `주문완료`, `주문 완료`, `ORDER_COMPLETE`처럼 같은 의미가 여러 값으로 저장될 수 있다.
- 오타 버그: `배송중`을 `배송증`으로 입력해도 DB는 오타인지 알 수 없다.
- 변경 어려움: 표시 문구가 바뀌면 기존 데이터를 대량 `UPDATE`해야 한다.
- 표시 이름과 코드값 혼재: 한글 표시 이름을 DB에 직접 저장하면 다국어 처리나 화면별 표시 변경이 복잡해진다.

### 코드값과 표시 이름 분리

해결책은 DB에는 안정적인 코드값만 저장하고, 사용자에게 보여줄 이름은 별도로 관리하는 것이다.

| 코드값 | 한글 이름 | 영문 이름 |
| --- | --- | --- |
| `ORDER` | 주문접수 | Order Received |
| `PAID` | 결제완료 | Payment Complete |
| `SHIPPING` | 배송중 | Shipping |
| `DELIVERED` | 배송완료 | Delivered |
| `CANCEL` | 주문취소 | Cancelled |

이 매핑 정보를 관리하는 테이블이 공통 코드 테이블이다.

## 공통 코드 테이블 설계

### 단순 설계의 한계

가장 단순한 설계는 코드와 이름만 두는 방식이다.

```sql
CREATE TABLE common_code (
  code VARCHAR(50) PRIMARY KEY,
  name VARCHAR(100) NOT NULL
);
```

이 방식은 주문 상태만 관리할 때는 괜찮지만, 회원 등급, 결제 상태, 결제 수단이 추가되면 코드들이 한 테이블에 뒤섞인다. 더 큰 문제는 `CANCEL`처럼 여러 도메인에서 같은 코드값을 쓰고 싶을 때 PK 충돌이 발생한다는 점이다.

### 그룹화 설계

실무에서는 코드를 종류별로 분리하기 위해 그룹 코드 테이블과 상세 코드 테이블로 나눈다.

```sql
CREATE TABLE common_code_group (
  group_code VARCHAR(50) PRIMARY KEY,
  group_name VARCHAR(100) NOT NULL,
  description VARCHAR(500),
  use_yn CHAR(1) NOT NULL DEFAULT 'Y',
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

CREATE TABLE common_code_detail (
  group_code VARCHAR(50) NOT NULL,
  code VARCHAR(50) NOT NULL,
  name VARCHAR(100) NOT NULL,
  description VARCHAR(500),
  sort_order INT NOT NULL DEFAULT 0,
  use_yn CHAR(1) NOT NULL DEFAULT 'Y',
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (group_code, code),
  FOREIGN KEY (group_code) REFERENCES common_code_group(group_code)
);
```

주요 컬럼의 역할은 다음과 같다.

| 컬럼 | 의미 |
| --- | --- |
| `group_code` | 코드 그룹 식별자. 예: `ORDER_STATUS`, `MEMBER_GRADE` |
| `code` | 실제 코드값. 예: `ORDER`, `VIP`, `CARD` |
| `name` | 화면 표시 이름 |
| `description` | 코드 또는 그룹 설명 |
| `sort_order` | 드롭다운이나 목록 표시 순서 |
| `use_yn` | 사용 여부. 기존 데이터는 유지하면서 신규 선택지만 막을 때 사용 |
| `created_at`, `updated_at` | 생성 및 수정 시각 |

복합 PK를 `(group_code, code)`로 잡으면 서로 다른 그룹에서 같은 코드값을 사용할 수 있다. 예를 들어 `ORDER_STATUS / CANCEL`과 `PAYMENT_STATUS / CANCEL`이 동시에 존재할 수 있다.

### 적용 예시

주문 테이블에는 표시 이름이 아니라 코드값만 저장한다.

```sql
CREATE TABLE orders (
  order_id BIGINT PRIMARY KEY AUTO_INCREMENT,
  member_id BIGINT NOT NULL,
  order_status VARCHAR(20) NOT NULL DEFAULT 'ORDER',
  total_amount INT NOT NULL,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

화면에서 주문 상태 이름이 필요하면 공통 코드 상세 테이블과 조인한다.

```sql
SELECT
  o.order_id,
  o.order_status,
  c.name AS order_status_name,
  o.total_amount
FROM orders o
JOIN common_code_detail c
  ON c.group_code = 'ORDER_STATUS'
 AND o.order_status = c.code;
```

드롭다운 목록은 특정 그룹만 조회한다.

```sql
SELECT code, name
FROM common_code_detail
WHERE group_code = 'ORDER_STATUS'
  AND use_yn = 'Y'
ORDER BY sort_order;
```

## 추가 속성 관리

공통 코드에는 표시 이름 외에도 할인율, 수수료율, 무료배송 여부 같은 속성이 붙을 수 있다.

### 방법 1. 범용 속성 컬럼 추가

속성이 적고 비교적 고정적이면 `attr1`, `attr2`, `attr3` 같은 범용 컬럼을 추가할 수 있다.

```sql
ALTER TABLE common_code_detail
ADD COLUMN attr1 VARCHAR(100),
ADD COLUMN attr2 VARCHAR(100),
ADD COLUMN attr3 VARCHAR(100);
```

단점은 `attr1`이 어떤 의미인지 그룹마다 달라질 수 있다는 점이다. 이를 보완하려면 그룹 테이블에 속성 이름을 함께 둔다.

```sql
ALTER TABLE common_code_group
ADD COLUMN attr1_name VARCHAR(50),
ADD COLUMN attr2_name VARCHAR(50),
ADD COLUMN attr3_name VARCHAR(50);
```

예를 들어 `MEMBER_GRADE`의 `attr1`은 할인율, `PAYMENT_METHOD`의 `attr1`은 수수료율로 해석할 수 있다.

### 방법 2. EAV 방식

속성 수가 많고 그룹마다 필요한 속성이 크게 다르면 EAV(Entity-Attribute-Value) 방식을 고려할 수 있다. 다만 EAV는 조회 성능이 떨어지고, 조인이나 피벗 처리가 복잡해질 수 있다.

실용적인 기준은 다음과 같다.

| 상황 | 추천 |
| --- | --- |
| 속성이 2~3개 정도로 고정 | 범용 속성 컬럼 |
| 속성이 많고 자주 변함 | EAV 고려 |
| 프로젝트 규모가 작거나 단순함 | 범용 속성 컬럼 우선 |

## 공통 코드의 단점

공통 코드는 데이터 일관성을 높이지만, 조회 로직에는 새로운 비용을 만든다.

### SQL 복잡도 증가

주문 상태 이름만 필요하면 공통 코드 조인 1개로 끝난다. 하지만 회원 등급, 결제 수단, 결제 상태까지 함께 보여줘야 하면 공통 코드 테이블을 여러 번 조인해야 한다.

결과적으로 비즈니스 조인보다 코드 이름을 가져오기 위한 조인이 더 많아지는 상황이 생긴다. 문서에서는 이를 조인 지옥(Join Hell)으로 설명한다.

### 코드 이름 조회 로직 중복

관리자 주문 목록, 사용자 마이페이지, 배송 관리 화면 등 여러 SQL에서 같은 공통 코드 조인이 반복된다. 공통 코드 테이블 구조가 바뀌면 여러 쿼리를 함께 수정해야 한다.

### 불필요한 데이터 전송

어떤 화면은 코드값만 필요하고, 어떤 화면은 코드 이름까지 필요하다. 범용 SQL 하나로 처리하면 필요 없는 이름 컬럼까지 조회하고, SQL을 나누면 비슷한 쿼리가 중복된다.

## 해결 방안 1. 애플리케이션 매핑

SQL에서는 코드값만 조회하고, 애플리케이션에서 공통 코드 목록을 별도로 가져와 `Map` 형태로 변환한다.

```java
String getCodeName(String groupCode, String code);

order.statusName = code.getCodeName("ORDER_STATUS", order.status);
order.gradeName = code.getCodeName("MEMBER_GRADE", order.grade);
```

장점은 SQL이 단순해지고 코드 이름 변환 로직을 중앙화할 수 있다는 점이다. 단점은 요청마다 공통 코드를 조회하면 추가 쿼리와 네트워크 호출이 생긴다는 점이다. 각 행마다 코드를 개별 조회하면 N+1 문제가 발생할 수 있다.

## 해결 방안 2. 캐싱

공통 코드는 캐싱에 매우 적합한 데이터다.

- 데이터 양이 작다. 보통 수백 건 수준이다.
- 변경이 드물다. 일주일에 몇 번 바뀌면 많은 편이다.
- 조회가 빈번하다. 거의 모든 화면에서 사용된다.

### 로컬 캐시

애플리케이션 시작 시 공통 코드를 모두 메모리에 올려두고, 이후에는 DB 조회 없이 메모리에서 이름을 찾는다.

```java
class CommonCodeCache {
  private Map<String, Map<String, String>> cache;

  String getName(String groupCode, String code) {
    return cache.get(groupCode).get(code);
  }
}
```

공통 코드 수백 건은 메모리 사용량이 매우 작기 때문에 부담이 거의 없다.

### 캐시 동기화 문제

로컬 캐시는 빠르지만 DB 값이 변경되어도 각 서버의 캐시가 즉시 바뀌지 않는다. 특히 서버가 여러 대라면 어떤 서버는 새 이름을 보여주고, 어떤 서버는 예전 이름을 보여줄 수 있다.

해결책으로는 다음이 있다.

| 방식 | 장점 | 단점 |
| --- | --- | --- |
| 관리자 페이지에서 캐시 갱신 | 직관적 | 다중 서버 동기화가 어렵다 |
| 모든 서버에 갱신 요청 | 서버별 캐시 갱신 가능 | 서버 목록 관리와 실패 처리가 복잡하다 |
| Redis 중앙 캐시 | 동기화가 쉽다 | Redis 조회도 네트워크 호출이다 |
| 로컬 캐시 + TTL | 빠르고 자동 동기화 가능 | TTL 기간만큼 반영 지연이 있다 |

### 권장 전략: 로컬 캐시 + TTL

문서의 권장 방향은 로컬 메모리 캐시에 TTL(Time To Live)을 두는 방식이다.

```java
class CommonCodeCache {
  private Map<String, Map<String, String>> cache;
  private DateTime lastLoadTime;
  private int ttlSeconds = 60;

  String getName(String groupCode, String code) {
    if (isExpired()) {
      refresh();
    }
    return cache.get(groupCode).get(code);
  }
}
```

일반적으로 60초 TTL이면 충분하다. 빠른 반영이 필요하면 10~30초, 변경이 거의 없으면 300초 이상도 가능하다. 캐시는 직접 구현하기보다 언어나 프레임워크에서 검증된 캐시 라이브러리를 사용하는 것이 좋다.

## 공통 코드 vs 애플리케이션 ENUM

공통 코드와 ENUM은 각각 다른 장단점이 있다.

| 구분 | 공통 코드 테이블 | 애플리케이션 ENUM |
| --- | --- | --- |
| 주요 용도 | 목록 표시, 운영 설정 | 비즈니스 로직 분기 |
| 타입 안전성 | 낮음. 문자열 오타 위험 | 높음. 컴파일 타임 검증 |
| 변경 유연성 | 높음. DB 수정으로 반영 | 낮음. 코드 수정과 배포 필요 |
| 자동 완성 | 지원 어려움 | IDE 지원 |
| 관리 주체 | 운영자, DBA, 개발자 | 개발자 |

### 공통 코드만 쓰기 좋은 경우

비즈니스 로직에 관여하지 않고 화면 표시나 선택 목록 제공이 목적이면 공통 코드 테이블만 사용한다.

예시는 다음과 같다.

- 은행 코드
- 지역 코드
- 가입 경로
- 취미 목록
- 단순 드롭다운 목록

이 경우 운영자가 항목이나 표시 이름을 바꿀 수 있는 유연성이 더 중요하다.

### ENUM을 쓰기 좋은 경우

코드값에 따라 `if`, `switch` 같은 비즈니스 분기가 일어나면 ENUM을 사용한다.

문자열을 직접 쓰면 다음 문제가 생긴다.

- 오타가 컴파일 단계에서 잡히지 않는다.
- 코드 변경 시 문자열 검색에 의존해야 한다.
- IDE 자동 완성을 받을 수 없다.
- 아무 문자열이나 들어갈 수 있어 타입 안전성이 없다.

ENUM을 사용하면 오타를 컴파일 타임에 잡고, 리팩토링과 자동 완성의 도움을 받을 수 있다.

```java
public enum OrderStatus {
  ORDER,
  PAID,
  SHIPPING,
  DELIVERED,
  CANCEL
}
```

DB에는 보통 ENUM 이름을 문자열로 저장한다.

```java
String statusCode = order.status.name();          // ENUM -> 문자열
OrderStatus status = OrderStatus.valueOf(code);   // 문자열 -> ENUM
```

### ENUM만 쓸 때의 한계

ENUM에 표시 이름이나 할인율 같은 속성을 함께 넣으면 운영 변경 때마다 애플리케이션을 다시 빌드하고 배포해야 한다. 단순한 표시 이름 변경이나 VIP 할인율 변경도 배포 대상이 되므로 운영 유연성이 떨어진다.

## 하이브리드 전략

하이브리드 전략은 ENUM과 공통 코드 테이블을 함께 사용한다. 코드값은 두 곳에 중복으로 유지하되 역할을 나눈다.

| 역할 | ENUM | 공통 코드 테이블 |
| --- | --- | --- |
| 코드값 정의 | O | O |
| 비즈니스 로직 | O | X |
| 타입 안전성 | O | X |
| 표시 이름 | X | O |
| 추가 속성 | X | O |
| 운영 중 변경 | X | O |

핵심은 다음과 같다.

- ENUM은 코드값 정의와 비즈니스 로직의 안전성을 담당한다.
- 공통 코드 테이블은 표시 이름, 다국어 이름, 할인율, 수수료율 같은 운영 속성을 담당한다.
- 화면 표시 시에는 ENUM의 `name()`으로 코드값을 얻고, 공통 코드 캐시에서 이름을 조회한다.

```java
String getStatusDisplayName(OrderStatus status) {
  return CommonCodeCache.getName("ORDER_STATUS", status.name());
}
```