---
title: 'H-MAS release notes (2026-03-01~03-07)'
date: 2026-03-08
permalink: /posts/2026/03/h-mas-weekly-release-notes-0301-0307/
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
## 2026년 3월 1일 - 3월 7일 (Iteration 12)

### 주요 개발 내용 요약

이번 주는 **실제 GPU 배포 파이프라인 안정화**, **HuggingFace 생태계 통합 (비-gated 모델 프리셋 + 토큰 관리)**, **배포 상세 페이지 강화**, **정책-배포 구조 재설계 착수**에 중점을 둔 기간이었습니다. 총 **9개의 커밋**, **9개의 PR 머지**, **12개의 이슈 생성**이 완료되었으며, Kubernetes + RTX 5060 Ti 환경에서 실제 GPU 추론 배포를 최초로 성공하면서 발견된 크리티컬 버그 3건(runtimeClassName 누락, 삭제 시 리소스 잔류, 이름 충돌)을 즉시 해결하고, 멀티 클러스터 전파 체인의 올바른 삭제 순서를 확립했습니다.

{% include youtube.html id="-Wj9mCw_kbk" autoplay=true %}

---

## 새로운 기능 (New Features)

### 1. 배포 상세 페이지에 런타임 상세 정보 및 Pod 이벤트 노출

**구현 완료**: 배포 상세 페이지(`/instances/[id]`)에 런타임 설정, Pod 상태, 이벤트 섹션을 추가하여 k9s 없이 웹 콘솔에서 운영/디버깅 정보 확인 가능

**주요 성과**:
- **Backend**:
  - `DeploymentItem` API 응답에 런타임 상세 필드 추가 (`gpuMode`, `image`, `hfRepo`, `hfFile`, `pvcName`, `zone`, `schedulerName`)
  - `getMemberClient()` 헬퍼 추출로 xxxxxxx Client 리팩터링
  - 멤버 클러스터에서 Pod 상태/이벤트를 직접 조회하는 `GetServingPodDetails`, `GetServingDeploymentEvents` 메서드 추가
  - `GET /api/deployments/:id/events` 엔드포인트 신설 (Pod Events 별도 조회)
- **Frontend**:
  - 배포 상세 페이지에 **런타임 설정**, **Pod 상태**, **이벤트** 섹션 추가 (Card 기반 수직 레이아웃)
  - 이벤트 섹션은 collapsible + lazy-loading으로 초기 로딩 영향 최소화
  - Warning/Normal 이벤트 시각적 구분, 멤버 클러스터 접근 불가 시 graceful 빈 상태 처리

**의사결정 사항**:

| 결정 | 내용 |
|------|------|
| Pod Events 조회 방식 | 별도 엔드포인트로 분리 — 상세 페이지 로딩 시간에 영향 없도록 lazy-loading |
| 멤버 클러스터 접근 | `getMemberClient()` 헬퍼 추출로 `GetClusterNodeIP()` 등 기존 코드 리팩터링 |

**관련 커밋**: `14e94ea` (PR #68, Closes #64)

---

### 2. HuggingFace 토큰 관리 기능 (gated 모델 지원)

**구현 완료**: gated 모델(Meta Llama 등) 배포를 위한 HuggingFace 토큰 등록/조회/삭제 API + 설정 UI + 배포 시 `secretKeyRef` 기반 자동 주입

**주요 성과**:
- **Backend**:
  - `POST/GET/DELETE /api/settings/hf-token` — 토큰 CRUD + HuggingFace `whoami-v2` API 유효성 검증
  - K8s Secret CRUD + 전용 PropagationPolicy로 멀티클러스터 자동 전파
  - 배포 시 HF 다운로드 런타임(`arg`/`hf-download`)에서 토큰 Secret 존재 확인 후 `secretKeyRef` 기반 `HF_TOKEN` env 자동 주입
- **Frontend**:
  - `/settings` 설정 페이지 신설 + HF 토큰 관리 UI (등록/갱신/삭제, 마스킹된 토큰 표시)
  - 배포 폼의 gated 모델 경고를 토큰 등록 상태에 따라 동적 3분기 (미등록→빨간 경고, 등록됨→파란 안내, 해당없음→미표시)
  - 사이드바 + 헤더 드롭다운에 설정 내비게이션 추가

**의사결정 사항**:

| 결정 | 내용 |
|------|------|
| 토큰 저장 | K8s Secret only (DB 미사용) — 보안 원칙 |
| 토큰 주입 | `secretKeyRef` env — 토큰이 Deployment manifest/로그에 평문 노출 방지 |
| 멀티클러스터 전파 | 전용 PropagationPolicy — 배포 라이프사이클과 독립적 |
| 유효성 검증 | 등록 시 HF API 검증 + 배포 시 존재 확인 — 등록 품질 보장 + 배포 시 외부 의존 제거 |

**관련 커밋**: `fe6d0c6` (PR #78, Closes #73)

---

### 3. 비-gated HuggingFace 모델 프리셋 추가 및 gated 모델 경고 UI

**구현 완료**: 승인 없이 즉시 사용 가능한 비-gated 모델 프리셋 추가 + gated 모델 플래그 시스템 도입

**주요 성과**:
- `ModelPreset` 인터페이스에 `gated?: boolean` 필드 추가
- Qwen 2.5 1.5B, Qwen 2.5 3B 등 비-gated 모델 프리셋 추가 (목록 상단 배치)
- 드롭다운에서 gated 모델에 `Lock` 아이콘 + "HF Gated" 라벨 표시
- `isHfDirectRuntime` 조건으로 경고 배너를 vLLM/TGI 런타임에서만 조건부 렌더링 (Ollama는 미표시)

**관련 커밋**: `d3be3eb`의 일부 (PR #74, Closes #72)

---

### 4. conflictResolution / preserveResourcesOnDeletion을 xxxxxxx PP에 전달

**구현 완료**: PlacementStrategy에 저장된 `conflictResolution`과 `preserveResourcesOnDeletion` 값이 xxxxxxx PropagationPolicy 생성 시 실제로 적용되도록 수정

**주요 성과**:
- `ServingDeploymentOpts`에 두 필드 추가, PP 생성부에서 조건부 적용
- PlacementStrategy 없이 배포 시 xxxxxxx 기본값(`Abort` / `false`)에 위임
- `ConflictResolution`: 빈 문자열이 아닐 때만 설정, `PreserveResourcesOnDeletion`: `true`일 때만 포인터 설정

**관련 커밋**: `2a99e2f` (PR #88, Closes #81)

---

## 버그 수정 (Bug Fixes)

### 1. GPU Pod runtimeClassName 누락 수정 및 vLLM 모델명 호환성 개선

**수정 완료**: GPU Pod 배포 시 `runtimeClassName: nvidia` 누락으로 `ContainerCreating` 무한 대기하던 크리티컬 버그 수정 + vLLM/TGI 런타임의 HuggingFace 모델 ID 자동 전달

**수정 내용**:
- **runtimeClassName 수정**: `ServingDeploymentOpts`에 `GPURuntimeClassName` 필드 추가, `GPU > 0`일 때 PodSpec에 `runtimeClassName: nvidia` 자동 설정 (`buildPodSpec` 헬퍼 분리)
- **모델명 호환성**: arg 기반 런타임(vLLM, TGI)에 대해 모델명에 `/`가 없으면 400 에러 반환 (Ollama 스타일 모델명 방어)
- **프리셋 개선**: 모델 프리셋에 `hfModelId` 필드 추가, vLLM/TGI 선택 시 HuggingFace 모델 ID 자동 전달
- **문서 추가**: `GPU_DEPLOYMENT_TROUBLESHOOTING.md` 신규 (RuntimeClass, 모델명, gated 모델, 초기 로딩 지연 등 4개 시나리오)
- **실제 검증**: Kubernetes (RTX 5060 Ti) 환경에서 vLLM + `Qwen/Qwen2.5-1.5B-Instruct` 배포 → Pod Running + `/v1/chat/completions` 추론 성공 확인

**해결된 기술 문제**:

| 문제 | 원인 | 해결 |
|------|------|------|
| GPU Pod `ContainerCreating` 무한 대기 | NVIDIA GPU Operator 환경에서 `runtimeClassName: nvidia` 미설정 시 기본 runc로 GPU 디바이스 설정 불가 | `GPU > 0`일 때 `runtimeClassName: nvidia` 자동 설정 |
| vLLM에 Ollama 스타일 모델명 전달 | `llama3.2:1b` 형식은 HuggingFace ID가 아님 | `hfModelId` 프리셋 필드 + `/` 포함 검증 |

**관련 커밋**: `d3be3eb` (PR #75, Closes #71, #72)

---

### 2. 인스턴스 삭제 시 멤버 클러스터 리소스 미삭제 수정

**수정 완료**: `DeleteServingDeployment`에서 PropagationPolicy를 Deployment보다 먼저 삭제하여 xxxxxxx 전파 체인이 끊기고, 멤버 클러스터의 Deployment·Service·Pod·PVC가 고아 상태로 남는 크리티컬 버그 수정

**수정 내용**:
- 삭제 순서를 워크로드 리소스 우선으로 변경:
  - **Before**: OverridePolicy → PropagationPolicy (전파 체인 끊김) → Service → PVC → Deployment
  - **After**: Deployment → Service → PVC → PropagationPolicy → OverridePolicy
- Deployment 삭제 시 `IsNotFound` 에러를 무시하도록 방어 로직 추가
- GPU/CPU 무관 모든 인스턴스 삭제에 해당하는 버그

**해결된 기술 문제**:

| 문제 | 원인 | 해결 |
|------|------|------|
| 웹 콘솔 삭제 후 멤버 클러스터에 리소스 잔류 | PropagationPolicy 선삭제로 ResourceBinding 제거 → 전파 체인 끊김 | 워크로드(Deployment/Service/PVC) 먼저 삭제 후 정책(PP/OP) 삭제 |

**관련 커밋**: `36b9331` (PR #77, Closes #76)

---

### 3. 배포 이름 자동 생성 시 이름 충돌 방지

**수정 완료**: 동일 모델 재배포 시 배포 이름이 항상 동일하게 생성되어 409 충돌이 발생하던 문제를 4자리 랜덤 접미사 추가로 해결

**수정 내용**:
- 자동 생성 이름에 4자리 랜덤 접미사 추가 (예: `llama3-2-1b-k8m2`)
- 모델명 유지로 식별성 보장, 사용자 직접 수정도 가능

**관련 커밋**: `3bf0f7b` (PR #70, Closes #69)

---

### 4. 배치 전략 사용 시 베이스 리소스 Zone fallback 버그 수정 + 클러스터 타입 매핑 중앙화

**수정 완료**: 배치 전략(PlacementStrategy) 사용 시 `req.Zone`이 빈 값이 되어 리소스 프로파일의 `general` 오버라이드가 항상 베이스 Deployment에 적용되던 버그 수정 + 3곳에 중복된 클러스터 타입↔라벨 매핑을 단일 정의로 중앙화

**수정 내용**:
- **Zone fallback 수정**: 배치 전략 사용 시 전략의 `clusterAffinity.labelSelector`에서 `gpu-tier` 값을 읽어 Zone 자동 추론
- **클러스터 타입 매핑 중앙화**: `xxxxxxx/cluster_types.go`에 매핑 상수 + 헬퍼 함수 (`ClusterTypeFromLabels`, `LabelsForClusterType`, `DefaultSchedulerForClusterType`) 단일 정의
- **유닛 테스트**: 매핑 라운드트립 일관성, 폴백, 방어적 복사 등 테스트 추가
- 배치 전략 해석을 리소스 프로파일 해석보다 **먼저** 수행하도록 순서 변경

**해결된 기술 문제**:

| 문제 | 원인 | 해결 |
|------|------|------|
| 배치 전략 사용 시 항상 `general` 오버라이드 적용 | `req.Zone` 빈 값 → 무조건 `general` fallback | `ClusterTypeFromLabels(clusterLabelSelector)` 기반 Zone 자동 추론 |
| 클러스터 타입↔라벨 매핑 3곳 중복 | `deployment.go`, `client.go`에 동일 switch문 반복 | `cluster_types.go`에 단일 정의 + 헬퍼 함수 |

**관련 커밋**: `ab1ad87` (PR #87, Closes #80)

---

## 리팩토링 (Refactoring)

### 1. 배치 전략에서 defaultProfileId 연결 제거

**리팩토링 완료**: 배치 전략("어디에 배포")과 리소스 프로파일("얼마만큼의 리소스로 배포")의 직교하는 관심사를 분리하기 위해 `defaultProfileId` 결합 해제

**주요 변경**:
- **Backend (7 files)**: DB 마이그레이션 (`default_profile_id` 컬럼 DROP), 모델/핸들러/레포지토리에서 `DefaultProfileId` 로직 전체 제거, `CountLinkedStrategies()` 메서드 삭제
- **Frontend (7 files)**: 폼에서 "기본 프로파일 연결" UI 섹션 전체 삭제, 카드에서 프로파일 배지 제거, 프로파일 삭제 시 전략 참조 확인 로직 제거
- **Swagger**: 재생성

**의사결정 사항**:

| 결정 | 내용 |
|------|------|
| 제거 근거 | 배치 전략과 리소스 프로파일은 직교하는 관심사 — 같은 전략으로 서로 다른 크기의 모델 배포 시 프로파일 강제가 유연성 저하 |
| 방향 변경 | Phase 1-1 원래 계획(자동 적용)에서 "제거"로 전환 — 역할 분리 원칙 우선 |

**관련 커밋**: `38f477d` (PR #86, Closes #79)

---

## 문서화 (Documentation)

### 신규 기술 문서 (1개)

| 문서 | 내용 | 관련 커밋 |
|------|------|-----------|
| `GPU_DEPLOYMENT_TROUBLESHOOTING.md` | GPU 배포 트러블슈팅 가이드 — RuntimeClass, 모델명, gated 모델, 초기 로딩 지연 등 4개 시나리오 | `d3be3eb` (PR #75) |

---

## 이전 Iteration 계획 달성도

Iteration 11에서 계획한 3개 항목 **1개 완료, 1개 부분 달성, 1개 미착수**:

| 계획 | 이슈 | 상태 | 비고 |
|------|------|------|------|
| 배포 상세 페이지에 런타임 상세 정보 및 Pod 이벤트 노출 | #64 | ✅ 완료 | PR #68 |
| 모델 프리셋 리스트 최신화 | #62 | ⚠️ 부분 달성 | 비-gated 모델 추가(PR #74)는 완료, 전체 후속 세대 교체는 미완 |
| 스케줄러 어노테이션 변환 구현 준비 | #66 | ❌ 미착수 | GPU 배포 안정화 + HF 토큰 기능이 우선 |

**미착수 사유**: 실제 GPU 환경 테스트에서 크리티컬 버그 3건(#71, #76, #69)이 발견되어 즉시 수정이 필요했고, HuggingFace gated 모델 지원(#73)과 정책-배포 구조 재설계(#79, #80, #81)가 추가 진행되면서 스케줄러 어노테이션 작업의 우선순위가 밀림.

---

## 정책-배포 구조 재설계 진행 현황

이번 Iteration에서 정책-배포 연동의 구조적 문제를 발견하고 4-Phase 재설계를 계획·착수:

| Phase | 내용 | 상태 | 관련 이슈/PR |
|-------|------|------|-------------|
| Phase 1-1 | defaultProfileId 제거 (역할 분리) | ✅ 완료 | #79 → PR #86 |
| Phase 1-2 | Zone fallback 수정 + 클러스터 타입 매핑 중앙화 | ✅ 완료 | #80 → PR #87 |
| Phase 1-3 | conflictResolution/preserveResourcesOnDeletion PP 전달 | ✅ 완료 | #81 → PR #88 |
| Phase 1-4 | 자동 배치(auto) 규칙 백엔드 평가 로직 | 📋 계획 | #82 |
| Phase 2 | xxx/xxxxxxxxxxx 스케줄러 설정 정리 | 📋 계획 | #83 |
| Phase 3 | 배포 폼 리소스/정책 UX 구조 재설계 | 📋 계획 | #84 |
| Phase 4 | 문서 업데이트 및 Swagger 재생성 | 📋 계획 | #85 |

---

## 미해결 이슈 (Open Issues)

### 신규 (이번 Iteration에서 생성)

### 1. 자동 배치(auto) 규칙 백엔드 평가 로직 구현 (#82)

- `type: auto` 배치 전략의 조건 기반 클러스터 선택 규칙(`rules`)이 백엔드에서 전혀 평가되지 않음
- `rules.condition`이 자유 텍스트로 되어 있어 파서 구현 필요 — MVP에서는 단순 매칭으로 시작
- 정책-배포 구조 재설계 Phase 1-4

### 2. 리소스 프로파일에서 xxx/xxxxxxxxxxx 스케줄러 설정 제거 (#83)

- xxx/xxxxxxxxxxx 관련 설정이 프론트엔드에 UI가 있지만 백엔드에서 완전히 무시됨
- 사용자 혼란 방지를 위해 UI 제거 + 백엔드 `omitempty` 태그 추가
- 정책-배포 구조 재설계 Phase 2

### 3. 배포 폼 리소스/정책 UX 구조 재설계 (#84)

- 현재 3개 카드(배치 전략/리소스 설정/정책 적용)를 2개 카드로 통합
- 리소스 프로파일 선택 시 수동 입력 필드를 읽기 전용으로 전환
- 정책-배포 구조 재설계 Phase 3

### 4. 정책-배포 연동 구조 문서 업데이트 (#85)

- 아키텍처/가이드/제품 문서 업데이트 + Swagger 재생성
- Phase 1~3 완료 후 진행 예정

### 기존 미해결 이슈 (이전 Iteration에서 이관)

- 스케줄러 어노테이션 변환 (xxx/xxxxxxxxxxx → xxxxxxx OverridePolicy) (#66)
- 모델 프리셋 리스트 최신화 (#62)
- Failover / 재배치 정책 구현 (#39) — v0.95 마일스톤
- NUMA / 토폴로지 인식 스케줄링 구현 (#38) — v0.95 마일스톤
- 추론 API 프록시 구현 — 통합 엔드포인트 (#36)
- H-MAS 래퍼 컨테이너 이미지 도입 검토 (#35)

---

## 이번 주 통계

| 항목 | 수치 |
|------|------|
| 총 커밋 수 | 9개 |
| 머지된 PR | 9개 (#68, #70, #74, #75, #77, #78, #86, #87, #88) |
| 생성된 이슈 | 12개 (#69, #71, #72, #73, #76, #79, #80, #81, #82, #83, #84, #85) |
| 해결된 이슈 | 9개 (#64, #69, #71, #72, #73, #76, #79, #80, #81) |
| 미해결 이슈 | 10개 (#35, #36, #38, #39, #62, #66, #82, #83, #84, #85) |
| 신규 기술 문서 | 1개 |
| 코드 변경량 | +3,007줄 / -365줄 |

---

## 결론

이번 주는 **실제 GPU 추론 환경에서의 첫 end-to-end 배포 성공과 함께, 발견된 크리티컬 버그들을 즉시 해결하고, HuggingFace 생태계 통합을 완성한 기간**이었습니다. Kubernetes + RTX 5060 Ti 환경에서 vLLM을 통한 실제 LLM 추론 성공이 핵심 마일스톤이며, 이 과정에서 드러난 xxxxxxx 전파 체인 삭제 순서, GPU RuntimeClass 설정, 모델명 호환성 등의 문제를 근본적으로 해결했습니다. 또한 정책-배포 구조의 구조적 문제를 식별하고 4-Phase 재설계를 착수하여 Phase 1을 완료했습니다.

**핵심 성과**:
1. **실제 GPU 추론 배포 최초 성공**: RTX 5060 Ti 클러스터에서 vLLM + Qwen 2.5 1.5B 배포 → `/v1/chat/completions` 추론 정상 동작 확인
2. **xxxxxxx 삭제 전파 체인 수정**: 워크로드 → 정책 순서로 삭제 순서를 교정하여 멤버 클러스터 리소스 고아 상태 방지
3. **HuggingFace 토큰 관리**: K8s Secret 기반 토큰 저장 + `secretKeyRef` 주입 + 멀티클러스터 전파 + 설정 UI
4. **배포 상세 페이지 강화**: 런타임 설정, Pod 상태, 이벤트를 웹 콘솔에서 직접 확인 가능
5. **정책-배포 구조 재설계 Phase 1 완료**: defaultProfileId 제거, Zone fallback 수정, 클러스터 타입 매핑 중앙화, PP 정책 옵션 전달

**다음 주 계획**:
- 정책-배포 구조 재설계 Phase 2: xxx/xxxxxxxxxxx 스케줄러 설정 정리 (#83)
- 배포 폼 리소스/정책 UX 구조 재설계 (#84)
- 자동 배치(auto) 규칙 백엔드 평가 로직 구현 (#82)

---

**문서 작성일**: 2026년 3월 8일  

