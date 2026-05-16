DBA 인턴십 중 실제로 발생한 쿼리 성능 문제와 해결 과정을 정리했습니다.  
실행계획(EXPLAIN PLAN) 분석을 기반으로 합니다.

---

## Case 01. 인덱스 컬럼 가공으로 인한 Index Full Scan → Index Range Scan

### 상황
일별 주식 시세 테이블(`stock_history`) 조회 쿼리가 인덱스가 존재함에도 응답 속도가 매우 느린 문제.

### 원인 분석
실행계획 확인 결과 `TO_CHAR(trade_date, 'YYYYMMDD')` 함수 가공으로 인해  
옵티마이저가 인덱스 트리 구조와 검색값 형식이 달라 인덱스를 사용하지 못하는 **Index Suppression** 발생.

### Before (문제 쿼리)

```sql
-- [AS-IS] 인덱스 사용 불가 — 컬럼 가공으로 Index Suppression 발생
SELECT *
FROM   stock_history
WHERE  TO_CHAR(trade_date, 'YYYYMMDD') = '20260315';
```

```
실행계획 (Before)
| Id | Operation          | Name         | Rows    | Cost |
|----|--------------------|--------------|---------|------|
|  0 | SELECT STATEMENT   |              |         | 8,450|
|  1 |   TABLE ACCESS FULL| STOCK_HISTORY | 980,000 | 8,450| ← Full Scan
```

### After (튜닝 쿼리)

```sql
-- [TO-BE] 컬럼은 그대로, 비교 대상(상수)을 가공 → SARGable Query
SELECT /*+ INDEX(stock_history idx_trade_date) */ *
FROM   stock_history
WHERE  trade_date >= TO_DATE('20260315', 'YYYYMMDD')
  AND  trade_date <  TO_DATE('20260316', 'YYYYMMDD');
```

```
실행계획 (After)
| Id | Operation                  | Name           | Rows | Cost |
|----|----------------------------|----------------|------|------|
|  0 | SELECT STATEMENT           |                |      |  94  |
|  1 |   TABLE ACCESS BY INDEX ROWID | STOCK_HISTORY | 850 |  94  |
|  2 |    INDEX RANGE SCAN        | IDX_TRADE_DATE |  850 |  12  | ← Index 활용
```

| 지표 | Before | After | 개선율 |
|------|--------|-------|--------|
| 실행시간 | 3.5s | 0.1s | **97% 감소** |
| Operation | TABLE ACCESS FULL | INDEX RANGE SCAN | — |

**핵심 원칙 (SARGable)** : WHERE 절 좌변(컬럼)에는 함수를 절대 적용하지 않는다.  
`TO_CHAR`, `TRUNC`, `SUBSTR` 등 모두 동일하게 적용됨.

---

## Case 02. EXISTS vs IN 성능 분석 및 적용

### 상황
대용량 메인 테이블에서 서브쿼리 테이블의 존재 여부를 체크하는 쿼리에서 타임아웃 발생.

### 원인 분석

| 구분 | 동작 방식 | 특징 |
|------|-----------|------|
| `IN` | 서브쿼리 결과를 **전부** 추출 후 비교 | 실제 집합 생성, 메모리 사용 |
| `EXISTS` | 조건 만족 행 **1건 발견 즉시** 검색 종료 | Semi-Join, Short-circuit |

### Before

```sql
SELECT txn_id, txn_amt
FROM   transactions T
WHERE  account_id IN (
           SELECT account_id
           FROM   vip_accounts
       );
```

### After

```sql
SELECT txn_id, txn_amt
FROM   transactions T
WHERE  EXISTS (
           SELECT 1
           FROM   vip_accounts V
           WHERE  V.account_id = T.account_id  -- ← 서브쿼리 컬럼에 인덱스 존재
       );
```

**튜닝 결과** : 서브쿼리 컬럼(`account_id`)에 인덱스가 있을 때 EXISTS 효과 극대화.  
메인 테이블 대용량 + 서브쿼리 소규모일수록 EXISTS가 유리함을 실무에서 확인.

---

## Case 03. 힌트(Hint)를 이용한 옵티마이저 제어

### 상황
데이터 통계 정보가 최신화되지 않아 옵티마이저가 잘못된 실행 경로를 선택하는 문제.

### 해결

```sql
-- 특정 인덱스 강제 사용
SELECT /*+ INDEX(stock_history idx_trade_date) */ *
FROM   stock_history
WHERE  trade_date >= TO_DATE('20260301', 'YYYYMMDD');

-- 조인 순서 결정
SELECT /*+ LEADING(A B) USE_NL(B) */
       A.account_id, B.txn_amt
FROM   accounts    A
JOIN   transactions B ON B.account_id = A.account_id;
```

| 힌트 | 용도 |
|------|------|
| `/*+ INDEX(table idx) */` | 특정 인덱스 사용 강제 |
| `/*+ LEADING(A B) */` | 조인 순서 결정 (A → B) |
| `/*+ USE_NL(B) */` | Nested Loop 조인 유도 |
| `/*+ FULL(table) */` | Full Table Scan 강제 |

> ⚠️ 힌트는 통계 정보 문제나 특수 케이스에만 사용. 남용 시 오히려 성능 저하 가능.  
> 근본 해결책은 `DBMS_STATS`를 통한 통계 정보 최신화.

---

## 튜닝 체크리스트

```
□ WHERE 좌변 컬럼에 함수 적용 여부 확인 (TO_CHAR, TRUNC, SUBSTR, NVL 등)
□ 암묵적 형변환 발생 여부 확인 (숫자 ↔ 문자)
□ 중첩 IN 절 → EXISTS / JOIN 전환 검토
□ 파티션 테이블 — Partition Pruning 작동 여부 확인
□ LIKE '%값' 패턴 — 선행 와일드카드 시 Full Scan 불가피
□ 통계 정보 최신화 여부 확인 (DBMS_STATS)
□ 힌트 사용 전 통계 수집 먼저 시도
```
