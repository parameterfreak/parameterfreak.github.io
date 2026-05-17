---
title: 'H-MAS release notes (2026-05-10~05-16)'
date: 2026-05-17
permalink: /posts/2026/05/h-mas-weekly-release-notes-0510-0516
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
## 2026년 5월 10일 - 5월 16일 (Iteration 22)

### 주요 개발 내용 요약

이번 주는 **추론 API Key 인증 시스템 구현(#254)**, **추론 API Rate Limiting 구현(#255)**, **이미지 워머 OOM 버그 수정(#260, #262)**에 중점을 둔 기간이었습니다. 총 **10개의 커밋**, **5개의 PR 머지**, **12개의 이슈 생성**이 완료되었으며, 추론 프록시의 인증/인가 및 사용량 제한 체계를 완성하여 상위 이슈 #194(추론 API 키 인증 및 Rate Limiting)를 종료하고, 인증/인가 고도화를 위한 후속 이슈 로드맵(사용자 관리, RBAC, Redis 전환 등)을 수립한 주간이었습니다.

{% include youtube.html id="HI6WF86uMHY" autoplay=true %}

---

## 새로운 기능 (New Features)

### 1. 추론 API Key 인증 시스템 (#254 → PR #256)

**구현 완료**: JWT 외에 프로그래매틱 접근(curl, OpenAI SDK 등)을 위한 API Key 인증 시스템 추가 — `sk-hmas-` 접두사 기반 키 발급, 통합 인증 미들웨어(JWT + API Key), 배포 범위 제한, 만료 관리

**주요 성과**:

- **Backend**:
  - `api_keys` 테이블 + 마이그레이션(000025, 000026) — SHA-256 해싱, 소유자/만료/스코프/활성 상태 관리
  - 통합 인증 미들웨어 리팩터링 — `Authorization: Bearer sk-hmas-xxx`(API Key) / `Bearer eyJ...`(JWT) 자동 분기
  - API Key CRUD 엔드포인트(`POST/GET/DELETE /api/api-keys`)
  - 추론 프록시에 배포 범위 제한(`allowed_deployments`) 적용
  - 추론 로그에 인증 방식(`auth_method: api_key/jwt`)과 키 ID 기록, `lastUsedAt` 자동 갱신

- **Frontend**:
  - `/api-keys` 페이지: 키 목록(이름, 생성일, 만료, 마지막 사용일), 생성 다이얼로그(만료일/스코프 설정), 키 원문 1회 복사 UI, 폐기 확인
  - 사이드바에 "API 키" 메뉴 추가

**의사결정 사항**:

| 결정 | 내용 |
|------|------|
| 키 접두사 | `sk-hmas-` — OpenAI 호환 형식, 사용자가 인지하기 쉬운 네이밍 |
| 인증 분기 | Bearer 토큰 패턴 자동 감지 — 기존 JWT 클라이언트 무영향 |
| 키 저장 | SHA-256 해싱 — 원문은 생성 시 1회만 반환, DB 유출 시에도 키 복원 불가 |

**검증 완료**: API Key 인증으로 추론 프록시 비스트리밍/SSE 스트리밍 요청, OpenAI SDK 호환성, 만료/폐기 키 거부, 추론 로그 기록 확인

**관련 커밋**: `c6b9f96` (PR #256, Closes #254)

---

### 2. 추론 API Rate Limiting (#255 → PR #265)

**구현 완료**: 추론 프록시에 사용자/API 키별 RPM(분당 요청 수)/TPM(분당 토큰 수) 제한 적용 — 429 응답 + 표준 헤더, 환경변수 글로벌 기본값 + API 키별 커스텀 한도 오버라이드

**주요 성과**:

- **Backend**:
  - RPM: Token Bucket 알고리즘 기반 실시간 요청 제한
  - TPM: Sliding Window 기반 사후 토큰 사용량 추적
  - 429 Too Many Requests 응답 + 표준 헤더(`X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`, `Retry-After`)
  - 환경변수 글로벌 기본값(`HMAS_RATE_LIMIT_RPM=60`, `TPM=0`) + API 키별 커스텀 오버라이드
  - `GET /api/rate-limits` — 현재 사용량/한도/글로벌 기본값 조회 API
  - `hmas_inference_rate_limited_total` Prometheus 메트릭

- **Frontend**:
  - API 키 생성 다이얼로그에 RPM/TPM 입력 필드 + 기본값 placeholder
  - API 키 목록에 Rate Limit 컬럼 — 기본값 수치 + "(기본)" 표시

**의사결정 사항**:

| 결정 | 내용 |
|------|------|
| RPM 알고리즘 | Token Bucket — 버스트 허용 + 평균 속도 제한, API 게이트웨이 업계 표준 |
| TPM 방식 | 사후 추적(Sliding Window) — 프록시 레이턴시 무영향, 프롬프트 토큰 사전 추정은 #270으로 분리 |
| 저장소 | In-Memory — v0.9 단일 인스턴스 환경에 적합, Redis 전환은 #264로 분리 |

**관련 커밋**: `10f03b9` (PR #265, Closes #255)

---

## 버그 수정 (Bug Fixes)

### 1. 이미지 워머 OOM 해결 (#260 → PR #261)

**수정 완료**: 이미지 워머 DaemonSet Pod이 CrashLoopBackOff(OOM-killed)에 빠지는 문제 해결 — 메모리 limit `4Mi` → `32Mi`, CPU limit `1m` → `10m` 상향

**수정 내용**:
- 서빙 런타임 이미지(vLLM 등)의 base layer(Ubuntu 24.04)가 init 시 4Mi 이상 메모리를 요구하여 OOM 발생
- `sleep infinity` 실행 상태에서 실사용량은 1~2Mi이므로 노드 리소스 부담 없음

**관련 커밋**: `870ce1b` (PR #261, Closes #260)

---

### 2. 이미지 워머 메모리 request/limit 분리 (#262 → PR #263)

**수정 완료**: #260 수정 시 request도 `32Mi`로 설정되어 발생한 불필요한 노드 메모리 예약 문제 해결 — request `32Mi` → `4Mi`로 분리

**수정 내용**:
- Kubernetes 스케줄러가 request 기준(32Mi)으로 노드 리소스를 예약하여 실사용량 대비 16~32배 과다 예약 발생
- request `4Mi`(실사용 + 여유) / limit `32Mi`(init OOM 방지)로 분리
- Reconciler 60초 주기 갱신으로 기존 DaemonSet에 자동 반영

**관련 커밋**: `56580b5` (PR #263, Closes #262)

---

## 문서화 (Documentation)

### 기술 문서 업데이트 (7건)

| 문서 | 내용 | 관련 커밋 |
|------|------|-----------|
| Control Plane 재설치 후 멤버 클러스터 정리 가이드 | 고아 리소스 진단 → 정리 → 재등록 절차 문서화 | `ac02bdd` (PR #259, Closes #258) |
| Rate Limiting 가이드 | 설정, 동작 방식, 응답 헤더, 클라이언트 대응, 운영 체크리스트 | `10f03b9` (PR #265) |
| 추론 프록시 설계 문서 | Phase 3-1(API Key), 3-2(Rate Limiting) 완료 반영, 아키텍처 다이어그램 추가 | `c6b9f96`, `10f03b9` |
| 전체 문서 최신화 | v0.4 이후 변경사항 반영 및 미구현 기능 명시 | `e7aa617` |
| V04_AUTH_DESIGN | 제외 항목에 후속 이슈(#268, #269) 링크 추가 | `21996b6` |
| 추론 테스트 UI 취소 반영 | #195 취소 상태 문서화 — Open WebUI 등 외부 서비스 대체 방향 | `1cc78c8` |

---

## 이전 Iteration 계획 달성도

Iteration 21에서 계획한 4개 항목 **1개 완료, 3개 미착수** — 추론 API 키 인증 및 Rate Limiting(#194)을 완성하는 데 집중하고, 나머지 시간은 이미지 워머 OOM 긴급 수정 및 인증/인가 고도화 후속 이슈 로드맵 수립에 활용:

| 계획 | 이슈 | 상태 | 비고 |
|------|------|------|------|
| 추론 API 키 인증 및 Rate Limiting | #194 | **완료** | #254(API Key, PR #256) + #255(Rate Limiting, PR #265) 완료 → 상위 이슈 #194 종료 |
| 추론 프록시 Phase 2: 멀티클러스터 라우팅 | #192 | 미착수 | API Key + Rate Limiting 구현에 우선순위 전환 |
| 런타임 자동 튜닝 설계 상세화 | #114 | 미착수 | 동일 사유 |
| hmas-agent 핵심 컴포넌트 구현 착수 | #190 | 미착수 | 동일 사유 |

**추가 달성**: 계획에 없던 이미지 워머 OOM 긴급 수정(#260, #262), Control Plane 정리 가이드(#258), 전체 문서 최신화, 인증/인가 후속 이슈 7건 생성(#264, #266~#270).

---

## 미해결 이슈 (Open Issues)

### 신규 이슈 (7개, 이번 Iteration 생성)

| 이슈 | 제목 | 라벨 |
|------|------|------|
| #270 | 리서치: TPM Rate Limiting 프롬프트 토큰 사전 추정 방안 조사 | research, backend |
| #269 | RBAC 인가 미들웨어 구현 (역할별 권한 분기) | backend, frontend |
| #268 | 사용자 관리 시스템 구현 (CRUD API + UI + 프로필 + 비밀번호 변경) | backend, frontend |
| #267 | JWT 사용자별 커스텀 Rate Limit 지원 | backend |
| #266 | API Key 동적 수정 API (PATCH /api/api-keys/:id) | backend, frontend |
| #264 | Rate Limit 상태 저장소 Redis 전환 | backend |
| #257 | 이미지 사전 캐싱 자동화: 클러스터 등록 시 기본 활성화 + 설정 UI 개선 | enhancement, backend, frontend |

> #268 → #269 → #267 순으로 의존: 사용자 관리(멀티 사용자) → RBAC(역할별 권한) → 사용자별 Rate Limit
> #264(Redis 전환)는 수평 확장 시점에 착수, 현재 단일 인스턴스 환경에서는 In-Memory로 충분
> #270(TPM 토큰 사전 추정)은 동시 요청 패턴 분석 후 구현 여부 판단

### 추론 프록시 이슈 체인

| 이슈 | 제목 | 상태 |
|------|------|------|
| #191 | 추론 프록시 Phase 1: 기본 프록시 구현 (Push 모드) | Iteration 20 완료 |
| #193 | 추론 요청 로깅 및 메트릭 수집 파이프라인 | Iteration 21 완료 |
| #227 | 추론 요청 로그 조회 UI | Iteration 21 완료 |
| #254 | 추론 API Key 인증 시스템 | **Iteration 22 완료** |
| #255 | 추론 API Rate Limiting | **Iteration 22 완료** |
| #194 | 추론 API 키 인증 및 Rate Limiting (상위) | **Iteration 22 완료** |
| #192 | 추론 프록시 Phase 2: 멀티클러스터 라우팅 (agent 터널 통합) | 미착수 |

### 인증/인가 고도화 이슈 체인 (신규)

| 이슈 | 제목 | 상태 |
|------|------|------|
| #254 | API Key 인증 시스템 | **Iteration 22 완료** |
| #255 | Rate Limiting | **Iteration 22 완료** |
| #266 | API Key 동적 수정 API (PATCH) | 미착수 |
| #268 | 사용자 관리 시스템 (CRUD + 프로필) | 미착수 |
| #269 | RBAC 인가 미들웨어 (역할별 권한 분기) | 미착수 — #268 선행 |
| #267 | JWT 사용자별 커스텀 Rate Limit | 미착수 — #268 선행 |
| #264 | Rate Limit Redis 전환 (수평 확장 대비) | 미착수 |
| #270 | TPM 토큰 사전 추정 리서치 | 미착수 |


---

## 이번 주 통계

| 항목 | 수치 |
|------|------|
| 총 커밋 수 | 10개 (squash merge 5건 + docs 5건) |
| 머지된 PR | 5개 (#256, #259, #261, #263, #265) |
| 생성된 이슈 | 12개 (#254, #255, #257, #258, #260, #262, #264, #266, #267, #268, #269, #270) |
| 해결된 이슈 | 6개 (#194, #254, #255, #258, #260, #262) |
| 미해결 이슈 | 34개 |
| 신규 기술 문서 | 7건 (Rate Limiting 가이드, CP 정리 가이드, 프록시 설계 현행화, 전체 문서 최신화, AUTH 문서, 추론 테스트 UI 취소 반영, 지원서 업데이트) |
| 코드 변경량 | +4,527줄 / -194줄 |

---

## 결론

이번 주는 **추론 프록시 인증/인가 및 사용량 제한 체계를 완성한 기간**이었습니다. API Key 인증 시스템(#254)으로 외부 시스템(OpenAI SDK, curl 등)의 프로그래매틱 접근이 가능해졌고, Rate Limiting(#255)으로 사용자/키별 RPM·TPM 제한을 적용하여 남용 방지 및 공정한 자원 분배 기반을 마련했습니다. 이로써 v0.4부터 이어진 추론 프록시 Phase 3(#194)를 종료하고, 인증/인가 고도화 후속 로드맵(사용자 관리, RBAC, Redis 전환, API Key PATCH, TPM 토큰 추정)을 수립하여 v0.9 이후 방향성을 구체화했습니다.

**핵심 성과**:
1. **추론 API Key 인증 완성**: `sk-hmas-` 접두사 키 발급, SHA-256 해싱, JWT/API Key 통합 인증 미들웨어, 배포 범위 제한, OpenAI SDK 호환
2. **추론 API Rate Limiting 완성**: Token Bucket(RPM) + Sliding Window(TPM), 429 응답 + 표준 헤더, 글로벌 기본값 + 키별 커스텀 오버라이드
3. **상위 이슈 #194 종료**: 추론 프록시 Phase 3(API Key + Rate Limiting) 전체 완료
4. **인증/인가 후속 로드맵 수립**: 사용자 관리(#268) → RBAC(#269) → 사용자별 Rate Limit(#267) 의존 체인 및 Redis 전환(#264) 계획 구체화

**다음 주 계획**:
- 추론 프록시 Phase 2: 멀티클러스터 라우팅 (agent 터널 통합) (#192)
- hmas-agent 핵심 컴포넌트 구현 착수 (#190)
- 런타임 자동 튜닝 설계 상세화 (#114)
- API Key 동적 수정 API (#266)

---

**문서 작성일**: 2026년 5월 17일


