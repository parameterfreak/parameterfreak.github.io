---
title: 'H-MAS release notes (2026-02-22~02-28)'
date: 2026-02-28
permalink: /posts/2026/02/h-mas-weekly-release-notes-0222-0228/
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
## 2026년 2월 22일 - 2월 28일

### 주요 개발 내용 요약

이번 주는 **정책↔배포 연동 시스템 구축**, **서빙 런타임 GPU/CPU 모드 선택 지원**, **클러스터 운영 가시성 강화**, **CPU 전용 Zone의 1등 시민 격상**에 중점을 둔 기간이었습니다. 총 **9개의 커밋**, **6개의 PR 머지**, **5개의 이슈 생성**이 완료되었으며, 이전 Iteration에서 계획한 4개 항목을 **모두 달성**하고 정책 시스템과 배포 파이프라인이 하나의 워크플로로 통합되었습니다.

{% include youtube.html id="CKIPhkRtR1w" %}

---

## 새로운 기능 (New Features)

### 1. 정책↔배포 연동 — 배치 전략·리소스 프로파일·장애 복구 통합

**구현 완료**: 배포 시 정책 메뉴의 배치 전략, 리소스 프로파일, 장애 복구 정책을 선택·적용할 수 있도록 3단계 연동 구현

**주요 성과**:
- **Backend**:
  - `placement_strategies` 테이블 및 CRUD API 6개 엔드포인트 신설
  - `resource_profiles` 테이블 및 CRUD API 구현
  - `failover_policies` 테이블 및 CRUD API 구현
  - 배포 API에 `placementStrategyId`, `resourceProfileId`, `failoverPolicyId` 파라미터 추가
  - 배치 전략 → xxxxxxx `PropagationPolicy` 자동 변환
  - 리소스 프로파일 → xxxxxxx `ClusterOverridePolicy` 변환
  - Failover 설정 → `PropagationPolicy.spec.failover` merge
  - 모델 사이즈 기반 전략 자동 추천 로직
- **Frontend**:
  - 프론트엔드 mock → 실제 API 전환, 배포 폼에 '정책 기반 배치' 모드 추가
  - 기존 zone/clusterName 인라인 방식과 하위 호환 유지
  - 드로어 폼 스크롤 버그 수정 (`min-h-0`)
- **문서**:
  - 정책↔배포 연동 아키텍처 설계 문서 작성

**의사결정 사항**:

| 결정 | 내용 |
|------|------|
| 정책 적용 방식 | 인라인 vs 정책 기반 토글 — 기존 인라인과 하위 호환 유지 |
| 전략 추천 로직 | 모델 사이즈(파라미터 수) 기반 자동 추천 |
| DB 아키텍처 | 배치 전략·리소스 프로파일·장애 복구를 독립 테이블로 분리 (Vertical Slice) |
| Failover merge 전략 | 장애 복구 설정을 PropagationPolicy에 merge하여 단일 정책 유지 |

**관련 커밋**: `1266acd` (PR #65, Closes #50)

---

### 2. 서빙 런타임 GPU/CPU 모드 선택 지원 (llama.cpp, Ollama)

**구현 완료**: `RequiresGPU(bool)` → `GPUSupport("required"|"optional"|"none")` 3단계 모델 전환으로 런타임별 GPU/CPU 모드 사용자 선택 가능

**주요 성과**:
- **Backend**:
  - `GPUSupport` 커스텀 타입 + `GPUModeOverride` 중첩 구조체로 런타임별 GPU 모드 오버라이드 안전하게 처리
  - `ResolveForGPUMode()` 헬퍼: `required` → 항상 GPU, `optional` → 사용자 선택, `none` → 항상 CPU
  - llama.cpp GPU 모드: `:server-cuda` 이미지 + `-ngl 99` args 자동 적용
  - Ollama GPU 모드: 동일 이미지 + `nvidia.com/gpu: 1` 리소스 요청
  - GPU 수량 자동 보정 (GPU 모드 시 최소 1)
  - DB 마이그레이션 000007: `gpu_mode` 컬럼 추가
- **Frontend**:
  - optional 런타임 선택 시 CPU/GPU 모드 토글 UI
  - GPU 모드 전환 시 최소 리소스(메모리 등) 자동 조정

**해결된 기술 문제**:

| 문제 | 원인 | 해결 |
|------|------|------|
| GPU 클러스터에서 llama.cpp가 CPU로만 추론 | `:server` 이미지에 CUDA 미포함 | GPU 모드 시 `:server-cuda` 이미지 + `-ngl 99` 자동 적용 |
| Ollama가 GPU를 할당받지 못함 | `nvidia.com/gpu` 리소스 미요청 | GPU 모드 시 리소스 요청 자동 추가 |
| `bool` 타입으로 선택적 GPU 표현 불가 | 2-state(true/false)로 3-state 불가 | `GPUSupport` enum 3단계 모델 전환 |

**관련 커밋**: `2467039` (PR #57, Closes #55)

---

### 3. 클러스터 인스턴스 수 표시, 배포 상태 DB 동기화, 가용 GPU 정확도 개선

**구현 완료**: 클러스터 카드에 실제 배포 인스턴스 수 표시, DB 상태 동기화 파이프라인 구축, GPU 가용량 계산 정확도 향상

**주요 성과**:
- **인스턴스 수 구현**:
  - `serving_deployments` 테이블의 `GROUP BY cluster_name` 집계 → 클러스터 API 응답에 `instanceCount` 포함
  - 프론트엔드 하드코딩 `"-"` 제거, 실제 값 바인딩
- **배포 상태 DB 동기화**:
  - `ListDeployments`/`GetDeployment` 조회 시 xxxxxxx 실시간 상태를 DB에 write-back
  - Reconciler 확장: 60초 주기 전체 배포 상태 동기화
- **가용 GPU 정확도 개선**:
  - `convertCluster()`에서 `Allocating` 리소스도 차감하여 스케줄링 중 Pod의 GPU 이중 할당 방지
  - `gpu-tier` 레이블 기반 GPU 미감지 클러스터에 경고 UI 추가

**해결된 기술 문제**:

| 문제 | 원인 | 해결 |
|------|------|------|
| 인스턴스 수 항상 `-` 표시 | 전 레이어(UI/API/모델/핸들러) 미구현 | `DeploymentRepository.CountByCluster()` 집계 + 전 레이어 구현 |
| DB 배포 상태가 항상 `pending` | xxxxxxx 조회 후 DB write-back 누락 | 조회 시 write-back + Reconciler 주기적 동기화 |
| 스케줄링 중 Pod의 GPU가 "가용"으로 표시 | `Allocating` 리소스 미반영 | `Allocating` 차감 로직 추가 |

**관련 커밋**: `3308469` (PR #61, Closes #58)

---

### 4. CPU 전용 Zone 확장 — 문서/폼/API 정합성

**구현 완료**: CPU 전용 Zone을 정책 시스템의 1등 시민(first-class)으로 격상, 문서·폼·API 전반의 갭 해소

**주요 성과**:
- **문서 업데이트 (3개)**:
  - `03-resource-profile-zone-design.md`: 2-Zone → 3-Zone 확장 확정, Zone C 스펙/영향 범위/변경 이력 추가
  - `02-policy-menu-spec.md`: CPU 전용 시나리오 전반 추가 — `auto-cpu-only` 전략, Zone C 폼/검증/xxxxxxx 변환, CPU 전용 추천 경로
  - `04-policy-deployment-integration.md`: CPU 전용 Failover, 교차 Failover 정책(CPU→GPU 단방향 허용), Phase 3 항목 추가
- **Frontend**:
  - `resource-profile-form.tsx`: Zod 스키마에 `cpuOnlyEnabled`/`cpuOnlyCpu`/`cpuOnlyMemory` 추가, 토글 기반 Zone C UI(teal)
  - `resource-profile-card.tsx`: Zone C (CPU Only) teal 색상 표시 섹션 추가
- **Backend**:
  - `deployment.go`: Zone enum에 `cpu-only` 추가
  - `xxxxxxx/client.go`: `cpu-only` 타입일 때 `nvidia.com/gpu` 리소스 명시적 제거 로직
  - Swagger 스펙 업데이트

**의사결정 사항**:

| 결정 | 내용 |
|------|------|
| Zone 확장 범위 | v0.9에서 3-Zone 고정 (HP/GP/CPU-Only), v1.0+에서 동적 Zone 검토 |
| 교차 Failover 정책 | CPU→GPU 단방향 허용 (리소스 과사용이지만 서비스 유지), GPU→CPU 비허용 (서빙 불가) |
| Zone C 색상 테마 | teal 계열 (클러스터 카드와 일치: `bg-teal-100 text-teal-700`) |

**관련 커밋**: `930b4a4` (PR #67, Closes #51)

---

## 버그 수정 (Bug Fixes)

### 1. llama.cpp 런타임 기본 컨텍스트 크기 제한으로 OOM 방지

**수정 완료**: llama.cpp `ExtraArgs`에 `--ctx-size 4096` 추가하여 KV 캐시 14GB+ 메모리 요구 OOM 문제 해결

**수정 내용**:
- llama.cpp가 모델 메타데이터의 기본 컨텍스트 길이(128K)를 그대로 사용하면서 KV 캐시 14GB+ 메모리 요구
- `--ctx-size 4096` 기본값 추가: Ollama의 `OLLAMA_NUM_CTX=4096`과 일관된 기본값
- 3B 모델 기준 KV 캐시 ~448MB로 6Gi 내 충분
- CPU/GPU 양쪽 모드에서 `ResolveForGPUMode()`를 통해 자동 적용

**의사결정 근거**:

| 결정 | 내용 |
|------|------|
| 기본값 4096 선정 | Ollama와의 일관성, 3B 모델 KV 캐시 ~448MB로 6Gi 내 충분 |
| ExtraArgs 하드코딩 방식 | Technical Preview 단계에서 크래시 방지 최우선, 사용자 커스텀 CLI 인자는 별도 이슈로 분리 |

**관련 커밋**: `5f7534e` (PR #63, Closes #52)

---

## 문서화 (Documentation)

### 신규 기술 문서 (2개)

| 문서 | 내용 | 관련 커밋 |
|------|------|-----------|
| `GPU_RESOURCE_ALLOCATION.md` | `nvidia.com/gpu` 정수 독점 할당의 전체 데이터 흐름, 스케줄러별(기본 K8s / xxxxxxxxxx Device Share / xxxxxxxxx MIG) GPU 공유 전략 비교, xxxxxxxxxx GPU 공유 구현 시 변경 범위, 모델 크기별 GPU 할당 가이드 (426줄) | `0e93bf0` (PR #60, Closes #59) |
| 정책↔배포 연동 아키텍처 문서 | 연동 흐름도, API 스펙, 데이터 모델 | `1266acd` (PR #65) |

### 기획 문서 업데이트 (3개)

| 문서 | 변경 | 관련 커밋 |
|------|------|-----------|
| `02-policy-menu-spec.md` | CPU 전용 시나리오 전반 추가 (§1.1/§3.1/§3.4/§3.5/§4/§6/§7/§11) | `930b4a4` (PR #67) |
| `03-resource-profile-zone-design.md` | 2-Zone → 3-Zone 확장 확정, Zone C 스펙/영향 범위 | `930b4a4` (PR #67) |
| `04-policy-deployment-integration.md` | CPU 전용 Failover, 교차 Failover 정책 | `930b4a4` (PR #67) |

---

## 기타 작업 (Chores)

- docs 참조 기반 작업 계획 Cursor rule 추가 (`89e7699`)
- 사이드바 로고를 서버 랙 이미지로 교체 (`29a742a`)
- Iteration 10 데모 영상 시나리오 및 썸네일 설정 추가 (`d2a4d96`)

---

## 이전 Iteration 계획 달성도

Iteration 10에서 계획한 4개 항목 **전체 달성**:

| 계획 | 이슈 | 상태 | 관련 PR |
|------|------|------|---------|
| 정책↔배포 연동 구현 | #50 | ✅ 완료 | #65 |
| 모델 프리셋 리소스 값 최적화 및 llama.cpp 컨텍스트 크기 조정 | #52 | ✅ 완료 | #63 |
| 서빙 런타임 GPU/CPU 모드 선택 지원 | #55 | ✅ 완료 | #57 |
| CPU 전용 Zone 기획/기술 문서 확장 | #51 | ✅ 완료 | #67 |

---

## 미해결 이슈 (Open Issues)

### 1. 스케줄러 어노테이션 변환 (xxx/xxxxxxxxxx → xxxxxxx OverridePolicy) (#66)

- #50에서 리소스 프로파일 기본 오버라이드(CPU/메모리/GPU)는 구현 완료, 스케줄러별 어노테이션 변환은 보류
- xxx: topology, NVLink, MIG 관련 어노테이션 / xxxxxxxxxx: QoS, GPU Share, Gang Scheduling 어노테이션
- 선행 조건: xxx 스케줄러가 HP 클러스터에 설치, xxxxxxxxxx GP 클러스터에 설치

### 2. 배포 상세 페이지에 런타임 상세 정보 및 Pod 이벤트 노출 (#64)

- 배포 상세 페이지에서 GPU 모드, Restart Count, Pod Events, 컨테이너 이미지, PVC 정보 등 운영/디버깅에 필요한 정보 미노출
- 백엔드 응답 모델 확장 + 프론트엔드 "런타임 설정", "Pod 상태", "이벤트" 섹션 추가 필요

### 3. 모델 프리셋 리스트 최신화 (#62)

- 현재 `MODEL_PRESETS`가 2024년 하반기 기준으로 1~2세대 이전 모델
- Llama 3.2 → Gemma 3, Phi-3 → Phi-4, Qwen 2.5 → Qwen 3 등 후속 모델로 교체 검토
- GGUF 파일 가용성 확인 및 실제 메모리 사용량 측정 필요

### 4. 기존 미해결 이슈 (이전 Iteration에서 이관)

- NUMA / 토폴로지 인식 스케줄링 구현 (#38) — v0.95 마일스톤
- Failover / 재배치 정책 구현 (#39) — v0.95 마일스톤
- 추론 API 프록시 구현 — 통합 엔드포인트 (#36)
- H-MAS 래퍼 컨테이너 이미지 도입 검토 (#35)

---

## 이번 주 통계

| 항목 | 수치 |
|------|------|
| 총 커밋 수 | 9개 |
| 머지된 PR | 6개 (#57, #60, #61, #63, #65, #67) |
| 생성된 이슈 | 5개 (#58, #59, #62, #64, #66) |
| 해결된 이슈 | 6개 (#50, #51, #52, #55, #58, #59) |
| 미해결 이슈 | 7개 (#35, #36, #38, #39, #62, #64, #66) |
| 신규 기술 문서 | 2개 |
| 기획 문서 업데이트 | 3개 |
| 코드 변경량 | +5,931줄 / -366줄 |

---

## 결론

이번 주는 **이전 Iteration에서 수립한 계획을 100% 달성하고, 정책 시스템과 배포 파이프라인을 하나의 통합 워크플로로 완성한 기간**이었습니다. 정책↔배포 연동으로 배치 전략·리소스 프로파일·장애 복구가 배포 시 자동 적용되며, GPU/CPU 모드 선택과 CPU 전용 Zone 1등 시민화로 하드웨어 다양성에 대한 대응 범위가 크게 확장되었습니다.

**핵심 성과**:
1. **정책↔배포 통합 워크플로**: 3개 정책 유형(배치 전략/리소스 프로파일/장애 복구)이 배포 폼에서 선택·적용 가능, 모델 사이즈 기반 자동 추천으로 UX 개선
2. **GPU/CPU 모드 선택**: `GPUSupport` 3단계 모델로 전환하여 llama.cpp·Ollama에서 배포 시 CPU/GPU 모드를 사용자가 직접 선택
3. **운영 가시성 강화**: 클러스터 인스턴스 수 실시간 표시, 배포 상태 DB 동기화(60초 주기), GPU 가용량 계산 정확도 개선
4. **CPU 전용 Zone 1등 시민화**: 기획 문서·기술 문서·프론트엔드 폼·백엔드 API 전반에서 CPU-Only를 3-Zone 체계의 공식 구성원으로 확립
5. **llama.cpp OOM 해결**: `--ctx-size 4096` 기본값으로 KV 캐시 메모리 폭발 방지
6. **GPU 리소스 할당 기술 문서화**: 현재 정수 독점 할당의 한계와 향후 xxxxxxxxxx GPU 공유 구현 경로를 체계적으로 정리

**다음 주 계획**:
- 배포 상세 페이지에 런타임 상세 정보 및 Pod 이벤트 노출 (#64)
- 모델 프리셋 리스트 최신화 (#62)
- 스케줄러 어노테이션 변환 구현 준비 (#66)

---

**문서 작성일**: 2026년 3월 1일  
