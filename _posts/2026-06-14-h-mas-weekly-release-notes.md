---
title: 'H-MAS release notes (2026-06-07~06-13)'
date: 2026-06-14
permalink: /posts/2026/06/h-mas-weekly-release-notes-0607-0613
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
## 2026년 6월 7일 - 6월 13일 (Iteration 26)

### 주요 개발 내용 요약

이번 주는 **멀티테넌시 보안 강화(Cross-Org 접근 차단·Org/Group 정합성 수정)**, **CI/CD 보안 파이프라인 구축(PR 체크·SAST·시크릿 스캔·이미지 스캔·컨테이너 린트)**, **보안 핫픽스(의존성 취약점 즉시 대응)**에 중점을 둔 기간이었습니다. 총 **23개의 커밋**, **20개의 PR 머지**, **26개의 이슈 생성**, **23개의 이슈 해결**이 완료되었으며, 특히 **멀티테넌시 인가 체계의 보안 취약점 5건을 전면 수정**하고 **심층 방어(Defense in Depth) 보안 파이프라인 6종을 1주 만에 구축 완료**하여 v0.7 출시 전 보안 기반을 확립했습니다.

---

## 새로운 기능 (New Features)

### 1. 클러스터 조직 할당 변경/회수 (#326, #327 → PR #328, #329)

**구현 완료**: 클러스터 등록 이후에도 소속 조직을 변경·회수할 수 있는 API 및 Frontend UI 전체 구현

**주요 성과**:
- **Backend**:
  - `PATCH /api/clusters/:name`에 `organizationId` 필드 추가 — 조직 할당(`값 지정`), 회수(`null`), 유지(`필드 미포함`) 3-state 구분
  - 실행 중 배포가 있는 클러스터는 `?force=true` 없이 조직 변경 시 `409 Conflict` 반환으로 안전장치 제공
  - 유닛 테스트 17개 + E2E 테스트 8개 완료
- **Frontend**:
  - 클러스터 상세 Sheet: 소속 조직 표시, System Admin 전용 "변경" 버튼 → 조직 변경 Dialog
  - 클러스터 카드: 소속 조직 뱃지 표시 (미할당 시 경고색 `amber`)
  - System Admin 전용 조직별 필터 Select 추가

**의사결정 사항**:

| 결정 | 내용 |
|------|------|
| 3-state 파싱 | `json.RawMessage` 기반으로 null/미포함/값 구분 — 부분 업데이트 API의 의미론적 일관성 확보 |
| 배포 안전장치 | 활성 배포 있는 클러스터는 force 파라미터 없으면 차단 — 운영 중 격리 경계 변경 방지 |

**관련 커밋**: `42a566c`, `29fb808` (PR #328, #329, Closes #326, #327)

---

### 2. 배포 폼 클러스터 가용량 초과 경고 (#330 → PR #331)

**구현 완료**: 배포 폼에서 CPU/메모리/GPU 입력값이 선택한 클러스터의 가용 리소스를 초과할 경우 실시간 Soft Warning 표시

**주요 성과**:
- **Frontend**:
  - 배치 모드별(직접 선택·타입별·정책 기반) 최대 가용 리소스 동적 계산
  - 가용 GPU 수, 총 CPU, 총 메모리 힌트 표시로 사용자 입력 가이드 제공
  - GPU `max` 하드코딩 제거 → 클러스터 가용량 기반 동적 설정
  - 경고 시에도 배포 버튼 활성화 유지 (Soft Warning 정책)

**관련 커밋**: `e133716` (PR #331, Closes #330)

---

### 3. 시스템 관리자 추론 로그 조직 단위 스코핑 (#336 → PR #337)

**구현 완료**: 시스템 관리자가 사이드바에서 조직을 전환하면 해당 조직의 추론 로그만 표시되도록 스코핑 변경 (기존: 사이드바 컨텍스트 무관 전체 조회)

**주요 성과**:
- **Backend**:
  - 시스템 관리자: 전체 조회 → 선택된 조직 내 모든 그룹 배포 로그로 스코핑
  - 일반 사용자: 기존 그룹 스코핑 동작 유지
  - 삭제된 배포의 감사 로그 보존 (`deleted_at IS NULL` 조건 제거)

**관련 커밋**: `645997d` (PR #337, Closes #336)

---

### 4. Organization 멤버 추가 시 Group 동시 가입 (#339 → PR #341)

**구현 완료**: 조직에 멤버를 추가할 때 그룹을 지정하여 한 번에 가입 처리 — 기존 Org 멤버십만 생성하던 방식을 Org + Group 멤버십 원자적 동시 생성으로 개선

**주요 성과**:
- **Backend**:
  - `AddOrgMemberWithGroup`: Org/Group 멤버십을 단일 트랜잭션에서 원자적으로 생성 (기존 `RemoveOrgMemberWithGroups`와 대칭 구조)
  - 그룹 미지정 시 Default Group 기본값 적용으로 하위 호환 유지
- **Frontend**:
  - 멤버 추가 다이얼로그에 그룹·그룹 역할 셀렉터 추가
  - 조직 역할 변경 시 그룹 역할 자동 연동, 수동 변경 시 분리

**관련 커밋**: `ece7e43` (PR #341, Closes #339)

---

### 5. CI 보안 파이프라인: PR 체크 워크플로우 (#342 → PR #343)

**구현 완료**: PR 생성 시 Backend(Go)/Frontend(Next.js) 코드 품질을 자동 검증하는 `.github/workflows/pr-check.yaml` 워크플로우 신설

**주요 성과**:
- **CI**:
  - Backend: 빌드·Vet·테스트 3단계 병렬 실행, Go 버전 소스(`go.mod`) 동기화
  - Frontend: 빌드 자동 검증
  - 같은 PR에 새 커밋 푸시 시 이전 실행 자동 취소(concurrency 설정)

**관련 커밋**: `dc9b90c` (PR #343, Closes #342)

---

### 6. 심층 방어(Defense in Depth) 보안 파이프라인 구축 (#349~#361)

**구현 완료**: 6종의 보안 스캔 도구를 CI에 순차 도입하여 H-MAS의 공급망 보안·코드 보안·컨테이너 보안을 전 계층에서 자동화

#### 6-1. Semgrep OSS SAST (#349 → PR #357)

**완료**: Go + TypeScript/JavaScript SAST 정적 분석 워크플로우 도입 — CodeQL 대신 Private + GHAS 미활성 환경에 최적화

- OWASP Top 10·보안 감사·시크릿 탐지 6개 룰셋 적용
- PR·push·주간 schedule 트리거, 초기 non-blocking → 트리아지 후 ERROR 승격 예정
- 결과: Job Summary (severity별 카운트) + 워크플로우 아티팩트 30일 보관
- 외부 액션 전체 SHA 핀(#347 정책 준수)

**관련 커밋**: `df5e473` (PR #357, Closes #349)

---

#### 6-2. 의존성 취약점 스캔 CI 추가 (#350 → #356에 통합)

**완료**: 백엔드 `govulncheck`, 프론트엔드 `npm audit` 기반 의존성 취약점 스캔을 PR 단계에서 지속 감시 체계 확립

**관련 PR**: #356 (Closes #350, #348)

---

#### 6-3. gosec 보안 린터 도입 (#358 → PR #362)

**완료**: Go 코드 보안 결함 정적 탐지 도구를 기존 `golangci-lint` 파이프라인에 통합 — 초기 발견 14건 트리아지 완료

- 기존 lint 잡 안에서 함께 실행되어 추가 워크플로우 불필요
- 발견 사항 14건 전량 false positive 또는 실용적 안전 케이스로 판단, 인라인 억제 처리
- `clearCookies`에 `SameSite` 속성 보강(set 시점과 일관성 확보) 부가 개선 포함

**관련 커밋**: `5cf52d4` (PR #362, Closes #358)

---

#### 6-4. gitleaks 시크릿 스캔 도입 (#359 → PR #363)

**완료**: CI + pre-commit 양쪽에 시크릿 스캔 도입 — PR diff 신규 누설 즉시 차단, 기존 히스토리 non-blocking 가시화

- PR diff 스캔(blocking) / push·주간 schedule·dispatch 전체 히스토리 스캔(non-blocking) 이중 정책
- 공식 릴리스 바이너리 + sha256 체크섬 검증 방식으로 설치 (외부 액션 상용 라이선스 모순 회피)
- `.gitleaks.toml`: 내장 룰셋 확장 + 테스트 픽스처·잠금 파일·자동 생성물 allowlist
- pre-commit 훅으로 커밋 전 차단까지 지원

**관련 커밋**: `370d29b` (PR #363, Closes #359)

---

#### 6-5. Trivy 이미지 스캔 + IaC 스캔 도입 (#360 → PR #364)

**완료**: 컨테이너 이미지 취약점 스캔(release 파이프라인 통합)과 Dockerfile/Helm 차트 미스컨피그 스캔(별도 워크플로우) 동시 도입

**의사결정 사항**:

| 결정 | 내용 |
|------|------|
| 이미지 스캔 통합 방식 | amd64 이미지 로컬 로드 → Trivy 스캔 → 통과 시에만 멀티 아키 GHCR 푸시 (취약 산출물 레지스트리 유입 사전 차단) |
| Helm 스캔 방식 | `helm template`으로 prod/dev 두 프로파일 렌더 후 실제 배포 형태 기준 점검 |
| 초기 차단 정책 | CRITICAL만 차단, HIGH 이하 보고만 → 트리아지 1회전 후 HIGH 승격 예정(#366) |

**관련 커밋**: `b442f4f` (PR #364, Closes #360)

---

#### 6-6. hadolint Dockerfile 린트 도입 (#361 → PR #365)

**완료**: backend/frontend Dockerfile 정적 린트 도입 — Trivy IaC 스캔의 보안 미스컨피그 검사를 보완하는 셸 품질·운영 베스트 프랙티스 계층 추가

- PR + push:main + 주간 schedule 트리거, 4단 구조(보고→차단→요약→아티팩트)
- 초기 error 등급만 차단 → 트리아지 후 warning 승격 예정(#367)
- 로컬 검증: config 적용 후 발견 사항 0건

**관련 커밋**: `df23bb7` (PR #365, Closes #361)

---

## 버그 수정 (Bug Fixes)

### 1. 배포 시 다른 조직 클러스터에 배포되는 멀티테넌시 인가 누락 (#332 → PR #333)

**수정 완료**: `CreateDeployment` 핸들러에 조직 기반 클러스터 인가 검증 추가 — 사용자가 다른 조직의 클러스터에 배포 불가하도록 수정, xxxxxxx PropagationPolicy에 조직 클러스터 화이트리스트 주입으로 스케줄링 범위 제한

**수정 내용**:
- 조직-클러스터 인가 검증: 조직 미할당·조직 외 클러스터 지정 시 403 반환
- Pre-flight Validation: 조직 클러스터 중 요청 조건(GPU 등) 만족 클러스터 없으면 400 즉시 반환
- 단위 테스트 7개 + 조직 경계 배포 E2E 테스트 5개 신규 추가

**해결된 기술 문제**:

| 문제 | 원인 | 해결 |
|------|------|------|
| 다른 조직 클러스터에 배포 가능 | 배포 핸들러의 조직-클러스터 인가 검증 부재 | H-MAS 독자적인 조직 클러스터 화이트리스트 검증 도입 및 xxxxxxx PP 스코핑 적용 |

**관련 커밋**: `c014a28` (PR #333, Closes #332)

---

### 2. 배포 API Cross-Org 리소스 접근 보안 취약점 (#334 → PR #335)

**수정 완료**: 시스템 관리자의 `groupID=0` 바이패스로 전체 배포 노출, 단건 API 13개에서 조직/그룹 경계 검증 없이 다른 조직 리소스 접근 가능했던 보안 취약점 일괄 수정

**수정 내용**:
- `ListDeployments`: 시스템 관리자의 전체 배포 노출 → 선택된 조직 범위로 제한
- 단건 API 13개: `GetByID` 단독 조회 → 조직/그룹 경계 검증 헬퍼(`resolveAndValidateDeployment`) 적용
- `WithOrgScopeViaGroup` GORM 스코프 함수 추가로 재사용 가능한 조직 필터링 확보

**관련 커밋**: `da8fc23` (PR #335, Closes #334)

---

### 3. Organization 생성 시 Default Group 자동 생성 누락 (#338 → PR #340)

**수정 완료**: `CreateOrganization()`이 Org 레코드만 생성하고 Default Group을 생성하지 않아 신규 Org의 배포·모델·추론 기능이 즉시 403으로 차단되던 버그 수정

**수정 내용**:
- Org + Default Group을 단일 트랜잭션으로 원자적 생성
- DB 마이그레이션 `000040`: 기존 Org 중 Default Group 누락 건 일괄 백필

**관련 커밋**: `e38e7e3` (PR #340, Closes #338)

---

### 4. 대시보드 최근 활동 위젯 비관리자 403 에러 (#324 → PR #325)

**수정 완료**: 시스템 관리자 전용 감사 로그 API를 비관리자도 호출하여 대시보드에서 항상 403이 표시되던 문제 수정 — 비관리자는 위젯 숨김 + 그리드 2열 재배치

**관련 커밋**: `12c10db` (PR #325, Closes #324)

---

### 5. 보안 핫픽스: 알려진 취약점 의존성 즉시 업그레이드 (#348 → PR #356)

**수정 완료**: 백엔드 `golang.org/x/net` v0.48.0 → v0.56.0, 프론트엔드 Next.js 16.1.6 → 16.2.9, postcss XSS 잔존 취약점 `overrides`로 강제 패치

**수정 내용**:
- 백엔드: `govulncheck` 기준 x/net 귀속 reachable 취약점 0건 달성
- 프론트엔드: `npm audit` 0건 달성 (high/moderate 전량 해소)
- xxxxxxx `GO-2024-2817`: 상위 패치 미존재로 추적 메모 작성, 영향 범위 제한 확인

**관련 커밋**: `76865a8` (PR #356, Closes #348, #350)

---

### 6. 백엔드 golangci-lint 지적 해소 및 PR 차단 승격 (#346 → PR #354)

**수정 완료**: golangci-lint 지적 16건(errcheck 6·staticcheck 8·ineffassign 1·unused 1) 해소 및 `.golangci.yml` 신설, PR 체크 차단 게이트로 승격

**관련 커밋**: `3affe02` (PR #354, Closes #346)

---

### 7. 프론트엔드 ESLint 오류 18건 해소 및 PR 차단 승격 (#344 → PR #352)

**수정 완료**: `react-hooks/set-state-in-effect`, `static-components`, `immutability` 등 ESLint 오류 18건 해소 및 PR Lint 체크 차단 게이트로 승격

**수정 내용**:
- 모달/Sheet 패턴 14건: `{open && <Inner key=... />}` 패턴으로 상태 초기화 자연화
- 타이머 패턴 2건: 렌더 중 직접 호출 → state 추적 방식으로 리팩터링
- 파생 상태 2건: effect 없이 렌더 중 계산으로 변경

**관련 커밋**: `d632f55` (PR #352, Closes #344)

---

### 8. apiPut 단위 테스트 갱신 및 PR Test 차단 승격 (#345 → PR #353)

**수정 완료**: `fetchWithAuth` 타임아웃 처리를 위한 `AbortController` 도입 후 `apiPut` 단위 테스트 2건의 기대값 미갱신 문제 수정 및 PR Test 체크 차단 게이트 승격

**관련 커밋**: `0c75b75` (PR #353, Closes #345)

---

## 리팩토링 (Refactoring)

### 1. 클러스터 상세보기 Dialog → Sheet(Drawer) 전환 (#322 → PR #323)

**완료**: 클러스터 상세보기 UI를 중앙 모달에서 우측 슬라이드 패널(Sheet)로 변환 — 프로젝트 UI 컨벤션 일관성 확보

**관련 커밋**: `9b497d8` (PR #323, Closes #322)

---

## 기타 작업 (Chores)

- GitHub Actions SHA 핀 자동 갱신을 위한 Dependabot 설정 추가 — SHA 핀 트레이드오프 해소 (#347, Closes #347)
- 심층 방어 보안 도구 도입 로드맵 및 6종 도구 선정 문서화 (#351)
- H-MAS 프로젝트 가이드 문서 추가 (`c3dc00c`)

---

## 결론

이번 주는 **멀티테넌시 인가 보안 취약점을 전면 수정하고 심층 방어(Defense in Depth) 보안 파이프라인 6종을 1주 만에 구축 완료한 기간**이었습니다. 멀티테넌시 인가 취약점(#332, #334, #338, #339) 수정으로 Cross-Org 리소스 접근 차단·Default Group 정합성·멤버 가입 원자성을 확보했으며, PR 체크 워크플로우(#342) → 코드 품질 정비(#344~#346) → SAST(#349) → 의존성 핫픽스(#348) → 심층 방어 6종(#358~#361) 순서로 보안 기반을 계층적으로 구축했습니다. 클러스터 조직 관리 기능(#326, #327)과 배포 폼 UX 개선(#330)도 함께 완료하여 멀티테넌시 운영 완성도를 높였습니다.

**핵심 성과**:
1. **멀티테넌시 인가 보안 강화**: Cross-Org 배포·리소스 접근 차단 (#332, #334), Default Group 정합성 확보(#338), Org 멤버 가입 원자화(#339) — 멀티테넌시 경계 보안 전면 정비
2. **심층 방어 보안 파이프라인 6종 구축**: PR 체크(#342) + Semgrep SAST(#349) + govulncheck/npm audit(#350) + gosec(#358) + gitleaks(#359) + Trivy(#360) + hadolint(#361) — v0.7 출시 전 전 계층 자동 보안 검증 확보
3. **보안 핫픽스 즉시 대응**: x/net, Next.js, postcss 알려진 취약점 0건 달성(#348)

**다음 주 계획**:
- Trivy HIGH 차단 승격 및 hadolint warning 승격 트리아지 (#366, #367)
- QoS-only 요청 OverridePolicy 오버라이더 누락 수정 (#355)
- xxxxxxx 고아 Work 자동 정리 구현 (#316)
- 추론 프록시 Phase 2: 멀티클러스터 라우팅 (#192)

---

**문서 작성일**: 2026년 6월 14일

