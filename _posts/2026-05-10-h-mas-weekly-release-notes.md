---
title: 'H-MAS release notes (2026-05-03~05-09)'
date: 2026-05-10
permalink: /posts/2026/05/h-mas-weekly-release-notes-0503-0509
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
## 2026년 5월 3일 - 5월 9일 (Iteration 21)

### 주요 개발 내용 요약

이번 주는 **추론 요청 로깅 파이프라인 구현(#193)**, **추론 로그 뷰어 UI 및 토큰 사용량 파싱(#227)**, **배포 이벤트 히스토리 DB 저장 및 조회(#250)**에 중점을 둔 기간이었습니다. 총 **12개의 커밋**, **8개의 PR 머지**, **8개의 이슈 생성**이 완료되었으며, 추론 프록시를 경유하는 모든 요청에 대해 DB 로깅·Prometheus 메트릭·토큰 사용량 추적이 가능해졌고, K8s 이벤트 TTL(~1시간) 이후에도 배포 이벤트 이력을 조회할 수 있는 운영 가시성 기반을 구축한 주간이었습니다.

{% include youtube.html id="mqqKaZVdrVw" autoplay=true %}

---

## 새로운 기능 (New Features)

### 1. 추론 요청 로깅 및 메트릭 수집 파이프라인 (#193 → PR #245)

**구현 완료**: 추론 프록시를 경유하는 모든 요청에 대해 DB 로깅, Prometheus 메트릭 5종 수집, 페이지네이션/필터 지원 조회 API를 구현하여 추론 사용 현황 추적 기반 확보

**주요 성과**:

- **Backend**:
  - `inference_logs` 테이블 + 마이그레이션(000023) — request_id, username, deployment, model, cluster, TTFB/Duration, 상태 코드, 스트리밍 여부, 에러 메시지 기록. 인덱스 4개
  - 비동기 채널 기반 배치 writer (`InferenceLogWriter`) — 프록시 레이턴시 무영향 (buffer=1000, batch=100, flush=5s). Non-blocking drop + 유실 카운터 메트릭
  - Prometheus 메트릭 5종: `requests_total`, `duration_seconds`, `ttfb_seconds`, `active_requests`, `log_drops_total`
  - 로그 조회 API (`GET /api/inference-logs`) — deployment, username, status_code, 시간 범위 필터 + 페이지네이션
  - 30일 보존 정책 정리 goroutine (`LogCleaner`) — 1시간 주기, 1000건 배치 삭제

**의사결정 사항**:

| 결정 | 내용 |
|------|------|
| 저장소 선택 | PostgreSQL (v0.9 규모에서 추가 인프라 없이 빠른 구현, v1.0에서 Loki 전환 예정) |
| Writer 구조 | 비동기 채널 기반 분리 — 프록시 핸들러 블로킹 방지, 향후 Loki writer로 교체 용이 |
| 채널 오버플로 | Non-blocking drop + Prometheus Counter 추적 (프록시 안정성 최우선) |

**관련 커밋**: `70a4d0f` (PR #245, Closes #193)

---

### 2. 추론 요청 로그 뷰어 UI 및 토큰 사용량 파싱 (#227 → PR #249)

**구현 완료**: 추론 로그 전용 페이지(`/inference-logs`) 신규 추가 및 비스트리밍/스트리밍 응답에서 토큰 사용량을 투명하게 추출하는 파싱 체계 구현

**주요 성과**:

- **Frontend**:
  - `/inference-logs` 경로에 독립 페이지 생성, 사이드바에 "추론 로그" 메뉴 추가
  - 테이블 UI (타임스탬프, 요청자, 배포, 모델, 상태, TTFB, 소요시간, 토큰)
  - 필터 바 (배포명, 사용자, 상태코드, 시간범위) + 상세 정보 Sheet + 페이지네이션
  - SWR 30초 자동 갱신

- **Backend**:
  - 비스트리밍 응답 바디에서 `usage` JSON 추출
  - 스트리밍 응답은 SSE 이벤트 스캐너(`SSEUsageScanner`)로 투명 토큰 추출
  - opt-in 런타임 지원: vLLM, TGI, llama.cpp, Ollama에 `stream_options.include_usage=true` 자동 주입
  - `hmas_inference_proxy_tokens_total` Prometheus 카운터 (deployment × direction)

**검증 완료**: vLLM/Ollama 비스트리밍·스트리밍 요청에서 토큰 정상 기록 확인, 추론 로그 페이지 필터링·상세 정보 Sheet 동작 확인

**관련 커밋**: `fad3e19` (PR #249, Closes #227)

---

### 3. 배포 이벤트 히스토리 DB 저장 및 조회 (#250 → PR #251)

**구현 완료**: K8s 이벤트 TTL(~1시간) 만료 후에도 배포 이벤트를 조회할 수 있도록 Reconciler 기반 이벤트 수집 및 DB 저장 파이프라인 구현, 모니터링 페이지에 실시간/히스토리 탭 UI 추가

**주요 성과**:

- **Backend**:
  - `deployment_events` 테이블 + 마이그레이션(000024) — fingerprint unique index 기반 중복 제거
  - Reconciler 확장 (`syncDeploymentEvents()`) — 60초 주기, (cluster, namespace) 그룹핑으로 API 호출 최적화
  - 히스토리 조회 API (`GET /api/deployments/:id/events/history`) — 페이지네이션, 타입/시간 필터
  - `EventCleaner` goroutine — 30일 보존 정리 (1시간 주기, 배치 삭제)

- **Frontend**:
  - 모니터링 페이지 배포 이벤트 섹션에 실시간/히스토리 탭 전환 UI
  - 히스토리 탭: Warning/Normal 필터, 페이지네이션, 경고 카운트/타이머 표시

**의사결정 사항**:

| 결정 | 내용 |
|------|------|
| 수집 위치 | 기존 Reconciler 확장 (별도 goroutine 대비 중복 감소, 기존 클라이언트 재사용) |
| NS 최적화 | 배치 수집 — 동일 (cluster, namespace)의 N개 배포 → 이벤트 API 1회 호출 |
| UPSERT 방식 | `clause.OnConflict` — 대량 이벤트 배치 처리에 적합 |

**관련 커밋**: `04892d7` (PR #251, Closes #250)

---

### 4. 이벤트 히스토리 시간 필터 및 pageSize 조절 UI (#252 → PR #253)

**구현 완료**: 배포 이벤트 히스토리 탭에 시간 범위 프리셋(1시간/24시간/7일/30일) 및 사용자 지정 시간 필터, pageSize 조절(25/50/100건) UI 추가

**수정 내용**:
- 시간 범위 프리셋 버튼 및 `datetime-local` 사용자 지정 입력
- `lib/utils/time.ts` 유틸리티 분리 — RFC3339 변환 중앙화 (v0.9 글로벌 타임존 설정 연동 대비)
- pageSize 변경 시 page 1로 리셋

**관련 커밋**: `e5163e9` (PR #253, Closes #252)

---

### 5. 감사 로그 API 페이지네이션 및 필터 지원 (#241 → PR #248)

**구현 완료**: 감사 로그 API에 offset 기반 페이지네이션과 다중 필터(action, actor, from, to) 추가, 프론트엔드에 필터 UI 및 페이지 네비게이션 구현

**수정 내용**:
- **Backend**: `AuditLogFilter` 구조체, `ListByFilter()`/`CountByFilter()` 추가, 응답에 `limit`/`offset` 필드 추가
- **Frontend**: 필터 바(액션, 실행자, 날짜) + 결과 총계 표시 + 첫/이전/다음/마지막 네비게이션

**관련 커밋**: `131f868` (PR #248, Closes #241)

---

### 6. 감사 로그를 모니터링에서 설정 페이지로 이동 (#240 → PR #242)

**구현 완료**: 모니터링 페이지(인스턴스 중심)에서 성격이 맞지 않는 감사 로그를 분리하여 관리 영역인 설정 페이지(`/settings/audit-logs`)로 이동, 아이콘/라벨/색상 매핑 공통 유틸 추출

**수정 내용**:
- `lib/utils/audit-log.ts` 공통 유틸 신규 생성 (대시보드 위젯과 설정 뷰어 공유)
- 모니터링 페이지에서 감사 로그 패널 제거, 설정 페이지에 전용 뷰어 추가
- `FEATURE_SPEC.md` 업데이트 (감사 로그 위치 변경 반영)

**관련 커밋**: `e434843` (PR #242, Closes #240)

---

## 인프라 개선 (Infrastructure)

### Graceful Shutdown 적용 (#246 → PR #247)

**구현 완료**: `signal.NotifyContext` 기반 공유 `rootCtx`를 생성하여 모든 장기 실행 goroutine(reconciler 3개, inference log writer/cleaner)에 전달, Echo HTTP 서버 graceful shutdown 적용

**종료 흐름**: SIGINT/SIGTERM → rootCtx 취소 → reconciler 자연 종료 → HTTP 서버 graceful shutdown (10초 timeout) → inference LogWriter drain → 프로세스 종료

**관련 커밋**: `eedcf17` (PR #247, Closes #246)

---

## 기타 작업 (Chores)

- 서빙 런타임 표시 순서를 프로덕션 우선순위로 재배치: `vLLM → TGI → TEI → llama.cpp → Ollama` (`5817d74`, PR #239, Closes #238)

---

## 문서화 (Documentation)

### 기술 문서 업데이트 (4건)

| 문서 | 내용 | 관련 커밋 |
|------|------|-----------|
| 추론 API 프록시 설계 문서 | Phase 1 완료 상태 반영으로 현행화 | `bb312a8` |
| Hermes Agent 연동 분석 | hmas-agent 향후 연동 분석 추가 | `19f7c7d` |
| 외부 서비스 연동 리서치 | 외부 서비스 연동 리서치 문서 신규 | `0deb093` |
| 추론 런타임 카테고리 UI 개선 계획 | 런타임 카테고리 기반 UI 설계 추가 | `5b9889d` |

---

## 이전 Iteration 계획 달성도

Iteration 20에서 계획한 4개 항목 **2개 완료, 2개 미착수** — 추론 로깅 파이프라인과 추론 테스트 UI 이슈를 종료하고, 나머지 시간은 배포 이벤트 히스토리·감사 로그 개선 등 운영 가시성 강화에 집중:

| 계획 | 이슈 | 상태 | 비고 |
|------|------|------|------|
| 추론 테스트 UI (Chat 인터페이스) 구현 | #195 | ✅ 종료 | PR #249에서 추론 로그 뷰어 및 토큰 추적 구현으로 범위 조정, 이슈 수동 종료 |
| 추론 요청 로깅 및 메트릭 수집 파이프라인 구현 | #193 | ✅ 완료 | PR #245 — DB 로깅 + Prometheus 메트릭 5종 + 조회 API + 30일 보존 |
| 런타임 자동 튜닝 설계 상세화 | #114 | ⏸ 미착수 | 추론 로깅 및 운영 도구 구현에 우선순위 전환 |
| hmas-agent 설계 상세화 | #190 | ⏸ 미착수 | 동일 사유 |

**추가 달성**: 계획에 없던 추론 로그 뷰어 UI(#227), 배포 이벤트 히스토리(#250, #252), 감사 로그 설정 페이지 이동(#240) 및 페이지네이션(#241), Graceful shutdown(#246), 런타임 표시 순서 변경(#238) 완료. 로그 통합 인프라(#243)·파티셔닝(#244) 후속 이슈 생성.

---

## 미해결 이슈 (Open Issues)

### 신규 이슈 (2개, 이번 Iteration 생성)

| 이슈 | 제목 | 라벨 |
|------|------|------|
| #243 | Loki 기반 로그 통합 인프라 구축 | infra |
| #244 | inference_logs 테이블 파티셔닝 적용 | backend |

> #243은 v1.0 목표로, 추론/Pod/앱 로그를 Loki로 통합하는 장기 과제
> #244는 #243 Loki 전환까지 PostgreSQL 부하 완화를 위한 조건부 중간 경로 (실 운영 데이터 기반 판단)

### 추론 프록시 이슈 체인

| 이슈 | 제목 | 상태 |
|------|------|------|
| #191 | 추론 프록시 Phase 1: 기본 프록시 구현 (Push 모드) | ✅ Iteration 20 완료 |
| #193 | 추론 요청 로깅 및 메트릭 수집 파이프라인 | ✅ **Iteration 21 완료** |
| #227 | 추론 요청 로그 조회 UI | ✅ **Iteration 21 완료** |
| #192 | 추론 프록시 Phase 2: 멀티클러스터 라우팅 (agent 터널 통합) | 🔒 미착수 |
| #194 | 추론 API 키 인증 및 Rate Limiting | 🔒 미착수 |

### 추론 로그 확장 경로

| 이슈 | 제목 | 상태 |
|------|------|------|
| #193 | PostgreSQL 기반 추론 로깅 (v0.9) | ✅ **Iteration 21 완료** |
| #244 | inference_logs 파티셔닝 (조건부) | 🔒 실 운영 부하 검증 후 판단 |
| #243 | Loki 기반 로그 통합 (v1.0 최종 목표) | 🔒 미착수 |

### 런타임 자동 튜닝 선행 조건 이슈 체인

| 이슈 | 제목 | 상태 |
|------|------|------|
| #125 | 서빙 배포 인플레이스 업데이트 API (PATCH) | ✅ Iteration 19 완료 |
| #126 | 서빙 배포 파라미터 변경 시 Rolling Restart 파이프라인 | ✅ Iteration 19 완료 |
| #127 | ~~서빙 런타임 메트릭 수집 파이프라인 구축~~ | ✅ Iteration 18 완료 |
| #128 | 서빙 최적화 파라미터 변경 이력 관리 및 감사 로그 | ✅ Iteration 19 완료 |
| #114 | 런타임 자동 튜닝 (메트릭 기반 파라미터 자동 조정) | 🔓 선행 조건 4/4 완료 — 착수 가능 |


---

## 이번 주 통계

| 항목 | 수치 |
|------|------|
| 총 커밋 수 | 12개 (squash merge 8건 + docs 4건) |
| 머지된 PR | 8개 (#239, #242, #245, #247, #248, #249, #251, #253) |
| 생성된 이슈 | 8개 (#238, #240, #241, #243, #244, #246, #250, #252) |
| 해결된 이슈 | 9개 (#193, #195, #227, #238, #240, #241, #246, #250, #252) |
| 미해결 이슈 | 28개 |
| 신규 기술 문서 | 4건 (프록시 설계 현행화, xxxxxxx Agent 연동, 외부 서비스 연동, 런타임 카테고리 UI) |
| 코드 변경량 | +4,780줄 / -360줄 |

---

## 결론

이번 주는 **추론 프록시 운영 가시성 확보와 플랫폼 운영 도구 고도화에 집중한 기간**이었습니다. 추론 요청 로깅 파이프라인(#193)으로 모든 추론 요청의 사용자·배포·토큰·레이턴시를 DB에 기록하고 Prometheus 메트릭으로 노출하는 기반을 마련했으며, 추론 로그 뷰어 UI(#227)와 SSE 스트리밍 토큰 파싱을 통해 실시간 사용 현황을 추적할 수 있게 되었습니다. 배포 이벤트 히스토리(#250)로 K8s 이벤트 TTL 이후에도 장애 원인 추적이 가능해졌고, 감사 로그 설정 페이지 이동(#240) 및 페이지네이션(#241)으로 관리 기능을 체계화했습니다.

**핵심 성과**:
1. **추론 로깅 파이프라인 완성**: 비동기 채널 배치 writer + Prometheus 메트릭 5종 + 30일 보존 — 프록시 레이턴시 무영향 설계
2. **추론 로그 뷰어 + 토큰 파싱**: 비스트리밍/SSE 스트리밍 모두 투명 토큰 추출, vLLM·TGI·llama.cpp·Ollama opt-in 자동 주입
3. **배포 이벤트 히스토리**: Reconciler 기반 수집 + fingerprint 중복 제거 + 실시간/히스토리 탭 UI
4. **운영 도구 고도화**: 감사 로그 설정 이동 + 페이지네이션/필터, Graceful shutdown 전체 적용

**다음 주 계획**:
- 추론 프록시 Phase 2: 멀티클러스터 라우팅 (agent 터널 통합) (#192)
- 런타임 자동 튜닝 설계 상세화 (#114)
- hmas-agent 핵심 컴포넌트 구현 착수 (#190)
- 추론 API 키 인증 및 Rate Limiting (#194)

---

**문서 작성일**: 2026년 5월 10일
