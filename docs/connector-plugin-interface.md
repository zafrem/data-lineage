# 커넥터 플러그인 인터페이스 (모듈 설계 v0.1)

> 전체 구조는 `data-lineage-saas-architecture.md`, 엔티티 스키마는
> `inventory-entity-schema.md`, 온보딩 흐름은 `onboarding-orchestration.md`
> 참고. 이 문서는 메인 문서의 "커넥터/스캔 에이전트"(4.1)를 상세화한다.

## 1. 목적과 범위

소스 타입별로 지원할 구체 타겟/버전을 정하고, 커넥터가 구현해야 하는 공통
계약과 인터페이스(정형/비정형)를 정의한다. **운영 환경에 영향을 주지 않는다는
원칙이 다른 모든 설계보다 우선한다.**

---

## 2. 운영 환경 안전 원칙 (최우선 — 예외 없음)

### 2.1 절대 금지

- 운영(Production) 프라이머리 인스턴스에 대한 전체 스캔
- `COUNT(*)` — 엔진이 이미 들고 있는 근사 통계로 대체 (PostgreSQL
  `pg_class.reltuples`, MySQL `information_schema.TABLES.TABLE_ROWS`,
  Snowflake/BigQuery의 테이블 메타데이터 등)
- `ORDER BY RANDOM()` / `RAND()` — 무작위처럼 보이지만 실제로는 전체 테이블을
  정렬해야 해서 풀 스캔보다 무거울 수 있다
- `SELECT *` (LIMIT 없이)

### 2.2 허용되는 샘플링 방식

- **엔진 네이티브 블록 샘플링**: `TABLESAMPLE SYSTEM(n)` / `BERNOULLI(n)`
  (PostgreSQL) 등 — 블록 단위 확률적 추출로 부하가 낮다
- **PK/인덱스 기반 안정적 LIMIT + 해시 모듈로 보정**: 인덱스 스캔으로 끝나
  안전하지만 앞쪽 편향이 생기므로, `WHERE MOD(hash(pk), 100) = 0` 같은 해시
  모듈로 조건을 병행해 편향을 줄인다
- **카탈로그 통계 우선**: 가능하면 실제 데이터를 안 읽고 시스템 카탈로그의
  히스토그램/통계만으로 분포를 추정 — 프로파일링 워커가 우선 시도할 경로

### 2.3 두 계층 모니터링

| 계층 | 대상 | 허용 범위 |
|---|---|---|
| 상시 모니터링 | 전체 오브젝트 | 메타데이터 조회(정보스키마) + 2.2의 안전한 소량 샘플링만 |
| 집중 모니터링 | 리스크 신호가 잡혔거나 담당자가 지정한 일부 오브젝트 | 더 깊은/잦은 스캔 허용 — 단, **반드시 읽기 전용 복제본 또는 스냅샷/백업 사본**에서만 실행. 복제본이 없는 소스는 집중 모니터링 자체를 비활성화하거나 고객이 스냅샷을 제공해야만 가능 |

### 2.4 연결 라우팅 규칙

- 커넥터는 기본적으로 읽기 복제본 연결을 우선 시도하고, 복제본이 없으면
  메타데이터 전용(상시 모니터링) 모드로 자동 제한한다 — 프라이머리 직접 연결은
  기본값에서 막는다
- 모든 쿼리에 `statement_timeout`(엔진별 동등 설정)과 동시 커넥션 수 제한을
  강제로 건다 — 설정 누락에 대비한 이중 안전장치

---

## 3. 지원 타겟 및 버전

버전 하한선은 "메타데이터 API 계약이 안정적으로 유지되는 지점" 기준이다.

### 3.1 1단계 (MVP) — 구조화 스토리지형

| 소스 | 구체 타겟 | 버전 |
|---|---|---|
| RDB | PostgreSQL | 12 이상 |
| RDB | MySQL | 8.0 이상 (5.7 제외 권장) |
| DW | Snowflake | 관리형 SaaS, Snowflake SQL API 버전 고정 |
| DW | Amazon Redshift | 현재 세대(RA3) |
| DW | Google BigQuery | BigQuery API v2 |
| Object Storage | S3 / GCS / Azure Blob | Parquet 2.6+, Avro 1.11+ (CSV/JSON 버전 없음) |

### 3.2 2단계 — 스트리밍/NoSQL/온프레미스 DW·OLAP

| 소스 | 구체 타겟 | 버전 |
|---|---|---|
| 스트리밍 | Apache Kafka | 2.8 이상 (3.x 권장) |
| 스트리밍 | Confluent Schema Registry | 7.x 이상 |
| NoSQL | MongoDB | 5.0 이상 |
| NoSQL | Amazon DynamoDB | 관리형, 버전 없음 |
| NoSQL | Apache Cassandra | 4.0 이상 |
| DW (온프레미스) | Apache Hive | 3.1 이상 (Metastore Thrift API 기준) |
| DW (실시간 분석) | ClickHouse | 22.8 이상 (LTS) |

### 3.3 3단계 — SaaS/비정형 콘텐츠

| 소스 | 구체 타겟 | 버전 |
|---|---|---|
| SaaS 애플리케이션 | Salesforce | REST API v58 이상 |
| SaaS 애플리케이션 | Jira Cloud | REST API v3 |
| SaaS 애플리케이션 | Jira Data Center/Server | 9.x 이상, REST API v2 |
| 사용자 콘텐츠 스토어 | Google Drive | Drive API v3 (Shared Drive 스코프 기본, 개인 드라이브는 별도 정책 필요) |
| 사용자 콘텐츠 스토어 | OneDrive/SharePoint | Microsoft Graph API v1.0 |
| 사용자 콘텐츠 스토어 | Dropbox (선택) | Dropbox API v2 |

**스코프 제외**: dbt, Airflow 등 변환/오케스트레이션형 시스템은 Object/Field를
채우는 게 아니라 리니지 그래프의 FLOWS_TO 엣지를 직접 만들어내는 원천이라,
이 문서의 커넥터 인터페이스 대상이 아니다 — 다음 모듈인 "리니지 추출
방법론"에서 다룬다.

---

## 4. 커넥터 분류: 정형 vs 비정형

| 구분 | 대상 | 특징 |
|---|---|---|
| 정형 커넥터 | RDB, DW, NoSQL, Kafka, Object Storage | 고정/반고정 스키마가 있어 구조 추출과 분류가 분리된 두 단계 |
| 비정형 커넥터 | Jira, Google Drive, OneDrive | 고정 구조가 없어, 텍스트를 바로 NER 엔진에 태우는 것이 사실상 구조 발견이자 분류 |

---

## 5. 공통 계약 (모든 커넥터)

- 자격증명은 Vault/KMS 참조로만 받는다 — 직접 저장 금지
- 샘플 데이터는 처리 즉시 폐기 — 커넥터 자체가 이 휘발성을 보장
- 결과는 온보딩 문서(7장)의 잡 상태 코드로 보고 — 성공/실패/재시도 필요
  여부를 오케스트레이터가 판단할 수 있는 공통 신호 체계
- 2장의 운영 환경 안전 원칙은 커넥터 구현체 레벨에서 강제(설정으로 끌 수 없음)

---

## 6. 정형 커넥터 인터페이스

```
discover_objects() -> [Object 후보 목록]           # 테이블/토픽/파일 목록
extract_structure(object) -> [Field 목록]          # 컬럼명, 구조적 타입, nullable
sample(object, field, n) -> 샘플 값 (휘발성)        # 분류/프로파일링용, 2.2 원칙 적용
get_characteristics() -> JSON                       # 소스 타입별 특성 (인벤토리 스키마 3.6)
```

`extract_structure`와 `sample`은 분리된 두 호출이다 — 구조는 카탈로그에서,
값은 표본에서 가져온다.

---

## 7. 비정형 커넥터 인터페이스

```
discover_containers() -> [Object 후보 목록]         # object_type=DOCUMENT/ISSUE
extract_text(container) -> 원문 텍스트 스트림        # NER 엔진(entiscope)으로 직접 전달
```

`extract_structure` 단계가 없다 — 컬럼 같은 고정 구조가 없으므로 텍스트
추출과 분류가 한 흐름으로 이어진다. 결과로 생기는 Field는 "컬럼"이 아니라
**문서 안에서 발견된 개별 PII 언급(span)** 이라, 인벤토리 스키마의 Field에
`structural_type` 대신 `span_location`(문서 내 위치) 같은 필드가 필요할 수
있다 — 인벤토리 스키마 문서에 반영 필요 항목으로 남겨둔다.

---

## 8. 커넥터 등록 매니페스트

```
connector_id, source_type, is_structured (bool),
tested_against: [버전 목록],
known_issues: {버전: 이슈 설명},
capabilities: { supports_incremental, supports_sampling, requires_replica }
```

`is_structured` 플래그로 오케스트레이터가 6장/7장 중 어느 인터페이스를
호출할지 분기한다. `requires_replica`가 true인 커넥터는 2.3의 집중 모니터링
요청 시 복제본/스냅샷 연결 정보가 없으면 자동으로 거부한다.

---

## 9. Jira의 이중성 — 인바운드/아웃바운드 분리

Jira는 두 가지 역할을 겸할 수 있다.
- **인바운드**: 이슈/코멘트/첨부파일 스캔 (7장 비정형 커넥터 대상)
- **아웃바운드**: 블래스트 레이디어스 리포트를 Jira 티켓으로 발행하거나,
  리니지 그래프의 Object 노드에 관련 티켓을 링크하는 거버넌스 알림 연동 —
  이건 커넥터가 아니라 "진행 이벤트 버스"에서 나가는 별도 통합기(integration)
  컴포넌트다. 향후 "변경 관리 게이트" 모듈에서 다룬다.

---

## 10. 다음 모듈 후보

- 리니지 추출 방법론 (dbt/Airflow 등 변환/오케스트레이션형의 FLOWS_TO 엣지 생성)
- 변경 관리 게이트 (Jira 아웃바운드 연동 포함)
- Object 제외 규칙 엔진 상세 설계
