# 인벤토리 스토어 엔티티 스키마 (모듈 설계 v0.1)

> 전체 구조는 `data-lineage-saas-architecture.md` 참고. 이 문서는 그 중 인벤토리
> 스토어(4.5) 모듈의 상세 설계다.

## 1. 목적과 범위

Domain/Source/Object/Field 계층의 엔티티 스키마를 정의하고, 이 스키마가 어떻게
세 개의 저장소(관계형 DB / 검색 인덱스 / 그래프 스토어)로 나뉘어 반영되는지를
정한다.

---

## 2. 저장소 토폴로지 요약

- **관계형 DB**: source of truth. 모든 쓰기가 여기 먼저 반영된다.
- **검색 인덱스**: 카탈로그 검색/필터용 파생 저장소. CDC로 비동기 갱신.
- **그래프 스토어**: 블래스트 레이디어스/경로 탐색용 파생 저장소. 동일하게
  CDC로 비동기 갱신.

세 저장소는 최종적 일관성(eventual consistency)을 가지며, 실시간 트랜잭션
일관성은 요구하지 않는다.

### 2.1 동기화 방식 — CDC (애플리케이션 이벤트 발행이 아님)

검색 인덱스/그래프 스토어로의 동기화는 애플리케이션 코드가 "쓰기 후 이벤트도
발행"하는 방식이 아니라, **CDC(Change Data Capture)** 로 한다 — Debezium 등으로
관계형 DB의 WAL(write-ahead log)을 직접 읽어 카프카 토픽으로 흘려보낸다.

이유:
- **이중 쓰기 문제 회피**: 애플리케이션이 DB 쓰기와 이벤트 발행을 각각 하면,
  한쪽만 성공하는 경우(부분 실패)에 파생 저장소가 조용히 어긋난다. CDC는
  DB에 커밋된 것만 캡처하므로 이 문제가 구조적으로 없다.
- **재생 가능**: 검색 인덱스/그래프 스토어가 어긋났을 때, 토픽을 처음부터
  재생해 완전히 다시 만들 수 있다.
- **앱 코드를 안 거친 변경도 캡처**: 마이그레이션 스크립트로 DB를 직접
  고쳐도 그 변경이 자동으로 이벤트화된다.

이 카프카 토픽은 온보딩 문서의 "잡 디스패치용 태스크 큐"(커넥터→오케스트레이터
→워커 구간)와는 별개의 파이프다 — 잡 디스패치는 재시도/완료 확인이 중요해
태스크 큐(Celery/SQS/Temporal류)를 쓰고, 이 구간(인벤토리→파생 저장소)만 CDC+
카프카를 쓴다.

배포 토폴로지별로 카프카 클러스터의 위치가 달라진다 (메인 아키텍처 문서 5장
참고): 하이브리드 배포에서는 컨트롤 플레인(벤더 클라우드) 쪽에 두고, 완전
온프레미스에서는 고객 내부에 둬야 하므로 운영 부담이 커진다 — 이 티어에서는
풀 카프카 대신 Redis Streams/NATS JetStream 같은 경량 대안도 검토 대상이다
(트레이드오프로 남겨둠).

---

## 3. 엔티티 모델

```
Domain
  └─ Source (소스 객체)
       └─ Object (테이블 / 토픽 / 파일-버킷 경로)
            └─ Field (컬럼 / 메시지 필드 / 파일 스키마 필드)
                 ├─ Classification (분류 워커 결과)
                 └─ Profiling (프로파일링 워커 결과)

Model (모델 아티팩트) --trained_from--> Object (그래프 스토어 엣지로만 존재)
```

### 3.1 Domain
| 필드 | 설명 |
|---|---|
| domain_id (PK) | |
| name | UNIQUE |
| owner_team | |
| description | |
| created_at | |

### 3.2 Source
| 필드 | 설명 |
|---|---|
| source_id (PK) | |
| domain_id (FK → Domain.domain_id) | INDEX |
| source_type | RDB / DW / KAFKA / OBJECT_STORAGE 등 고정 taxonomy. INDEX |
| name | |
| connection_ref | Vault/KMS 참조 키 — 자격증명 원본은 저장하지 않음 |
| environment | prod / dev / staging |
| criticality_tier | 블래스트 레이디어스 리포트의 우선순위 산정에 사용 |
| characteristics | JSONB — 소스 타입별 특성 (3.6 참고) |
| registered_at | |
| last_scanned_at | |

**제약**: `UNIQUE(domain_id, name)` — 같은 도메인 안에서 소스 이름 중복 방지

### 3.3 Object
| 필드 | 설명 |
|---|---|
| object_id (PK) | |
| source_id (FK → Source.source_id) | INDEX |
| object_type | TABLE / VIEW / TOPIC / FILE_PATH 등 |
| name | 예: schema.table_name, topic 이름, 버킷 경로 |
| owner | nullable — 미배정이면 거버넌스 갭 리포트에 노출. INDEX (WHERE owner IS NULL) |
| row_count_estimate | 소스 타입에 따라 message rate 등으로 대체 가능 |
| last_modified_at | 신선도 지표. INDEX (정렬용) |
| scan_status | pending / in_progress / done / failed / excluded_by_rule / excluded_manual — 단계 진행 추적. INDEX (오케스트레이터 조회용) |
| exclusion_reason | nullable — 규칙 이름 또는 수동 제외 사유 (예: `rule:name_suffix_tmp`, `manual: 담당자 김OO`) |
| valid_from / valid_to | SCD Type 2 버저닝 (5장 참고) |

**제약**: `UNIQUE(source_id, name) WHERE valid_to IS NULL` — 같은 소스 안에서
현재 유효한 오브젝트 이름 중복 방지 (부분 인덱스, 과거 버전은 중복 허용)

### 3.4 Field
| 필드 | 설명 |
|---|---|
| field_id (PK) | |
| object_id (FK → Object.object_id) | INDEX |
| name | |
| structural_type | 구조 추출 워커 결과 (예: VARCHAR(255), INT) |
| nullable | |
| ordinal_position | |
| eligible_for_training | eligible / requires_approval / blocked — 카탈로그 검색 결과에 노출되는 "학습 가능 여부" 배지의 근거 필드. INDEX |
| valid_from / valid_to | SCD Type 2 버저닝 (5장 참고) |

**제약**: `UNIQUE(object_id, name) WHERE valid_to IS NULL`

### 3.5 Classification / Profiling (Field에 종속)
| 필드 (Classification) | 설명 |
|---|---|
| classification_id (PK) | |
| field_id (FK → Field.field_id) | INDEX `(field_id, classified_at DESC)` — 최신 판정 조회용 |
| semantic_type | 이메일/전화번호/주민번호 등. INDEX (리스크 히트맵 집계용) |
| confidence_score | |
| method | rule_based / ner_model / manual_override |
| reviewed_by | nullable — 사람이 검수했으면 기록 |
| classified_at | |

| 필드 (Profiling) | 설명 |
|---|---|
| profiling_id (PK) | |
| field_id (FK → Field.field_id) | INDEX `(field_id, profiled_at DESC)` — 최신 결과 조회용 |
| null_ratio | |
| cardinality | |
| distribution_summary | JSONB — 상위 값/히스토그램 요약 (선택적) |
| profiled_at | |

두 테이블 모두 판정 이력을 남기는 append-only 구조다(같은 field_id에 여러 row
허용). "현재 값"은 `classified_at`/`profiled_at` 최신 row로 조회한다.
Classification과 Profiling 사이에는 FK 관계가 없다 — 서로 독립적으로 쓰는
워커가 각자의 시점에 병렬로 기록한다.

### 3.6 소스 타입별 특성 처리 (Source.characteristics)
소스 타입마다 다른 특성을 별도 테이블로 나누지 않고, `characteristics` 컬럼에
JSONB로 저장한다. 타입별 스키마는 별도 taxonomy 설정(코드/구성 파일)으로
정의해 검증한다.

| 소스 타입 | characteristics 예시 |
|---|---|
| RDB | `{engine, version}` |
| DW | `{warehouse_type, refresh_interval}` |
| Kafka | `{schema_registry_url, serialization_format}` |
| Object Storage | `{file_format, partitioning_rule}` |

새 소스 타입 추가 시 테이블 스키마 변경 없이 taxonomy 설정만 추가하면 된다.

### 3.7 Model (모델 아티팩트)
관계형 DB에는 최소 정보만 둔다 (model_id, name, version, owner, trained_at).
"어떤 Object/Field로 학습됐는가"는 관계형 FK가 아니라 **그래프 스토어의
`TRAINED_FROM` 엣지**로 표현한다 — 학습에 쓰인 데이터셋이 여러 Object에 걸친
쿼리 결과일 수 있어 N:M 관계이고, 리니지 그래프 순회(영향받는 모델 조회)와
동일한 질의 패턴이기 때문이다.

### 3.8 UnclaimedAsset (미할당 자산 — 인프라 대조용, Domain 계층과 별개)
Source로 등록되지 않은 인프라 자산(그레이 영역)을 담는 테이블. Domain/Source
계층에 속하지 않으므로 FK로 연결하지 않고, 나중에 Source로 편입될 때만
연결된다.

| 필드 | 설명 |
|---|---|
| unclaimed_asset_id (PK) | |
| infra_identifier | 클라우드 리소스 식별자(ARN 등). UNIQUE `(cloud_account_id, infra_identifier)` |
| cloud_account_id | |
| resource_type | RDS_INSTANCE / S3_BUCKET / KAFKA_CLUSTER 등 인프라 API가 주는 원시 타입 |
| discovered_at | |
| match_status | unmatched / matched — 등록된 Source의 connection 정보와 자동 대조 결과. INDEX |
| triage_status | pending / assigned_to_domain / ignored — 담당자 트리아지 상태. INDEX |
| matched_source_id (FK → Source.source_id, nullable) | matched 상태일 때만 채워짐 |

이 테이블은 4장 쓰기 흐름의 파이프라인과 별개로, **베스트 에포트 인프라
스캐너**가 채운다 — 클라우드 계정 API 접근 권한이 없는 고객에게는 이 기능
자체가 비활성화되며, 없어도 나머지 파이프라인은 정상 동작한다.

---

## 4. 쓰기 흐름 (Write Flow)

FK 방향이 곧 쓰기 순서다 — 아래 순서를 벗어난 쓰기(예: Object 없이 Field 생성)는
제약 위반으로 막힌다.

| 단계/컴포넌트 | 쓰는 테이블 | 시점 |
|---|---|---|
| 1a 도메인 범위 (사용자 입력) | Domain | 온보딩 시 |
| 1b 소스 객체 등록 (사용자 입력) | Source | 온보딩 시 |
| 스캔 오케스트레이터 (오브젝트 디스커버리) | Object 생성 (`scan_status=pending`) | 도메인 스캔 시작 시 |
| 구조 추출 워커 | Object 갱신(`scan_status`, `row_count_estimate` 등), Field 생성 | 오브젝트별 스캔 완료 시 |
| 분류 워커 (entiscope) | Classification 생성 | Field 생성 이후, 비동기 |
| 프로파일링 워커 | Profiling 생성 | Field 생성 이후, 비동기 |
| 검색 인덱스 빌더 (파생) | 검색 인덱스 문서 | CDC 토픽 구독 (2.1 참고) |
| 그래프 스토어 빌더 (파생) | FLOWS_TO / TRAINED_FROM 엣지 | 리니지 추출 결과 + CDC 토픽 구독 |
| 인프라 자산 스캐너 (베스트 에포트, 별도 트리거) | UnclaimedAsset 생성/갱신 (`match_status`) | 클라우드 계정 API 접근 권한이 있을 때, 파이프라인과 무관한 주기로 |

분류 워커와 프로파일링 워커는 서로 FK 의존이 없어 **병렬로** 쓴다 — 순서 보장이
필요 없다.

---

## 5. 버저닝 전략

Object/Field는 스키마가 바뀔 수 있으므로 SCD Type 2 방식(변경 시 새 row 추가,
`valid_from`/`valid_to` 컬럼)으로 이력을 관리한다. 현재 상태를 보여줄 때는
`valid_to IS NULL`인 row만 조회하고, 특정 시점 리니지 재현이 필요할 때는
`valid_from/valid_to` 범위로 조회한다.

---

## 6. 검색 인덱스 파생 스키마

Field를 기본 문서 단위로 삼고, 상위 계층(Domain/Source/Object 이름, owner,
criticality_tier)과 Classification/Profiling 결과를 비정규화해 하나의 문서로
색인한다.

```json
{
  "field_id": "...",
  "field_name": "...",
  "object_name": "...",
  "source_name": "...",
  "domain_name": "...",
  "owner": "...",
  "semantic_type": "...",
  "eligible_for_training": "...",
  "null_ratio": 0.0
}
```

Field 단위 색인을 쓰는 이유: "이메일 컬럼 찾기" 같은 검색이 Object 단위보다
정확하고, 화면에서 Object로 묶어 보여줄 땐 집계 질의로 처리 가능하기 때문이다.

---

## 7. 그래프 스토어 파생 스키마

**노드 타입**: Object, Field, Model
**엣지 타입**:
| 엣지 | 방향 | 속성 |
|---|---|---|
| FLOWS_TO | Object→Object 또는 Field→Field | extraction_method, confidence, extracted_at, reviewed_by |
| TRAINED_FROM | Model→Object | trained_at |

Domain/Source는 그래프 순회 대상이 아니라 필터링용 속성으로만 노드에
비정규화해 붙인다 (예: Object 노드에 domain_id, source_type을 속성으로 포함).

---

## 8. 다음 모듈 후보

- 소스 타입별 커넥터 플러그인 인터페이스 (이 스키마의 characteristics/Object/
  Field를 실제로 채워 넣는 쪽)
- 리니지 추출 방법론 (FLOWS_TO 엣지를 어떻게 생성할지)
- Object 제외 규칙 엔진 (규칙 정의 방식, 규칙에 안 걸리는 예외 케이스의 수동
  처리 흐름)
