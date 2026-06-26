# 3. 계층 구조 설계

## 핵심 요약

계층 구조는 쇼핑몰 카테고리, 조직도, 댓글과 대댓글, 파일 시스템, 메뉴 구조처럼 부모-자식 관계가 반복되는 트리 형태의 데이터이다. 관계형 데이터베이스는 기본적으로 행과 열로 이루어진 평면 구조이므로, 트리 구조를 어떻게 저장하고 조회할지 별도의 설계 전략이 필요하다.

이 문서의 핵심은 다음과 같다.

- 가장 직관적인 방식은 각 행이 자신의 부모를 참조하는 인접 리스트 모델이다.
- 인접 리스트 모델은 저장과 수정이 단순하지만, 모든 자손이나 모든 조상을 조회할 때 어려움이 있다.
- 계층 깊이가 가변적이면 반복 호출, 고정 JOIN, `UNION ALL` 방식은 한계가 있다.
- MySQL 8.0 이상에서는 재귀 CTE(`WITH RECURSIVE`)로 가변 깊이 계층을 한 번의 쿼리로 조회할 수 있다.
- 조회 성능이 극도로 중요하고 구조 변경이 드문 경우에는 모든 조상-자손 관계를 미리 저장하는 폐쇄 테이블 모델을 고려할 수 있다.
- 대부분의 실무 상황은 `인접 리스트 모델 + 재귀 CTE` 조합으로 충분하다.

## 계층 구조가 필요한 이유

대표적인 계층 구조 예시는 다음과 같다.

```text
전자제품
├── 컴퓨터
│   ├── 노트북
│   ├── 데스크탑
│   └── 태블릿
├── 스마트폰
│   ├── 애플
│   └── 삼성
└── 가전제품
    ├── TV
    └── 냉장고
```

실무에서 자주 만나는 계층 데이터는 다음과 같다.

- 쇼핑몰 상품 카테고리
- 회사 조직도
- 게시판 댓글과 대댓글
- 파일 시스템 폴더
- 사이트 메뉴 구조

관계형 DB에 이런 트리를 저장하려면 부모-자식 관계를 표현하고, 특정 노드의 직속 자식, 모든 자손, 모든 조상, 전체 경로 등을 조회할 수 있어야 한다.

## 인접 리스트 모델

인접 리스트 모델(Adjacency List Model)은 각 노드가 자신의 부모 노드를 참조하는 방식이다.

핵심 아이디어는 단순하다.

- 각 행은 `parent_id`를 가진다.
- `parent_id`는 같은 테이블의 PK를 참조한다.
- 최상위 루트 노드는 부모가 없으므로 `parent_id`가 `NULL`이다.

### 테이블 설계

```sql
CREATE TABLE category (
  category_id BIGINT NOT NULL AUTO_INCREMENT,
  name VARCHAR(100) NOT NULL,
  parent_id BIGINT NULL,
  PRIMARY KEY (category_id),
  FOREIGN KEY (parent_id) REFERENCES category(category_id)
);
```

컬럼의 의미는 다음과 같다.

| 컬럼 | 의미 |
| --- | --- |
| `category_id` | 카테고리 고유 식별자 |
| `name` | 카테고리 이름 |
| `parent_id` | 부모 카테고리 ID. 같은 테이블의 `category_id`를 참조 |

`parent_id`가 자기 자신의 테이블을 참조하므로 자기 참조(Self-Referencing) 관계라고 부른다.

### 기본 조회

루트 카테고리는 `parent_id IS NULL`로 조회한다.

```sql
SELECT *
FROM category
WHERE parent_id IS NULL;
```

특정 카테고리의 직속 자식은 `parent_id`로 조회한다.

```sql
SELECT *
FROM category
WHERE parent_id = 1;
```

특정 카테고리의 부모는 셀프 조인으로 조회한다.

```sql
SELECT p.*
FROM category c
JOIN category p ON c.parent_id = p.category_id
WHERE c.category_id = 7;
```

### 장점

인접 리스트 모델의 장점은 다음과 같다.

- 구조가 직관적이다. 부모-자식 관계를 외래 키 하나로 표현한다.
- 노드 추가가 간단하다. 새 행을 넣고 `parent_id`만 지정하면 된다.
- 노드 이동이 쉽다. `parent_id`만 바꾸면 된다.
- 저장 공간이 효율적이다. 노드 하나당 행 하나만 저장한다.
- 외래 키 제약조건으로 부모 참조 무결성을 보장할 수 있다.

```sql
-- 새로운 카테고리 추가
INSERT INTO category (name, parent_id)
VALUES ('게이밍노트북', 7);

-- 카테고리 이동
UPDATE category
SET parent_id = 6
WHERE category_id = 7;

-- 자식이 없는 카테고리 삭제
DELETE FROM category
WHERE category_id = 17;
```

### 한계

직속 자식이나 직속 부모는 쉽게 조회할 수 있지만, 특정 카테고리 아래의 모든 자손이나 특정 카테고리 위의 모든 조상을 조회하는 것은 어렵다.

예를 들어 `전자제품`의 모든 하위 카테고리를 조회하려면 `컴퓨터`, `스마트폰`, `가전제품`뿐 아니라 그 아래의 `노트북`, `데스크탑`, `애플`, `삼성`, `TV`, `냉장고`까지 모두 찾아야 한다. 계층의 깊이를 모르면 단순한 `WHERE parent_id = ?`만으로는 해결할 수 없다.

## 계층 구조 조회의 어려움

### 방법 1. 애플리케이션 반복 호출

가장 단순한 방법은 애플리케이션에서 단계별로 DB를 여러 번 호출하는 것이다.

```sql
-- 1단계: 루트 조회
SELECT * FROM category WHERE category_id = 1;

-- 2단계: 루트의 자식 조회
SELECT * FROM category WHERE parent_id = 1;

-- 3단계: 자식들의 자식 조회
SELECT * FROM category WHERE parent_id IN (4, 5, 6);

-- 4단계: 더 이상 자식이 없으면 종료
SELECT * FROM category WHERE parent_id IN (7, 8, 9, 10, 11, 12, 13);
```

구현은 쉽지만 계층 깊이만큼 쿼리가 필요하다. 깊이가 10단계라면 최소 10번의 DB 호출이 필요하고, 네트워크 비용이 커진다.

### 방법 2. JOIN을 사용한 고정 깊이 조회

계층 최대 깊이가 정해져 있다면 여러 번 셀프 조인을 할 수 있다.

```sql
SELECT
  c1.category_id AS level1_id,
  c1.name AS level1_name,
  c2.category_id AS level2_id,
  c2.name AS level2_name,
  c3.category_id AS level3_id,
  c3.name AS level3_name
FROM category c1
LEFT JOIN category c2 ON c2.parent_id = c1.category_id
LEFT JOIN category c3 ON c3.parent_id = c2.category_id
WHERE c1.category_id = 1;
```

한 번의 쿼리로 처리할 수 있지만 깊이가 고정되어 있어야 한다. 깊이가 늘어나면 JOIN도 계속 늘어나고, 결과가 중복 행처럼 펼쳐져 후처리가 필요하다.

### 방법 3. UNION ALL을 사용한 조회

각 레벨별 쿼리를 작성하고 `UNION ALL`로 합칠 수도 있다.

```sql
SELECT category_id, name, parent_id, 1 AS level
FROM category
WHERE category_id = 1

UNION ALL

SELECT category_id, name, parent_id, 2 AS level
FROM category
WHERE parent_id = 1

UNION ALL

SELECT c.category_id, c.name, c.parent_id, 3 AS level
FROM category c
WHERE c.parent_id IN (
  SELECT category_id FROM category WHERE parent_id = 1
);
```

이 방식도 계층 깊이만큼 `UNION ALL`을 추가해야 하므로 가변 깊이에 대응하기 어렵다.

### 한계 정리

| 방법 | 장점 | 단점 |
| --- | --- | --- |
| 애플리케이션 반복 호출 | 구현이 간단 | 네트워크 비용, 성능 저하 |
| 고정 깊이 JOIN | 한 번의 쿼리 | 깊이 고정, JOIN 증가, 결과 중복 |
| `UNION ALL` | 레벨별 분리 가능 | 깊이 고정, 쿼리 복잡 |

실무의 계층 구조는 깊이가 고정되지 않는 경우가 많다. 상품 카테고리는 3단계일 수도 있고 10단계일 수도 있다. 댓글은 대댓글, 대대댓글처럼 깊이를 예측하기 어렵다. 이런 경우 재귀 쿼리가 필요하다.

## CTE와 재귀 쿼리

CTE(Common Table Expression)는 쿼리 안에서 임시로 사용할 수 있는 이름 있는 결과 집합이다. `WITH` 절로 정의하며, 해당 SQL 안에서만 존재하는 임시 뷰처럼 사용할 수 있다.

```sql
WITH top_categories AS (
  SELECT *
  FROM category
  WHERE parent_id IS NULL
)
SELECT *
FROM top_categories;
```

CTE의 장점은 다음과 같다.

- 쿼리 가독성이 좋아진다.
- 같은 결과 집합을 여러 번 참조할 수 있다.
- 재귀 쿼리를 작성할 수 있다.

### 재귀 CTE 기본 구조

재귀 CTE는 자기 자신을 참조하는 CTE이다. `WITH RECURSIVE` 키워드를 사용한다.

```sql
WITH RECURSIVE cte_name AS (
  -- 기본 케이스: 재귀 시작점
  SELECT ...

  UNION ALL

  -- 재귀 케이스: CTE 자신을 참조
  SELECT ...
  FROM cte_name
  WHERE ...
)
SELECT *
FROM cte_name;
```

재귀 CTE는 두 부분으로 나뉜다.

| 구성 | 의미 |
| --- | --- |
| 기본 케이스(Anchor Member) | 재귀의 시작점 |
| 재귀 케이스(Recursive Member) | CTE 자신을 참조하여 반복 실행되는 쿼리 |

실행 흐름은 다음과 같다.

1. 기본 케이스를 실행해 초기 결과를 만든다.
2. 재귀 케이스를 실행해 다음 결과를 만든다.
3. 새 결과가 없을 때까지 반복한다.
4. 지금까지의 모든 결과를 합쳐 반환한다.

MySQL은 8.0부터 재귀 CTE를 지원한다.

## 재귀 CTE 활용

### 모든 자손 조회

`전자제품(category_id = 1)`과 그 아래 모든 하위 카테고리를 조회하는 예시이다.

```sql
WITH RECURSIVE descendants AS (
  -- 기본 케이스: 시작 노드
  SELECT category_id, name, parent_id, 1 AS depth
  FROM category
  WHERE category_id = 1

  UNION ALL

  -- 재귀 케이스: 이전 결과의 자식 조회
  SELECT c.category_id, c.name, c.parent_id, d.depth + 1
  FROM category c
  JOIN descendants d ON c.parent_id = d.category_id
)
SELECT *
FROM descendants;
```

핵심은 재귀 케이스의 `JOIN descendants` 부분이다. 이전 단계에서 찾은 노드들을 부모로 삼아 다시 자식을 찾는다.

자손 조회의 조인 조건은 다음과 같다.

```sql
c.parent_id = d.category_id
```

즉, 자식의 `parent_id`가 현재 노드의 `category_id`와 같은 행을 찾는다.

### 모든 조상 조회

반대로 `노트북(category_id = 7)`의 모든 조상을 조회할 수도 있다.

```sql
WITH RECURSIVE ancestors AS (
  -- 기본 케이스: 시작 노드
  SELECT category_id, name, parent_id, 1 AS depth
  FROM category
  WHERE category_id = 7

  UNION ALL

  -- 재귀 케이스: 부모를 찾아 올라감
  SELECT c.category_id, c.name, c.parent_id, a.depth + 1
  FROM category c
  JOIN ancestors a ON c.category_id = a.parent_id
)
SELECT *
FROM ancestors;
```

조상 조회의 조인 조건은 다음과 같다.

```sql
c.category_id = a.parent_id
```

즉, 현재 노드의 `parent_id`와 같은 `category_id`를 가진 부모를 찾는다.

### 특정 깊이까지만 조회

재귀 CTE에 조건을 추가하면 특정 깊이까지만 조회할 수 있다.

```sql
WITH RECURSIVE descendants AS (
  SELECT category_id, name, parent_id, 1 AS depth
  FROM category
  WHERE category_id = 1

  UNION ALL

  SELECT c.category_id, c.name, c.parent_id, d.depth + 1
  FROM category c
  JOIN descendants d ON c.parent_id = d.category_id
  WHERE d.depth < 2
)
SELECT *
FROM descendants;
```

2단계까지만 조회하고 싶을 때 조건은 `d.depth < 2`이다. 이 조건은 새로 만들어질 자식이 아니라, 현재 부모 역할을 하는 `d`의 깊이를 검사한다.

- `d.depth < 2`: 깊이 1인 부모까지만 자식을 찾는다. 결과는 깊이 2까지 생성된다.
- `d.depth <= 2`: 깊이 2인 부모도 자식을 찾으므로 결과가 깊이 3까지 생성된다.

재귀 쿼리에서 깊이를 제한할 때는 "원하는 깊이보다 작은 부모까지만 재귀한다"는 관점으로 조건을 작성해야 한다.