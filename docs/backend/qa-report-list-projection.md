# 리포트 목록 조회 DTO Projection 적용

> `GET /api/qa/reports` 응답 시간 단축 — 사용 안 하는 `reportJson` / `graph` 컬럼을 DB SELECT 에서 제외.

## 문제

[QaReport.reportJson](../backend/src/main/java/com/autoqa/backend/domain/qa/entity/QaReport.java) 은 `json` 컬럼으로 실행 결과 raw JSON 을 통째로 저장 (운영 환경 기준 50KB ~ 수 MB). Hibernate 기본 동작은 `EAGER` 라 `findAll()` 호출 시 모든 컬럼을 fetch.

### 기존 흐름
```
GET /api/qa/reports
   ↓
QaQueryService.listReports()
   ↓
reportRepository.findAll()         ← SELECT * FROM qa_report
   ↓ (qa_report 100건 × reportJson 50KB = 5MB 를 DB → 백엔드로 전송)
.stream().map(this::toReportListResponse)
   ↓ (toReportListResponse 는 reportJson 사용 X — 다른 필드만 추출)
Jackson 직렬화 → 응답
   ↓ (응답 본문엔 reportJson 없음)
```

**아이러니**: 응답에 안 들어가는 `reportJson` 을 DB 에서 매번 5MB 씩 끌어와 메모리에 올렸다가 버리는 것.

## 해결 — JPQL Projection

[QaReportRepository](../backend/src/main/java/com/autoqa/backend/domain/qa/repository/QaReportRepository.java) 에 SELECT 컬럼을 명시한 쿼리 추가:

```java
@Query("""
    SELECT new com.autoqa.backend.domain.qa.dto.response.ReportListResponse(
        r.id, r.userId, r.pageUrl, r.summary,
        r.totalScenarios, r.successCount, r.failCount,
        r.duration, r.testBrowser, r.createdAt
    )
    FROM QaReport r
    ORDER BY r.createdAt DESC
""")
List<ReportListResponse> findAllAsListResponse();
```

[ReportListResponse](../backend/src/main/java/com/autoqa/backend/domain/qa/dto/response/ReportListResponse.java) 에 `@AllArgsConstructor` 추가 (JPQL projection 생성자용).

[QaQueryService.listReports](../backend/src/main/java/com/autoqa/backend/domain/qa/service/QaQueryService.java) 단순화:
```java
public List<ReportListResponse> listReports() {
    return reportRepository.findAllAsListResponse();
}
```

## 측정 결과

### 환경
- 로컬 docker-compose (PostgreSQL 16, Redis 7)
- 시드 데이터: `qa_report` 100건, `report_json` 컬럼에 50KB 더미 JSON 채움
- k6 25 VU × 2분 sustained
- 부하 스크립트: `loadtest/03-scenarios-read.js`

### 비교

| 지표 | BEFORE (Projection 없음) | AFTER (Projection 적용) | 변화 |
|---|---|---|---|
| `reports-list` p95 | **387.76 ms** | **48.11 ms** | **8.1배 단축** |
| `reports-list` avg | 220.26 ms | 20.59 ms | 10.7배 |
| `reports-list` median | 220 ms | 17 ms | 13배 |
| `reports-list` max | 555 ms | 102 ms | 5.4배 |
| 전체 throughput | 39 req/s | 54 req/s | 1.4배 |
| 완료 iteration | 2573 | 3545 | 1.4배 |
| `scenarios` p95 (대조군) | 32.93 ms | 29.01 ms | 변화 없음 |
| 실패율 | 0% | 0% | — |

### 응답 크기 — 사용자 체감은 동일

| | BEFORE | AFTER |
|---|---|---|
| `data_received` 총량 | 66 MB | 90 MB |
| 요청당 크기 | ~13 KB | ~13 KB |

**응답 본문 크기는 비슷.** 차이는 백엔드 내부 처리 비용에서 발생.

## 효과의 출처

응답 본문이 그대로인데 8배 빨라진 이유:

1. **DB ↔ 백엔드 네트워크 트래픽 감소**
   - BEFORE: 매 요청마다 5 MB raw 데이터 전송 (100건 × 50KB)
   - AFTER: 같은 응답을 ~10 KB 로 fetch
2. **Hibernate 마샬링 비용 제거**
   - 100 개 `QaReport` 엔티티 (5MB String 필드 포함) 객체 생성 → 사용 안 되고 GC
   - Projection 은 바로 DTO 생성, Hibernate 영속 컨텍스트 거치지 않음
3. **메모리 점유 감소**
   - 동시 요청 × 5MB 만큼의 힙 메모리 사용량 절감
   - GC 압박 감소

## 영향도

- **단일 요청**: 응답 시간 220ms → 21ms (avg)
- **동시 요청**: throughput 1.4배 (54 req/s)
- **운영 환경 추가 이득**:
  - RDS ↔ EC2 사이 트래픽 비용 감소
  - JVM heap 사용량 감소
  - DB 디스크 IO 부하 감소 (toast 컬럼 fetch 회피)

## 적용 범위

- ✅ `GET /api/qa/reports` (목록 조회)
- ❌ `GET /api/qa/reports/{reportId}` (상세 조회) — `reportJson` 이 화면에 표시되므로 그대로 fetch 필요

상세 조회는 단건이라 5MB 가져와도 부담 적음. 변경하지 않음.

## 참고 — 검증 절차

```bash
# 1. seed 데이터의 reportJson 을 50KB 더미로 채움
bash loadtest/db-seed-reportjson.sh 50

# 2. AFTER 측정 (현재 코드 = Projection 적용)
cd loadtest
k6 run --summary-export=results/03-after-large.json \
       03-scenarios-read.js \
       | tee results/03-after-large.txt

# 3. BEFORE 측정 — projection 임시 제거 후
git stash                          # 또는 코드 직접 revert
# (백엔드 재시작 필요)
k6 run --summary-export=results/03-before-large.json \
       03-scenarios-read.js \
       | tee results/03-before-large.txt
git stash pop                      # projection 복원
# (백엔드 재시작)

# 4. 비교
grep -E "endpoint:reports-list" results/03-before-large.txt results/03-after-large.txt
```

## 변경 파일

| 파일 | 변경 |
|---|---|
| [QaReportRepository.java](../backend/src/main/java/com/autoqa/backend/domain/qa/repository/QaReportRepository.java) | `findAllAsListResponse()` JPQL projection 쿼리 추가 |
| [ReportListResponse.java](../backend/src/main/java/com/autoqa/backend/domain/qa/dto/response/ReportListResponse.java) | `@AllArgsConstructor` 추가 (projection 생성자) |
| [QaQueryService.java](../backend/src/main/java/com/autoqa/backend/domain/qa/service/QaQueryService.java) | `listReports()` 가 새 메서드 사용, 옛 `toReportListResponse` 메서드 제거 |
