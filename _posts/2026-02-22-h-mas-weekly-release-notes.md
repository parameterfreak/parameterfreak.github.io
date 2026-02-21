---
title: 'H-MAS release notes (2026-02-15~02-21)'
date: 2026-02-22
permalink: /posts/2026/02/h-mas-weekly-release-notes-0215-0221/
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
## 2026년 2월 15일 - 2월 21일

### 주요 개발 내용 요약

이번 주는 **Zone 기반 Dual-Track 배포 정책 레이어 구현**, **CPU 전용 클러스터 서빙 지원**, **런타임 고급 설정 및 PVC 기반 모델 캐싱**, **클러스터 레이블 관리 체계 확립**에 중점을 둔 기간이었습니다. 총 **9개의 커밋**, **7개의 PR 머지**, **12개의 이슈 생성**이 완료되었으며, 배포 파이프라인 위에 **Zone 기반 정책 레이어**가 구축되고, **CPU 전용 클러스터까지 포괄하는 3-Zone 분류 체계**로 확장되었습니다.

{% include youtube.html id="-Q4kQahBkLU" %}

---

## 새로운 기능 (New Features)

### 1. 배포 정책 레이어 및 스케줄러 제어 구현 (Zone 기반 Dual-Track Serving)

**구현 완료**: Zone 기반 배치(LabelSelector ClusterAffinity)와 스케줄러 오버라이드를 통한 Dual-Track Serving 아키텍처의 정책 레이어

**주요 성과**:
- **Backend**:
  - `PropagationPolicy`: `ClusterNames` 직접 지정 외에 `LabelSelector` 기반 `ClusterAffinity` 지원 (`gpu-tier: high/general/none`)
  - `OverridePolicy` 자동 생성: Zone 배포 시 `schedulerName`을 Pod spec에 자동 주입
  - Policy CRUD API: `POST/DELETE` for PropagationPolicy, OverridePolicy
  - 클러스터 라벨 수정 API: `PATCH /api/clusters/:name/labels` — 기존 클러스터에 Zone 라벨 추가/변경
  - DB 마이그레이션 v4: `zone`, `scheduler_name`, `xxxxxxx_override_policy_name` 컬럼 추가
- **Frontend**:
  - 배포 폼: Zone 기반 배치 / 클러스터 직접 선택 토글, Zone별 매칭 클러스터 수 표시
  - 클러스터 등록: "기본 K8s" 선택 시 Zone 포함 여부(Zone A / Zone B / 없음) 선택 UI 추가
  - API 클라이언트: `apiPatch` 헬퍼, Policy CRUD 클라이언트 (`policies.ts`)

**의사결정 사항**:

| 결정 | 내용 |
|------|------|
| Policy 데이터 소유권 | xxxxxxx = source of truth, H-MAS DB는 메타데이터 캐시 |
| ClusterAffinity 전략 | `LabelSelector` 기반 (Zone → `gpu-tier` 라벨 매핑) |
| OverridePolicy 단위 | Deployment 당 1개 자동 생성 |
| 배포 폼 UX | Zone/클러스터 직접선택 토글 방식 |

**관련 커밋**: `bbf94d9` (PR #40, Closes #33)

---

### 2. GPU 미보유 클러스터 서빙 지원 및 3-Zone 분류 체계

**구현 완료**: CPU 전용 클러스터를 위한 분류 체계 확장 및 레이블 일관성 확보

**주요 성과**:
- **분류 체계 확장**: 기존 2-Zone(HP/GP) → 3-Zone(HP/GP/CPU Only)으로 확장
  - `gpu-tier: none` / `cluster-type: cpu-only` 레이블 체계 추가
  - UI 분류: `Standard` → `Unclassified`로 전환하여 미분류 클러스터 모호성 해소
- **Reconciler 레이블 자동화**: 외부 등록(`xxxxxxxctl join`) 클러스터에 핵심 레이블 미설정 시 기본값 자동 write-back
- **배포 폼**: CPU Only Zone 선택 옵션 추가 (3-column grid)
- **Backend**: Zone `cpu-only` → `gpu-tier: none` labelSelector 매핑, Reconciler audit log 기록
- **Frontend**: CPU Only(teal) 배지, Unclassified(amber) 배지, 정책 폼에 `cpu-only` 추가

**관련 커밋**: `5235987` (PR #46, Closes #43)

---

### 3. 클러스터 상세 다이얼로그 레이블 편집 UI

**구현 완료**: 클러스터 상세 다이얼로그에서 레이블을 직접 편집/추가/삭제할 수 있는 인라인 편집 모드

**주요 기능**:
- 레이블 탭에 편집 모드 전환 버튼, 기존 레이블 값 수정/삭제 + 새 레이블 추가
- `gpu-tier` 변경 시 `cluster-type` 자동 파생 (`none` → `cpu-only`, 기타 → `standard`)
- 핵심 레이블 변경 시 Zone 재분류 경고 배너 표시
- 저장 성공 후 클러스터 목록 자동 갱신(Zone 재분류 반영)
- 빈 키, 중복 키 입력 시 유효성 검증 에러 메시지
- **버그 수정**: `UpdateClusterLabels` 핸들러의 `gpu-tier` → `cluster-type` DB 파생 매핑 누락 수정

**관련 커밋**: `9ad0d5c` (PR #47, Closes #44)

---

### 4. 런타임 고급 설정(환경변수) 및 PVC 기반 모델 캐시 스토리지

**구현 완료**: 배포 시 런타임 환경변수 사용자 지정 + emptyDir → PVC 전환으로 모델 캐시 영속화

**주요 성과**:
- **Backend**:
  - `CreateDeploymentRequest`에 `envVars`, `storageSize`, `existingPvcName` 필드 추가
  - 런타임 프리셋 Env와 사용자 envVars merge (사용자 값이 override)
  - 배포별 PVC 동적 생성 (`hmas-{name}-model-cache`), 런타임별 캐시 마운트 경로 결정
  - PropagationPolicy에 PVC ResourceSelector 포함하여 xxxxxxx 멀티클러스터 전파
  - 배포 삭제 시 자동 생성된 PVC 정리 (기존 PVC 지정 시 보존)
  - DB 마이그레이션 000006
- **Frontend**:
  - 접이식 "고급 설정" 섹션: 환경변수 key-value 입력 + 런타임별 프리셋 버튼 (Ollama 4종, vLLM 1종, TGI 1종)
  - 모델 캐시 스토리지 크기 및 기존 PVC 지정 옵션

**해결된 기술 문제**:

| 문제 | 원인 | 해결 |
|------|------|------|
| Pod 재시작 시 모델 재다운로드 | emptyDir 사용으로 데이터 비영속 | PVC 기반 모델 캐시로 전환 |
| 런타임 동작 커스텀 불가 | 환경변수 설정 인터페이스 부재 | 고급 설정 UI + envVars merge 로직 |
| 멀티클러스터 PVC 전파 | PropagationPolicy에 PVC 미포함 | PVC ResourceSelector 추가 |

**관련 커밋**: `b29abcf` (PR #53, Closes #34)

---

### 5. 클러스터 등록 시 커스텀 GPU 타입 입력 지원 및 Zone 헤더 동적화

**구현 완료**: 프리셋에 없는 GPU 타입 직접 입력 + Zone 그룹 헤더 실제 데이터 기반 동적 표시

**주요 기능**:
- **커스텀 GPU 입력**:
  - 기존 권장/기타 프리셋 Badge UI 유지, 하단에 "직접 입력" 텍스트 필드 + 추가 버튼 배치
  - 입력 검증: trim, 중복 방지, 빈값 방지, Enter 키 지원
  - 추가된 커스텀 GPU는 보라색 Badge + X 삭제 버튼으로 표시
- **Zone 헤더 동적화**:
  - `getZoneSummary()` 헬퍼: Zone 내 클러스터들의 GPU 타입과 스케줄러를 합집합으로 수집
  - GPU 5종 초과 시 "외 N종"으로 축약
  - 스케줄러별 색상 배지 자동 표시

**관련 커밋**: `7e2d858` (PR #56, Closes #54)

---

## 버그 수정 (Bug Fixes)

### 1. llama.cpp 런타임 GGUF 모델 로딩 실패 수정 및 전체 런타임 헬스 프로브 추가

**수정 완료**: llama.cpp가 Ollama 태그 형식(`llama3.2:1b`)을 GGUF 파일 경로로 인식하지 못하는 문제 해결

**수정 내용**:
- **`hf-download` ModelLoadStrategy 도입**: llama.cpp가 `--hf-repo`/`--hf-file` 인자로 HuggingFace에서 GGUF를 자동 다운로드
- **런타임 최소 리소스 floor 적용**: 프리셋 메모리(2Gi)가 런타임 최소(6Gi)보다 낮을 때 `resourceMax()` 함수로 OOMKilled 방지
- **전체 런타임 헬스 프로브 추가**: StartupProbe + ReadinessProbe로 모델 로딩 완료 전 잘못된 "실행 중" 상태 표시 방지
  - HTTP `/health` (vLLM, TGI, llama.cpp) 또는 exec `ollama show` (Ollama)
- **DB 마이그레이션 000005**: `hf_repo`/`hf_file` 컬럼 추가
- **Frontend**: 모델 프리셋에 GGUF HF repo/file 매핑 추가, 런타임 변경 시 `max(프리셋, 런타임)` 리소스 자동 설정

**관련 커밋**: `5dc20fc` (PR #48, Closes #45)

---

## 문서화 (Documentation)

### 신규 기술 문서 (3개)

| 문서 | 내용 | 관련 커밋 |
|------|------|-----------|
| `05-cluster-label-system.md` | xxxxxxx Cluster CRD 레이블 저장소, 생명주기, 사용처, 키 규격, Zone↔레이블 매핑 관계 체계적 정리 (446줄) | `12bab31` (PR #42) |
| `RUNTIME_SETTINGS_MODEL_LOADING.md` | 환경변수 주입, PVC 전략, 멀티클러스터 동작, L2/L3 캐싱 계층 설계 | `b29abcf` (PR #53) |
| `CLUSTER_RESOURCE_COLLECTION.md` | xxxxxxx → 백엔드 → 프론트엔드 리소스 수집 파이프라인, GPU가 0으로 표시될 때 트러블슈팅 가이드 | `7e2d858` (PR #56) |

---

## 기타 작업 (Chores)

- Swagger 문서에서 Apache 2.0 라이센스 표기 제거 (`037204f`)
- video-generator 썸네일 추출 스크립트 추가 (`6438d1a`)
- 저사양 GPU 구입 완료 (#49)

---

## 미해결 이슈 (Open Issues)

### 1. NUMA / 토폴로지 인식 스케줄링 구현 (#38)

- #33에서 분리된 Step 3, GPU 하드웨어 토폴로지(NVLink, NVSwitch, PCIe) 인식 스케줄링
- GPU 토폴로지 정보 수집, NUMA 노드 인식 Pod 배치, 토폴로지 시각화
- v0.95 마일스톤 대상

### 2. Failover / 재배치 정책 구현 (#39)

- #33에서 분리된 Step 4, 클러스터 장애 시 자동 재배치 및 고가용성 보장
- xxxxxxx Cluster Status 기반 장애 감지, Graceful Migration
- v0.95 마일스톤 대상

### 3. 정책↔배포 연동 설계 (#50)

- 현재 배포와 정책이 완전히 독립적으로 동작하는 문제
- 배포 과정에서 정책 메뉴의 배치 전략/리소스 프로파일/장애 복구 설정을 선택·적용할 수 있도록 연동 필요
- 기획 문서에 설계된 흐름과 실제 구현 간 갭 해소

### 4. CPU 전용 Zone 확장 — 기획/기술 문서 (#51)

- 현재 정책 시스템이 GPU 클러스터 중심으로 설계되어 CPU 전용 시나리오 미포함
- 리소스 프로파일, 기본 배치 전략, 기획 문서에 CPU 전용 Zone 반영 필요

### 5. 모델 프리셋 리소스 값 및 llama.cpp 기본 컨텍스트 크기 조정 (#52)

- llama.cpp가 모델의 기본 컨텍스트 길이(128K)를 사용하면서 KV 캐시 14GB+ 메모리 요구
- 프리셋 `minMemory`와 실제 필요량 간 괴리로 OOMKill 발생

### 6. 서빙 런타임 GPU/CPU 모드 선택 지원 (#55)

- `RequiresGPU` 필드가 `bool` 타입으로 "선택적 GPU 사용"을 표현할 수 없음
- llama.cpp, Ollama 등 GPU 선택적 사용 런타임이 항상 CPU 모드로만 배포되는 문제

### 7. 기존 미해결 이슈 (이전 Iteration에서 이관)

- H-MAS 래퍼 컨테이너 이미지 도입 검토 (#35)
- 추론 API 프록시 구현 — 통합 엔드포인트 (#36)

---

## 이번 주 통계

| 항목 | 수치 |
|------|------|
| 총 커밋 수 | 9개 |
| 머지된 PR | 7개 (#40, #42, #46, #47, #48, #53, #56) |
| 생성된 이슈 | 12개 (#38, #39, #41, #43~#45, #49~#52, #54, #55) |
| 해결된 이슈 | 8개 (#33, #34, #41, #43~#45, #49, #54) |
| 미해결 이슈 | 8개 (#35, #36, #38, #39, #50~#52, #55) |
| 신규 기술 문서 | 3개 |
| 코드 변경량 | +3,909줄 / -356줄 |

---

## 결론

이번 주는 **배포 파이프라인 위에 정책 기반 지능형 배치 시스템을 구축하고, 실제 운영 환경의 다양한 하드웨어 구성을 포괄하는 방향으로 확장한 기간**이었습니다. Zone 기반 Dual-Track Serving 아키텍처가 구현되어 GPU 성능 등급별 최적 스케줄러가 자동 선택되며, CPU 전용 클러스터까지 포괄하는 3-Zone 분류 체계가 확립되었습니다.

**핵심 성과**:
1. **Zone 기반 정책 레이어**: LabelSelector ClusterAffinity + OverridePolicy 스케줄러 자동 주입으로 모델 특성에 맞는 지능적 배치
2. **3-Zone 분류 체계**: GPU High Performance / GPU General / CPU Only + Unclassified로 모든 클러스터 유형 포괄
3. **PVC 기반 모델 캐싱**: emptyDir → PVC 전환으로 Pod 재시작 시 30분+ 모델 재다운로드 문제 해결
4. **런타임 고급 설정**: 환경변수 사용자 지정으로 컨텍스트 길이, 동시 요청, GPU 레이어 등 세밀한 튜닝 가능
5. **llama.cpp HF 다운로드**: `hf-download` ModelLoadStrategy 도입으로 GGUF 모델 자동 다운로드 + 전 런타임 헬스 프로브 추가
6. **클러스터 레이블 관리**: 인라인 편집 UI + Reconciler 자동 write-back으로 레이블 일관성 확보

**다음 주 계획**:
- 정책↔배포 연동 구현: 정책 메뉴 설정이 실제 배포에 적용되도록 연동 (#50)
- 모델 프리셋 리소스 값 최적화 및 llama.cpp 기본 컨텍스트 크기 조정 (#52)
- 서빙 런타임 GPU/CPU 모드 선택 지원 (#55)
- CPU 전용 Zone 기획/기술 문서 확장 (#51)

---

**문서 작성일**: 2026년 2월 22일  


