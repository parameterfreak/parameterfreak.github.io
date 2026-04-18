---
title: 'H-MAS release notes (2026-04-12~04-18)'
date: 2026-04-19
permalink: /posts/2026/04/h-mas-weekly-release-notes-0412-0418/
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
## 2026년 4월 12일 - 4월 18일 (Iteration 18)

### 주요 개발 내용 요약

이번 주는 **서빙 런타임 메트릭 수집 파이프라인 전체 구현**, **배포 일시정지/재시작 기능**, **브랜치 전략 및 릴리즈 프로세스 문서화**에 중점을 둔 기간이었습니다. 총 **23개의 커밋**, **6개의 PR 머지**, **9개의 이슈 생성**이 완료되었으며, #127(메트릭 수집 파이프라인)의 3개 서브이슈(인프라→Backend API→Frontend)를 1주일 만에 순차 완주하여 목 데이터 기반 대시보드를 Prometheus 실데이터로 전면 전환한 주간이었습니다.

{% include youtube.html id="QS24P_KsbK0" autoplay=true %}

---

## 새로운 기능 (New Features)

### 1. 서빙 배포 일시정지/재시작 (Pause & Resume) (#185 → PR #189)

**구현 완료**: 서빙 배포의 일시정지(Pause) 및 재시작(Resume) 기능을 xxxxxxx Dual-Update 패턴으로 구현하여, 멤버 클러스터 Pod를 종료/재생성하면서 xxxxxxx 전파 지연 문제를 해결

**주요 성과**:

- **Backend**:
  - `POST /api/deployments/:id/pause`, `POST /api/deployments/:id/resume` 엔드포인트 추가
  - xxxxxxx Dual-Update: xxxxxxx API Server + 멤버 클러스터 직접 변경을 동시에 수행하여 전파 지연 문제 해결 (#100 패턴 적용)
  - `ScaleServingDeployment()`, `CheckMemberDeploymentRunning()`, `IsServingDeploymentPaused()` xxxxxxx 클라이언트 함수 추가
  - Resume 후 xxxxxxx가 pending으로 보고할 때 멤버 클러스터를 직접 조회하여 running 상태로 보정
  - Reconciler에 paused 상태 교차 검증 (DB ↔ K8s annotation) 및 외부 변경 감지 로직 추가

- **Frontend**:
  - 인스턴스 목록 드롭다운 메뉴에 일시정지/재시작 액션 추가
  - 인스턴스 상세 페이지 일시정지/재시작 버튼
  - 대시보드 파이 차트에 paused 카운트 반영
  - `deployment-status.ts`에 paused 배지 스타일 추가

- **Docs**:
  - `xxxxxxx_STATE_MANAGEMENT.md`: xxxxxxx 상태 관리 패턴 아키텍처 문서 (Dual-Update, Direct Status Verification, Cross-Validation)

**의사결정 사항**:

| 결정 | 내용 |
|------|------|
| 스케일 방식 | `replicas: 0` 기반 일시정지 — Deployment/Pod을 삭제하지 않으므로 Resume 시 빠른 복구 가능 |
| xxxxxxx 상태 보정 | Resume 후 멤버 클러스터 직접 조회로 running 보정 — xxxxxxx 전파 지연(수 초) 동안 UI에 pending 표시 방지 |
| Reconciler 교차 검증 | DB `paused` 상태 ↔ K8s annotation 비교 — 외부에서 수동 스케일 변경 시 감지 |

**관련 커밋**: `05eaf73` (PR #189, Closes #185)

---

### 2. 모니터링 인프라 구축 — Prometheus + DCGM Exporter 배포 (#196 → PR #199)

**구현 완료**: Control Plane에 Prometheus Server를 Helm subchart로 배포하고, 멤버 클러스터에 Prometheus Agent/DCGM Exporter/Node Exporter를 배포하여 멀티클러스터 메트릭 수집 파이프라인의 기반 인프라를 구축

**주요 성과**:

- **Helm Chart**:
  - `charts/h-mas/Chart.yaml`에 Prometheus subchart dependency 추가 (`condition: prometheus.enabled`)
  - `values.yaml`: 프로덕션 기본값 (retention 15d, PV 50Gi, remote-write-receiver, servicePort 9090)
  - `values-dev.yaml`: 개발환경 오버라이드 (retention 3d, PV 비활성화, NodePort 30090)

- **Backend**:
  - `PROMETHEUS_URL` 환경변수 주입 (Backend Deployment 템플릿)
  - `CreateServingDeployment()`에 Prometheus scrape annotation 자동 추가 (`prometheus.io/scrape`, `/port`, `/path`)

- **멤버 클러스터**:
  - `charts/prometheus-agent-values.yaml`: Prometheus 3.x Agent 모드 (`--agent` 플래그), Node Exporter 포함
  - Makefile: `deploy-prometheus-agent`, `deploy-dcgm`, `undeploy-monitoring` 타겟 추가
  - `deploy-dcgm` annotation 타입 버그 수정 (`--set` → `--set-string`)

- **Docs**:
  - `MONITORING_ARCHITECTURE.md`, `MONITORING_SETUP_GUIDE.md` 업데이트

**검증 결과 (xxxstack + research 클러스터 — RTX 5060 Ti)**:

| 항목 | 결과 |
|------|------|
| Prometheus Server 배포 + PromQL 쿼리 | v |
| Prometheus Agent → Remote Write → Central Prometheus 메트릭 도착 | v |
| vLLM 런타임 메트릭 수집 (97종) | v |
| DCGM GPU 메트릭 (사용률, 온도, 전력, 메모리) | v |
| Node Exporter (메모리, 디스크) | v |

**관련 커밋**: `a53e1d6`, `2def76c`, `15289a1`, `6d43c59`, `4f1225f`, `b4558dc` (PR #199, Closes #196)

---

### 3. Backend 메트릭 조회 API 구현 — PromQL 연동 (#197 → PR #200)

**구현 완료**: Prometheus HTTP API 클라이언트를 통한 서빙 런타임/GPU 메트릭 조회 REST API를 구현하고, 런타임별(vLLM, TGI, llama.cpp) 메트릭 이름을 통합 메트릭으로 정규화

**주요 성과**:

- **Backend (신규 파일 6개)**:
  - `internal/metrics/client.go`: Prometheus HTTP API 래퍼 (InstantQuery, RangeQuery)
  - `internal/metrics/queries.go`: 런타임별 PromQL 매핑 + 통합 메트릭 정의
  - `internal/metrics/service.go`: 비즈니스 로직 (DB 조회 → PromQL 구성 → 정규화)
  - `internal/metrics/middleware.go`: Echo 미들웨어 (HTTP 메트릭 수집) + `/metrics` 핸들러
  - `internal/handler/metrics.go`: API 핸들러 + Swagger 주석
  - `internal/model/metrics.go`: API 응답 구조체

- **API 엔드포인트**:

  | 엔드포인트 | 설명 |
  |-----------|------|
  | `GET /api/deployments/:id/metrics` | 배포별 실시간 메트릭 스냅샷 |
  | `GET /api/deployments/:id/metrics/history` | 시계열 메트릭 이력 |
  | `GET /api/clusters/:name/gpu-metrics` | 클러스터별 GPU 메트릭 요약 |
  | `GET /metrics` | Backend 자체 Prometheus exposition |

- **Docs**:
  - `PROMETHEUS_RECORDING_RULES_DESIGN.md`: v1.0 이후 메트릭 쿼리 최적화를 위한 Recording Rules 설계

**의사결정 사항**:

| 결정 | 내용 |
|------|------|
| Prometheus 라이브러리 | `prometheus/client_golang` 전체 — API 클라이언트 + instrumentation 동시 해결 |
| 배포→메트릭 매핑 | DB 조회 → K8s 레이블 기반 PromQL — 추가 레이블 없이 즉시 동작 |
| 캐싱 | 없음 (v0.9 규모에서 불필요) — Recording Rules를 향후 검토 문서로 정리 |
| Prometheus 미연결 시 | 503 + `PROMETHEUS_NOT_CONFIGURED` — 프론트엔드에서 안내 UI 분기 가능 |

**관련 커밋**: `a96b2f5`, `b0bc9ea`, `675e060` (PR #200, Closes #197)

---

### 4. 프론트엔드 모니터링 실데이터 연동 및 대시보드 개선 (#198 → PR #201)

**구현 완료**: SWR 기반 실시간 폴링으로 배포별 13개 메트릭 지표를 게이지 및 시계열 차트로 표시하고, 대시보드에 서빙 성능 요약 카드·비정상 인스턴스 경고 배너·클러스터 GPU 상태 도트 히트맵을 추가

**주요 성과**:

- **Frontend (신규 파일 13개)**:
  - `lib/api/metrics.ts`: 메트릭 API 클라이언트 + TypeScript 타입
  - `lib/hooks/use-deployment-metrics.ts`: 배포별 메트릭 SWR 훅
  - `lib/hooks/use-cluster-gpu-metrics.ts`: 클러스터 GPU 메트릭 SWR 훅
  - `lib/hooks/use-dashboard-metrics.ts`: 대시보드 GPU 사용률 + 클러스터별 GPU 상세 훅
  - `lib/hooks/use-aggregated-metrics.ts`: running 배포 메트릭 집계 훅
  - `components/metrics/deployment-metrics-panel.tsx`: 게이지 + 시계열 차트 통합 패널
  - `components/metrics/metric-gauge.tsx`: 개별 메트릭 게이지 컴포넌트
  - `components/metrics/metrics-time-series-chart.tsx`: Recharts 시계열 차트
  - `components/metrics/metrics-status-bar.tsx`: 인스턴스 상세용 컴팩트 메트릭 바
  - `components/metrics/cluster-gpu-metrics-panel.tsx`: 클러스터 GPU 패널
  - `components/metrics/monitoring-not-configured.tsx`: Prometheus 미설정 폴백 UI
  - `components/dashboard/serving-performance-summary.tsx`: 서빙 성능 요약 카드
  - `components/dashboard/unhealthy-instances-banner.tsx`: 비정상 인스턴스 배너

- **대시보드 개선**:
  - StatsCards: GPU 실사용률 (Prometheus 연결 시) / GPU 할당률 (미연결 시) 자동 전환
  - ServingPerformanceSummary: 총 RPS, 총 TPS, 평균 TTFT, 활성 요청, 최대 KV Cache, 평균 GPU 온도
  - ClusterOverview: 클러스터별 GPU 온도 점 표시
  - 사이드바 메뉴명 "로그" → "모니터링" 변경

- **모니터링 페이지**:
  - 풀 메트릭 패널 + Pod 로그/이벤트 사이드바 통합 레이아웃
  - 인스턴스 상세 페이지에 컴팩트 MetricsStatusBar 배치
  - SWR polling: 스냅샷 10초, 히스토리 60초, 대시보드 GPU 30초, 집계 메트릭 15초

**관련 커밋**: `2589b9b` (PR #201, Closes #198)

---

### 5. Latency p95/p99 분할 표시 및 Ollama 런타임 미지원 안내 (#133 → PR #202)

**구현 완료**: TTFT, E2E Latency, ITL에 대해 p95/p99 퍼센타일 분할 표시를 추가하고, Ollama 런타임의 추론 메트릭 미지원 안내 UI를 구현하여 #133 대시보드 메트릭 이슈를 완료

**주요 성과**:

- **Backend**:
  - `queries.go`에 p95/p99 PromQL `histogram_quantile` 쿼리 추가 (vLLM 전체, TGI는 TTFT만)
  - `MetricsSnapshot` 모델에 6개 필드 추가 (`ttftP95`, `ttftP99`, `e2eLatencyP95`, `e2eLatencyP99`, `itlP95`, `itlP99`)
  - `runtimeQueries`에 `"ollama"` 엔트리 추가 — GPU 메트릭만 포함

- **Frontend**:
  - `MetricGauge`에 `percentiles` prop 추가 — 게이지 하단에 p95/p99 서브텍스트 + 임계값 초과 시 색상 강조
  - 시계열 차트에 E2E Latency p95(긴 점선), p99(짧은 점선) 라인 추가
  - Ollama 배포 선택 시 추론 메트릭 영역을 안내 메시지로 교체 (GPU 메트릭은 정상 표시)

**버그 수정 (실 환경 테스트 중 발견)**:

| 문제 | 원인 | 해결 |
|------|------|------|
| Ollama 배포에서 메트릭 히스토리 페이지 크래시 | PromQL 매핑이 없는 런타임에서 `timestamps: null` 반환 → `null.length` 접근 | null 가드 추가 |
| Ollama 배포에서 GPU 메트릭 미표시 | `runtimeQueries`에 `"ollama"` 키 부재 → `BuildQuery`가 GPU 포함 모든 쿼리 스킵 | GPU 전용 ollama 엔트리 추가 |

**관련 커밋**: `903106c`, `01f1a9d`, `6ee1328`, `8e45905` (PR #202, Closes #133)

---

## 문서화 (Documentation)

### 1. 브랜치 전략 및 릴리즈/배포 프로세스 정립 (#166 → PR #203)

**작성 완료**: 노션 정리 내용 + 이슈 검토 항목 + 기존 프로젝트 문서를 종합 분석하여 H-MAS 프로젝트 맞춤 전략 수립

| 문서 | 내용 | 관련 커밋 |
|------|------|-----------|
| `docs/operations/BRANCH_RELEASE_STRATEGY.md` | 신규 — 14개 섹션, `dev/stg/prd` 3-브랜치 모델, SemVer + VERSION 파일, 릴리즈 체크리스트, 배포/롤백, CI/CD 확장, Feature Flag, 장애 대응 체크리스트, 도입 시점 판단 기준 | `ac3ce99` (PR #203) |

**문서 구성 (14개 섹션)**:
1. 배경 — 현재 전략 vs v0.9 변화 요인, 브랜치 모델 비교 및 채택 근거
2. 브랜치 정책 — 상시 브랜치(dev/stg/prd) + 임시 브랜치, 네이밍 규칙, 보호 규칙
3. 개발-배포 흐름 — 평소 개발 → 릴리즈 준비 → 운영 배포 → 긴급 대응 다이어그램
4. 버전 정책 — SemVer, `dev.N` / `rc.N` 표기, VERSION 파일 운영
5. 릴리즈 프로세스 — 체크리스트 템플릿, 릴리즈 노트 규칙
6. 배포 전략 — 파트너별 Helm values, 업그레이드/롤백, DB migration 원칙
7. CI/CD 확장 — 현재(v0.4) → 목표(v0.9) 워크플로, main→dev 전환 계획
8. 버전 지원 정책 — EOL 기준, 구버전 예외 패치 절차
9. 라이센스와 버전 지원 — 연간 라이센스-지원 범위 관계
10. Feature Flag 운영 — 환경별 flag 정책, 관리 규칙
11. 커밋/PR 규칙 — Conventional Commits, PR 컨벤션
12. 장애 대응 체크리스트 — 1차 확인 → 즉시 조치 → 수정/배포 흐름
13. 도입 시점 판단 기준 — 즉시 / 파트너 2곳+ / 팀원 2명+ 단계별 도입
14. 운영 원칙 요약 — 핵심 규칙 한눈에 보기

### 2. 아키텍처/설계 문서

| 문서 | 내용 | 관련 커밋 |
|------|------|-----------|
| `docs/architecture/MONITORING_ARCHITECTURE.md` | 모니터링 아키텍처 설계 — Prometheus 경유 결정, 멀티클러스터 Remote Write, 메트릭 정규화 | `206cb7a` |
| `docs/architecture/INFERENCE_PROXY_DESIGN.md` | 추론 API 프록시 설계 — Phase 1 Push/Phase 2 Pull, 라우팅, SSE 스트리밍 | `a01a9cd` |
| `docs/architecture/xxxxxxx_STATE_MANAGEMENT.md` | xxxxxxx 상태 관리 패턴 — Dual-Update, Direct Status Verification, Cross-Validation | `05eaf73` (PR #189) |
| `docs/research/PROMETHEUS_RECORDING_RULES_DESIGN.md` | Recording Rules 사전 집계 설계 — v1.0 이후 메트릭 쿼리 최적화 | `675e060` (PR #200) |
| `docs/product/V09_IMPLEMENTATION_PLAN.md` | v0.9 구현 계획 문서 | `efe9d00` |

### 3. 기존 문서 최신화

| 문서 | 내용 | 관련 커밋 |
|------|------|-----------|
| `README.md` | 전체 최신화 — 누락 문서 반영 및 v0.4 이후 현황 업데이트 | `fc81be4` |
| `MONITORING_ARCHITECTURE.md`, `MONITORING_SETUP_GUIDE.md` | 실제 구현과 일치하도록 업데이트 | `6d43c59` (PR #199) |
| 전체 문서 (다수) | v0.4 이후 기능 고도화 반영 — 전체 문서 최신화 | `875c284` |

---

## 리팩토링 (Refactoring)

### 1. GP 클러스터 타입 표시명 변경

- GP 클러스터 타입 표시명 `"General"` → `"General Performance"` 일괄 변경
- **관련 커밋**: `16e8d49`


---

## 이전 Iteration 계획 달성도

Iteration 17에서 계획한 4개 항목 **1개 완료, 1개 부분 달성, 2개 미착수** — 모니터링 파이프라인(#127 → #196/#197/#198) 전체 구축 및 Pause/Resume(#185) 구현에 집중:

| 계획 | 이슈 | 상태 | 비고 |
|------|------|------|------|
| 서빙 배포 일시정지/재시작 기능 설계 | #185 | 완료 | PR #189로 설계 + 구현 + 테스트 완료, xxxxxxx 상태 관리 패턴 문서까지 작성 |
| 추론 API 프록시 구현 — 통합 엔드포인트 설계 | #36 | 부분 달성 | 설계 문서 `INFERENCE_PROXY_DESIGN.md` 작성 완료, 구현 이슈 4건(#191, #192, #193, #194) + UI 이슈(#195)로 분해 |
| 서빙 배포 인플레이스 업데이트 API (PATCH) 구현 착수 | #125 | ⏸ 미착수 | 모니터링 파이프라인 구축에 우선순위 전환 |
| xxxxxxxxxxx 스케줄러 리서치 진행 | #179 | ⏸ 미착수 | 모니터링 파이프라인 구축에 우선순위 전환 |

**추가 달성**: 계획에 없던 모니터링 파이프라인 전체 구축(#127의 서브이슈 #196→#197→#198 순차 완주), 대시보드 Latency p95/p99 + Ollama 안내(#133 완료), 브랜치/릴리즈 전략 문서(#166 완료), v0.9 구현 계획 문서, 아키텍처 설계 문서 3건, h-mas-agent 이슈(#190) 생성. 목 데이터 기반이던 대시보드를 Prometheus 실데이터 기반으로 전면 전환하여 운영 가시성을 확보.

---

## 미해결 이슈 (Open Issues)

### 신규 이슈 (9개, 이번 Iteration 생성)

| 이슈 | 제목 | 라벨 |
|------|------|------|
| #195 | 프론트엔드: 추론 테스트 UI (Chat 인터페이스) | frontend |
| #194 | 추론 API 키 인증 및 Rate Limiting | backend, frontend |
| #193 | 추론 요청 로깅 및 메트릭 수집 파이프라인 | backend |
| #192 | 추론 프록시 Phase 2: 멀티클러스터 라우팅 (agent 터널 통합) | backend, agent |
| #191 | 추론 프록시 Phase 1: 기본 프록시 구현 (Push 모드) | backend |
| #190 | h-mas-agent 핵심 컴포넌트 구현 | agent |

> #191→#192→#193→#194는 #36(추론 API 프록시)의 단계별 구현 이슈 체인으로, 이번 iteration에 `INFERENCE_PROXY_DESIGN.md` 설계 문서를 기반으로 분해

### 런타임 자동 튜닝 관련 선행 조건 이슈 체인 (5개)

| 이슈 | 제목 | 역할 |
|------|------|------|
| #125 | 서빙 배포 인플레이스 업데이트 API (PATCH) | 파라미터 변경 API |
| #126 | 서빙 배포 파라미터 변경 시 Rolling Restart 파이프라인 | 안전한 재시작 |
| #127 | ~~서빙 런타임 메트릭 수집 파이프라인 구축~~ | **이번 iteration 완료** |
| #128 | 서빙 최적화 파라미터 변경 이력 관리 및 감사 로그 | 변경 추적/롤백 |
| #114 | 런타임 자동 튜닝 (메트릭 기반 파라미터 자동 조정) | 최종 목표 — 위 4개 이슈 모두 선행 |

### 프리셋 확장 이슈 (3개)

| 이슈 | 제목 |
|------|------|
| #120 | DB 기반 최적화 프리셋 관리 (CRUD API + UI) |
| #122 | 사용자 정의 최적화 프리셋 CRUD (DB 기반) |
| #123 | 프리셋 벤치마크 인프라 및 GPU 스펙별 최적화 값 분기 |

### 대시보드 v0.9 이슈 (1개)

| 이슈 | 제목 |
|------|------|
| #134 | 대시보드 GPU 토폴로지 컴팩트 위젯 실데이터 연동 (h-mas-agent) |

> #133(대시보드 실시간 GPU 사용률 및 추론 메트릭)은 **이번 iteration PR #202로 완료**

### 리서치 이슈 (4개, Iteration 17에서 생성)

| 이슈 | 제목 |
|------|------|
| #186 | 리서치: 서버리스 서빙(Scale-to-Zero) 방식 분석 및 H-MAS 적용 가능성 검토 |
| #184 | 리서치: 폐쇄망(Air-Gapped) 환경 배포 전략 수립 |
| #179 | 리서치: xxxxxxxxxxx 스케줄러 도입을 위한 기술 조사 및 통합 설계 |
| #96 | 리서치: GPU 메모리 인지형 동적 모델 Bin Packing 전략 |

### 기존 미해결 이슈 (이전 Iteration에서 이관)

### 1. 스케줄러 어노테이션 변환 (xxx/xxxxxxxxxxx → xxxxxxx OverridePolicy) (#66)

- #179 xxxxxxxxxxx 리서치 이슈와 연계하여 검토 예정

### 2. 모델 프리셋 리스트 최신화 (#62)

- 잔여 작업: 최신 모델 시드 데이터 추가 필요

### 3. Failover / 재배치 정책 구현 (#39) — v0.95 마일스톤

### 4. NUMA / 토폴로지 인식 스케줄링 구현 (#38) — v0.95 마일스톤

### 5. 추론 API 프록시 구현 — 통합 엔드포인트 (#36)

- 설계 문서 완료, Phase 1~4 이슈(#191~#194)로 분해 완료, Phase 1 구현 착수 예정

---

## 이번 주 통계

| 항목 | 수치 |
|------|------|
| 총 커밋 수 | 23개 (+ merge 5건) |
| 머지된 PR | 6개 (#189, #199, #200, #201, #202, #203) |
| 생성된 이슈 | 9개 (#190, #191, #192, #193, #194, #195, #196, #197, #198) |
| 해결된 이슈 | 7개 (#127, #133, #166, #185, #196, #197, #198) |
| 미해결 이슈 | 23개 (#36, #38, #39, #62, #66, #96, #114, #120, #122, #123, #125, #126, #128, #134, #179, #184, #186, #190, #191, #192, #193, #194, #195) |
| 신규 기술 문서 | 6개 (브랜치/릴리즈 전략, 모니터링 아키텍처, 추론 프록시 설계, xxxxxxx 상태 관리, Recording Rules, v0.9 구현 계획) |
| 코드 변경량 | +5,849줄 / -502줄 (85개 파일) |

---

## 결론

이번 주는 **서빙 런타임 메트릭 수집 파이프라인(#127)을 인프라→Backend→Frontend 3단계로 1주일 만에 완주하여 운영 모니터링 가시성을 확보한 기간**이었습니다. Prometheus Server/Agent/DCGM Exporter 기반 멀티클러스터 메트릭 인프라(#196)를 구축하고, PromQL 연동 Backend API(#197)로 런타임별 메트릭을 정규화하며, 13개 신규 프론트엔드 컴포넌트(#198)로 목 데이터를 실데이터로 전면 교체했습니다.

배포 일시정지/재시작(#185)에서는 xxxxxxx Dual-Update 패턴을 적용하여 전파 지연 문제를 해결했고, 이를 `xxxxxxx_STATE_MANAGEMENT.md` 패턴 문서로 정리하여 향후 유사 기능 구현의 참조 아키텍처를 마련했습니다. Latency p95/p99 분할 표시 및 Ollama 런타임 안내(#133)로 대시보드 메트릭 이슈를 마무리하고, 브랜치/릴리즈 전략 문서(#166)로 장기 미해결 운영 이슈를 클로즈했습니다.

추론 API 프록시(#36)는 설계 문서와 4단계 구현 이슈로 분해하여 Phase 1 착수를 준비했고, h-mas-agent(#190) 이슈를 생성하여 Pull 모드 클러스터 지원의 기반을 마련했습니다.

**핵심 성과**:
1. **메트릭 수집 파이프라인 완주**: #127(상위 이슈) + 3개 서브이슈(#196→#197→#198) — Prometheus 인프라, Backend PromQL API, Frontend 13개 컴포넌트
2. **대시보드 Latency p95/p99**: 게이지 퍼센타일 분할 + 시계열 점선 + Ollama 안내 — #133 완료
3. **배포 일시정지/재시작**: xxxxxxx Dual-Update + Reconciler 교차 검증 — #185 완료
4. **브랜치/릴리즈 전략**: 14개 섹션, 3-브랜치 모델 + SemVer + 장애 대응 체크리스트 — #166 완료
5. **추론 프록시 설계**: `INFERENCE_PROXY_DESIGN.md` + Phase 1~4 이슈 분해 — #36 착수 준비 완료
6. **v0.9 방향 구체화**: 구현 계획 문서, 아키텍처 설계 문서 3건, h-mas-agent 이슈 생성

**다음 주 계획**:
- 추론 프록시 Phase 1 구현 착수 (#191) — Push 모드 기본 프록시
- 서빙 배포 인플레이스 업데이트 API (PATCH) 구현 (#125)
- 추론 테스트 UI (Chat 인터페이스) 구현 (#195)
- h-mas-agent 설계 상세화 (#190)

---

**문서 작성일**: 2026년 4월 19일


