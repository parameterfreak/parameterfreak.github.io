---
title: 'H-MAS release notes (2026-03-08~03-14)'
date: 2026-03-15
permalink: /posts/2026/03/h-mas-weekly-release-notes-0308-0314/
categories:
  - H-MAS
tags:
  - MLOps
  - GPU
  - Scheduling
  - K8s
  - Inference
---

# H-MAS 주간 작업 노트
## 2026년 3월 8일 - 3월 14일 (Iteration 13)

### 주요 개발 내용 요약

이번 주는 **정책-배포 구조 재설계 Phase 2~4 완료**, **Zone 개념 전면 제거 및 클러스터 타입 기반 용어 통일**, **리소스 프로파일 클러스터 타입 동적 구성**에 중점을 둔 기간이었습니다. 총 **6개의 커밋**, **5개의 PR 머지**, **2개의 이슈 생성**이 완료되었으며, Iteration 12에서 착수한 정책-배포 구조 재설계를 Phase 2(xxx/xxxxxxxxxxx 제거)~Phase 4(문서 업데이트)까지 마무리하고, H-MAS 고유의 "Zone A/B/C" 추상 계층을 완전히 걷어내어 "클러스터 타입(HP/GP/CPU Only)"으로 전면 통일한 대규모 리팩토링 주간이었습니다.

{% include youtube.html id="OameWF5U9tA" autoplay=true %}

---

## 새로운 기능 (New Features)

### 1. 리소스 프로파일 클러스터 타입 동적 구성 (ClusterTypeRegistry 기반)

**구현 완료**: 리소스 프로파일 폼에서 HP/GP/CPU-Only 클러스터 타입이 하드코딩되어 있던 구조를 백엔드 ClusterTypeRegistry 기반 동적 렌더링으로 전환

**주요 성과**:
- **Backend**:
  - `cluster_type_registry.go` — 클러스터 타입 메타데이터 레지스트리 (HP, GP, CPU-Only) 신규 구현
  - `GET /api/cluster-types` 핸들러 — xxxxxxx 클러스터 라벨(`gpu-tier`) 기반 실시간 가용성(등록된 클러스터 수) 조회
  - `ClusterTypeInfo` 응답 모델 추가
- **Frontend**:
  - `resource-profile-form.tsx` — `useFieldArray` 기반 동적 폼으로 완전 재작성
  - `resource-profile-card.tsx` — 동적 렌더링으로 전환
  - `useClusterTypes` 훅 + API 클라이언트 신규 구현
  - 미등록 클러스터 타입은 토글 비활성화 + "클러스터 미등록" 배지로 표시
  - 기존 프로파일에 포함된 미등록 클러스터 override 편집(제거) 가능하도록 토글 로직 수정 (끄기는 항상 허용)

**의사결정 사항**:

| 결정 | 내용 |
|------|------|
| 클러스터 타입 소스 | 별도 `GET /api/cluster-types` API — 클러스터 목록 전체 조회보다 가볍고, 레지스트리 기반으로 향후 타입 확장 용이 |
| 미등록 타입 처리 | 토글 비활성화 + 배지 — 숨기지 않고 존재를 표시하여 인프라 구성 의도 전달 |
| 폼 구조 | `useFieldArray` 기반 — 하드코딩 제거, 레지스트리에 타입 추가만으로 폼 자동 확장 |

**관련 커밋**: `7578cfe` (PR #95, Closes #94)

---

## 리팩토링 (Refactoring)

### 1. Zone 개념 전면 제거 — 클러스터 타입 기반 용어 통일 및 API 필드 리네임

**리팩토링 완료**: H-MAS 고유의 "Zone A/B/C" 개념을 완전히 제거하고, "클러스터 타입(cluster type)"으로 용어를 전면 통일 + 백엔드 API 필드 `zone` → `targetClusterType` 리네임 (Breaking Change)

**변경 범위**: 32개 파일

**주요 변경**:
- **Backend (7개 파일)**:
  - `model/deployment.go`: `Zone string` → `TargetClusterType string` (2개 struct)
  - `database/models.go`: DB 모델 필드 리네임 + `gorm:"column:target_cluster_type"` 명시
  - `handler/deployment.go`: `req.Zone` → `req.TargetClusterType` (6곳)
  - DB 마이그레이션 `000012`: `ALTER TABLE serving_deployments RENAME COLUMN zone TO target_cluster_type`
  - Swagger 재생성
- **Frontend (10개 파일)**:
  - `placement-card.tsx`: `placementMode: 'zone'` → `'type'`, `ZoneButton` → `ClusterTypeButton`, "Zone 기반 배치" → "타입별 배치"
  - `deploy-form.tsx`: `formData.zone` → `formData.targetClusterType`
  - `resource-profile-form.tsx`, `resource-profile-card.tsx`: "Zone A/B/C — 클러스터" → "HP/GP/CPU Only 클러스터"
  - `cluster-register-dialog.tsx`: `defaultSchedulerZone` → `defaultClusterType`, Zone UI 텍스트 전면 교체
  - `cluster-detail-dialog.tsx`: "Zone 재분류 안내" → "클러스터 타입 재분류 안내"
  - `instances/[id]/page.tsx`: `zoneLabels` → `clusterTypeLabels`
- **문서 (8개 파일)**: README.md, ROADMAP.md, FEATURE_SPEC.md, 리서치 문서 4개 및 기타

**의사결정 사항**:

| 결정 | 내용 |
|------|------|
| 제거 근거 | #83에서 xxx/xxxxxxxxxxx 제거 후 Zone이 순수 리소스 값만 보유 → 추상 계층 의미 상실 + xxxxxxx/K8s topology zone과 혼동 방지 |
| Breaking Change 허용 | v0.9 Technical Preview 단계, 외부 API 소비자 없음 |
| 변경하지 않은 것 | `cluster.Zones []string` (AWS/GCP 가용 영역), `SpreadConstraints.SpreadByField: 'zone'` (xxxxxxx 표준), 기존 마이그레이션 파일명 |

**관련 커밋**: `ed51704` (PR #93, Closes #92)

---

### 2. 배포 폼 리소스/정책 UX 구조 재설계

**리팩토링 완료**: 배포 폼의 3개 카드(배치 전략, 리소스 설정, 정책 적용)를 2개 카드로 통합하여 사용자 혼란 해소 + `deploy-form.tsx` 1,599줄 → 1,132줄 (-29%, 카드 단위 컴포넌트 분리)

**구조 변경**:
```
Before: [배치 전략] → [리소스 설정] → [정책 적용]
After:  [배치 전략 + 장애 복구] → [리소스 설정 (프로파일 통합)]
```

**주요 변경**:
- `PlacementCard` 컴포넌트 분리 (457줄) — 배치 전략 + 장애 복구 통합, 선택된 전략의 `conflictResolution`/`preserveResourcesOnDeletion` info 배지 추가
- `ResourceCard` 컴포넌트 분리 (210줄) — 리소스 설정 + 프로파일 통합, 프로파일 선택 시 수동 입력 비활성화 + 클러스터 타입별 override 요약 표시
- "정책 적용" 카드 완전 제거
- `ClusterOverride` 타입, API 클라이언트, mock 데이터에서 `xxxConfig`/`xxxxxxxxxxxConfig` 잔존 코드 정리

**관련 커밋**: `d18a33c` (PR #90, Closes #84)

---

### 3. 리소스 프로파일에서 xxx/xxxxxxxxxxx 스케줄러 설정 UI 제거

**리팩토링 완료**: 백엔드가 사용하지 않는 xxx/xxxxxxxxxxx 스케줄러 설정 UI를 제거하여 사용자 혼란 방지. 배포 시 스케줄러는 `DefaultSchedulerForClusterType()`으로 클러스터 타입 기반 자동 결정되므로 프로파일에서 별도 설정 불필요

**주요 변경**:
- `resource-profile-form.tsx` (-400줄): xxx 설정(토폴로지, NVLink, MIG) + xxxxxxxxxxx 설정(QoS, GPU 공유, Gang 스케줄링, 오버커밋) 전체 제거, 관련 Zod 스키마·검증·submit 로직 제거
- `resource-profile-card.tsx` (-35줄): 스케줄러 정보 표시 블록 제거, 리소스 표시를 단일 행으로 정리
- `policies/page.tsx`: 목업 데이터 `xxxConfig`/`xxxxxxxxxxxConfig` 제거
- `SCHEDULER_CONFIGURATION.md`: 구현 상태 테이블에 UI 제거 반영

**의사결정 사항**:

| 결정 | 내용 |
|------|------|
| 백엔드 필드 유지 | `ClusterOverrideItem`의 포인터 필드 + `omitempty` 유지 — DB 호환성, #66에서 재사용 |
| 프론트엔드 타입 유지 | `xxxSchedulerConfig`, `xxxxxxxxxxxSchedulerConfig` 타입 유지 — #66에서 재사용 예정 |

**관련 커밋**: `b74b6da` (PR #89, Closes #83)

---

## 문서화 (Documentation)

### 정책-배포 연동 구조 문서 업데이트 (6개 문서)

| 문서 | 내용 | 관련 커밋 |
|------|------|-----------|
| `BACKEND_ARCHITECTURE.md` | 배포 시 정책 적용 순서도(ASCII), 스케줄러 자동 추론 규칙, OverridePolicy 생성 조건 추가. 프로젝트 구조 및 데이터 모델에 정책 관련 핸들러/리포지토리/테이블 반영 | `71bcbe9` (PR #91) |
| `RUNTIME_SETTINGS_MODEL_LOADING.md` | 리소스 프로파일 흐름은 별도 문서 참조 링크 추가 | `71bcbe9` (PR #91) |
| `IMPLEMENTATION_GUIDE.md` | 배포 폼(§5.6)을 #84 기준 5카드 구조(모델 선택, 런타임, PlacementCard, ResourceCard, 고급 설정)로 재작성 | `71bcbe9` (PR #91) |
| `FEATURE_SPEC.md` | §3 정책 관리를 4개 하위 섹션으로 확장, 정책 적용 우선순위 명세 추가, §6 배포 기능 12개 항목 구현 완료 반영 | `71bcbe9` (PR #91) |
| `ROADMAP.md` | Phase 2.5에 #79~#85 정책-배포 재설계 완료 테이블 추가, Backend 현황 갱신 | `71bcbe9` (PR #91) |
| `04-policy-deployment-integration.md` | defaultProfileId 제거(#79), preserveResourcesOnDeletion(#81) 반영, Phase 2/3 체크리스트 동기화 | `71bcbe9` (PR #91) |


---

## 정책-배포 구조 재설계 최종 현황

Iteration 12에서 착수한 4-Phase 재설계가 이번 Iteration에서 완료:

| Phase | 내용 | 상태 | 관련 이슈/PR |
|-------|------|------|-------------|
| Phase 1-1 | defaultProfileId 제거 (역할 분리) | 완료 (Iter 12) | #79 → PR #86 |
| Phase 1-2 | Zone fallback 수정 + 클러스터 타입 매핑 중앙화 | 완료 (Iter 12) | #80 → PR #87 |
| Phase 1-3 | conflictResolution/preserveResourcesOnDeletion PP 전달 | 완료 (Iter 12) | #81 → PR #88 |
| Phase 1-4 | 자동 배치(auto) 규칙 백엔드 평가 로직 | 보류 (NOT_PLANNED) | #82 |
| Phase 2 | xxx/xxxxxxxxxxx 스케줄러 설정 정리 | 완료 | #83 → PR #89 |
| Phase 3 | 배포 폼 리소스/정책 UX 구조 재설계 | 완료 | #84 → PR #90 |
| Phase 4 | 문서 업데이트 및 Swagger 재생성 | 완료 | #85 → PR #91 |

**Phase 1-4 보류 사유**: `type: auto` 배치 전략의 `rules.condition`이 자유 텍스트로 되어 있어 파서 구현이 필요하며, 현재 단일 클러스터 환경에서 우선순위가 낮음. 향후 멀티클러스터 운영 환경에서 필요 시 재개 예정.

---

## 이전 Iteration 계획 달성도

Iteration 12에서 계획한 3개 항목 **2개 완료, 1개 보류**:

| 계획 | 이슈 | 상태 | 비고 |
|------|------|------|------|
| 정책-배포 구조 재설계 Phase 2: xxx/xxxxxxxxxxx 스케줄러 설정 정리 | #83 | 완료 | PR #89 |
| 배포 폼 리소스/정책 UX 구조 재설계 | #84 | 완료 | PR #90 |
| 자동 배치(auto) 규칙 백엔드 평가 로직 구현 | #82 | NOT_PLANNED | 조건 파서 복잡도 + 단일 클러스터 환경에서 우선순위 낮음 |

**추가 달성**: 계획에 없던 Zone 개념 전면 제거(#92 → PR #93), 정책-배포 문서 업데이트(#85 → PR #91), 리소스 프로파일 동적 구성(#94 → PR #95)을 추가로 완료. #83에서 xxx/xxxxxxxxxxx 제거 후 Zone 추상 계층의 의미가 상실된 것을 포착하여 코드베이스 전반의 용어 통일까지 진행.

---

## 미해결 이슈 (Open Issues)

### 기존 미해결 이슈 (이전 Iteration에서 이관)

### 1. 스케줄러 어노테이션 변환 (xxx/xxxxxxxxxxx → xxxxxxx OverridePolicy) (#66)

- xxx/xxxxxxxxxxx 스케줄러 어노테이션을 xxxxxxx OverridePolicy로 변환하는 로직 구현 필요
- #89에서 프론트엔드 UI는 제거했으나 백엔드 타입은 유지 — 재사용 예정

### 2. 모델 프리셋 리스트 최신화 (#62)

- Iteration 12에서 비-gated 모델 추가(PR #74)는 완료
- 전체 후속 세대 모델 교체는 미완

### 3. Failover / 재배치 정책 구현 (#39) — v0.95 마일스톤

### 4. NUMA / 토폴로지 인식 스케줄링 구현 (#38) — v0.95 마일스톤

### 5. 추론 API 프록시 구현 — 통합 엔드포인트 (#36)

### 6. H-MAS 래퍼 컨테이너 이미지 도입 검토 (#35)

---

## 이번 주 통계

| 항목 | 수치 |
|------|------|
| 총 커밋 수 | 6개 |
| 머지된 PR | 5개 (#89, #90, #91, #93, #95) |
| 생성된 이슈 | 2개 (#92, #94) |
| 해결된 이슈 | 5개 (#83, #84, #85, #92, #94) |
| 보류된 이슈 | 1개 (#82 — NOT_PLANNED) |
| 미해결 이슈 | 6개 (#35, #36, #38, #39, #62, #66) |
| 신규 기술 문서 | 0개 (기존 6개 문서 업데이트) |
| 코드 변경량 | +3,297줄 / -2,901줄 |

---

## 결론

이번 주는 **정책-배포 구조 재설계를 Phase 4까지 완료하고, Zone 개념을 전면 제거하여 코드베이스의 개념적 일관성을 확보한 대규모 리팩토링 기간**이었습니다. Iteration 12에서 Phase 1을 완료한 데 이어, Phase 2(xxx/xxxxxxxxxxx 제거), Phase 3(배포 폼 UX 재설계), Phase 4(문서 업데이트)를 연속 완료하여 전체 재설계를 마무리했습니다. 특히 Phase 2에서 xxx/xxxxxxxxxxx 설정을 제거한 후 Zone 추상 계층의 의미 상실을 포착하고, 32개 파일에 걸친 "Zone → 클러스터 타입" 용어 통일 + API Breaking Change를 과감히 진행한 것이 특징적입니다. 또한 리소스 프로파일의 클러스터 타입 섹션을 하드코딩에서 백엔드 레지스트리 기반 동적 구성으로 전환하여, 향후 클러스터 타입 확장 시 프론트엔드 수정 없이 대응할 수 있는 구조를 확보했습니다.

**핵심 성과**:
1. **정책-배포 구조 재설계 완료**: Phase 1~4 중 실행 가능한 6개 Phase 모두 완료 (Phase 1-4만 보류), 배포 폼 3카드→2카드 통합으로 UX 개선
2. **Zone 개념 전면 제거**: 32개 파일, 백엔드 API Breaking Change 포함한 전면 리네임으로 xxxxxxx/K8s 용어와의 혼동 해소
3. **리소스 프로파일 동적 구성**: ClusterTypeRegistry + `GET /api/cluster-types` API 기반 동적 렌더링으로 단일 클러스터 환경 지원 + 확장성 확보
4. **코드 품질**: `deploy-form.tsx` -29% (1,599줄→1,132줄), xxx/xxxxxxxxxxx 잔존 코드 정리, 6개 문서 동기화

**다음 주 계획**:
- 모델 프리셋 리스트 최신화 — 후속 세대 모델 교체 (#62)
- 추론 API 프록시 구현 — 통합 엔드포인트 설계 착수 (#36)
- 스케줄러 어노테이션 변환 검토 (#66)

---

**문서 작성일**: 2026년 3월 15일  

