---
title: 'H-MAS release notes (2026-02-08~02-14)'
date: 2026-02-15
permalink: /posts/2026/02/h-mas-weekly-release-notes-0208-0214/
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
## 2026년 2월 8일 - 2월 14일

### 주요 개발 내용 요약

이번 주는 **정책 관리 UI 전면 재구현**, **실제 모델 서빙 배포 파이프라인 구축**, **클러스터 관리 안정화**에 중점을 둔 기간이었습니다. 총 **12개의 커밋**, **12개의 PR 머지**, **15개의 이슈 생성**이 완료되었으며, 멀티 클러스터 관리 정책 시스템 기반의 **정책 관리 화면이 기획서 기반으로 전면 재구현**되고, mock 데이터 기반 배포 UI가 **실제 멀티 클러스터 관리를 통한 모델 서빙 배포 파이프라인**으로 전환되었습니다.

{% include youtube.html id="N1h9psWRRm4" %}

---

## 새로운 기능 (New Features)

### 1. 모델 서빙 배포 파이프라인 구현 (Mock → Real)

**구현 완료**: 배포 폼 제출 → 멀티 클러스터 관리를 통해 멤버 클러스터에 실제 모델 서빙 Pod 생성되는 엔드투엔드 파이프라인

**주요 성과**:
- **Backend (신규)**:
  - 배포 CRUD API (`POST/GET/DELETE /api/deployments`, `GET /api/runtimes`)
  - 런타임 추상화 (`runtime/config.go`): Ollama, vLLM, TGI, llama-cpp 프리셋 정의
  - `serving_deployments` 테이블 + `node_port`, `node_ip` 컬럼 마이그레이션
  - 멀티 클러스터 관리를 통한 Deployment + Service(NodePort) + PropagationPolicy 생성/삭제/상태 조회
  - Ollama startup script 방식 적용 (CrashLoopBackOff 해결)
  - DNS-1035 네이밍 검증 (`sanitizeDNS1035`): 모델명의 `.`, `:` 등 특수문자 처리
  - 멤버 클러스터 노드 IP 자동 조회 (`GetClusterNodeIP`)
- **Frontend (신규)**:
  - 배포 API 클라이언트 (`lib/api/deployments.ts`): fetch, create, delete, runtimes
  - 인스턴스 목록 컴포넌트 (`instance-list.tsx`): 실시간 폴링, 검색, 삭제
- **Frontend (변경)**:
  - 배포 폼: mock → 실제 API, 클러스터/런타임 동적 로드, 모델 프리셋
  - 인스턴스 상세: 외부 엔드포인트 카드(`http://<nodeIP>:<nodePort>`), curl 예시, 자동 폴링

**해결된 기술 문제**:
| 문제 | 원인 | 해결 |
|------|------|------|
| DNS-1035 네이밍 오류 | 모델명(`llama3.2:1b`)의 `.`, `:` 등이 K8s 리소스명에 포함 | `sanitizeDNS1035` 적용 |
| Ollama CrashLoopBackOff | `ollama pull`이 `ollama serve` 없이 실행 불가 | startup script 방식으로 전환 |

**관련 커밋**: `17fde33` (PR #37, Closes #32)

---

### 2. 정책 관리 화면 전면 재구현

**구현 완료**: 멀티 클러스터 관리 정책 시스템 기술 리서치 → 기획서 작성 → UI/UX 전면 재구현 (5개 PR)

#### 2-1. 배치 전략(PlacementStrategy) 생성/편집 폼

- Sheet(슬라이드 패널) 레이아웃, `react-hook-form` + `Zod` 기반 폼 상태 관리 및 검증
- 배치 모드 선택 (자동/수동), 대상 클러스터 선택 방식 (라벨 기반 / 클러스터 이름 지정)
- 자동 배치 규칙 편집기: 조건 타입(모델 크기/GPU 타입/런타임), 연산자, 동적 규칙 추가/삭제 (`useFieldArray`)
- 기본 프로파일 연결, 고급 옵션 (기본 전략 설정, 리소스 보존, 충돌 해결)
- shadcn/ui 컴포넌트 9종 추가 (Sheet, Form, RadioGroup, Checkbox, Label, Textarea, Tooltip, Sonner 등)

**관련 커밋**: `2f5b4e3` (PR #28, Closes #23)

#### 2-2. 리소스 프로파일(ResourceProfile) 생성/편집 폼

- Zone A (High Performance): GPU/메모리/CPU + 스케줄러 설정 (토폴로지, NVLink, MIG + 조건부 UI)
- Zone B (General Purpose): GPU/메모리/CPU + 스케줄러 설정 (QoS, GPU 공유, Gang Scheduling, 오버커밋)
- 인라인 경고 메시지: MIG+NVLink 충돌, LSE/LSR 오버커밋, BE 오버커밋 안내

**관련 커밋**: `e6f6a30` (PR #29, Closes #24)

#### 2-3. 장애 복구(FailoverPolicy) 생성/편집 폼 + 타임라인 시각화

- Sheet 폼 (Zod 검증, 실시간 타임라인 프리뷰, 활성화 토글)
- 공용 타임라인 시각화 컴포넌트 (`failover-timeline.tsx`): mini(카드용) / full(폼 미리보기) variant
- SVG 수평 타임라인 + 바 시각화, `watch`로 입력값 변경 시 실시간 업데이트
- 초→분 자동 환산, 이름 중복 검사

**관련 커밋**: `15a1287` (PR #30, Closes #25)

#### 2-4. 빈 상태 UI + 경고/검증 시스템 + 삭제 확인

- 공용 `EmptyState` 컴포넌트: 각 탭에서 데이터가 없을 때 빈 상태 화면 표시
- 기본 전략 세트 일괄 생성: 배치 전략 3개 + 리소스 프로파일 3개 + 장애 복구 1개를 원클릭 생성
- `DeletePolicyDialog`: 사용 중인 전략 삭제 차단(blocked) + 일반 삭제 확인(confirm) + 기본 정책 추가 경고
- 경고 다이얼로그: 기본 전략 변경 / Failover 비활성화 시 확인 다이얼로그

**관련 커밋**: `c961c90` (PR #31, Closes #26)

---

### 3. 클러스터 해제(삭제) UI 구현

**구현 완료**: 클러스터 카드 및 상세 다이얼로그에서 삭제 기능 추가

**주요 기능**:
- 클러스터 카드에 `Trash2` 삭제 아이콘 버튼, 상세 다이얼로그 하단에 "클러스터 해제" 버튼 추가
- shadcn/ui `AlertDialog` 기반 삭제 확인 다이얼로그 (운영 중 클러스터에 대한 상태별 경고 메시지)
- 삭제 성공 시 클러스터 목록 자동 새로고침, 삭제 중 로딩 상태 표시
- **버그 수정**: GPU 타입(`NVIDIA T4` 등)을 K8s 라벨에 저장하면 공백/쉼표로 인해 라벨 유효성 검사 실패 — DB(`cluster_metas`)에만 저장하도록 변경

**관련 커밋**: `895938a` (PR #11, Closes #10)

---

## 버그 수정 (Bug Fixes)

### 1. Push 모드 클러스터 등록 시 SA/RBAC 자동 프로비저닝

**수정 완료**: Push 모드 클러스터가 `not-ready` 상태로 유지되던 문제 해결

**수정 내용**:
- **SA/RBAC 자동 프로비저닝** (`멀티 클러스터 관리/client.go`):
  - `provisionMemberClusterRBAC`: 멤버 클러스터에 Namespace → ServiceAccount → ClusterRoleBinding(cluster-admin) → Token Secret 생성, SA 토큰 반환
  - `cleanupMemberClusterRBAC`: 클러스터 삭제 시 멤버 클러스터 리소스 정리 (best-effort)
- **Client certificate 인증 지원** (프론트엔드 + 백엔드):
  - kubeconfig의 `client-certificate-data` / `client-key-data` 추출 및 백엔드 전달
  - 토큰 없이 client cert로도 멤버 클러스터 접속 및 프로비저닝 가능
- **연결 테스트 강화**: `ServerVersion()` 호출에서 노드 목록 조회(`Nodes().List`)로 변경하여 실제 인증/권한 검증
- **cluster_metas 중복 등록 수정**: `Create`를 upsert 방식으로 변경하여 재등록 시 duplicate key 에러 방지

**관련 커밋**: `b2c3b9b` (PR #12, Closes #8)

---

### 2. 멀티 클러스터 관리 etcd PVC 미설정으로 인한 데이터 유실 및 컨트롤 플레인 장애

**수정 완료**: 멀티 클러스터 관리 Helm 설치 시 etcd에 PVC가 없어 Pod 재시작 시 데이터가 유실되는 문제 해결

**수정 내용**:
- `Makefile`의 `멀티 클러스터 관리-setup`에 `etcd.internal.storageType=pvc`, `pvc.size=5Gi` 설정 추가
- `docs/troubleshooting/` 디렉토리 신설:
  - `멀티 클러스터 관리_ETCD_DATA_LOSS.md`: 장애 증상, 원인, 해결, 재발 방지
  - `멀티 클러스터 관리_DIAGNOSIS_GUIDE.md`: 일반 진단 절차 및 유용한 명령어 모음

**관련 커밋**: `adf6807` (PR #14, Closes #13)

---

### 3. 클러스터 카드에서 중복 삭제 버튼 제거

**수정 완료**: 클러스터 카드의 삭제 버튼을 제거하고 상세 다이얼로그 내 "클러스터 해제" 버튼만 유지

- 파괴적 작업(클러스터 해제)에 대한 실수 클릭 방지 및 UI 정리
- `ClusterDeleteDialog`, `Trash2` 아이콘, `showDeleteConfirm` state 제거

**관련 커밋**: `3944243` (PR #16, Closes #15)

---

## 리팩토링 (Refactoring)

### 1. 정책 관리 타입 모델 재정의 + Custom Hook + 카드 개선

- **용어 통일**: `PropagationPolicy` → `PlacementStrategy`, `OverridePolicy` → `ResourceProfile`로 H-MAS 추상화에 맞게 변경 (하위 호환 alias 유지)
- **Custom Hook 도입**: `usePlacementStrategies`, `useResourceProfiles`, `useFailoverPolicies` 훅으로 데이터 로직 분리 (향후 API 연동 대비)
- **카드 컴포넌트 개선**: 배치 전략 카드에 기본 프로파일/인스턴스 수 표시, 리소스 프로파일 Zone A/B 비교 레이아웃, 장애 복구 타임라인 시각화 추가
- **Mock 데이터 갱신**: 기획서 §11.4 기반 기본 전략 세트 mock 데이터로 교체

**관련 커밋**: `ab9e31b` (PR #27, Closes #22)

---

## 문서화 (Documentation)

### 신규 기술 문서 (5개)

| 문서 | 내용 | 관련 커밋 |
|------|------|-----------|
| `SCHEDULER_CONFIGURATION.md` | 스케줄러 타입별 기능, 설정 체계, 배포 파이프라인 역할 정리 (752줄) | `54731ce` |
| `INFERENCE_RUNTIME_ANALYSIS.md` | 인퍼런스 런타임 비교 분석 (Ollama, vLLM, TGI, llama.cpp, SGLang) | `17fde33` |
| `03-resource-profile-zone-design.md` | v0.9 Zone 2개(HP/GP) 고정 결정 근거 및 v1.0+ 동적 Zone 확장 설계 방향 | `e6f6a30` |
| `멀티 클러스터 관리_ETCD_DATA_LOSS.md` | 멀티 클러스터 관리 etcd 데이터 유실 장애 케이스 문서 (증상, 원인, 해결, 재발 방지) | `adf6807` |
| `멀티 클러스터 관리_DIAGNOSIS_GUIDE.md` | 멀티 클러스터 관리 일반 진단 절차 및 유용한 명령어 모음 | `adf6807` |

### 대규모 기획 문서 (멀티 클러스터 관리 정책 관리 리서치)

**멀티 클러스터 관리 정책 관리 기능 기획 리서치** (PR #21, 17개 파일, +9,383줄):

| 구분 | 문서 | 내용 |
|------|------|------|
| 기술 리서치 | `tech/01-policy-system.md` | 멀티 클러스터 관리 정책 시스템 개요 |
| 기술 리서치 | `tech/02-usage-patterns.md` | 사용 패턴 및 베스트 프랙티스 |
| 기술 리서치 | `tech/03-deep-dive.md` | 심화 (CLI, 아키텍처, 경쟁 분석) |
| 기술 리서치 | `tech/04-implications.md` | 기획 시사점 통합 |
| H-MAS 분석 | `analysis/01-policy-scope.md` | 정책 관리 범위 분석 |
| H-MAS 분석 | `analysis/02-policy-menu-spec.md` | 정책 메뉴 상세 기획서 (화면 설계, API, DB 스키마, 멀티 클러스터 관리 매핑, UX 가이드라인) |
| 기획 | `planning/01~08` | 코어 기능, 페르소나, 사용자 여정, 기능 스펙, IA, 기술 아키텍처, NFR, 종합 계획 |

**핵심 결정 사항**:
| 결정 | 내용 |
|------|------|
| 정책 소유권 | H-MAS DB 저장 + 멀티 클러스터 관리 동기화 (Reconciler) |
| 추상화 수준 | 멀티 클러스터 관리 용어 숨김, 모델 중심 UX |
| 멀티 클러스터 관리 범위 | PropagationPolicy + OverridePolicy + Failover (3가지만) |
| Suspension | v1 제외 |

**관련 커밋**: `a6204ce` (PR #21, Closes #19)

---

## 미해결 이슈 (Open Issues)

### 1. 배포 정책 레이어 및 스케줄러 통제 구현 (#33)

- #32 (배포 파이프라인)의 후속 이슈
- ClusterAffinity 기반 배치 → Zone A/B 분류 + 스케줄러 선택 → NUMA/토폴로지 인식 → Failover/재배치까지 4단계 점진적 확장 계획
- GPU 토폴로지(NUMA, NVLink, PCIe) 인식 하드웨어 최적화 스케줄링 구현 필요

### 2. 배포 시 런타임 고급 설정 및 모델 로딩 최적화 (#34)

- **런타임 고급 설정**: Ollama 환경변수(컨텍스트 길이, 동시 요청, GPU 레이어 등) 및 vLLM/TGI 설정 지원 필요
- **모델 로딩 최적화**: 현재 `emptyDir` 사용으로 Pod 시작 시 매번 모델 재다운로드 → PVC 기반 모델 영속화, 사전 로드 Job, readinessProbe 개선 필요
- 대형 모델(70B, ~40GB)에서 30분+ 다운로드 시간 문제

### 3. H-MAS 래퍼 컨테이너 이미지 도입 검토 (#35)

- 공식 런타임 이미지 위에 H-MAS 전용 래퍼 이미지 빌드 검토
- 엔트리포인트 스크립트, 헬스체크, 메트릭 수집, 환경변수 표준화, 보안 하드닝 등
- GPU 확보 후 프로덕션 서빙 시작 시점(2026년 4월~) 도입 목표

### 4. 추론 API 프록시 구현 — 통합 엔드포인트 (#36)

- 현재 각 모델별 개별 NodePort Service로 노출 → 사용자가 모델마다 다른 IP:포트를 알아야 하는 문제
- H-MAS 백엔드에 추론 프록시 구현: `POST /api/inference/{deployment-name}/v1/chat/completions`
- SSE 스트리밍 지원, 요청 로깅, 메트릭 수집, API 키 인증, Rate Limiting 등 점진적 확장 계획

---

## 이번 주 통계

| 항목 | 수치 |
|------|------|
| 총 커밋 수 | 12개 |
| 머지된 PR | 12개 (#11, #12, #14, #16, #18, #21, #27, #28, #29, #30, #31, #37) |
| 생성된 이슈 | 15개 (#13, #15, #17, #19, #20, #22~#26, #32~#36) |
| 해결된 이슈 | 13개 (#8, #10, #13, #15, #17, #19, #20, #22~#26, #32) |
| 미해결 이슈 | 4개 (#33, #34, #35, #36) |
| 신규 기술/기획 문서 | 22개+ (리서치 17개 파일 포함) |
| 코드 변경량 | +22,031줄 / -1,751줄 |

---

## 결론

이번 주는 **H-MAS가 클러스터 관리 플랫폼에서 실제 AI 모델 서빙 플랫폼으로 도약한 전환점**이었습니다. 멀티 클러스터 관리 정책 시스템에 대한 심층 리서치를 바탕으로 정책 관리 UI를 전면 재구현했으며, mock 데이터 기반의 배포 UI를 실제 멀티 클러스터 관리를 통한 모델 서빙 배포 파이프라인으로 전환하여 **배포 폼 제출부터 멤버 클러스터 Pod 생성까지 엔드투엔드 동작**을 달성했습니다.

**핵심 성과**:
1. **실제 모델 서빙 배포**: Ollama 런타임으로 모델 서빙 Pod 생성 → NodePort 외부 접근 → curl 추론 요청까지 전 과정 동작
2. **정책 관리 UI 전면 재구현**: 배치 전략/리소스 프로파일/장애 복구 3종 생성/편집/삭제 폼 + 빈 상태 UI + 경고/검증 시스템
3. **클러스터 관리 안정화**: Push 모드 SA/RBAC 자동 프로비저닝, etcd PVC 영속화, 삭제 UI UX 개선
4. **체계적 기획 기반 개발**: 멀티 클러스터 관리 정책 리서치 → 기획서 → UI 구현의 체계적 워크플로우 확립
5. **대규모 기술 문서화**: 스케줄러 설정 아키텍처, 인퍼런스 런타임 분석, Zone 설계 결정 등 22개+ 문서

**다음 주 계획**:
- 배포 정책 레이어 구현 (ClusterAffinity 기반 배치) (#33)
- 런타임 고급 설정 및 모델 로딩 최적화 (#34)
- 추론 API 프록시 구현 (통합 엔드포인트) (#36)
- 정책 관리 UI를 실제 멀티 클러스터 관리 API와 연동

---

**문서 작성일**: 2026년 2월 15일 

