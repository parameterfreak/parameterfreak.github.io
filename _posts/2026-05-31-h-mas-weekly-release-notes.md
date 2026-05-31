---
title: 'H-MAS release notes (2026-05-24~05-30)'
date: 2026-05-31
permalink: /posts/2026/05/h-mas-weekly-release-notes-0524-0530
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
## 2026년 5월 24일 - 5월 30일 (Iteration 24)

### 주요 개발 내용 요약

이번 주는 **Organization/Group 멀티테넌시 전체 파이프라인 구현(리서치→DB→CRUD→인가→Frontend)**, **Rate Limit 인프라 Redis 전환 및 실시간 사용량 UI**, **최적화 프리셋 복제/내보내기/가져오기 기능 완성**에 중점을 둔 기간이었습니다. 총 **68개의 커밋**, **10개의 PR 머지**, **11개의 이슈 생성**, **10개의 이슈 해결**이 완료되었으며, 특히 **멀티테넌시 Phase 2를 리서치부터 Frontend까지 7일 만에 전 계층 구현 완료**하여 7월 데모를 위한 고객사별 독립 환경의 기반을 확보했습니다.

{% include youtube.html id="hKYzkB-x7VE" autoplay=true %}

---

## 새로운 기능 (New Features)

### 1. 멀티테넌시 Phase 2: Organization/Group 전 계층 구현 (#284, #286~#289)

**구현 완료**: Flat RBAC(admin/member/viewer) 단일 계층에서 Organization > Group 계층적 멀티테넌시 모델로 전환 — 리서치(#284) → DB(#286) → CRUD API(#287) → 인가(#288) → Frontend(#289)를 의존 순서대로 연속 착수·완료하여 **고객사별 완전 격리 환경** 달성

#### 1-1. 리서치: Flat RBAC → Organization/Group 전환 계획 수립 (#284 → PR #285)

**완료**: 현행 Flat RBAC와 목표 Org/Group 모델 간 갭 분석, 10개 핵심 기술 결정사항 문서화

**주요 성과**:
- 23개 리소스에 대한 Org/Group 범위 재분류 및 권한 매트릭스 수립
- 클러스터 할당, GPU 쿼터, 모델 저장소, 네트워크 격리 등 10개 영역별 설계 결정 완료
- DB 마이그레이션 7단계 순차 계획 수립 (무중단 전환)
- Phase 2 → Phase 3-Early → Phase 3-Late 진화 경로 정의
- v0.7 데모용 최소 범위(Phase 2a)와 v1.0 프로덕션 범위(Phase 2b) 분리

**의사결정 사항**:

| 결정 | 내용 |
|------|------|
| 클러스터 할당 방식 | xxxxxxx 레이블 기반 하이브리드(Label Pool) 채택 |
| GPU 쿼터 모델 | Dedicated + Org 공유 풀 혼합 모드 |
| 모델 저장소 범위 | Organization 단위 (Org admin CRUD, Group 읽기) |
| 네트워크 격리 | 계층적 NetworkPolicy 3계층 + 물리 분리 선택적 적용 |

**관련 커밋**: `df20150` 외 9건 (PR #285, Closes #284)

---

#### 1-2. Phase 2-1: DB 마이그레이션 + 데이터 모델 (#286 → PR #294)

**구현 완료**: Organization/Group 멀티테넌시를 위한 DB 스키마 변경 및 GORM 모델 정의 — 기존 환경 하위 호환 유지

**주요 성과**:
- **Backend**:
  - 3개 마이그레이션 파일(000029~000031)로 논리적 분리: 테이블 생성 → 사용자 매핑 → FK 추가
  - `organizations`, `groups`, `org_members`, `group_members` 테이블 + Default Organization/Group 시드
  - 기존 사용자를 Default Org/Group에 자동 매핑 (admin → system_admin 전환)
  - 6개 기존 테이블에 FK 추가 (serving_deployments, models, optimization_presets, audit_logs, api_keys)

**관련 커밋**: `31f72e4` 외 2건 (PR #294, Closes #286)

---

#### 1-3. Phase 2-2: Backend Org/Group CRUD + 멤버십 API (#287 → PR #295)

**구현 완료**: Organization/Group/Membership 관리를 위한 **18개 엔드포인트** 구현 — Handler → Service → Repository 3계층 아키텍처로 비즈니스 로직 분리

**주요 성과**:
- **Backend**:
  - Organization CRUD 5개 + Group CRUD 5개 + Membership 관리 8개 = 18개 엔드포인트
  - 비즈니스 규칙: Default Org/Group 삭제 방지, 마지막 admin 보호, Org⊇Group 멤버십 검증
  - slug DNS-safe 유효성 검증, namespace 자동생성(`hmas-{org_slug}-{group_slug}`)
  - Cross-org IDOR 방지, TOCTOU race 방어, 멤버 제거 트랜잭션 원자화

**해결된 기술 문제**:

| 문제 | 원인 | 해결 |
|------|------|------|
| Cross-org 리소스 접근 가능 | Group 수정/삭제 시 orgID 소속 미검증 | 서비스 레이어에 org 소속 검증 추가 |
| 멤버 제거 시 부분 삭제 발생 | Org 멤버십과 Group 멤버십 삭제가 별도 트랜잭션 | 단일 DB 트랜잭션으로 원자화 |

**관련 커밋**: `0813430` 외 7건 (PR #295, Closes #287)

---

#### 1-4. Phase 2-3: Backend 인가 미들웨어 확장 + 리소스 스코핑 (#288 → PR #296)

**구현 완료**: 기존 RequireMinRole 단일 미들웨어를 Org/Group 멤버십 기반 3계층 인가 체계로 확장 — **고객사 A의 데이터가 고객사 B에게 보이지 않는** 핵심 격리 달성

**주요 성과**:
- **Backend**:
  - JWT Claims에 `system_role` 필드 추가
  - `InjectOrgGroupContext` → `RequireSystemAdmin` / `RequireOrgRole` / `RequireGroupRole` 3단계 미들웨어 체인
  - 배포(Group), 모델(Org), API Key(Org), 추론로그(Group), 감사로그(Org+Group) 핸들러에 리소스 스코핑 적용
  - X-Org-ID/X-Group-ID 헤더 위장 방어 (멤버십 검증 + Group-to-Org 소속 검증)
  - 명시적 Org/Group 요청 실패 시 fail closed 적용 (Default 폴백 차단)

**해결된 기술 문제**:

| 문제 | 원인 | 해결 |
|------|------|------|
| 비-admin이 Org/Group 없이 리소스 생성 시 고아 레코드 | 컨텍스트 필수 검증 누락 | orgID/groupID=0 && !IsSystemAdmin 시 403 반환 |
| DB 에러가 도메인 에러로 마스킹 | 모든 에러를 NotFound로 변환 | ErrRecordNotFound만 매핑, 나머지 원본 전파 |

**관련 커밋**: `da6afde` 외 9건 (PR #296, Closes #288)

---

#### 1-5. Phase 2-4: Frontend Org/Group 셀렉터 + 관리 UI + 리소스 조직 스코핑 (#289 → PR #297)

**구현 완료**: Organization/Group 컨텍스트 전환 UI, 관리 페이지, 전 리소스 조직 스코핑 적용 — **7월 데모에서 고객사별 독립 환경을 UI로 시연** 가능

**주요 성과**:
- **Frontend**:
  - 헤더에 Organization/Group 셀렉터 배치, 전환 시 전체 리소스 즉시 리페치
  - 조직/그룹 관리 페이지: 생성/수정/삭제, 멤버 관리 (드로어 UI)
  - 도메인 기반 멤버십: 조직 생성 시 이메일 도메인 설정, 도메인 일치 사용자만 추가 가능
  - 모델/정책 3종: 빌트인(전역) + 사용자 추가(조직별) 2계층 구분, 빌트인 수정/삭제 차단
  - 클러스터: 조직 소유 자원으로 완전 격리, 등록 시 현재 조직 자동 연결
  - 배포 네임스페이스: 현재 그룹의 K8s 네임스페이스를 기본값으로 연동
  - System Admin / Org Admin / Group Admin 3계층 역할 기반 메뉴 가시성 제어
  - localStorage 기반 헤더 즉시 복원으로 stale 데이터 방지

**관련 커밋**: `94258a2` 외 14건 (PR #297, Closes #289)

---

### 2. Rate Limit 인프라 강화: Redis 전환 + 실시간 사용량 UI (#264, #281)

#### 2-1. Rate Limit 상태 저장소 Redis 전환 (#264 → PR #282)

**구현 완료**: In-Memory 전용이었던 Rate Limit 저장소를 Redis로 전환하여 멀티 인스턴스 환경에서 통합 카운팅 지원 — 수평 확장 대비 인프라 기반 확보

**주요 성과**:
- **Backend**:
  - `TPMStore` 인터페이스 추출, `RedisStore`(Lua Token Bucket) + `RedisTPMTracker`(Sorted Set Sliding Window) 구현
  - Redis 장애 시 폴백 전략 (open/memory) 및 헬스체크 프로브
  - `NewStores()` 팩토리로 환경변수 기반 저장소 선택
  - 단위 테스트 9개 추가
- **Infra**:
  - Helm Chart에 Bitnami Redis subchart 추가 (기본 비활성화, 하위 호환)
  - Helm release name `h-mas` → `hmas` 통일

**해결된 기술 문제**:

| 문제 | 원인 | 해결 |
|------|------|------|
| Redis 미연결 + FallbackOpen 시 서버 기동 불가 | Ping 실패 시 무조건 에러 반환 | open 모드 기동 허용, Redis 복구 시 자동 전환 |
| 폴백 전환 시 Redis client 리소스 누수 | Close 미호출 | 조기 반환 시 client 리소스 명시적 해제 |

**관련 커밋**: `735b990` 외 7건 (PR #282, Closes #264)

---

#### 2-2. Rate Limit 실시간 사용량 UI (#281 → PR #283)

**구현 완료**: API Key 목록에서 각 키의 실시간 RPM/TPM 사용량을 프로그레스 바와 리셋 카운트다운으로 표시

**주요 성과**:
- **Backend**: `GET /api/api-keys` 응답에 키별 usage(rpm/tpm remaining, resetAt) 추가
- **Frontend**: `RateLimitUsage` 컴포넌트(MiniBar + 툴팁 + 카운트다운), SWR 30초 polling 실시간 갱신, 키보드 접근성 지원

**관련 커밋**: `5d174ab` 외 2건 (PR #283, Closes #281)

---

### 3. 최적화 프리셋 복제(Clone) + JSON 내보내기/가져오기 (#122 → PR #280)

**구현 완료**: #120에서 DB 전환 완료 후 잔여 범위인 프리셋 복제 및 JSON Import/Export 기능 구현 — v0.2부터 이어온 장기 과제(#122) 최종 완료

**주요 성과**:
- **Backend**: `POST /api/optimization-presets/import` 엔드포인트 — 충돌 전략(skip/overwrite/rename) 지원
- **Frontend**:
  - 프리셋 관리 드롭다운에 "복제" 메뉴 (slug `-copy` / 이름 `(복제)` 자동 생성)
  - 테이블 체크박스 선택 → JSON 내보내기 다운로드
  - 가져오기 Sheet: 파일 업로드 → 전략 선택 → 결과 요약 표시
  - 배포 폼에서 커스텀 프리셋 별도 "커스텀 프리셋" 섹션으로 표시

**관련 커밋**: `d15ab58` 외 5건 (PR #280, Closes #122)

---

### 4. 이미지 사전 캐싱 자동화 (#257 → PR #277)

**구현 완료**: GPU 클러스터 등록 시 이미지 캐싱을 자동 활성화하고, 설정 페이지에서 클러스터별 관리 UI 제공 — 사용자가 내부 레이블이나 API를 알 필요 없이 토글로 관리

**주요 성과**:
- **Backend**: 클러스터 등록 시 GPU 유무에 따라 `image_cache_enabled` 자동 설정 + xxxxxxx 레이블 즉시 적용
- **Frontend**:
  - 클러스터 등록 위자드 Step 3에 이미지 사전 캐싱 토글 (GPU 선택과 연동)
  - 설정 > 런타임 이미지 관리 페이지를 탭으로 분리 (이미지 설정 / 캐싱 대상 클러스터)
  - `ImageCacheClusterCard` 컴포넌트: GPU/CPU 클러스터 분리 표시, 개별 토글

**관련 커밋**: `5706226` 외 3건 (PR #277, Closes #257)

---

## 버그 수정 (Bug Fixes)

### 1. 커스텀 GPU 뱃지 X 버튼 접근성 개선 (#278 → PR #279)

**수정 완료**: 클러스터 등록 위자드 Step 3의 커스텀 GPU 뱃지 삭제 버튼을 네이티브 `<button>`으로 감싸 키보드 접근(Tab/Enter/Space), `aria-label`, 포커스 링 지원

**관련 커밋**: `7add69b` (PR #279, Closes #278)

---

## 문서화 (Documentation)

### 신규 기술 문서 (2건)

| 문서 | 내용 | 관련 커밋 |
|------|------|-----------|
| `ORG_GROUP_MODEL_REDESIGN.md` | Org/Group 재설계 SSOT — 10개 핵심 기술 결정, DB 마이그레이션 계획, 인가/프론트엔드 확장 설계 (1,578줄) | `df20150` (PR #285) |
| Phase 2a/2b 분리 문서 | v0.7 데모용 최소 범위와 v1.0 프로덕션 범위 구분, 후순위 이슈(#290~#293) 참조 | `f1685e8` |

---

## 이전 Iteration 계획 달성도

Iteration 23에서 계획한 4개 항목 **모두 미착수** — 멀티테넌시 Phase 2 전체 구현(리서치→DB→CRUD→인가→Frontend)을 7월 데모 일정에 맞춰 우선 착수하면서 기존 계획을 전면 조정:

| 계획 | 이슈 | 상태 | 비고 |
|------|------|------|------|
| 추론 프록시 Phase 2: 멀티클러스터 라우팅 | #192 | 미착수 | 멀티테넌시 Phase 2 우선 |
| hmas-agent 핵심 컴포넌트 구현 착수 | #190 | 미착수 | 동일 사유 |
| 런타임 자동 튜닝 설계 상세화 | #114 | 미착수 | 동일 사유 |
| JWT 사용자별 커스텀 Rate Limit | #267 | 미착수 | Rate Limit Redis 전환(#264) 선 진행 |

**대신 달성**: 멀티테넌시 Phase 2 전 계층 구현(#284~#289), Rate Limit Redis 전환(#264) + 사용량 UI(#281), 프리셋 복제/내보내기(#122), 이미지 캐싱 자동화(#257), GPU 뱃지 접근성(#278) — 총 10건 추가 완료.

---

## 미해결 이슈 (Open Issues)

### 멀티테넌시 후속 이슈 체인 (Phase 2b / Phase 3)

| 이슈 | 제목 | 상태 |
|------|------|------|
| #284 | 사용자·조직 모델 재설계 리서치 | **Iteration 24 완료** |
| #286 | 멀티테넌시 Phase 2-1: DB 마이그레이션 + 데이터 모델 | **Iteration 24 완료** |
| #287 | 멀티테넌시 Phase 2-2: Backend Org/Group CRUD + 멤버십 API | **Iteration 24 완료** |
| #288 | 멀티테넌시 Phase 2-3: Backend 인가 미들웨어 + 리소스 스코핑 | **Iteration 24 완료** |
| #289 | 멀티테넌시 Phase 2-4: Frontend Org/Group 셀렉터 + 관리 UI | **Iteration 24 완료** |
| #290 | Phase 2b-1: GPU 쿼터 혼합 모드 (Dedicated + Org 공유 풀) | 미착수 |
| #291 | Phase 2b-2: Org 프리셋 분리 + Cross-Group 가시성 + Group 삭제 전략 | 미착수 |
| #292 | Phase 3-Early: Org별 클러스터 격리 + NetworkPolicy + PP org-access | 미착수 |
| #293 | Phase 3-Late: 미터링/과금 파이프라인 (Prometheus + PostgreSQL) | 미착수 |

> 멀티테넌시 Phase 2a(v0.7 Demo-Ready) 5건 전체 완료, Phase 2b(v1.0) 2건 + Phase 3 2건 잔여

### 인증/인가 후속 이슈 체인

| 이슈 | 제목 | 상태 |
|------|------|------|
| #264 | Rate Limit Redis 전환 | **Iteration 24 완료** |
| #267 | JWT 사용자별 커스텀 Rate Limit | 미착수 |
| #270 | TPM 토큰 사전 추정 리서치 | 미착수 |

> 인증/인가 고도화 로드맵 8건 중 6건 완료 (#254, #255, #266, #268, #269, #264), 잔여 2건 (#267, #270)

### 추론 프록시 이슈 체인

| 이슈 | 제목 | 상태 |
|------|------|------|
| #191 | Phase 1: 기본 프록시 구현 | Iteration 20 완료 |
| #193 | 요청 로깅 및 메트릭 수집 | Iteration 21 완료 |
| #227 | 요청 로그 조회 UI | Iteration 21 완료 |
| #194 | API Key 인증 + Rate Limiting (상위) | Iteration 22 완료 |
| #192 | Phase 2: 멀티클러스터 라우팅 (agent 터널 통합) | 미착수 |

---

## 이번 주 통계

| 항목 | 수치 |
|------|------|
| 총 커밋 수 | 68개 (10개 PR 내 feature branch 커밋) |
| 머지된 PR | 10개 (#277, #279, #280, #282, #283, #285, #294, #295, #296, #297) |
| 생성된 이슈 | 11개 (#278, #281, #284, #286, #287, #288, #289, #290, #291, #292, #293) |
| 해결된 이슈 | 10개 (#122, #257, #264, #278, #281, #284, #286, #287, #288, #289) |
| 미해결 이슈 | 30개 |
| 신규 기술 문서 | 2건 (Org/Group 재설계 SSOT, Phase 2a/2b 분리) |
| 코드 변경량 | +10,567줄 / -326줄 |

---

## 결론

이번 주는 **멀티테넌시 Phase 2를 리서치부터 Frontend까지 전 계층을 7일 만에 구현 완료하고, Rate Limit 인프라를 수평 확장 가능하게 전환한 기간**이었습니다. Flat RBAC 단일 계층에서 Organization > Group 계층적 멀티테넌시로 전환하여 고객사별 완전 격리 환경의 기반을 확보했으며, 7월 데모에서 고객사별 독립 환경을 UI로 시연할 수 있게 되었습니다. Rate Limit Redis 전환(#264)으로 멀티 인스턴스 통합 카운팅 기반을 마련하고 실시간 사용량 UI(#281)로 운영 가시성을 확보했습니다. 프리셋 복제/내보내기(#122)와 이미지 캐싱 자동화(#257) 완료로 기존 백로그 과제도 정리했습니다.

**핵심 성과**:
1. **멀티테넌시 Phase 2a 전체 완료**: 리서치(#284) → DB(#286) → CRUD(#287) → 인가(#288) → Frontend(#289) — 5개 이슈를 의존 순서대로 연속 구현하여 고객사별 완전 격리 달성
2. **Rate Limit 수평 확장 기반 확보**: Redis 전환(#264) + 실시간 사용량 UI(#281) — 멀티 인스턴스 통합 카운팅 + 운영 가시성
3. **장기 과제 2건 최종 완료**: 프리셋 복제/내보내기(#122, v0.2~), 이미지 캐싱 자동화(#257)

**다음 주 계획**:
- 멀티테넌시 안정화 및 7월 데모 시나리오 검증
- 추론 프록시 Phase 2: 멀티클러스터 라우팅 (#192)
- hmas-agent 핵심 컴포넌트 구현 착수 (#190)
- JWT 사용자별 커스텀 Rate Limit (#267)

---

**문서 작성일**: 2026년 5월 31일
