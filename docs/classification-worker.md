# 분류 워커 상세 설계 (모듈 설계 v0.1)

> 전체 구조는 `data-lineage-saas-architecture.md`(4.3 분류 워커), 엔티티
> 스키마는 `inventory-entity-schema.md`(3.4 Field, 3.5 Classification),
> 적응형 샘플링은 `onboarding-orchestration.md`(13장) 참고.

## 1. 목적과 범위

entiscope 연동 방식, 신뢰도 임계값 정책, 사람 검수 워크플로우, 그리고
`eligible_for_training`("학습 가능 여부") 판정 로직을 정의한다.

---

## 2. entiscope 연동

분류 워커는 entiscope를 내부 라이브러리/서비스 호출로 사용한다.

```
input: { field_id, sample_values: [...], language_hint }
output: { semantic_type, confidence_score, matched_pattern }
```

- entiscope는 한국어/영어/일본어/중국어(간체)를 지원한다 — 그 외 언어는
  `language_hint`가 없거나 미지원으로 판정되며, 이 경우 결과를 신뢰하지
  않는다 (7장 참고).
- 샘플은 `onboarding-orchestration.md` 13장의 적응형 샘플링 결과를 그대로
  받는다 — 분류 워커가 직접 샘플링하지 않는다(샘플링은 오케스트레이터/커넥터
  책임, 분류 워커는 판정만 담당).

---

## 3. 신뢰도 임계값 정책 (3단계)

단일 임계값이 아니라 3단계로 나눈다.

| 구간 | 처리 |
|---|---|
| 높음 (예: ≥0.85) | 자동 확정. `method=ner_model`, 검수 없이 `eligible_for_training` 판정에 바로 반영 |
| 중간 (예: 0.5~0.85) | `pending_review` — 검수 큐에 쌓임 (5장) |
| 낮음 (예: <0.5) | 판정 보류(`semantic_type=unknown`) — 자동으로 "민감할 수 있음"으로 안전하게 취급 (6장 기본 정책과 연결) |

구간 경계값은 초기 추정치이며, 실사용 데이터로 조정 대상이다.

---

## 4. 적응형 샘플링과의 연동 (에스컬레이션)

`onboarding-orchestration.md` 13장의 원칙을 구체화한다.

```
1차: 100행 샘플로 분류
    confidence ≥ 임계값 → 확정, 종료
    confidence < 임계값 → 2차 진행
2차: 1,000행 샘플로 재분류
    confidence ≥ 임계값 → 확정, 종료
    여전히 낮음 → 3장의 "중간/낮음" 구간 정책에 따라 검수 큐 또는 unknown 처리
    (3차 이상 확대는 하지 않음 — 부하 대비 이득이 낮음)
```

에스컬레이션은 필드당 최대 2회로 제한한다 — 무한정 샘플을 키우면
2.2절(커넥터 문서)의 운영 환경 안전 원칙과 충돌할 수 있다.

---

## 5. 사람 검수 워크플로우

- `pending_review` 상태의 Classification은 별도 큐 화면에 노출된다 —
  필드명, 샘플 일부(마스킹된 형태), 추정된 semantic_type, confidence를 함께
  보여준다.
- 검수자가 승인/수정하면 새 Classification row가 추가된다
  (`method=manual_override`, `reviewed_by`, `classified_at`) — 기존 행을
  덮어쓰지 않고 이력으로 남긴다 (인벤토리 스키마 3.5의 append-only 원칙과
  동일).
- 검수 완료 즉시 `eligible_for_training` 재판정이 트리거된다(6장).

---

## 6. "학습 가능 여부"(eligible_for_training) 판정 로직

Field의 `eligible_for_training`은 Classification 결과만으로 정하지 않는다 —
아래 규칙을 우선순위대로 적용한다.

```
1. Classification이 pending_review 또는 unknown → blocked
   (기본값은 항상 안전 쪽 — "확실하지 않으면 차단", 확실하지 않으면 허용이
   아니다)
2. Classification이 확정(semantic_type 판정 완료)이고 민감 유형이 아님
   → eligible
3. Classification이 확정이고 민감 유형(PII 등)이며, 아직 비식별화 처리
   기록이 없음 → requires_approval
4. 민감 유형이지만 비식별화 파이프라인 처리 완료 기록이 연결됨(향후
   비식별화 모듈에서 채움) → eligible (단, "비식별화됨" 라벨을 함께 노출)
```

**기본값이 "차단"이라는 게 핵심이다.** 분류가 불확실한 상태를 "괜찮을 것"으로
가정하면 안 되고, 확실해질 때까지 학습 데이터로 노출하지 않는다.

---

## 7. 비지원 언어 처리

entiscope가 지원하지 않는 언어의 샘플은 신뢰도 산정 자체가 무의미하므로,
`confidence_score`를 강제로 낮은 구간에 두어 3장의 "낮음" 정책(민감할 수
있음으로 안전 처리)을 그대로 타게 한다 — 별도 예외 경로를 만들지 않는다.

---

## 8. 재분류 트리거

- **데이터 변경**: `onboarding-orchestration.md`의 증분 재스캔에서 오브젝트
  스키마 변경이 감지되면 해당 Field는 재분류 대상이 된다.
- **분류 룰/모델 업데이트**: 메인 아키텍처 문서(5장)에서 정의한 "분류 룰/패턴
  업데이트"가 배포되면, 전체 Field를 재분류하지 않고 **새 룰이 다루는
  semantic_type과 연관 가능성이 있는 Field만** 선별 재분류한다(예: 새로
  추가된 "여권번호" 패턴이면 문자열 타입 Field 중 기존에 `unknown`이거나
  낮은 confidence였던 것 위주로). 비용이 크므로 전체 재실행은 기본값이
  아니다.

---

## 9. 다음 모듈 후보

- 프로파일링 워커 상세 설계 (결측치/카디널리티/분포/신선도 1차 범위 확정)
- 비식별화 파이프라인 및 재식별 위험 스코어링 (6장의 4번 규칙이 참조하는
  "비식별화 처리 완료 기록"을 실제로 어떻게 남길지)
