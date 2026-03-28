---
title: 'H-MAS release notes (2026-03-22~03-28)'
date: 2026-03-29
permalink: /posts/2026/03/h-mas-weekly-release-notes-0322-0328/
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
## 2026년 3월 22일 - 3월 28일 (Iteration 15)

### 주요 개발 내용 요약

이번 주는 **대시보드 실데이터 연동 완료(mock→API 전환)**, **프론트엔드 데이터 페칭 SWR 마이그레이션**, **클러스터 Terminating 고착 버그 수정**에 중점을 둔 기간이었습니다. 총 **10개의 커밋**, **7개의 PR 머지**, **3개의 이슈 생성**이 완료되었으며, Iteration 14에서 계획한 대시보드 실데이터 연동 4건을 모두 달성하고, SWR 도입으로 프론트엔드 데이터 페칭 아키텍처를 근본적으로 개선한 주간이었습니다.

{% include youtube.html id="B5GGjatZfWc" %}

---

## 새로운 기능 (New Features)

### 1. 대시보드 실데이터 연동 — mock 데이터 전면 교체 (4개 PR)

**구현 완료**: 대시보드의 모든 mock 데이터 의존성을 제거하고, 실제 백엔드 API 호출로 전환하여 운영 환경에서 정확한 실시간 데이터를 표시하도록 완성

**주요 성과**:

#### 1-1. 클러스터/GPU 통계 위젯 실데이터 연동 (#129 → PR #135)

- **StatsCards**: 클러스터 수, Ready 클러스터 수, 전체 GPU 수, GPU 할당률을 `GET /api/clusters` 데이터 기반으로 실시간 표시
- **ClusterOverview**: `mockClusters` → `useClusters()` 훅 전환, Skeleton 로딩 UI 및 에러 배너 추가
- **사이드바**: 하단 클러스터 상태 요약을 `useClusters()` 훅으로 전환
- **Client Component 전환**: `page.tsx`를 Server → Client Component로 전환하여 훅 기반 데이터 페칭 가능
- `skeleton.tsx` shadcn/ui 패턴 Skeleton 컴포넌트 신규 생성

**관련 커밋**: `9718fca` (PR #135, Closes #129)

#### 1-2. 배포 현황 위젯 실데이터 연동 (#130 → PR #137)

- `useDeployments` 커스텀 훅 추가 (`useClusters` 패턴 일관성 유지)
- 백엔드 배포 상태(`running/pending/failed/unknown`) → 프론트 상태(`running/stopped/error/pending`) 매핑 함수 구현
- `RuntimeDistribution` 컴포넌트에서 `ServingInstance[]` + mock 의존성 제거, 집계된 `Record<string, number>` props로 리팩토링
- InstanceChart, RuntimeDistribution, StatsCards 3개 위젯에 로딩/에러/빈 상태 UI 추가

**관련 커밋**: `09d09e9` (PR #137, Closes #130)

#### 1-3. 최근 활동 위젯 감사 로그 API 연동 (#131 → PR #138)

- `AuditLog` 타입 및 API 클라이언트(`audit-logs.ts`), `useAuditLogs` 커스텀 훅 신규 추가
- `RecentActivity` 컴포넌트를 AuditLog 기반으로 리팩토링: action×resourceType 2차원 아이콘/색상 매핑
- Skeleton/에러/빈 상태 UI 완비
- `page.tsx`에서 `mockActivities` 제거 → `useAuditLogs(10)` 훅으로 교체

**관련 커밋**: `21f7691` (PR #138, Closes #131)

#### 1-4. 미구현 위젯 placeholder 전환 (#132 → PR #139)

- **GPU 토폴로지 컴팩트(mock)** → **최근 배포 위젯**으로 교체: `useDeployments()` 데이터를 재사용하여 추가 API 호출 없이 최근 5건 배포 표시
- **요청/초(mock)** → **정책 요약 카드**로 교체: `GET /api/policies/summary` 연동 (`usePolicySummary` hook 신규 생성)
- **GPU 사용량** 라벨 → **GPU 할당률**로 변경: 실제 계산 방식(`(total - available) / total`)과 일치하도록 명확화
- `mockMetrics`, `mockNodeTopologies` import 완전 제거

**관련 커밋**: `325347d` (PR #139, Closes #132)

---

### 2. 대시보드 레이아웃 개선 — 뷰포트 높이 고정 (#140 → PR #141)

**구현 완료**: 대시보드 4행 수직 레이아웃을 CSS Grid 기반 3행 구조로 재구성하여 1080p 이상에서 스크롤 없이 한 화면에 모든 위젯 표시

**주요 성과**:
- `h-full` + `grid-rows-[auto_1fr_1fr]`로 뷰포트 고정
- 하단 2행을 최근배포/최근활동/클러스터현황 3컬럼 1행으로 병합
- 리스트 위젯에 `overflow-y-auto` 내부 스크롤 적용
- 인스턴스 차트 고정 높이(`h-[190px]`) 제거 → flex 기반 유연 높이

```
┌──────────────────────────────────────────────┐
│ [클러스터] [인스턴스] [GPU] [GPU할당률] [정책] │ auto
├─────────────────────┬────────────────────────┤
│ 인스턴스 현황 (차트) │ 런타임별 분포          │ 1fr
├────────┬────────────┼────────────────────────┤
│ 최근   │ 최근       │ 클러스터               │ 1fr
│ 배포   │ 활동       │ 현황                   │
└────────┴────────────┴────────────────────────┘
```

**관련 커밋**: `4925687` (PR #141, Closes #140)

---

### 3. 프론트엔드 데이터 페칭 훅 SWR 마이그레이션 (#136 → PR #142)

**구현 완료**: `useState`+`useEffect`+`useCallback` 기반 커스텀 훅 9개와 인라인 페칭 코드 5개를 **SWR**로 전환하여 보일러플레이트를 대폭 제거하고, 자동 캐싱·중복 요청 제거·포커스 재검증을 확보

**주요 성과**:

**인프라 구축**:
- `SWRProvider` 글로벌 설정 및 `lib/swr/keys.ts` 캐시 키 팩토리 도입 (17개 엔드포인트)
- `docs/research/SWR_MIGRATION_DESIGN.md` 설계 의사결정 기록

**SWR 전환된 훅 (9개)**:

| 훅 | 유형 | SWR 키 |
|----|------|--------|
| `useClusters` | 읽기 전용 | `/api/clusters` |
| `useDeployments` | 읽기 전용 | `/api/deployments` |
| `useAuditLogs` | 읽기 전용 (파라미터) | `['/api/audit-logs', limit]` |
| `usePolicySummary` | 읽기 전용 | `/api/policies/summary` |
| `useClusterTypes` | 읽기 전용 | `/api/cluster-types` |
| `useModels` | CRUD | `/api/v1/models` |
| `usePlacementStrategies` | CRUD | `/api/placement-strategies` |
| `useResourceProfiles` | CRUD | `/api/resource-profiles` |
| `useFailoverPolicies` | CRUD | `/api/failover-policies` |

**SWR 전환된 컴포넌트 (5개)**:

| 컴포넌트 | 변경 전 | 변경 후 |
|----------|---------|---------| 
| `instance-list.tsx` | `setInterval` 동적 폴링 | SWR `refreshInterval` 함수 |
| `instances/[id]/page.tsx` | `loadData()` + `setInterval` | SWR 조건부 폴링 2개 |
| `cluster-detail-dialog.tsx` | `useEffect` + fetch | SWR 조건부 페칭 (`open ? key : null`) |
| `deploy-form.tsx` | `Promise.all` 8개 API | 개별 SWR 8개 (캐시 공유) |
| `hf-token-settings.tsx` | `useCallback` + `useEffect` | SWR + mutation 후 `mutate(key)` |

**의사결정 사항**:

| 결정 | 내용 |
|------|------|
| SWR vs TanStack Query | 현재 규모(훅 9개, API ~12개)에서 번들 크기·Next.js 호환성·API 단순성 기준으로 SWR 선택 |
| Fetcher 구성 | 기존 `lib/api/*.ts` fetcher 함수를 그대로 재사용 (글로벌 fetcher 미설정) |
| 캐시 키 | 상수 기반 키 팩토리로 mutation 시 키 불일치 버그 방지 |
| Mutation 전략 | 단순 revalidation (낙관적 UI는 Phase 3에서 검토) |

**관련 커밋**: `2dc7c11` (PR #142, Closes #136)

---

## 버그 수정 (Bug Fixes)

### 1. 클러스터 해제 시 Terminating 고착 방지 및 강제 해제 지원

**수정 완료**: 원격 클러스터가 먼저 제거된 후 H-MAS에서 클러스터 해제 시 xxxxxxx finalizer가 해제되지 않아 Cluster CR이 Terminating 상태로 무한히 남는 문제 수정

**수정 내용**:
- **Backend**: `DeleteCluster`에서 멤버 클러스터가 not-ready이거나 이미 Terminating인 경우 finalizer를 patch로 강제 제거
- `Delete()` 후 NotFound 에러 허용 (finalizer 제거 시 즉시 GC되어 객체가 사라질 수 있음)
- `convertCluster`에서 `DeletionTimestamp` 확인하여 `"deleting"` status 매핑
- `model.Cluster` Status enum에 `"deleting"` 추가
- **Frontend**: 클러스터 카드 및 상세 다이얼로그에 "삭제 중" 상태 배지 (주황색) 추가
- "삭제 중" 클러스터 상세 다이얼로그에 안내 메시지와 "강제 해제" 버튼 제공

**해결된 기술 문제**:

| 문제 | 원인 | 해결 |
|------|------|------|
| 클러스터 해제 후 목록에서 사라지지 않음 | xxxxxxx finalizer(`xxxxxxx.io/cluster-controller`)가 원격 클러스터 미접근으로 해제 불가 → Terminating 고착 | Delete 전 finalizer patch 강제 제거 + `DeletionTimestamp` 기반 `"deleting"` 상태 매핑 |

**관련 커밋**: `97141e1` (PR #144, Closes #143)

---

## 문서화 (Documentation)

### 1. 네트워크 IP 고갈 트러블슈팅 및 클러스터 네트워크 모니터링 설계

| 문서 | 내용 | 관련 커밋 |
|------|------|-----------|
| `docs/troubleshooting/NETWORK_IP_EXHAUSTION.md` | CNI별 IP 고갈 진단/복구 절차, 예방 모니터링 스크립트 | `8f72b8c` |
| `docs/research/CLUSTER_NETWORK_MONITORING.md` | CNI 독립적 네트워크 헬스 모니터링 설계 (h-mas-agent 통합 방안) | `8f72b8c` |
| `docs/INDEX.md` | 새 문서 링크 추가 | `8f72b8c` |

### 2. 외부 제품 소개 자료

| 문서 | 내용 | 관련 커밋 |
|------|------|-----------|
| 제품 발표 덱 (PDF/PPTX) | `docs/external/` 디렉토리에 제품 발표 자료 추가 | `6453a2b` |
| `PRODUCT_OVERVIEW.md` | 플레이스홀더를 실제 스크린샷으로 교체 | `6453a2b` |
| `PRODUCT_DECK_OUTLINE.md` | PILOT_DECK_OUTLINE → PRODUCT_DECK_OUTLINE 전환 | `6453a2b` |
| 스크린샷 에셋 14개 | `docs/external/assets/` 디렉토리에 UI 스크린샷 추가 | `6453a2b` |

### 3. SWR 마이그레이션 설계 문서

| 문서 | 내용 | 관련 커밋 |
|------|------|-----------|
| `docs/research/SWR_MIGRATION_DESIGN.md` | SWR vs TanStack Query 비교, Phase 3-4 향후 계획 | `2dc7c11` (PR #142) |
| `docs/guide/FRONTEND_INTEGRATION.md` | 상태 관리 패턴 섹션에 SWR 패턴 추가 | `2dc7c11` (PR #142) |


---

## 이전 Iteration 계획 달성도

Iteration 14에서 계획한 3개 항목 **1개 달성, 2개 미착수**:

| 계획 | 이슈 | 상태 | 비고 |
|------|------|------|------|
| 대시보드 실데이터 연동 — mock 데이터에서 실제 API 호출로 전환 | #129, #130, #131, #132 | 완료 | 4개 이슈 모두 달성 + SWR 마이그레이션(#136)으로 아키텍처 개선까지 완료 |
| 서빙 배포 인플레이스 업데이트 API (PATCH) 구현 착수 | #125 | 미착수 | 대시보드 연동 + SWR 마이그레이션 + 클러스터 버그 수정에 우선순위 배분 |
| 추론 API 프록시 구현 — 통합 엔드포인트 설계 | #36 | 미착수 | 프론트엔드 아키텍처 정비에 우선순위 배분 |

**추가 달성**: 계획에 없던 대시보드 레이아웃 개선(#140), SWR 마이그레이션(#136), 클러스터 Terminating 고착 버그 수정(#143), 네트워크 트러블슈팅/모니터링 설계 문서 작성, 외부 제품 소개 자료 업데이트. 특히 SWR 마이그레이션은 대시보드 실데이터 연동 완료 후 동일 패턴의 수동 페칭 훅이 다수 존재하는 것을 발견하여 착수하게 됨.

---

## 미해결 이슈 (Open Issues)

### 런타임 자동 튜닝 관련 선행 조건 이슈 체인 (5개)

| 이슈 | 제목 | 역할 |
|------|------|------|
| #125 | 서빙 배포 인플레이스 업데이트 API (PATCH) | 파라미터 변경 API |
| #126 | 서빙 배포 파라미터 변경 시 Rolling Restart 파이프라인 | 안전한 재시작 |
| #127 | 서빙 런타임 메트릭 수집 파이프라인 구축 (Prometheus 연동) | 메트릭 수집 인프라 |
| #128 | 서빙 최적화 파라미터 변경 이력 관리 및 감사 로그 | 변경 추적/롤백 |
| #114 | 런타임 자동 튜닝 (메트릭 기반 파라미터 자동 조정) | 최종 목표 — 위 4개 이슈 모두 선행 |

### 프리셋 확장 이슈 (3개)

| 이슈 | 제목 |
|------|------|
| #120 | DB 기반 최적화 프리셋 관리 (CRUD API + UI) |
| #122 | 사용자 정의 최적화 프리셋 CRUD (DB 기반) |
| #123 | 프리셋 벤치마크 인프라 및 GPU 스펙별 최적화 값 분기 |

### 대시보드 v0.9 이슈 (2개)

| 이슈 | 제목 |
|------|------|
| #133 | 대시보드 실시간 GPU 사용률 및 추론 메트릭 표시 (Prometheus 연동) |
| #134 | 대시보드 GPU 토폴로지 컴팩트 위젯 실데이터 연동 (h-mas-agent) |

### 기존 미해결 이슈 (이전 Iteration에서 이관)

### 1. 스케줄러 어노테이션 변환 (xxx/xxxxxxxxxxx → xxxxxxx OverridePolicy) (#66)

- Iteration 13 이후 계속 미착수. 런타임 최적화 시스템 완성 후 검토 예정

### 2. 모델 프리셋 리스트 최신화 (#62)

- Model Registry 구현으로 정적 프리셋 관리의 필요성 크게 감소
- 잔여 작업: 최신 모델 시드 데이터 추가 필요

### 3. Failover / 재배치 정책 구현 (#39) — v0.95 마일스톤

### 4. NUMA / 토폴로지 인식 스케줄링 구현 (#38) — v0.95 마일스톤

### 5. 추론 API 프록시 구현 — 통합 엔드포인트 (#36)

### 6. H-MAS 래퍼 컨테이너 이미지 도입 검토 (#35)

### 7. 리서치: GPU 메모리 인지형 동적 모델 Bin Packing 전략 (#96)

---

## 이번 주 통계

| 항목 | 수치 |
|------|------|
| 총 커밋 수 | 10개 |
| 머지된 PR | 7개 (#135, #137, #138, #139, #141, #142, #144) |
| 생성된 이슈 | 3개 (#136, #140, #143) |
| 해결된 이슈 | 7개 (#129, #130, #131, #132, #136, #140, #143) |
| 미해결 이슈 | 17개 (#35, #36, #38, #39, #62, #66, #96, #114, #120, #122, #123, #125~#128, #133, #134) |
| 신규 기술 문서 | 3개 (네트워크 트러블슈팅, 클러스터 네트워크 모니터링 설계, SWR 마이그레이션 설계) |
| 코드 변경량 | +3,221줄 / -735줄 |

---

## 결론

이번 주는 **대시보드 실데이터 연동 완료와 프론트엔드 데이터 페칭 아키텍처 SWR 전환으로, v0.4 Developer Preview의 프론트엔드 완성도를 크게 끌어올린 기간**이었습니다. Iteration 14에서 계획한 대시보드 실데이터 연동 4건(#129~#132)을 모두 달성하여, 대시보드의 모든 mock 데이터 의존성을 제거하고 실제 운영 데이터를 표시하는 상태가 되었습니다.

대시보드 실데이터 연동 완료 후, 동일 패턴의 수동 페칭 훅이 9개 + 인라인 페칭 컴포넌트 5개가 존재하는 것을 발견하여 SWR 마이그레이션(#136)을 착수했습니다. 이를 통해 `useState`+`useEffect`+`useCallback` 보일러플레이트를 대폭 제거하고, 자동 캐싱·중복 요청 제거·포커스 재검증·에러 재시도를 확보했습니다. 또한 운영 환경에서 발견된 클러스터 Terminating 고착 버그(#143)를 수정하여 xxxxxxx finalizer 관련 장애를 해소했습니다.

**핵심 성과**:
1. **대시보드 실데이터 연동 완료**: 클러스터/GPU 통계, 배포 현황, 최근 활동, 미구현 위젯 전환 — 4개 이슈 모두 달성, mock 데이터 의존성 전면 제거
2. **SWR 마이그레이션**: 커스텀 훅 9개 + 인라인 페칭 5개 전환, 캐시 키 팩토리 17개 엔드포인트, `SWRProvider` 글로벌 설정 — 프론트엔드 데이터 페칭 아키텍처 근본 개선
3. **클러스터 Terminating 고착 수정**: xxxxxxx finalizer 강제 제거 + "삭제 중" 상태 UI — 원격 클러스터 제거 후 해제 불가 장애 해소
4. **기술 문서 3건**: 네트워크 IP 고갈 트러블슈팅, CNI 독립적 네트워크 모니터링 설계, SWR 마이그레이션 설계 의사결정

**다음 주 계획**:
- 서빙 배포 인플레이스 업데이트 API (PATCH) 구현 착수 (#125)
- 추론 API 프록시 구현 — 통합 엔드포인트 설계 (#36)
- 대시보드 v0.9 메트릭 연동 사전 조사 (#133, #134)

---

**문서 작성일**: 2026년 3월 29일  


