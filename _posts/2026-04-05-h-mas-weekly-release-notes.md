---
title: 'H-MAS release notes (2026-03-29~04-04)'
date: 2026-04-05
permalink: /posts/2026/04/h-mas-weekly-release-notes-0329-0404/
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
## 2026년 3월 29일 - 4월 5일 (Iteration 16)

### 주요 개발 내용 요약

이번 주는 **v0.4.0 Developer Preview 릴리즈 완료**, **JWT 기반 인증 시스템 전체 구현**, **Helm Chart + GHCR CI/CD 기반 설치 자동화**에 중점을 둔 기간이었습니다. 총 **27개의 커밋**, **14개의 PR 머지**, **15개의 이슈 생성**이 완료되었으며, v0.4 릴리즈의 P0 Blocker 4개를 모두 해결하고 성공 기준 5/5를 달성하여 첫 외부 릴리즈를 출시한 주간이었습니다.

{% include youtube.html id="uyeynYfqWMs" %}

---

## 새로운 기능 (New Features)

### 1. JWT 기반 인증 시스템 — v0.4 최소 보안 (#152 → PR #158)

**구현 완료**: v0.4 Developer Preview의 P0-4 요건으로, 파일럿 파트너 환경에 최소한의 접근 제어를 제공하기 위해 JWT 기반 인증 시스템을 Backend/Frontend/K8s 전 계층에 걸쳐 구현

**주요 성과**:

- **Backend**:
  - `internal/auth/` 패키지 신규: `AuthProvider` 인터페이스, `BuiltinProvider`(bcrypt), `AuthService`, JWT 유틸, Echo 미들웨어
  - Auth API 4개: `POST /api/auth/login`, `POST /api/auth/refresh`, `GET /api/auth/me`, `POST /api/auth/logout`
  - DB 마이그레이션 `000017_auth`: `users`, `refresh_tokens` 테이블, `serving_deployments.created_by` 컬럼
  - HttpOnly Cookie(`hmas_access`/`hmas_refresh`) 기반 토큰 전달, Refresh Token Rotation 적용
  - JWT 미들웨어로 기존 `/api/*` 라우트 보호 (공개 라우트 제외)
  - 기존 감사 로그 actor를 인증된 사용자명(`auth.GetUsername(c)`)으로 연동
  - 환경변수 기반 초기 admin 계정 시드

- **Frontend**:
  - `app/(authenticated)/` Route Group으로 기존 페이지 재구성 (URL 변경 없음)
  - Server Layout Guard — 서버 사이드 쿠키 검증 후 미인증 시 `/login` 리다이렉트
  - `proxy.ts` — Next.js 16.1 네트워크 레벨 인증 게이트
  - 로그인 페이지 (`/login`) — ID/PW 입력 폼
  - `useAuth()` SWR 훅 + auth API 클라이언트
  - 헤더/사이드바에 실제 사용자명 표시 + 로그아웃 동작 연결

- **K8s / 운영**:
  - Auth Secret(`hmas-auth`)을 git에서 제거, `make deploy-backend` 시 동적 생성
  - 개발: `password123!` 기본값, JWT Secret 자동 생성 / 운영: 환경변수 명시 설정

**의사결정 사항**:

| 결정 | 내용 |
|------|------|
| JWT 라이브러리 | `echo-jwt/v4` (미들웨어) + `golang-jwt/jwt/v5` (토큰 생성) 병용 — Echo v4 통합성 + 토큰 생성 유연성 |
| 토큰 전달 | HttpOnly Cookie — XSS 방어 + Next.js rewrites 프록시에서 자동 전달 + 프론트엔드 변경 최소화 |
| 역할 모델 | 3-Tier (`admin`/`member`/`viewer`) — v0.4는 DB 컬럼만, 인가 체크는 v0.9 |
| 초기 계정 | 환경변수 시드 + DB 저장 — v0.9 다중 사용자 확장 시 마이그레이션 비용 최소화 |

**관련 커밋**: `9dfe161` (PR #158, Closes #152)

---

### 2. 설치 자동화 — Helm Chart + GHCR CI + install.sh (#165 → PR #167)

**구현 완료**: v0.4 릴리즈의 P0-3 요건으로, 파일럿 파트너가 자체 K8s 클러스터에 H-MAS를 원커맨드로 설치할 수 있는 인프라 전체를 구축

**주요 성과**:

- **Helm Chart 전환**: `k8s/` raw 매니페스트를 `charts/h-mas/` Helm Chart로 마이그레이션. `values.yaml`(프로덕션) / `values-dev.yaml`(개발) 환경 분리, Secret 자동 생성(`lookup` + `randAlphaNum`)
- **GHCR CI/CD**: `release.yaml` — `v*` 태그 푸시 시 backend/frontend 멀티아키텍처 이미지 빌드+푸시. `cleanup-ghcr.yaml` — 주간 자동 정리
- **install.sh 래퍼**: xxxxxxx 설치 → Secret 생성 → Helm 배포 → Health check 원커맨드 자동화. `--registry-token` 옵션으로 Private GHCR 인증까지 자동 처리
- **imagePullSecrets 지원**: Helm Chart + install.sh에 Private GHCR PAT 기반 인증 통합
- **Makefile 리팩토링**: Helm 기반 `deploy`/`undeploy`/`render` 타겟, `xxxxxxx-setup` 멱등성 개선
- **문서**: `INSTALLATION_GUIDE.md` 신규(파일럿/운영 설치 가이드), `DEVELOPMENT_GUIDE.md`에 CI/CD 파이프라인 섹션 추가

**의사결정 사항**:

| 결정 | 내용 |
|------|------|
| PostgreSQL | 자체 StatefulSet 그대로 템플릿화 — v0.4 파일럿 수준에서 적절, Bitnami 서브차트는 과도 |
| xxxxxxx kubeconfig 주입 | install.sh/Makefile이 Secret 생성 → Helm은 `existingSecret` 참조 — 민감 정보 Helm Chart 미포함 |
| auth Secret | Helm `lookup` + `randAlphaNum` 자동 생성 — 첫 설치 시 자동, upgrade 시 기존 값 유지 |
| dev-setup-lite | 제거 — xxxxxxx 없이는 핵심 기능 동작 불가 |

**관련 커밋**: `ab0fb05` (PR #167, Closes #165)

---

### 3. 로그 페이지 재구성 — 실제 API 연동 (#145 → PR #147)

**구현 완료**: `/logs` 페이지를 mock 데이터 기반 프로토타입에서, 실제 API(`GET /api/deployments`, `GET /api/deployments/:id/events`, `GET /api/audit-logs`) 기반 배포 트러블슈팅 도구로 전환

**주요 성과**:
- 배포 셀렉터, 배포 이벤트 탭, 감사 로그 탭, 메트릭 Coming Soon 플레이스홀더로 재구성
- 최초 진입 시 가장 최근 배포 자동 선택으로 빈 화면 방지
- 이벤트 헤더에 마지막 갱신 시간 표시 및 15초 자동 polling
- K8s 이벤트 만료 안내 등 빈 상태 메시지 개선

**관련 커밋**: `bf95d67` (PR #147, Closes #145)

---

### 4. K8s Pod 로그 조회 API 및 로그 뷰어 연동 (#146 → PR #149)

**구현 완료**: 배포된 서빙 인스턴스의 실제 K8s Pod 로그를 조회하는 백엔드 API(`GET /api/deployments/:id/logs`)를 구현하고, LogViewer 컴포넌트에 연동

**주요 성과**:
- **Backend**: `GetPodLogs()` 메서드 추가 — `getMemberClient` → `Pods().GetLogs()` 패턴, K8s 타임스탬프 활성화
- **Frontend**: Pod 선택 드롭다운, tailLines 조절(50~1000), K8s 타임스탬프 파싱/로컬 시각 표시, 15초 자동 polling
- **문서**: `API_REFERENCE.md`에 로그 API 문서화, `FEATURE_SPEC.md` 상태 업데이트, `GLOBAL_TIMEZONE_DESIGN.md` 향후 설계 문서 추가

**관련 커밋**: `8eb4c1c` (PR #149, Closes #146)

---

## 버그 수정 (Bug Fixes)

### 1. Ollama 배포 시 HuggingFace ID 대신 Ollama 모델명 사용 (#148 → PR #151)

**수정 완료**: Ollama 런타임으로 프리셋 모델 배포 시 HuggingFace ID(`BAAI/bge-m3`)가 `ollama pull`에 그대로 전달되어 `CrashLoopBackOff`가 발생하던 버그 수정

**수정 내용**:
- `serving_configs` 테이블에 `ollama_model` 컬럼 추가 (마이그레이션 `000016_add_ollama_model`)
- 기존 Ollama 지원 프리셋 모델 10개에 대한 매핑값 backfill
- 배포 핸들러에서 `pull` 전략 + `OllamaModel`이 있으면 자동 치환
- 모델 등록/수정 폼에 Ollama 모델명 입력 필드 추가 (조건부 렌더링)

**관련 커밋**: `8f7dd4d` (PR #151, Closes #148)

### 2. Pod 로그에서 ANSI 이스케이프 시퀀스 strip (#150 → PR #154)

**수정 완료**: `ollama pull` 등 진행률 표시 프로세스의 ANSI 제어 시퀀스(`[K`, `[A`, `[?25l` 등)가 Pod 로그 UI에 raw 텍스트로 노출되는 문제 수정

**수정 내용**:
- `GetPodLogs` 반환 전 `stripANSI` 함수로 CSI/OSC 시퀀스 및 `\r` 문자 제거
- 11개 테스트 케이스 추가 (실제 ollama pull 출력 시뮬레이션 포함)

**관련 커밋**: `dd22bf5` (PR #154, Closes #150)

### 3. Access Token 만료 시 자동 갱신(refresh) 구현 (#159 → PR #162)

**수정 완료**: `refreshToken()` 함수가 정의되어 있으나 호출되지 않아, Access Token 만료 시 자동 갱신 없이 즉시 `/login`으로 리다이렉트되던 문제 수정

**수정 내용**:
- `client.ts`에 `fetchWithAuth` 래퍼 추가: 401 응답 시 자동으로 `POST /api/auth/refresh` → 원래 요청 재시도
- Promise mutex 패턴으로 동시 요청 간 refresh 중복 호출 방지 (N개 동시 401 → refresh 1건만 발생)
- `/api/auth/refresh` 경로 자체는 재시도 대상에서 제외하여 무한 루프 방지
- Vitest 도입 및 단위 테스트 6건 추가

**관련 커밋**: `0c48981` (PR #162, Closes #159)

### 4. raw fetch를 apiPut으로 교체하여 401 인증 리다이렉트 통일 (#160 → PR #163)

**수정 완료**: 일부 모듈에서 공통 `apiGet`/`apiPost` 대신 직접 `fetch`를 호출하여 `fetchWithAuth`(토큰 자동 갱신) + `handleResponse`(`/login` 리다이렉트) 파이프라인을 우회하던 문제 수정

**수정 내용**:
- `client.ts`에 `apiPut` 공통 함수 추가
- raw `fetch`를 사용하던 **4개 파일 9개 함수**를 `apiPut`으로 교체: `models.ts`, `resource-profiles.ts`, `placement-strategies.ts`, `failover-policies.ts`
- `apiPut` 401 처리 검증 단위 테스트 4건 추가

**관련 커밋**: `6bbd47c` (PR #163, Closes #160)

### 5. proxy.ts를 Next.js 16 공식 컨벤션에 맞게 정리 (#161 → PR #164)

**수정 완료**: `proxy.ts`가 dead code라는 이슈 보고를 검증한 결과 Next.js 16.1.6에서 정상 동작 확인됨. 공식 컨벤션에 맞게 export 방식 변경 및 중복 경로 정리

**수정 내용**:
- `export function proxy` → `export default function proxy` (Next.js 16 공식 예시 일치)
- `PUBLIC_PATHS`에서 중복 포함된 경로 제거, 변수명 `PUBLIC_PAGE_PATHS`로 명확화
- 설계 문서 디렉토리 구조 정정

**관련 커밋**: `c621651` (PR #164, Closes #161)

### 6. GHCR 이미지 태그 소문자 변환으로 빌드 실패 해결 (#168 → PR #169)

**수정 완료**: `github.repository_owner`가 대문자 포함 조직명(`ShrinkLabs`)을 반환하여 Docker 이미지 태그가 `invalid tag` 오류를 발생시키던 문제 수정

**수정 내용**:
- `env` 레벨 이미지 변수를 제거하고, step에서 `tr '[:upper:]' '[:lower:]'`로 소문자 변환 후 `GITHUB_ENV`에 설정

**관련 커밋**: `2d07fb9` (PR #169, Closes #168)

### 7. Docker 멀티아키텍처 빌드 최적화 — QEMU 에뮬레이션 제거 (#170 → PR #171)

**수정 완료**: Backend arm64 빌드가 QEMU 에뮬레이션으로 30분+ 소요되던 문제를 네이티브 크로스 컴파일로 전환하여 **~3-4분**으로 단축

**수정 내용**:
- Backend/Frontend Dockerfile의 builder 스테이지에 `--platform=$BUILDPLATFORM` 적용
- Backend: Go 크로스 컴파일(`GOARCH=$TARGETARCH`) 활용
- Frontend: deps/builder 스테이지 네이티브 실행, runner 스테이지는 타겟 플랫폼 유지

**관련 커밋**: `fca3329` (PR #171, Closes #170)

### 8. 헤더의 미사용 글로벌 검색창 제거 (#155 → PR #156)

**수정 완료**: 검색 로직이 연결되지 않은 글로벌 검색 UI(Search 아이콘 + Input)를 제거하고 미사용 import 정리

**관련 커밋**: `3500cbb` (PR #156, Closes #155)

---

## 문서화 (Documentation)

### 1. 인증/인가 기술 리서치 문서 (#153 → PR #157)

| 문서 | 내용 | 관련 커밋 |
|------|------|-----------|
| `docs/research/auth/README.md` | 인덱스 — 문서 구조, 버전별 로드맵, 핵심 설계 원칙 | `db76c61` (PR #157) |
| `docs/research/auth/V04_AUTH_DESIGN.md` | v0.4 구현 결정 — JWT, HttpOnly Cookie, proxy.ts, 3-Tier 역할, 감사 추적, DB 스키마, API 설계 | `db76c61` (PR #157) |
| `docs/research/auth/PROJECT_MULTITENANCY.md` | v0.9 리서치 — 프로젝트 기반 멀티테넌시, 1 Project = 1 Namespace, 프로젝트 역할별 권한 매트릭스 | `db76c61` (PR #157) |
| `docs/research/auth/EXTERNAL_IDP_INTEGRATION.md` | v0.9+ 리서치 — OIDC/SAML/LDAP 비교, Auth Provider 추상화 패턴, JIT Provisioning | `db76c61` (PR #157) |

### 2. v0.4.0 릴리즈 문서 (#172 → PR #173)

| 문서 | 내용 | 관련 커밋 |
|------|------|-----------|
| `CHANGELOG.md` | 신규 작성 — Keep a Changelog 포맷, 기능 단위 요약 + 주요 이슈 링크 | `65ab3b0` (PR #173) |
| `README.md` | v0.4 DP 상태 "진행 중" → "완료" 전환, 성공 항목 체크 | `65ab3b0` (PR #173) |
| `ROADMAP.md` | v0.4 항목 전체 ✅ 완료, 성공 기준 5/5 달성 반영 | `65ab3b0` (PR #173) |
| `V04_RELEASE_CHECKLIST.md` | P0/P1 완료 처리, Iteration 16 실적 반영 | `218eff4`, `65ab3b0` |

### 3. 비즈니스 문서 (가격정책, 파트너 전략)

| 문서 | 내용 | 관련 커밋 |
|------|------|-----------|
| 가격정책 문서 | 구축형/구독형 가격 모델 설계 | `a4674cc` |
| 파트너 판매 모델 | 파트너 수익 구조 추가 | `bdd8dd6` |
| 파트너 전략 분리 | SI 파트너/리셀러/VAR 역할 분리 및 상세화 | `7fc4cea`, `4a40221`, `7481200` |
| 제품 소개서 | 실제 구현 상태 반영 및 내용 보강 | `e523098` |

### 4. 릴리즈 검증 기록

| 문서 | 내용 | 관련 커밋 |
|------|------|-----------|
| `V04_RELEASE_CHECKLIST.md` | P0-1/P0-2 E2E 검증 진행 상황, P0-4 인증 완료 반영 | `295a449`, `d0c7e32` |
| `V04_RELEASE_CHECKLIST.md` | P0-1 멀티 클러스터 E2E 완료 — 배포 분산 확인 | `441d346` |
| `V04_RELEASE_CHECKLIST.md` | P0-2 Standard 서빙 E2E 완료 — 배포 삭제/정리 확인 | `2e3a4cc` |

### 5. v0.4.0 출시 후 문서 (4월 5일)

| 문서 | 내용 | 관련 커밋 |
|------|------|-----------|
| 출시 문서 최신화 | v0.4.0 출시 반영 전체 문서 최신화 | `05daaf1` |
| 데모 영상 시나리오 | v0.4.0 데모 시나리오 추가 | `05daaf1` |

---

## 이전 Iteration 계획 달성도

Iteration 15에서 계획한 3개 항목 **0개 달성, 3개 미착수** — 그러나 v0.4.0 릴리즈라는 더 큰 목표를 달성:

| 계획 | 이슈 | 상태 | 비고 |
|------|------|------|------|
| 서빙 배포 인플레이스 업데이트 API (PATCH) 구현 착수 | #125 | ⏸ 미착수 | v0.4.0 릴리즈 P0 blocker 해결에 우선순위 전환 |
| 추론 API 프록시 구현 — 통합 엔드포인트 설계 | #36 | ⏸ 미착수 | JWT 인증 + 설치 자동화에 집중 |
| 대시보드 v0.9 메트릭 연동 사전 조사 | #133, #134 | ⏸ 미착수 | v0.4 릴리즈 후 v0.9 범위로 이관 |

**추가 달성**: 계획에 없던 v0.4.0 릴리즈 전체 — JWT 인증 시스템(#152), 인증 리서치(#153), 설치 자동화(#165), 로그 시스템 실데이터 연동(#145, #146), Ollama 모델명 버그 수정(#148), ANSI strip(#150), 인증 관련 후속 수정 3건(#159, #160, #161), CI/CD 버그 수정 2건(#168, #170), 릴리즈 문서(#172), 비즈니스 문서 다수. v0.4.0 Developer Preview를 성공적으로 출시하여 파일럿 파트너에게 첫 외부 릴리즈를 제공하게 됨.

---

## v0.4.0 Developer Preview 릴리즈 요약

이번 Iteration에서 v0.4.0 릴리즈의 **P0 Blocker 4개를 모두 해결**하고 **성공 기준 5/5를 달성**하여 릴리즈를 완료했습니다.

### P0 Blocker 달성 현황

| P0 항목 | 상태 | 주요 작업 |
|---------|------|-----------|
| P0-1: 멀티 클러스터 실 환경 E2E | ✅ 완료 | 배포 분산 확인, 성공 기준 달성 |
| P0-2: Standard 서빙 실 환경 검증 | ✅ 완료 | 배포 삭제/정리 확인, 성공 기준 달성 |
| P0-3: 설치 자동화 | ✅ 완료 | Helm Chart + GHCR CI + install.sh (PR #167) |
| P0-4: 기본 인증 (JWT) | ✅ 완료 | 인증 시스템 전체 구현 (PR #158) |

### 릴리즈 CI/CD 파이프라인

릴리즈 과정에서 GHCR 이미지 빌드 파이프라인의 2건의 문제를 발견하고 즉시 수정:
1. 조직명 대문자로 인한 `invalid tag` 오류 → 소문자 변환 적용 (#168 → PR #169)
2. arm64 빌드 QEMU 에뮬레이션 30분+ → 네이티브 크로스 컴파일로 ~3-4분 단축 (#170 → PR #171)

---

## 미해결 이슈 (Open Issues)

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

### v0.9 운영/인프라 이슈 (1개, 신규)

| 이슈 | 제목 |
|------|------|
| #166 | v0.9: 브랜치 전략 및 릴리즈/배포 프로세스 정립 |

### 기존 미해결 이슈 (이전 Iteration에서 이관)

### 1. 스케줄러 어노테이션 변환 (xxx/xxxxxxxxxxx → xxxxxxx OverridePolicy) (#66)

- Iteration 13 이후 계속 미착수. 런타임 최적화 시스템 완성 후 검토 예정

### 2. 모델 프리셋 리스트 최신화 (#62)

- Model Registry 구현으로 정적 프리셋 관리의 필요성 크게 감소
- 잔여 작업: 최신 모델 시드 데이터 추가 필요

### 3. Failover / 재배치 정책 구현 (#39) — v0.95 마일스톤

### 4. NUMA / 토폴로지 인식 스케줄링 구현 (#38) — v0.95 마일스톤

### 5. 추론 API 프록시 구현 — 통합 엔드포인트 (#36)

### 6. H-MAS 래퍼 컨테이너 이미지 도입 검토 (#35)

### 7. 리서치: GPU 메모리 인지형 동적 모델 Bin Packing 전략 (#96)

---

## 이번 주 통계

| 항목 | 수치 |
|------|------|
| 총 커밋 수 | 27개 (+ merge 1건) |
| 머지된 PR | 14개 (#147, #149, #151, #154, #156, #157, #158, #162, #163, #164, #167, #169, #171, #173) |
| 생성된 이슈 | 15개 (#145, #146, #148, #150, #152, #153, #155, #159, #160, #161, #165, #166, #168, #170, #172) |
| 해결된 이슈 | 14개 (#145, #146, #148, #150, #152, #153, #155, #159, #160, #161, #165, #168, #170, #172) |
| 미해결 이슈 | 18개 (#35, #36, #38, #39, #62, #66, #96, #114, #120, #122, #123, #125~#128, #133, #134, #166) |
| 신규 기술 문서 | 4개 (인증 리서치 3건, 글로벌 타임존 설계) |
| 비즈니스 문서 | 5개 (가격정책, 파트너 전략, 제품 소개서) |
| 코드 변경량 | +9,010줄 / -3,397줄 (116개 파일) |

---

## 결론

이번 주는 **v0.4.0 Developer Preview를 성공적으로 릴리즈하여 파일럿 파트너에게 제공 가능한 첫 외부 릴리즈를 완성한 기간**이었습니다. P0 Blocker 4개(멀티 클러스터 E2E, Standard 서빙 검증, 설치 자동화, 기본 인증)를 모두 해결하고 성공 기준 5/5를 달성했습니다.

특히 JWT 기반 인증 시스템을 리서치(#153) → 설계 문서 → 구현(#152) → 후속 수정(#159, #160, #161)까지 일주일 내에 완성했으며, Helm Chart + GHCR CI/CD + install.sh 기반 설치 자동화(#165)로 파일럿 파트너가 자체 환경에 원커맨드로 설치할 수 있는 인프라를 구축했습니다. 로그 시스템도 mock 데이터에서 실제 API 연동으로 전환하여 배포 트러블슈팅 기능을 완비했습니다.

릴리즈 과정에서 CI/CD 파이프라인의 이미지 태그 대소문자 문제와 arm64 빌드 성능 문제를 발견하고 즉시 수정하여, 안정적인 멀티아키텍처 이미지 빌드 파이프라인을 확보했습니다.

**핵심 성과**:
1. **v0.4.0 Developer Preview 릴리즈**: P0 4개 + 성공 기준 5/5 달성 — 파일럿 파트너 제공 가능 상태
2. **JWT 인증 시스템 전체 구현**: Backend(`internal/auth/` 패키지) + Frontend(`proxy.ts`, Layout Guard, `useAuth`) + 자동 갱신/리다이렉트 통일 — 4개 PR로 완성
3. **설치 자동화**: Helm Chart + GHCR CI + install.sh — 원커맨드 설치, 환경 분리, Secret 자동 생성
4. **로그 시스템 실데이터 연동**: 로그 페이지 재구성 + K8s Pod 로그 API — 배포 트러블슈팅 완비
5. **CI/CD 파이프라인**: GHCR 멀티아키텍처 빌드 + arm64 30분→4분 최적화

**다음 주 계획**:
- v0.9 Phase 계획 수립 및 우선순위 정리
- 서빙 배포 인플레이스 업데이트 API (PATCH) 구현 착수 (#125)
- 추론 API 프록시 구현 — 통합 엔드포인트 설계 (#36)
- 브랜치 전략 및 릴리즈 프로세스 정립 (#166)

---

**문서 작성일**: 2026년 4월 5일  

