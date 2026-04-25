---
title: 'H-MAS release notes (2026-04-19~04-25)'
date: 2026-04-26
permalink: /posts/2026/04/h-mas-weekly-release-notes-0419-0425/
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
## 2026년 4월 19일 - 4월 25일 (Iteration 19)

### 주요 개발 내용 요약

이번 주는 **런타임 자동 튜닝(#114)의 선행 조건 이슈 3건 완주 (#125→#126→#128)**, **클러스터 장애 시 인스턴스 상태 안정성 확보**, **xxxxxxxxxxx 스케줄러 통합 기술 조사 완료**에 중점을 둔 기간이었습니다. 총 **12개의 커밋**, **7개의 PR 머지**, **12개의 이슈 생성**이 완료되었으며, 인플레이스 업데이트 API(#125) → Rolling Restart 파이프라인(#126) → 변경 이력 관리(#128) 순서로 파라미터 변경 라이프사이클 전체를 1주일 만에 구축하여 #114 자동 튜닝의 기반 인프라를 마련한 주간이었습니다.

{% include youtube.html id="t6KCfkEupiM" autoplay=true %}

---

## 새로운 기능 (New Features)

### 1. 서빙 배포 인플레이스 업데이트 API — PATCH (#125 → PR #208)

**구현 완료**: 배포 삭제/재생성 없이 운영 중인 서빙 배포의 최적화 파라미터, 레플리카, 리소스 프로필을 변경할 수 있는 `PATCH /api/deployments/:id` 엔드포인트 및 프론트엔드 설정 수정 UI 구현

**주요 성과**:

- **Backend**:
  - `PATCH /api/deployments/:id` 엔드포인트 추가 (`replicas`, `optimization`, `resourceProfileId` 변경 지원)
  - `xxxxxxx.UpdateServingDeployment()` (Get-Modify-Update + 멤버 클러스터 dual-write) 구현
  - `UpdateServingOverridePolicy()` — 리소스 프로파일 OverridePolicy 생성/교체/삭제 통합
  - `GET /api/deployments/:id` 응답에 `optimization` 필드 추가 (기존 설정 표시용)
  - Swagger 문서 재생성

- **Frontend**:
  - Sheet(드로어) 기반 설정 수정 UI — 기존 `OptimizationCard` 재사용으로 프리셋/Tier 1·2/Raw Override 구조화 입력
  - 기존 설정 초기값 로드, 변경 diff 미리보기, 재시작 경고 제공
  - 인스턴스 상세 페이지에 "설정 수정" 버튼 (running/pending 상태에서 표시)

**의사결정 사항**:

| 결정 | 내용 |
|------|------|
| UI 패턴 | 모달 대신 Sheet(드로어) 채택 — 기존 OptimizationCard 재활용으로 일관된 UX |
| xxxxxxx 업데이트 방식 | Get-Modify-Update + 멤버 클러스터 dual-write — #189에서 검증된 Dual-Update 패턴 재사용 |
| Paused 배포 처리 | 변경 불가 — 400 에러로 명확히 차단 |

**관련 커밋**: `059a7da`, `5d85a21` (PR #208, Closes #125)

---

### 2. 서빙 배포 Rolling Restart 파이프라인 (#126 → PR #221)

**구현 완료**: 배포 파라미터 변경 시 안전한 롤아웃과 실패 시 자동 롤백을 수행하는 Rolling Restart 파이프라인 구현 — Dry-run 사전 분석, 롤아웃 상태 추적, CrashLoopBackOff/OOMKilled 감지 시 자동 복원

**주요 성과**:

- **Backend**:
  - Dry-run API (`?dryRun=true`): 변경 적용 전 재시작 필요 여부, 전략(RollingUpdate/Recreate), 예상 영향도 사전 분석
  - `deployment_rollouts` 테이블 (migration #21): 롤아웃 상태 추적 및 이전 설정 스냅샷 저장
  - RolloutReconciler: 10초 간격 활성 롤아웃 모니터링, 런타임별 startup grace period 적용 (Recreate 3분 / RollingUpdate 90초)
  - 자동 롤백: CrashLoopBackOff·OOMKilled·ProgressDeadlineExceeded 감지 시 이전 설정으로 자동 복원
  - 롤아웃 상태 조회, 이력 조회, 수동 롤백 엔드포인트 추가

- **Frontend**:
  - 설정 수정 다이얼로그에 dry-run 분석 결과 (전략, 영향도) 표시
  - 롤아웃 상태 배너: 실시간 진행률 표시 + 수동 롤백 버튼
  - 롤아웃 이력 목록 (collapsible)

**해결된 기술 문제**:

| 문제 | 원인 | 해결 |
|------|------|------|
| Ollama 최적화 변경 시 Pod 즉시 크래시 | 런타임 환경변수 누락 | 환경변수 생성 로직 보정 |
| 롤백 스냅샷에 변경 후 값 저장 | 스냅샷 캡처 시점 오류 | 변경 전 값을 정확히 캡처하도록 수정 |
| Recreate 전략 시 모델 로딩 중 오탐 | 대형 모델 초기화 시간을 실패로 오판 | 런타임별 grace period 적용 |

**관련 커밋**: `8c3c584` (PR #221, Closes #126)

---

### 3. 서빙 최적화 파라미터 변경 이력 관리 및 감사 로그 (#128 → PR #222)

**구현 완료**: 배포 최적화 파라미터 변경 시 변경 전/후 config, 변경 원인(manual/preset/auto_tuning/rollback), 적용 결과를 자동 기록하는 이력 관리 시스템 및 프론트엔드 통합 변경 이력 패널 구현

**주요 성과**:

- **Backend**:
  - `deployment_config_history` 테이블 신규 생성 (마이그레이션 #22), `rollout_id` FK로 롤아웃과 하이브리드 연결
  - config-history 조회(목록/상세), 특정 시점 롤백 API 3개 엔드포인트 추가

  | 엔드포인트 | 설명 |
  |-----------|------|
  | `GET /api/deployments/:id/config-history` | 변경 이력 목록 (페이지네이션) |
  | `GET /api/deployments/:id/config-history/:historyId` | 변경 상세 (전/후 diff) |
  | `POST /api/deployments/:id/config-history/:historyId/rollback` | 특정 시점으로 롤백 |

- **Frontend**:
  - 통합 변경 이력 패널: 롤아웃 상태 인라인 표시, JSON diff 뷰, 롤백 버튼
  - 기존 롤아웃 이력 섹션을 통합 패널로 흡수하여 중복 UI 제거 (활성 롤아웃 배너는 유지)

**의사결정 사항**:

| 결정 | 내용 |
|------|------|
| 테이블 설계 | 하이브리드 테이블 — 별도 `deployment_config_history` + 옵셔널 `rollout_id` FK로 롤아웃과 연결 |
| 추적 범위 | OptimizationConfig만 — #114 자동 튜닝 범위에 맞춤 (replicas는 기존 rollouts에서 관리) |
| 롤백 구현 | 기존 xxxxxxx 파이프라인 재사용 — `UpdateServingDeployment` 내부 호출로 일관성 유지 |

**관련 커밋**: `e223e85` (PR #222, Closes #128)

---

### 4. URL 경로 /logs → /monitoring 변경 (#204 → PR #205)

**구현 완료**: 사이드바 메뉴명 "모니터링"과 URL 경로(`/logs`) 불일치를 해소하여 Next.js 라우트 디렉토리를 `monitoring`으로 이동하고, 사이드바·내부 네비게이션·인스턴스 상세 페이지·데모 시나리오의 경로 참조를 일괄 변경

**관련 커밋**: `e2c9963` (PR #205, Closes #204)

---

## 버그 수정 (Bug Fixes)

### 1. 클러스터 종료 시 인스턴스 상세/모니터링 페이지 무한 로딩 (#209 → PR #211)

**수정 완료**: 멤버 클러스터 종료 시 인스턴스 상세 페이지와 모니터링 페이지가 무한 로딩에 빠지는 문제를 해결하여, 클러스터 미도달 시에도 DB 기반 정보를 정상 표시하고 사용자 친화적 안내 UI 제공

**수정 내용**:
- **Backend**: xxxxxxx/멤버 클러스터 조회를 병렬 실행 + 개별 5초 timeout 적용, 실패 시 DB fallback 응답 반환 (`statusSource: "cached"`)
- **Frontend**: `fetchWithAuth` AbortController 타임아웃 추가, `statusSource === "cached"` 시 amber 배너로 "마지막 저장된 상태" 안내, 모니터링 페이지 클러스터 연결 불가 안내 UI (내부 K8s 에러 미노출)

**관련 커밋**: `40dc15a` (PR #211, Closes #209)

---

### 2. 클러스터 종료 후 인스턴스 stale status 해결 (#210 → PR #212)

**수정 완료**: 클러스터 종료 후에도 인스턴스가 "실행 중"으로 표시되는 stale status 문제를 다층 방어(defense-in-depth)로 해결, 목록 API 응답 시간을 **10초 → 0.03초**로 대폭 개선

**수정 내용**:
- **Backend — API 핸들러**: 클러스터 readiness 사전 확인 → not-ready 클러스터 배포는 즉시 `unknown`/`cached` 반환, 순차 → 병렬 enrichment + 목록 전용 타임아웃(2초) 도입
- **Backend — Reconciler**: 연속 3회 API 실패 시 DB `LastKnownStatus`를 `"unknown"`으로 갱신, not-ready 클러스터 인스턴스 일괄 `"unknown"` 갱신 (60초 주기)
- **Backend — Repository**: `BulkUpdateStatusByCluster`, `ListActiveByCluster` 신규 메서드 추가
- **Frontend**: `statusSource="cached"` 시 WifiOff 아이콘 + 툴팁, 대시보드 파이차트에 `"unknown"` 별도 카테고리(회색) 분리, unhealthy 배너에 unknown 포함

**관련 커밋**: `8e223a7` (PR #212, Closes #210)

---

## 문서화 (Documentation)

### 1. xxxxxxxxxxx 스케줄러 통합 기술 조사 및 설계 (#179 → PR #220)

**작성 완료**: xxxxxxxxxxx 스케줄러 도입을 위한 기초 기술 조사 및 H-MAS 통합 설계 문서 4건 작성 — 멀티클러스터 이기종 스케줄러 환경 설계, 기능 계층 분류, GPU 공유/격리 방식 비교, Compute Plane GP Profile 설계를 포괄

| 문서 | 내용 | 관련 커밋 |
|------|------|-----------|
| `docs/research/xxxxxxxxxxx/` (허브 + 4개 문서) | xxxxxxxxxxx 기능 조사, GPU 공유 비교, H-MAS 통합 설계 | `0b1c1f2` (PR #220) |

**조사 범위**:

| 영역 | 내용 |
|------|------|
| 멀티클러스터 아키텍처 | 이기종 스케줄러 환경(기본 K8s/xxxxxxxxxxx) 설계, 클러스터별 독립 스케줄러 |
| 기능 계층 | L1~L4 내부 분류 + Capability 기반 사용자 노출 전략 |
| GPU 공유/격리 | xxx, xxxx, xxx, xxx, xxxx-xxxxxxx 5가지 비교, 방식 무관 추상화 설계 |
| 멀티 GPU 추론 | TP/PP/EP 병렬화 전략, 모델 크기별 GPU 요구량, HP vs GP 역할 분담 |
| 호환성 | K8s 버전 매트릭스, ARM64 이미지 지원 확인 |
| H-MAS 통합 | 백엔드 Capability 감지, 프론트엔드 동적 UI, Compute Plane GP Profile |

**생성된 후속 이슈**:
- #216 — xxxxxxxxxxx 실 클러스터 설치 및 환경 검증
- #217 — QoS/오버커밋 동작 검증
- #218 — 멀티 GPU 추론 (Gang Scheduling + 토폴로지) 검증
- #219 — H-MAS xxxxxxxxxxx 통합 구현

### 2. 기타 문서

| 문서 | 내용 | 관련 커밋 |
|------|------|-----------|
| `README.md` | 시스템 요구사항 섹션 신설 및 README 보강 | `c847274` |
| 다수 문서 | #210 구현에 따른 클러스터 장애 대응 관련 문서 최신화 | `6038f82` |

---

## 기타 작업 (Chores)

- 기술 스택 전체 버전 핀닝 (`c716bb5`)

---

## 이전 Iteration 계획 달성도

Iteration 18에서 계획한 4개 항목 **1개 완료, 3개 미착수** — 인플레이스 업데이트(#125)를 기점으로 Rolling Restart(#126)→변경 이력(#128) 선행 조건 체인 완주 및 클러스터 장애 안정성 확보에 집중:

| 계획 | 이슈 | 상태 | 비고 |
|------|------|------|------|
| 서빙 배포 인플레이스 업데이트 API (PATCH) 구현 | #125 | 완료 | PR #208 — PATCH API + Sheet UI + xxxxxxx Dual-Update |
| 추론 프록시 Phase 1 구현 착수 | #191 | ⏸ 미착수 | 인플레이스 업데이트 → Rolling Restart → 변경 이력 체인에 우선순위 전환 |
| 추론 테스트 UI (Chat 인터페이스) 구현 | #195 | ⏸ 미착수 | 동일 사유 |
| h-mas-agent 설계 상세화 | #190 | ⏸ 미착수 | 동일 사유 |

**추가 달성**: 계획에 없던 Rolling Restart 파이프라인(#126), 변경 이력 관리(#128) 구현으로 #114 자동 튜닝의 선행 조건 3/4 완료. 클러스터 장애 시 stale status(#210) 및 무한 로딩(#209) 버그 수정. xxxxxxxxxxx 기술 조사(#179) 완료 및 후속 검증 이슈 4건 + 리서치 이슈 3건 생성으로 v0.9 GP 클러스터 Phase 준비.

---

## 미해결 이슈 (Open Issues)

### 신규 이슈 (12개, 이번 Iteration 생성)

| 이슈 | 제목 | 라벨 |
|------|------|------|
| #219 | 구현: H-MAS xxxxxxxxxxx 통합 — Capability 감지, API, 프론트엔드, Compute Plane | research |
| #218 | 검증: 멀티 GPU 추론 지원 — Gang Scheduling 및 GPU 토폴로지 인식 검증 | research |
| #217 | 검증: xxxxxxxxxxx QoS 클래스 및 리소스 오버커밋 동작 검증 | research |
| #216 | 검증: xxxxxxxxxxx 실 클러스터 설치 및 환경 호환성 검증 | research |
| #215 | 리서치: GPU 사용량 기반 과금/미터링 시스템 설계 | research |
| #214 | 리서치: SaaS 멀티테넌트 자원 거버넌스 설계 (ElasticQuota 계층형 쿼터) | research |
| #213 | 리서치: GPU 공유/격리 방식 비교 조사 (xxx, xxxx, xxx, xxx, xxxx-xxxxxxx) | research |
| #207 | 서빙 배포 Desired State 기반 Reconciler 적용 (2-Phase 일관성 보장) | backend |
| #206 | 서빙 배포 PVC 인플레이스 리사이즈 지원 | backend |

> #216→#217→#218→#219는 #179(xxxxxxxxxxx 리서치)의 후속 검증/구현 이슈 체인으로, 이번 iteration에 기술 조사 문서를 기반으로 분해
> #213→#214→#215는 xxxxxxxxxxx 통합 과정에서 파생된 SaaS/GP 관련 리서치 이슈

### 런타임 자동 튜닝 선행 조건 이슈 체인

| 이슈 | 제목 | 상태 |
|------|------|------|
| #125 | 서빙 배포 인플레이스 업데이트 API (PATCH) | **Iteration 19 완료** |
| #126 | 서빙 배포 파라미터 변경 시 Rolling Restart 파이프라인 | **Iteration 19 완료** |
| #127 | ~~서빙 런타임 메트릭 수집 파이프라인 구축~~ | Iteration 18 완료 |
| #128 | 서빙 최적화 파라미터 변경 이력 관리 및 감사 로그 | **Iteration 19 완료** |
| #114 | 런타임 자동 튜닝 (메트릭 기반 파라미터 자동 조정) | **선행 조건 4/4 완료 — 착수 가능** |

### 추론 프록시 이슈 체인 (4개, 미착수)

| 이슈 | 제목 |
|------|------|
| #191 | 추론 프록시 Phase 1: 기본 프록시 구현 (Push 모드) |
| #192 | 추론 프록시 Phase 2: 멀티클러스터 라우팅 (agent 터널 통합) |
| #193 | 추론 요청 로깅 및 메트릭 수집 파이프라인 |
| #194 | 추론 API 키 인증 및 Rate Limiting |

### 기타 미해결 이슈

| 이슈 | 제목 |
|------|------|
| #195 | 프론트엔드: 추론 테스트 UI (Chat 인터페이스) |
| #190 | h-mas-agent 핵심 컴포넌트 구현 |
| #134 | 대시보드 GPU 토폴로지 컴팩트 위젯 실데이터 연동 (h-mas-agent) |
| #120 | DB 기반 최적화 프리셋 관리 (CRUD API + UI) |
| #122 | 사용자 정의 최적화 프리셋 CRUD (DB 기반) |
| #123 | 프리셋 벤치마크 인프라 및 GPU 스펙별 최적화 값 분기 |
| #96 | 리서치: GPU 메모리 인지형 동적 모델 Bin Packing 전략 |
| #186 | 리서치: 서버리스 서빙(Scale-to-Zero) 방식 분석 |
| #184 | 리서치: 폐쇄망(Air-Gapped) 환경 배포 전략 수립 |
| #66 | 스케줄러 어노테이션 변환 (xxx/xxxxxxxxxxx → xxxxxxx OverridePolicy) |
| #62 | 모델 프리셋 리스트 최신화 |
| #39 | Failover / 재배치 정책 구현 — v0.95 마일스톤 |
| #38 | NUMA / 토폴로지 인식 스케줄링 구현 — v0.95 마일스톤 |
| #36 | 추론 API 프록시 구현 (통합 엔드포인트) |

---

## 이번 주 통계

| 항목 | 수치 |
|------|------|
| 총 커밋 수 | 12개 (+ merge 7건) |
| 머지된 PR | 7개 (#205, #208, #211, #212, #220, #221, #222) |
| 생성된 이슈 | 12개 (#204, #206, #207, #209, #210, #213, #214, #215, #216, #217, #218, #219) |
| 해결된 이슈 | 7개 (#125, #126, #128, #179, #204, #209, #210) |
| 미해결 이슈 | 28개 |
| 신규 기술 문서 | 4개 (xxxxxxxxxxx 허브 + 기능 조사 + GPU 공유 비교 + H-MAS 통합 설계) |
| 코드 변경량 | +6,930줄 / -273줄 (62개 파일) |

---

## 결론

이번 주는 **런타임 자동 튜닝(#114)의 선행 조건 이슈 3건(#125→#126→#128)을 순차 완주하여 파라미터 변경 라이프사이클 전체를 구축한 기간**이었습니다. 인플레이스 업데이트 API(#125)로 운영 중 배포 파라미터 변경이 가능해졌고, Rolling Restart 파이프라인(#126)으로 CrashLoopBackOff/OOMKilled 감지 시 자동 롤백이 적용되며, 변경 이력 관리(#128)로 모든 설정 변경이 감사 로그로 기록됩니다. 이로써 Iteration 18에서 완료한 메트릭 수집(#127)과 합쳐 #114 자동 튜닝의 선행 조건 4/4가 모두 충족되었습니다.

클러스터 장애 대응에서는 stale status(#210)와 무한 로딩(#209) 문제를 다층 방어로 해결하여, 목록 API 응답 시간을 10초에서 0.03초로 개선하고 클러스터 미도달 시에도 DB 기반 정보를 안정적으로 제공하게 되었습니다.

xxxxxxxxxxx 스케줄러 기술 조사(#179)를 완료하고 후속 검증 이슈 4건(#216~#219)과 SaaS 관련 리서치 3건(#213~#215)을 생성하여, 6월 GP 서빙 엔진 Phase 진입을 위한 로드맵을 구체화했습니다.

**핵심 성과**:
1. **파라미터 변경 라이프사이클 완주**: #125(PATCH API) → #126(Rolling Restart + 자동 롤백) → #128(변경 이력 + 감사 로그) — #114 자동 튜닝 선행 조건 4/4 완료
2. **클러스터 장애 안정성 확보**: stale status 다층 방어 + 무한 로딩 해결 — 목록 API 10초 → 0.03초 개선
3. **xxxxxxxxxxx 기술 조사 완료**: 4건 리서치 문서 + 후속 검증/구현 이슈 4건 생성 — GP 클러스터 Phase 준비
4. **SaaS 로드맵 구체화**: GPU 공유/격리 비교(#213), 멀티테넌트 거버넌스(#214), 과금/미터링(#215) 리서치 이슈 생성

**다음 주 계획**:
- 추론 프록시 Phase 1 구현 착수 (#191) — Push 모드 기본 프록시
- 추론 테스트 UI (Chat 인터페이스) 구현 (#195)
- 런타임 자동 튜닝 설계 상세화 (#114) — 선행 조건 완료에 따른 착수
- h-mas-agent 설계 상세화 (#190)

---

**문서 작성일**: 2026년 4월 26일
