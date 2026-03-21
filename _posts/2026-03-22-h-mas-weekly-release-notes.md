---
title: 'H-MAS release notes (2026-03-15~03-21)'
date: 2026-03-22
permalink: /posts/2026/03/h-mas-weekly-release-notes-0315-0321/
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
## 2026년 3월 15일 - 3월 21일 (Iteration 14)

### 주요 개발 내용 요약

이번 주는 **운영 안정성 버그 3건 수정**, **Model Registry 풀스택 구현(데이터 통합→Backend API→CRUD UI→런타임 호환성)**, **런타임 최적화 파라미터 시스템 전체 구축(ParamSpec 스키마→적용 엔진→시나리오 프리셋)**에 중점을 둔 기간이었습니다. 총 **19개의 커밋**, **12개의 PR 머지**, **27개의 이슈 생성**이 완료되었으며, Iteration 13에서 완료한 정책-배포 구조 재설계에 이어 사용자 경험의 핵심 축인 "모델 관리"와 "서빙 최적화"를 본격적으로 구현하여 v0.4 Developer Preview의 기능 완성도를 크게 끌어올린 주간이었습니다.

{% include youtube.html id="oCWUrJ3vTvU" %}


---

## 버그 수정 (Bug Fixes)

### 1. 배포 시 클러스터의 실제 scheduler-type 기반으로 schedulerName 결정

**수정 완료**: `DefaultSchedulerForClusterType()`이 cluster-type만 보고 `schedulerName`을 하드코딩하여, `scheduler-type: default`인 클러스터에 `xxxxxxxxxxx`를 설정해 Pod이 무한 Pending되는 치명적 버그 수정

**수정 내용**:
- `handler/deployment.go`에 `clusterMetaRepo` 의존성 추가, `DefaultSchedulerForClusterType()` 호출 제거 → DB(`cluster_metas.scheduler_type`) 기반 조회로 교체
- 클러스터 미특정(Label 기반 매칭) 시 안전하게 `schedulerName`을 설정하지 않음 (기본 K8s 스케줄러 사용)
- `cmd/server/main.go`에서 `NewDeploymentHandler()` 호출 시 `clusterMetaRepo` 전달

**관련 커밋**: `7393105` (PR #99, Closes #98)

---

### 2. 배포 삭제 시 orphan 리소스 방지 및 ClusterName 자동 기록

**수정 완료**: 배포 삭제 시 xxxxxxx 리소스만 삭제되고 멤버 클러스터 리소스가 고아 상태로 남는 레이스 컨디션 해결

**수정 내용**:
- 삭제 순서를 **xxxxxxx API 서버 워크로드 삭제 → 멤버 클러스터 직접 삭제(safety net) → Policy 삭제**로 변경
- `resolveTargetClusters`: `ClusterName`이 비어있을 때 xxxxxxx ResourceBinding을 조회하여 실제 배포된 클러스터 파악
- `WaitForTargetCluster`: 배포 시점에 ResourceBinding 폴링(최대 5초)으로 실제 클러스터를 DB에 자동 기록 — `targetClusterType`/배치 전략 기반 배포에서도 `ClusterName`이 항상 저장됨

**관련 커밋**: `865c829` (PR #101, Closes #100)

---

### 3. 인스턴스 이벤트 조회 시 Pod 레이블 셀렉터 불일치 수정

**수정 완료**: 인스턴스 상세 페이지의 이벤트 섹션이 항상 "이벤트가 없습니다."로 표시되던 버그 수정

**수정 내용**:
- `GetServingDeploymentEvents`에서 Pod 조회 시 `app` 레이블을 사용했으나, 실제 Pod에는 `app.kubernetes.io/name` 레이블만 존재
- 레이블 셀렉터를 `app=%s` → `app.kubernetes.io/name=%s`로 변경

**관련 커밋**: `427fdb4` (PR #103, Closes #102)

---

## 새로운 기능 (New Features)

### 1. Model Registry 풀스택 구현 — 데이터 통합 → Backend API → CRUD UI

**구현 완료**: 배포 메뉴(`MODEL_PRESETS`)와 모델 저장소(`mockModels`)가 별개 하드코딩 데이터를 사용하던 문제를 해결하고, DB 기반 Model Registry를 풀스택으로 구현

**주요 성과**:

**1단계 — 데이터 통합 (#105 → PR #109)**:
- `Model` 인터페이스에 `ServingConfig` 추가하여 카탈로그 정보와 서빙 설정을 분리된 관심사로 관리
- 배포 메뉴의 `MODEL_PRESETS` 제거 → `mockModels` 참조로 통합 + 임베딩 모델 2종(BGE-M3, E5-Mistral-7B) 추가 → 총 17종
- 모델 카드 "배포하기" 클릭 → `/deploy?model={id}` query param 실제 연동

**2단계 — Backend API (#106 → PR #115)**:
- DB 마이그레이션: `models` + `serving_configs` 테이블 + 17개 시드 데이터
- CRUD API 7개 엔드포인트:
  ```
  GET/POST /api/v1/models
  GET/PUT/DELETE /api/v1/models/:id
  GET/PUT /api/v1/models/:id/serving-config
  ```
- 프론트엔드 `mockModels` → `useModels()` 훅 + `fetchModels()` API 호출로 전환

**3단계 — CRUD UI (#107 → PR #116)**:
- 모델 등록: 2단계 위자드 Dialog (기본 정보 → 서빙 설정), HuggingFace "정보 가져오기" 자동 채움, 파라미터 수 기반 리소스 자동 추천, VRAM 추정 배너
- 모델 수정: 기존 데이터 pre-fill, 생성/수정 모드 통합
- 모델 삭제: `AlertDialog` 확인 + 배포 중인 모델 삭제 차단(409 Conflict)
- 모델 카드에 DropdownMenu(⋮) 추가 + 등록 성공 toast에 "배포하기" 액션 링크

**의사결정 사항**:

| 결정 | 내용 |
|------|------|
| DB 스키마 분리 | `models` + `serving_configs` 2테이블 — 모델 메타데이터와 서빙 설정의 관심사 분리 |
| 시드 데이터 | 17개 프리셋 모델을 마이그레이션 시드로 포함 — 설치 즉시 사용 가능 |
| 삭제 차단 | 배포 중인 모델 삭제 시 409 Conflict — `CountDeploymentsByModelName` 패턴 (ResourceProfile 삭제와 일관) |

**관련 커밋**: `9e7d641` (PR #109, Closes #105), `85393f6` (PR #115, Closes #106), `6d5652a` (PR #116, Closes #107)

---

### 2. 런타임-모델 카테고리 호환성 관리 및 TEI 런타임 추가

**구현 완료**: 런타임별 서빙 가능한 모델 카테고리를 체계적으로 관리하고, 임베딩 전용 TEI(Text Embeddings Inference) 런타임 추가

**주요 성과**:
- **런타임-카테고리 호환성 매트릭스**: `runtime.Config`에 `SupportedCategories` 필드 추가 — Single Source of Truth
- **TEI 런타임 Preset 추가**: 임베딩 전용, `/v1/embeddings` API
- **런타임 ID 통일**: 백엔드 `llama-cpp` → `llamacpp`으로 변경 (프론트엔드/DB와 일치)
- **배포 시 이중 가드**: 프론트엔드(비호환 런타임 비활성화 + "비호환" 뱃지) + 백엔드(`POST /api/deployments`에서 카테고리 검증)

| 런타임 | LLM | Code | Vision | Embedding |
|--------|-----|------|--------|-----------|
| Ollama | v | v | - | v |
| vLLM | v | v | v | v |
| TGI | v | v | - | - |
| **TEI** | - | - | - | **v** |
| llama.cpp | v | v | - | - |

**관련 커밋**: `3a02682` (PR #117, Closes #108)

---

### 3. 런타임 최적화 파라미터 시스템 (ParamSpec → 적용 엔진 → 프리셋)

이번 Iteration에서 런타임 최적화 파라미터의 전체 파이프라인을 4단계로 구축:

#### 3-1. 런타임 Config 자기 완결화 (#110 → PR #118)

**리팩토링 완료**: `runtime.Config`에 `CacheMountPath`, `HealthProbe(ProbeConfig)` 필드를 추가하여 런타임 외부의 하드코딩 분기(switch문)를 Config 내부로 흡수

- `modelCacheMountPath()` 함수 삭제, 런타임 ID 기반 switch문 전량 제거
- TEI 런타임의 캐시 마운트 경로 누락 버그도 함께 수정
- **목표 달성**: 새 런타임 추가 시 `Presets()`에 Config 선언 하나만 추가하면 되는 구조

**관련 커밋**: `468fd9d` (PR #118, Closes #110)

#### 3-2. 런타임 최적화 파라미터 스키마 — ParamSpec + SemanticKey (#111 → PR #119)

**구현 완료**: 4개 런타임(vLLM, TGI, Ollama, llama.cpp)의 주요 최적화 파라미터 37개를 선언적 스키마(ParamSpec)로 정의하고, SemanticKey 9개로 런타임 간 크로스 매핑

**주요 성과**:
- 6개 파일 신규 생성: `paramspec.go`, `params_vllm.go`, `params_tgi.go`, `params_ollama.go`, `params_llamacpp.go`, `handler/runtime.go`
- `RuntimeHandler` 분리 — API 3개 엔드포인트:
  - `GET /api/runtimes` (optimizationParams 포함)
  - `GET /api/runtimes/{id}/params` (카테고리 필터)
  - `GET /api/runtimes/semantic-keys` (크로스 런타임 매핑)

**SemanticKey 매핑 테이블**:

| SemanticKey | vLLM | TGI | Ollama | llama.cpp |
|-------------|------|-----|--------|-----------|
| `max_concurrent` | `max_num_seqs` | `max_concurrent_requests` | `num_parallel` | `parallel` |
| `max_context` | `max_model_len` | `max_total_tokens` | `num_ctx` | `ctx_size` |
| `gpu_memory_util` | `gpu_memory_utilization` | — | `num_gpu` | `n_gpu_layers` |
| `tensor_parallel` | `tensor_parallel_size` | `num_shard` | — | — |
| `prefix_caching` | `enable_prefix_caching` | — | — | `flash_attn` |
| `max_batch_tokens` | `max_num_batched_tokens` | `max_batch_prefill_tokens` | — | `batch_size` |
| `quantization_mode` | `quantization` | `quantize` | — | — |
| `cpu_threads` | — | — | `num_threads` | `threads` |
| `chunked_prefill` | `enable_chunked_prefill` | — | — | — |

**관련 커밋**: `1f5e18f` (PR #119, Closes #111)

#### 3-3. 런타임 최적화 파라미터 적용 엔진 (#112 → PR #121)

**구현 완료**: 2-Tier + Raw Override 구조의 OptimizationConfig 도입 — 기존 `EnvVars` 필드 완전 대체

**주요 성과**:
- **2-Tier + Raw Override 아키텍처**:
  - Tier 1 (Common): SemanticKey 기반 cross-runtime 공통 설정
  - Tier 2 (Native): 런타임 고유 파라미터명 직접 지정
  - Raw Override: `extraArgs` / `extraEnv` 자유 입력
- **ResolveOptimization 엔진** (`resolve.go`): Common → Native → Raw 우선순위 해석, 충돌 감지, 타입/범위 검증, denylist 적용
- **DB 마이그레이션**: `env_vars` → `optimization_config` 데이터 이관 (up/down, NULL/edge case 안전 처리)
- **프론트엔드 OptimizationCard**: Progressive Disclosure UI, ParamSpec 기반 동적 입력 렌더링, CPU/GPU 모드별 파라미터 필터링, 클라이언트 충돌 감지
- **ParamSpec에 `gpuOnly`/`cpuOnly` 플래그** 추가 (vLLM, Ollama, llama.cpp)

**의사결정 사항**:

| 결정 | 내용 |
|------|------|
| 우선순위 체인 | Raw Override > Tier 2 (Native) > Tier 1 (Common) > Preset 기본값 — 숙련도별 Progressive Override |
| 기존 데이터 호환 | `env_vars` → `optimization_config` 마이그레이션으로 기존 배포 데이터 무손실 전환 |
| 충돌 감지 | Tier 1과 Tier 2가 동일 SemanticKey를 설정하면 경고 — 사용자가 의도적 override인지 확인 |

**관련 커밋**: `4c9fe08` (PR #121, Closes #112)

#### 3-4. 시나리오 기반 런타임 최적화 프리셋 (#113 → PR #124)

**구현 완료**: 5개 시나리오 × 3개 모델 크기(소형/중형/대형) 조합의 내장 최적화 프리셋 15개 구축 + Backend API + 배포 폼 UI

**주요 성과**:
- **프리셋 15개 구축**:
  | 시나리오 | 최적화 방향 |
  |---------|-----------|
  | 고 동시성 챗봇 | throughput 극대화, prefix_caching ON |
  | 긴 문서 처리/요약 | max_context ↑, gpu_memory_util 보수적 |
  | 코드 생성 | 낮은 latency, 중간 컨텍스트 |
  | RAG 파이프라인 | prefix 캐싱 극대화, 높은 throughput |
  | 단일 사용자 | latency 최소화 |
- **Backend**: `GET /api/optimization-presets`, `/{id}`, `/recommend` + `ResolveOptimization`에 Preset base layer (Tier 0) 통합
- **Frontend**: OptimizationCard에 시나리오 카드 클릭 UI → 모델 크기 자동 매칭 → Tier 1 자동 채움 → 부분 오버라이드 지원
- **우선순위 체인 완성**: Preset (Tier 0) → Common (Tier 1) → Native (Tier 2) → Raw Override

**관련 커밋**: `f486e12` (PR #124, Closes #113)

---

## 리팩토링 (Refactoring)

### 1. 런타임 Config 자기 완결화 — 하드코딩 분기 제거

위 §3-1에서 상세 기술. `runtime.Config`에 `CacheMountPath`, `HealthProbe` 필드를 흡수하여 런타임 외부의 switch문을 전량 제거.

**관련 커밋**: `468fd9d` (PR #118, Closes #110)

---

## 문서화 (Documentation)

### 1. xxxxxxxx 과제 제안서 초안

| 문서 | 내용 | 관련 커밋 |
|------|------|-----------|
| `docs/proposals/2026-04-xxxxxxxxxxxx/PROPOSAL.md` | xxxxxxx 과제 제안서 전체 8개 섹션 초안 | `03d86fd` (PR #104, Closes #97) |
| `docs/proposals/README.md` | proposals 디렉토리 설명 | `03d86fd` |
| `docs/INDEX.md` | proposals 섹션 추가 | `03d86fd` |

### 2. 제품 소개서 및 외부 자료

| 문서 | 내용 | 관련 커밋 |
|------|------|-----------|
| 제품 소개서 | Mermaid 다이어그램 적용 및 이미지 플레이스홀더 추가 | `55cefcf` |
| PPT 슬라이드 | PPT 슬라이드 구성안 추가 및 제품 소개서 범용화 | `5b55e6d` |
| 외부 자료 | 제품 소개서 및 파일럿 프로그램 외부 자료 추가 | `f67d6ac` |

### 3. 코드베이스 정비

| 작업 | 내용 | 관련 커밋 |
|------|------|-----------|
| README 최신화 | 프로젝트 구조, 문서 목록, 로드맵 체계 업데이트 | `8b76c0e` |
| 문서 디렉토리 정리 | 단수형 통일, `outreach` → `external` 리네임 | `de8e2da` |
| 이슈 트래킹 | 대시보드 실데이터 연동 이슈 6개(#129~#134) 추적 문서 추가 | `0125d7d` |



---

## 이전 Iteration 계획 달성도

Iteration 13에서 계획한 3개 항목 **1개 대체 달성, 2개 미착수**:

| 계획 | 이슈 | 상태 | 비고 |
|------|------|------|------|
| 모델 프리셋 리스트 최신화 — 후속 세대 모델 교체 | #62 | 🔄 대체 달성 | 정적 프리셋 업데이트 대신 Model Registry 풀스택 구현(#105→#117)으로 근본적 해결. DB 기반 동적 관리 체계 확보 |
| 추론 API 프록시 구현 — 통합 엔드포인트 설계 착수 | #36 | ⏸ 미착수 | 런타임 최적화 시스템 구축에 우선순위 배분 |
| 스케줄러 어노테이션 변환 검토 | #66 | ⏸ 미착수 | 런타임 최적화 시스템 구축에 우선순위 배분 |

**추가 달성**: 계획에 없던 운영 버그 3건 수정(#98, #100, #102), 런타임 최적화 파라미터 시스템 전체 구축(#110→#113, 4개 이슈 연쇄 완료), TEI 런타임 추가 및 호환성 매트릭스(#108), 디딤돌 R&D 제안서 초안(#97), 제품 소개서 및 외부 자료 작성. 특히 런타임 최적화 파라미터 시스템은 계획 외 작업이었으나, Model Registry 구현 과정에서 배포 UX의 "최적화 설정" 공백이 드러나 착수하게 됨.

---

## 미해결 이슈 (Open Issues)

### 이번 Iteration에서 신규 생성된 미해결 이슈

#### 런타임 자동 튜닝 관련 선행 조건 이슈 체인 (5개)

| 이슈 | 제목 | 역할 |
|------|------|------|
| #125 | 서빙 배포 인플레이스 업데이트 API (PATCH) | 파라미터 변경 API |
| #126 | 서빙 배포 파라미터 변경 시 Rolling Restart 파이프라인 | 안전한 재시작 |
| #127 | 서빙 런타임 메트릭 수집 파이프라인 구축 (Prometheus 연동) | 메트릭 수집 인프라 |
| #128 | 서빙 최적화 파라미터 변경 이력 관리 및 감사 로그 | 변경 추적/롤백 |
| #114 | 런타임 자동 튜닝 (메트릭 기반 파라미터 자동 조정) | 최종 목표 — 위 4개 이슈 모두 선행 |

#### 프리셋 확장 이슈 (3개)

| 이슈 | 제목 |
|------|------|
| #120 | DB 기반 최적화 프리셋 관리 (CRUD API + UI) |
| #122 | 사용자 정의 최적화 프리셋 CRUD (DB 기반) |
| #123 | 프리셋 벤치마크 인프라 및 GPU 스펙별 최적화 값 분기 |

#### 대시보드 실데이터 연동 이슈 (6개)

| 이슈 | 제목 | 단계 |
|------|------|------|
| #129 | 대시보드 클러스터/GPU 통계 위젯 실데이터 연동 | v0.4 |
| #130 | 대시보드 배포 현황 위젯 실데이터 연동 | v0.4 |
| #131 | 대시보드 최근 활동 로그 위젯 감사 로그 API 연동 | v0.4 |
| #132 | 대시보드 미구현 위젯 placeholder 전환 | v0.4 |
| #133 | 대시보드 실시간 GPU 사용률 및 추론 메트릭 표시 | v0.9 |
| #134 | 대시보드 GPU 토폴로지 컴팩트 위젯 실데이터 연동 | v0.9 |

#### 리서치

- #96 — 리서치: GPU 메모리 인지형 동적 모델 Bin Packing 전략

### 기존 미해결 이슈 (이전 Iteration에서 이관)

### 1. 스케줄러 어노테이션 변환 (xxx/xxxxxxxxxxx → xxxxxxx OverridePolicy) (#66)

- Iteration 13에서도 미착수. 런타임 최적화 시스템 완성 후 검토 예정

### 2. 모델 프리셋 리스트 최신화 (#62)

- Model Registry(#105→#117) 구현으로 정적 프리셋 관리의 필요성이 크게 감소
- DB 기반 관리 체계에서 모델 추가/수정이 UI로 가능해짐
- 잔여 작업: 최신 모델(Llama 3.2, Gemma 2 등) 시드 데이터 추가 필요

### 3. Failover / 재배치 정책 구현 (#39) — v0.95 마일스톤

### 4. NUMA / 토폴로지 인식 스케줄링 구현 (#38) — v0.95 마일스톤

### 5. 추론 API 프록시 구현 — 통합 엔드포인트 (#36)

### 6. H-MAS 래퍼 컨테이너 이미지 도입 검토 (#35)

---

## 이번 주 통계

| 항목 | 수치 |
|------|------|
| 총 커밋 수 | 19개 |
| 머지된 PR | 12개 (#99, #101, #103, #104, #109, #115, #116, #117, #118, #119, #121, #124) |
| 생성된 이슈 | 27개 (#96~#98, #100, #102, #105~#114, #120, #122~#123, #125~#134) |
| 해결된 이슈 | 12개 (#97, #98, #100, #102, #105~#108, #110~#113) |
| 미해결 이슈 | 21개 (#35, #36, #38, #39, #62, #66, #96, #114, #120, #122~#123, #125~#134) |
| 신규 기술 문서 | 1개 (디딤돌 R&D 제안서 초안) |
| 코드 변경량 | +10,112줄 / -866줄 |

---

## 결론

이번 주는 **Model Registry 풀스택 구현과 런타임 최적화 파라미터 시스템 전체 구축으로, 배포 UX의 두 핵심 축을 완성한 대규모 기능 개발 기간**이었습니다. Iteration 13에서 정책-배포 구조 재설계를 마무리한 데 이어, 이번 Iteration에서는 "무엇을(Model Registry)" + "어떻게 최적화할 것인가(Optimization System)"라는 배포 경험의 핵심 질문에 답하는 기능을 연쇄적으로 구현했습니다.

Model Registry는 데이터 통합(#105) → Backend API(#106) → CRUD UI(#107) → 런타임 호환성(#108)의 4단계를 3일 만에 완료하여, 모델 관리의 Single Source of Truth를 확립했습니다. 런타임 최적화 시스템은 Config 자기 완결화(#110) → ParamSpec 스키마(#111) → 2-Tier 적용 엔진(#112) → 시나리오 프리셋(#113)의 4단계를 2일 만에 완료하여, 초보 사용자(프리셋 선택)부터 파워 사용자(Raw Override)까지 지원하는 Progressive Disclosure 체계를 구축했습니다. 또한 운영 환경에서 발견된 3건의 치명적 버그(schedulerName 하드코딩, 삭제 시 orphan, 이벤트 미표시)를 즉시 수정하여 안정성을 확보했습니다.

**핵심 성과**:
1. **Model Registry 풀스택**: DB 스키마 + 7개 API + CRUD UI + 런타임-모델 호환성 매트릭스 + TEI 런타임 추가 — 모델 관리의 Single Source of Truth 확립
2. **런타임 최적화 파라미터 시스템**: ParamSpec 37개 + SemanticKey 9개 + ResolveOptimization 엔진 + 시나리오 프리셋 15개 — Preset(Tier 0) → Common(Tier 1) → Native(Tier 2) → Raw Override 4계층 우선순위 체인 완성
3. **운영 안정성**: schedulerName DB 기반 조회, 배포 삭제 orphan 방지(ResourceBinding 활용), 이벤트 레이블 셀렉터 수정 — 3건 모두 운영 환경 장애 유발 가능 버그
4. **외부 자료**: 디딤돌 R&D 2차 과제 제안서 초안 완성 + 제품 소개서/PPT 슬라이드 구성안

**다음 주 계획**:
- 대시보드 실데이터 연동 — mock 데이터에서 실제 API 호출로 전환 (#129, #130, #131, #132)
- 서빙 배포 인플레이스 업데이트 API (PATCH) 구현 착수 (#125)
- 추론 API 프록시 구현 — 통합 엔드포인트 설계 (#36)

---

**문서 작성일**: 2026년 3월 22일  

