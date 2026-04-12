---
title: 'H-MAS release notes (2026-04-05~04-11)'
date: 2026-04-12
permalink: /posts/2026/04/h-mas-weekly-release-notes-0405-0411/
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
## 2026년 4월 5일 - 4월 11일 (Iteration 17)

### 주요 개발 내용 요약

이번 주는 **서빙 런타임 이미지 관리 체계 구축**, **멀티클러스터 이미지 사전 캐싱**, **배포 상태 동기화 개선**에 중점을 둔 기간이었습니다. 총 **16개의 커밋**, **5개의 PR 머지**, **8개의 이슈 생성**이 완료되었으며, v0.4.0 릴리즈 이후 v0.9 방향의 첫 피처 개발 사이클로서 런타임 이미지 관리와 배포 안정성을 대폭 강화한 주간이었습니다.

{% include youtube.html id="axMXcVPf8H0" autoplay=true %}

---

## 새로운 기능 (New Features)

### 1. 서빙 런타임 이미지 관리 — 2-Layer 이미지 Resolution (#180 → PR #183)

**구현 완료**: 하드코딩된 `latest` 태그 기반 서빙 런타임 이미지 관리를 DB 기반 관리자 오버라이드 체계로 전환하여, 버전 고정·프라이빗 레지스트리 연동·재배포 시간 70% 단축(20분→6분)을 달성

**주요 성과**:

- **Backend**:
  - DB 마이그레이션: `runtime_image_settings`, `system_settings` 테이블 추가
  - `RuntimeImageSettingRepository`, `SystemSettingRepository` 신규 구현
  - `runtime.ResolveImage()` 핵심 로직: `배포별 customImage` > `DB 관리자 오버라이드` > `코드 프리셋 기본값` 3단 우선순위
  - `imagePullPolicy` 자동 결정: `latest` 태그 → `Always`, 특정 태그 → `IfNotPresent`
  - 런타임 이미지 설정 CRUD + 글로벌 레지스트리 설정 5개 API 엔드포인트
  - 배포 핸들러에 이미지 Resolution 통합, `imagePullSecrets` 주입 지원
  - `SplitImage`, `ResolveImage` 테이블 드리븐 단위 테스트

- **Frontend**:
  - 런타임 이미지 관리 설정 페이지 (`/settings/runtimes`) 신규
  - 런타임별 CPU/GPU 이미지 태그 설정 UI
  - 글로벌 레지스트리 프리픽스 설정 + 최종 이미지 미리보기
  - 레지스트리 태그 목록 외부 링크 제공

**의사결정 사항**:

| 결정 | 내용 |
|------|------|
| 이미지 Resolution 구조 | 2-Layer — 코드 프리셋(안전한 기본값) + DB 오버라이드(관리자) + 배포별 customImage 3단 우선순위 |
| imagePullPolicy 전략 | 태그 기반 자동 결정 — `latest`→`Always`, 특정 태그→`IfNotPresent`, 배포별 오버라이드 가능 |
| 설정 범위 | 글로벌 레지스트리 프리픽스 + 런타임별 이미지 태그 분리 — 온프레미스/에어갭 환경 대응 |

**해결된 기술 문제**:

| 문제 | 원인 | 해결 |
|------|------|------|
| 매 배포마다 이미지 풀로 10-20분 소요 | `latest` 태그 → `imagePullPolicy: Always` 고정 | 태그 기반 자동 결정 + 관리자 버전 고정 → `IfNotPresent`로 캐시 히트 |
| GPU 아키텍처 호환성 이슈 발견 | vLLM 실환경 검증 중 RTX 5060 Ti(Blackwell)에서 특정 버전 비호환 | 관리자가 호환 버전으로 태그 고정 가능, #182로 상태 감지 이슈 생성 |

**관련 커밋**: `7f38638` (PR #183, Closes #180)

---

### 2. 멀티클러스터 서빙 런타임 이미지 사전 캐싱 (#181 → PR #187)

**구현 완료**: #180의 이미지 관리 설정을 기반으로, 캐싱이 활성화된 멤버 클러스터의 GPU 노드에 서빙 런타임 이미지를 xxxxxxx DaemonSet으로 사전 배포하여 첫 배포 시에도 이미지 풀 시간을 제거

**주요 성과**:

- **Backend**:
  - DB 마이그레이션 (`000019`): `runtime_image_settings.cache_enabled`, `cluster_metas.image_cache_enabled` 컬럼 추가
  - `xxxxxxx/image_warmer.go`: 런타임별 DaemonSet + PropagationPolicy 생성/삭제/조회, 캐싱 상태 확인, 클러스터 레이블 관리
  - `sync/image_prepull_reconciler.go`: DB ↔ xxxxxxx 간 desired/actual 상태 주기적 보정 (60초 주기)
  - 클러스터 API: `PATCH /clusters/:name/image-cache` — 클러스터별 캐싱 토글 + xxxxxxx 레이블 동기화
  - 런타임 이미지 API: `cacheEnabled` 토글 시 DaemonSet 즉시 반영, `GET /settings/runtime-images/cache-status` 상태 조회
  - 기존 Reconciler에 `hmas/image-cache` 레이블 정합성 보정 로직 추가

- **Frontend**:
  - 런타임 이미지 설정에 캐싱 토글 Switch UI 추가
  - 클러스터 API 클라이언트에 `imageCacheEnabled` 필드 및 `updateClusterImageCache` 함수 추가

- **Docs**:
  - `IMAGE_DISTRIBUTION_OPTIMIZATION.md`: DaemonSet, P2P(Spegel), Lazy Pull(Nydus), Warm Pool 등 대안 기술 비교 및 단계별 개선 로드맵

**의사결정 사항**:

| 결정 | 내용 |
|------|------|
| 런타임별 DaemonSet 분리 | 단일 DaemonSet에 모든 런타임 initContainer를 담으면 노드 디스크 압박(20-40GB+), 직렬 실행(40분+), 전체 롤아웃 문제 발생 → 런타임별 개별 DaemonSet으로 독립적 롤아웃·선택적 캐싱·개별 모니터링 가능 |
| GC 방지 전략 | initContainer + pause 대신 main 컨테이너에서 런타임 이미지를 `sleep infinity`로 유지 → kubelet GC 대상에서 제외 |
| GPU 노드 타겟팅 | `nodeSelector: nvidia.com/gpu.present=true`로 GPU 노드에만 DaemonSet 스케줄링 |
| 클러스터별 선택적 캐싱 | 기본값 `false` — 관리자가 명시적으로 활성화, xxxxxxx 클러스터 레이블(`hmas/image-cache=enabled`)으로 PropagationPolicy 필터링 |
| 즉시 반영 + Reconciler 안전망 | API 핸들러에서 동기적으로 DaemonSet 반영 + 백그라운드 Reconciler(60초)가 드리프트/실패 보정 |

**관련 커밋**: `6894ed3`, `6263405`, `24be169`, `b15b99f` (PR #187, Closes #181)

---

### 3. 배포 상태 동기화 개선 — Pod 실패 상태 감지 (#182 → PR #188)

**구현 완료**: Kubernetes Deployment 조건만 확인하던 상태 동기화 로직을 Pod 레벨까지 확장하여, ImagePullBackOff·CrashLoopBackOff·OOMKilled·SchedulingFailed 등의 실패 상태를 UI에서 즉시 인지할 수 있도록 개선

**주요 성과**:

- **Backend**:
  - DB 마이그레이션 (`000020`): `status_reason` 컬럼 추가
  - `xxxxxxx/client.go`: `DiagnosePodsForDeployment` 구현 — 멤버 클러스터 Pod 컨테이너 상태 및 스케줄링 조건 직접 검사
  - `GetServingDeploymentStatusWithReason`: 2-tier 상태 모델 (`status` + `statusReason`)
  - `sync/reconciler.go`: 주기적 동기화 시 pending 배포에 대해 Pod 진단 실행, DB 상태 캐시
  - `repository/deployment.go`: `UpdateStatusWithReason` 추가

- **Frontend**:
  - `deployment-status.ts` 신규: 4개 컴포넌트에 분산되어 있던 상태 표시 로직을 공통 유틸로 통합
  - `instance-list.tsx`, `recent-deployments.tsx`, `logs-content.tsx`, `[id]/page.tsx` 리팩토링

**해결된 기술 문제**:

| 문제 | 원인 | 해결 |
|------|------|------|
| ImagePullBackOff/CrashLoopBackOff 시 UI에 "대기 중" 무한 표시 | 기존 로직이 Deployment `.status.conditions`만 확인, Pod 수준 실패 상태 미감지 | `DiagnosePodsForDeployment`로 Pod `containerStatuses[].state.waiting.reason` 직접 검사 |
| GPU/리소스 부족 시 스케줄링 실패 미감지 | Pod 스케줄링 조건 미확인 | Pod의 `conditions` 중 `PodScheduled=False` + reason 검사 추가 |
| 상태 표시 로직 4중 중복 | 4개 컴포넌트에서 각각 상태 뱃지/색상 로직 보유 | `deployment-status.ts` 공통 유틸로 통합 |

**상태 매핑**:

| Pod 상태 | H-MAS statusReason | UI 표시 |
|----------|-------------------|---------|
| `ImagePullBackOff` / `ErrImagePull` | `image_pull_failed` | "이미지 풀 실패" (경고) |
| `CrashLoopBackOff` | `crash_loop` | "크래시 반복" (에러) |
| `OOMKilled` | `oom_killed` | "메모리 부족" (에러) |
| `PodScheduled=False` (리소스) | `insufficient_resources` | "리소스 부족" (경고) |
| 그 외 에러 | `config_error` | "설정 오류" (에러) |

**관련 커밋**: `b902130`, `4fae41f`, `da9f786` (PR #188, Closes #182)

---

## 리팩토링 (Refactoring)

### 1. Ollama startup script 외부화 (#35 → PR #178)

**리팩토링 완료**: `client.go`에 30줄 인라인 하드코딩되어 있던 Ollama startup script를 `scripts/ollama-entrypoint.sh`로 분리하고 `go:embed`로 임베드

**주요 변경**:
- 모델명 전달: `fmt.Sprintf` → `MODEL_NAME` 환경변수 (스크립트를 정적 파일로 유지)
- SIGTERM/SIGINT graceful shutdown 추가 (`trap cleanup TERM INT`)
- `ollama show`로 PVC 캐시 확인 후 모델 존재 시 pull 스킵 (재시작 시간 단축, 에어갭 환경 지원)
- `TestOllamaEntrypointEmbed` 테스트 — 임베드 성공, shebang/필수 문자열 검증
- 실제 클러스터 배포 테스트에서 POSIX sh 호환성 확인 (`SIGTERM` → `TERM` 수정)

**관련 커밋**: `af2323d` (PR #178, Closes #35)

---

## 문서화 (Documentation)

### 1. v0.4.0 사용자 시나리오 워크스루 문서 (#176 → PR #177)

| 문서 | 내용 | 관련 커밋 |
|------|------|-----------|
| `docs/external/V04_WALKTHROUGH.md` | 신규 — v0.4.0 전체 사용자 시나리오를 실제 스크린샷(21장)과 함께 6개 구간으로 안내 | `2d6f40e` (PR #177) |
| `docs/external/PRODUCT_OVERVIEW.md` | v0.4.0 스크린샷으로 7개 이미지 교체, 문의처/이메일 수정 | `64d36f0` (PR #177) |
| `tools/pdf-generator/` | 신규 — marked + puppeteer 기반 Markdown → PDF 변환 도구 | `64d36f0` (PR #177) |
| 3종 PDF | V04_WALKTHROUGH.pdf (3.2MB), PRODUCT_OVERVIEW.pdf (2.1MB), H-MAS_Product_Deck.pdf (2.7MB, 12장 랜드스케이프 덱) | `64d36f0` (PR #177) |

### 2. 기술 리서치 문서

| 문서 | 내용 | 관련 커밋 |
|------|------|-----------|
| `IMAGE_DISTRIBUTION_OPTIMIZATION.md` | DaemonSet, P2P(Spegel), Lazy Pull(Nydus), Warm Pool 등 이미지 배포 최적화 기술 비교 및 단계별 개선 로드맵 | `b15b99f` (PR #187) |
| `SERVING_RUNTIME_IMAGE_MANAGEMENT.md` | 런타임 이미지 관리 아키텍처 설계 + 실환경 검증 결과 (배포 시간 측정, GPU 호환성) | `7f38638` (PR #183) |
| `CONTAINER_RUNTIME_MIRROR_GUIDE.md` | containerd 레지스트리 미러 설정 참고 가이드 | `7f38638` (PR #183) |
| 서버리스 서빙 리서치 초안 | Scale-to-Zero 방식 분석 문서 초안 (#186) | `27f3cb5` |


---

## 이전 Iteration 계획 달성도

Iteration 16에서 계획한 4개 항목 **1개 부분 달성, 3개 미착수** — 대신 런타임 이미지 관리 체계(#180, #181)와 배포 안정성(#182) 구축에 집중:

| 계획 | 이슈 | 상태 | 비고 |
|------|------|------|------|
| v0.9 Phase 계획 수립 및 우선순위 정리 | — | 🔄 부분 달성 | 이미지 관리/캐싱/상태 동기화 등 v0.9 방향의 첫 피처 사이클 완료, 리서치 이슈 4건 생성(#179, #184, #185, #186) |
| 서빙 배포 인플레이스 업데이트 API (PATCH) 구현 착수 | #125 | ⏸ 미착수 | 이미지 관리 체계 구축에 우선순위 전환 |
| 추론 API 프록시 구현 — 통합 엔드포인트 설계 | #36 | ⏸ 미착수 | 이미지 관리/캐싱 작업에 집중 |
| 브랜치 전략 및 릴리즈 프로세스 정립 | #166 | ⏸ 미착수 | v0.9 피처 개발 후 진행 예정 |

**추가 달성**: 계획에 없던 서빙 런타임 이미지 관리(#180), 멀티클러스터 이미지 사전 캐싱(#181), 배포 상태 동기화 개선(#182), Ollama 스크립트 외부화(#35 — 장기 미해결 이슈 클로즈), v0.4.0 워크스루 문서/PDF(#176), 리서치 이슈 4건 생성. v0.4.0 이후 첫 v0.9 피처 사이클로서 배포 운영 경험을 개선하는 핵심 인프라를 구축.

---

## 미해결 이슈 (Open Issues)

### 신규 리서치/피처 이슈 (4개, 이번 Iteration 생성)

| 이슈 | 제목 | 라벨 |
|------|------|------|
| #186 | 리서치: 서버리스 서빙(Scale-to-Zero) 방식 분석 및 H-MAS 적용 가능성 검토 | research |
| #185 | 서빙 배포 일시정지(Pause) 및 재시작(Resume) 기능 | enhancement |
| #184 | 리서치: 폐쇄망(Air-Gapped) 환경 배포 전략 수립 | research |
| #179 | 리서치: xxxxxxxxxxx 스케줄러 도입을 위한 기술 조사 및 통합 설계 | research |

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

### v0.9 운영/인프라 이슈 (1개)

| 이슈 | 제목 |
|------|------|
| #166 | v0.9: 브랜치 전략 및 릴리즈/배포 프로세스 정립 |

### 기존 미해결 이슈 (이전 Iteration에서 이관)

### 1. 스케줄러 어노테이션 변환 (xxx/xxxxxxxxxxx → xxxxxxx OverridePolicy) (#66)

- #179 xxxxxxxxxxx 리서치 이슈와 연계하여 검토 예정

### 2. 모델 프리셋 리스트 최신화 (#62)

- 잔여 작업: 최신 모델 시드 데이터 추가 필요

### 3. Failover / 재배치 정책 구현 (#39) — v0.95 마일스톤

### 4. NUMA / 토폴로지 인식 스케줄링 구현 (#38) — v0.95 마일스톤

### 5. 추론 API 프록시 구현 — 통합 엔드포인트 (#36)

### 6. 리서치: GPU 메모리 인지형 동적 모델 Bin Packing 전략 (#96)

---

## 이번 주 통계

| 항목 | 수치 |
|------|------|
| 총 커밋 수 | 16개 (+ merge 5건) |
| 머지된 PR | 5개 (#177, #178, #183, #187, #188) |
| 생성된 이슈 | 8개 (#176, #179, #180, #181, #182, #184, #185, #186) |
| 해결된 이슈 | 5개 (#35, #176, #180, #181, #182) |
| 미해결 이슈 | 21개 (#36, #38, #39, #62, #66, #96, #114, #120, #122, #123, #125~#128, #133, #134, #166, #179, #184, #185, #186) |
| 신규 기술 문서 | 4개 (이미지 배포 최적화 리서치, 런타임 이미지 관리 설계, containerd 미러 가이드, 서버리스 리서치 초안) |
| 코드 변경량 | +7,082줄 / -148줄 (68개 파일) |

---

## 결론

이번 주는 **v0.4.0 릴리즈 이후 첫 v0.9 피처 사이클로서 서빙 런타임 이미지 관리 체계를 전면 구축한 기간**이었습니다. 하드코딩된 `latest` 태그 기반 이미지 관리를 2-Layer Resolution 체계(#180)로 전환하고, 멀티클러스터 이미지 사전 캐싱(#181)으로 확장하여 재배포 시간 70% 단축과 첫 배포 이미지 풀 시간 제거를 달성했습니다.

배포 상태 동기화(#182)에서는 Pod 레벨 실패 상태(ImagePullBackOff, CrashLoopBackOff, OOMKilled, SchedulingFailed)를 감지하는 2-tier 상태 모델을 도입하여 관리자에게 정확한 배포 상태 정보를 제공하게 되었습니다. 또한 장기 미해결 이슈였던 Ollama startup script 외부화(#35)를 완료하여 graceful shutdown과 PVC 캐시 스킵 기능을 추가했습니다.

v0.4.0 워크스루 문서와 PDF(#176)로 파일럿 파트너향 외부 문서를 보강하고, 4건의 리서치 이슈(xxxxxxxxxxx, 폐쇄망, Pause/Resume, Scale-to-Zero)를 생성하여 v0.9 이후의 기술 방향을 구체화했습니다.

**핵심 성과**:
1. **서빙 런타임 이미지 관리**: 2-Layer Resolution + 관리자 UI — 버전 고정, 프라이빗 레지스트리, 재배포 70% 단축
2. **멀티클러스터 이미지 사전 캐싱**: xxxxxxx DaemonSet 기반 — 런타임별 분리, 클러스터별 선택적 캐싱, Reconciler 안전망
3. **배포 상태 동기화 개선**: Pod 진단 + 2-tier 상태 모델 — ImagePullBackOff/CrashLoopBackOff/OOMKilled/SchedulingFailed 감지
4. **Ollama 스크립트 외부화**: go:embed 기반 — graceful shutdown, PVC 캐시 스킵, 장기 이슈 클로즈
5. **v0.9 방향성 구체화**: 리서치 이슈 4건 — xxxxxxxxxxx 스케줄러, 폐쇄망, Pause/Resume, Scale-to-Zero

**다음 주 계획**:
- 서빙 배포 인플레이스 업데이트 API (PATCH) 구현 착수 (#125)
- 추론 API 프록시 구현 — 통합 엔드포인트 설계 (#36)
- xxxxxxxxxxx 스케줄러 리서치 진행 (#179)
- 서빙 배포 일시정지/재시작 기능 설계 (#185)

---

**문서 작성일**: 2026년 4월 12일  

