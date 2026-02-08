---
title: 'H-MAS release notes (2026-02-01~02-07)'
date: 2026-02-08
permalink: /posts/2026/02/h-mas-weekly-release-notes-0201-0207/
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
## 2026년 2월 1일 - 2월 7일

### 주요 개발 내용 요약

이번 주는 **Go 백엔드 API 서버 구축**, **클러스터 등록 UX 개선**, **다중 클러스터 배포 자동화**, **기술 설계 문서 대량 작성**에 중점을 둔 기간이었습니다. 총 **18개의 커밋**, **4개의 PR 머지**, **6개의 이슈 생성**이 완료되었으며, 프론트엔드의 mock 데이터 기반 UI가 **실제 다중 클러스터와 연동되는 완전한 풀스택 시스템**으로 전환되었습니다.

---

## 새로운 기능 (New Features)

### 1. Go 백엔드 API 서버 및 프론트엔드 클러스터 탭 연동

**구현 완료**: 다중 클러스터 데이터를 프론트엔드에 연동하는 백엔드 API 서버

**주요 성과**:
- **Backend (신규)**:
  - Echo v4 REST API 서버: 다중 클러스터 솔루션의 client-go를 통한 Cluster/Policy CRUD 및 조회
  - PostgreSQL 16: `ClusterMeta`, `AuditLog`, `ClusterEvent` 테이블 (golang-migrate 스키마 관리)
  - MultiCluster-DB Reconciler: 외부에서 등록된 클러스터를 DB에 자동 동기화 (60초 주기)
  - Swagger/OpenAPI: swaggo/swag 기반 자동 생성 API 문서 (`/swagger/index.html`)
  - K8s 매니페스트: backend Deployment/Service/SA, postgres StatefulSet/Service/Secret
- **Frontend**:
  - API 클라이언트 레이어: `lib/api/client.ts` (공통 fetch + ApiError), `lib/api/clusters.ts` (타입 매핑)
  - Next.js rewrites 프록시: Frontend→Backend Same-Origin 통신 (CORS 불필요)
  - 클러스터 탭을 mock 데이터에서 `GET /api/clusters` 실제 호출로 전환
  - 클러스터 등록 다이얼로그: `POST /api/clusters`, Bootstrap Token API 연동
  - 클러스터 상세 다이얼로그: 정보/레이블·Taints/이벤트 3탭 구현
  - 로딩/에러/빈 상태 UI: 스켈레톤, 스피너, 에러 배너, 재시도 버튼

**관련 커밋**: `65280b9`

---

### 2. Kubeconfig 파일 파싱 및 연결 테스트 기능

**구현 완료**: 클러스터 등록 시 kubeconfig 업로드로 필드 자동 채움 및 실시간 연결 테스트

**주요 기능**:
- **프론트엔드**: kubeconfig YAML 파싱(`js-yaml`) 후 `current-context` 기준으로 API 엔드포인트, CA 인증서, 토큰 필드 자동 채움
- **프론트엔드**: 드래그 앤 드롭 파일 업로드 지원 추가
- **프론트엔드**: 연결 테스트 성공 시 토큰 없이도 다음 단계 진행 가능하도록 유효성 검사 개선
- **백엔드**: `POST /api/clusters/test-connection` 엔드포인트 추가 (client-go `ServerVersion()` 호출로 실제 연결 테스트, 15초 타임아웃)

**설계 결정**:

| 항목 | 결정 |
|------|------|
| 필드 자동 채움 | 프론트엔드 파싱 (즉각 반응, 백엔드 변경 불필요) |
| 연결 테스트 | 백엔드 API (실제 네트워크 접속 필요) |
| Multi-context | `current-context` 기준 자동 선택 |
| 타임아웃 | 15초 |

**관련 커밋**: `d8f55b3`

---

### 3. 다중 클러스터 솔루션 Helm Chart 배포 자동화

**구현 완료**: Makefile을 통한 다중 클러스터 Control Plane 설치 및 멤버 클러스터 등록 자동화

**주요 기능**:
- `make dev-setup`에 다중 클러스터 Control Plane 설치 + 멤버 클러스터 등록 통합
- `make dev-setup-lite` 추가 (다중 클러스터 솔루션 없이 Frontend만 구축)
- `make install-tools` 추가 (kubectl, helm, multiclusterctl 자동 설치, macOS/Linux 지원)
- `make multicluster-setup/join/status/clean` 개별 타겟 추가
- 로컬 클러스터의 API 엔드포인트를 ClusterIP로 치환하여 Pod 내부 접근 문제 해결
- 모든 단계에 멱등성 체크 포함 (반복 실행 안전)

**관련 커밋**: `5c01445`

---

### 4. 프론트엔드 데모 동영상 생성 도구

**구현 완료**: Playwright 기반 자동 녹화 + SRT 자막 생성 + FFmpeg 자막 합성 도구

**관련 커밋**: `b0bd1e1`, `e9a8732` (빌드 오류로 `tools/video-generator/`로 위치 변경)

---

## 버그 수정 (Bug Fixes)

### 1. 클러스터 등록 기능 개선 및 백엔드 버그 수정

**수정 완료**: GPU 선택 필수 해제 + 백엔드 로직 버그 2건 수정

**수정 내용**:
- **Frontend**: GPU 선택을 필수에서 선택사항으로 변경하여 CPU 전용 클러스터 등록 가능
- **Frontend**: "GPU 없음 (CPU 전용)" 버튼 추가 및 안내 메시지 표시
- **Frontend**: `region`에 한글 표시명(`본사 IDC`) 대신 영문 value(`hq-idc`) 전송하도록 수정
- **Backend**: `ClusterMeta`에 `TableName()` 메서드 추가 — GORM이 테이블명을 잘못 생성하던 문제 해결
- **Backend**: `GetCluster` 에러를 `_`로 무시하던 코드 수정 — 항상 "already exists" 반환하던 문제 해결

**관련 커밋**: `0f7e939`

---

### 2. Kubeconfig 파일 선택 제한 해제

**수정 완료**: `.config` 확장자 또는 확장자 없는 파일 선택 불가 문제 해결

- `accept=".yaml,.yml"` 속성 제거로 모든 파일 형식 선택 가능
- Kubeconfig 파일은 `config`, `stg.config`, `kubeconfig.yaml` 등 다양한 형태 존재

**관련 커밋**: `a39b23a`

---

### 3. Frontend 빌드 실패 수정

**수정 완료**: 중복된 `video-generator` 디렉토리가 TypeScript 컴파일에 포함되어 Playwright 모듈 의존성으로 빌드 실패하던 문제 해결

**관련 커밋**: `e9a8732`

---

## 리팩토링 (Refactoring)

### 1. 온프레미스 중심 클러스터 관리 UI 변경

- 퍼블릭 클라우드 프로바이더(AWS, GCP, Azure) 선택 UI 제거
- 리전 개념을 데이터센터 위치로 변경 (본사 IDC, 서울 DC1 등)
- 클러스터 카드에서 provider 표시 제거
- 목업 데이터를 온프레미스 데이터센터 기반으로 수정

**관련 커밋**: `2129a61`

---

### 2. K8s 리소스명 간소화 및 정리 타겟 추가

- Deployment/Service 이름을 `hmas-frontend`에서 `frontend`로 변경 (네임스페이스 내 중복 prefix 제거)
- `make dev-clean` 추가: K8s 제거 + Docker 이미지/아티팩트 일괄 정리

**관련 커밋**: `a75d58a`

---

### 3. docs 디렉토리 재구조화

17개 문서를 목적별 4개 하위 폴더로 분류:
- `product/`: 제품 전략 (PRD, ROADMAP, FEATURE_SPEC)
- `architecture/`: 설계 문서 (BACKEND_ARCHITECTURE, MODEL_REGISTRY 등 6개)
- `guides/`: 개발 가이드 (DEVELOPMENT_GUIDE, API_REFERENCE 등 5개)
- `research/`: 향후 계획 (FUTURE_REBALANCING, MIG 관련 3개)

추가 작업:
- `docs/INDEX.md` 신설 (전체 문서 네비게이션, 읽는 순서 추천)
- 78개 상호 참조 링크를 새 경로에 맞게 전수 수정
- README.md 문서 섹션을 카테고리별 테이블로 재구성

**관련 커밋**: `de6c0de`

---

## 문서화 (Documentation)

### 신규 설계 문서 (6개)

| 문서 | 내용 | 관련 커밋 |
|------|------|-----------|
| `MODEL_REGISTRY_DESIGN.md` | 플러거블 어댑터 아키텍처 설계 (Built-in, MLflow, HuggingFace, W&B, S3 어댑터) | `4133807` |
| `FUTURE_REBALANCING_DESIGN.md` | Model-Infra-Aware 배치 리밸런싱 설계 (시나리오 5가지, 옵션 3가지) | `fd02df7` |
| `FUTURE_MIG_SUPPORT_DESIGN.md` | Multi-Instance GPU 지원 설계 (L1/L2/L3 기능 레벨별 방향) | `067199c` |
| `MIG_PERFORMANCE_ANALYSIS.md` | MIG 성능 분석 (학습 vs 인퍼런스, 벤치마크 수치 포함) | `2aa10c4` |
| `COMPUTE_PLANE_PACKAGE.md` | 멤버 클러스터 표준 설치 패키지 설계 (h-mas-agent, 설치 프로파일) | `1a4d3cd` |
| `SINGLE_CLUSTER_SERVING.md` | 단일 클러스터 서빙 관리 가이드 | `4bfdfe4` |

### 백엔드 기술 문서 (PR #2와 함께 추가, 6개)

| 문서 | 내용 |
|------|------|
| `BACKEND_ARCHITECTURE.md` | 아키텍처, 기술 스택 의사결정 |
| `DATABASE_MIGRATION.md` | golang-migrate 가이드 |
| `API_REFERENCE.md` | 전체 API 엔드포인트 레퍼런스 |
| `DEVELOPMENT_GUIDE.md` | 개발 환경 구축 워크플로우 |
| `FRONTEND_INTEGRATION.md` | 프론트엔드 연동 패턴 |
| `FEATURE_SPEC.md` | 전체 기능 구현 상태 추적 |

### 기존 문서 최신화

- **README.md**: Backend 기술 스택(Go/Echo/PostgreSQL) 반영, 프로젝트 구조 갱신, Phase 3 진행 상태 업데이트, 15개 문서 링크 테이블 추가
- **PRD.md**: 관련 문서 16개로 확장, 기술 스택 확정(Echo v4, GORM), 통신 패턴 신설, 마일스톤 Phase 3 완료 항목 반영
- **ROADMAP.md**: Backend 개발 현황 갱신, Phase 2.5 신설, Infrastructure 현황 업데이트

**관련 커밋**: `b7ae509`, `4a4236b`

---

## 미해결 이슈 (Open Issues)

### 1. Push 모드 클러스터 등록 후 비정상 상태 유지 

- Push 모드로 등록한 클러스터가 `not-ready` 상태로 유지되는 문제
- 다중 클러스터 관리 솔루션이 멤버 클러스터에서 정보를 수집하지 못함 (노드 0, 메모리 0, GPU 0)
- **원인**: `multiclusterctl join`과 달리 API를 통한 Push 모드 등록은 ServiceAccount/RBAC를 자동 생성하지 않음
- **해결 방향**: 멤버 클러스터에 ServiceAccount/RBAC 자동 생성 또는 credential 유효성 사전 검증

### 2. 클러스터 해제(삭제) UI 추가 

- 등록된 클러스터를 해제할 수 있는 UI가 없음
- 백엔드 API(`DELETE /api/clusters/:name`)와 프론트엔드 API 클라이언트(`deleteCluster()`)는 이미 구현됨
- **필요 작업**: 클러스터 카드/상세 다이얼로그에 삭제 버튼 + 확인 다이얼로그 구현

---

## 이번 주 통계

| 항목 | 수치 |
|------|------|
| 총 커밋 수 | 18개 |
| 머지된 PR | 4개 (#2, #4, #7, #9) |
| 생성된 이슈 | 6개 (#1, #3, #5, #6, #8, #10) |
| 해결된 이슈 | 4개 (#1, #3, #5, #6) |
| 미해결 이슈 | 2개 (#8, #10) |
| 신규 설계 문서 | 12개 |

---

## 결론

이번 주는 **H-MAS가 프론트엔드 프로토타입에서 풀스택 애플리케이션으로 진화한 전환점**이었습니다. Go 백엔드 API 서버를 구축하여 다중 클러스터 관리 솔루션의 멀티클러스터 데이터를 실제로 연동했으며, 클러스터 등록 UX를 Kubeconfig 자동 파싱 및 연결 테스트 기능으로 대폭 개선했습니다.

**핵심 성과**:
1. **풀스택 전환**: mock 데이터 기반 UI → 실제 다중 클러스터 솔루션 API 연동
2. **백엔드 신규 구축**: Go/Echo/PostgreSQL 기반 REST API + Swagger 문서
3. **배포 자동화**: 다중 클러스터 Helm Chart 배포를 Makefile에 완전 통합
4. **클러스터 등록 UX**: Kubeconfig 파싱, 드래그앤드롭, 연결 테스트, CPU 전용 클러스터 지원
5. **체계적 문서화**: 12개 설계/가이드 문서 작성, 17개 문서 카테고리별 재구조화

**다음 주 계획**:
- Push 모드 클러스터 등록 안정화 
- 클러스터 삭제 UI 구현 
- 클러스터 관리 기능 고도화 (상태 모니터링, 이벤트 알림)

---

**문서 작성일**: 2026년 2월 8일
