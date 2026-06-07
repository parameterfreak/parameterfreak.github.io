---
title: 'H-MAS release notes (2026-05-31~06-06)'
date: 2026-06-07
permalink: /posts/2026/06/h-mas-weekly-release-notes-0531-0606
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
## 2026년 5월 31일 - 6월 6일 (Iteration 25)

### 주요 개발 내용 요약

이번 주는 **xxxxxxxxxxx v0.7 통합(설치 검증→QoS 검증→디바이스 검증→API→OverridePolicy→Frontend→E2E 테스트)**, **멀티테넌시 Frontend 마무리 및 UX 개선(Popover 스위처)**, **런타임 전략 정비(Ollama/TGI 제거)**에 중점을 둔 기간이었습니다. 총 **22개의 커밋**, **13개의 PR 머지**, **12개의 이슈 생성**, **14개의 이슈 해결**이 완료되었으며, 특히 **xxxxxxxxxxx 통합을 L1 설치부터 E2E 테스트까지 1주 만에 전 계층 구현 완료**하여 v0.7 GP(General Purpose) 클러스터 서빙의 기반을 확보했습니다.

{% include youtube.html id="q99B-IkWmHg" autoplay=true %}

---

## 새로운 기능 (New Features)

### 1. xxxxxxxxxxx v0.7 통합: 설치 검증부터 E2E까지 (#216, #217, #218, #300~#303)

**구현 완료**: xxxxxxxxxxx v1.7.0 설치 → QoS 클래스 검증 → Device CRD 기반 GPU 토폴로지 인식 → 스케줄러 타입 변경 API → OverridePolicy 자동 생성 → Frontend QoS 선택 UI → E2E 통합 테스트를 의존 순서대로 연속 구현하여 **GP 클러스터에서 xxxxxxxxxxx 기반 서빙 동작** 달성

#### 1-1. L1 설치 검증: xxxxxxxxxxx v1.7.0 설치 및 환경 호환성 (#216 → PR #304)

**완료**: MicroK8s v1.33.12 환경에서 xxxxxxxxxxx v1.7.0 설치, 17개 검증 항목 전체 통과

**주요 성과**:
- xxxxxxxxxxx 4개 컴포넌트(controller-manager, scheduler, descheduler, xxxxxlet) 전체 정상 동작
- CRD 10개 + NodeMetric + Device CRD 생성 확인
- NVIDIA Device Plugin과의 공존 검증 (`nvidia.com/gpu: 1` 유지)
- kube-scheduler + xxxxx-scheduler 병행 구동 확인
- 기존 H-MAS 서빙 삭제 → 재배포 → 추론 회귀 테스트 통과
- 설치 가이드, Helm values 템플릿, K8s-xxxxxxxxxxx 버전 호환성 관리 가이드 작성

**의사결정 사항**:

| 결정 | 내용 |
|------|------|
| xxxxxxxxxxx 버전 | v1.8.0은 K8s 1.33 DRA API 미지원으로 v1.7.0 채택 |
| K8s 1.28~1.34 호환 규칙 | xxxxxxxxxxx v1.7.0 사용으로 표준화 |

**관련 커밋**: `b9fb25b` (PR #304, Closes #216)

---

#### 1-2. L2 QoS 검증: QoS 클래스별 동작 확인 및 H-MAS 기본값 결정 (#217 → PR #305)

**완료**: LSR/LS/BE QoS 클래스별 실동작 검증, H-MAS 배포 환경별 QoS 기본값 확정

**주요 성과**:
- LSR(CPU 코어 독점), LS(공유 풀), BE(오버커밋) 3개 클래스 정상 동작 확인
- ClusterColocationProfile 기반 QoS/schedulerName/priority 자동 주입 검증
- xxxxxlet BE CPU Suppress 동작 확인 (임계값 65% 초과 시 cpuset 축소)
- H-MAS QoS 기본값 결정: 프로덕션=LSR, 스테이징=LS, 개발=BE

**관련 커밋**: `b7434f1` (PR #305, Closes #217)

---

#### 1-3. L3 디바이스 검증: Device CRD 구조 분석 및 GPU 감지 (#218 → PR #309)

**완료**: xxxxx-device-daemon의 GPU 토폴로지 자동 수집 메커니즘 분석, GPU 감지 트러블슈팅 가이드 작성

**주요 성과**:
- Device CRD에 수집되는 GPU 토폴로지 데이터 구조(busID, nodeID, pcieID, VRAM 등) 분석
- xxxxx-scheduler의 Device 기반 개별 GPU 인스턴스 추적/할당 메커니즘 확인
- MicroK8s + GPU Operator 환경 NVML 미주입 문제 해결 가이드 작성

**해결된 기술 문제**:

| 문제 | 원인 | 해결 |
|------|------|------|
| Device CRD `spec.devices` 비어 있음 | xxxxxlet에 NVML 라이브러리 미주입 | 호스트 라이브러리 경로 마운트 패치 적용 |

**관련 커밋**: `cb2ade0` (PR #309, Closes #218)

---

#### 1-4. Backend: 클러스터 스케줄러 타입 변경 API 및 Capabilities 반환 (#300 → PR #306)

**구현 완료**: `PATCH /api/clusters/:name` 범용 클러스터 수정 API — schedulerType 변경 시 DB + xxxxxxx 레이블 동기화, Capabilities 정보 정적 반환

**주요 성과**:
- **Backend**:
  - `PATCH /api/clusters/:name` API 구현 (schedulerType 외 description 등 향후 확장 가능)
  - `ClusterCapabilities` 모델: schedulerType 기반 QoS/오버커밋/GPU 토폴로지 기능 정보 정적 생성
  - schedulerType → clusterType 자동 연동 + 명시적 오버라이드 지원
  - Swagger 문서 재생성

**관련 커밋**: `ea04d66` (PR #306, Closes #300)

---

#### 1-5. Backend: xxxxxxxxxxx QoS 자동 주입 OverridePolicy (#301 → PR #310)

**구현 완료**: GP 클러스터 배포 시 xxxxxxxxxxx 전용 설정(QoS 클래스, schedulerName)을 담은 OverridePolicy를 자동 생성, 배포 삭제 시 함께 정리

**주요 성과**:
- **Backend**:
  - GP 클러스터(`scheduler_type=xxxxxxxxxxx`) 배포 시 OverridePolicy 자동 생성 (schedulerName + qosClass 주입)
  - 배포 요청에서 `qosClass` 파라미터 수신 (LSE/LSR/BE, 기본값: LSR)
  - 배포 삭제 시 연관 OverridePolicy 자동 정리
  - Standard 클러스터 대상 시 기존 로직 유지 (OverridePolicy 생성 스킵)

**관련 커밋**: `4f2b7bf` (PR #310, Closes #301)

---

#### 1-6. Frontend: 배포 위자드 QoS 클래스 선택 UI (#302 → PR #311)

**구현 완료**: xxxxxxxxxxx 클러스터 선택 시 배포 위자드에 QoS 클래스 선택 드롭다운 표시, 검토 화면 및 인스턴스 목록/상세에 QoS 정보 반영

**주요 성과**:
- **Frontend**:
  - GP 클러스터 조건부 QoS 클래스(LSE/LSR/LS/BE) 선택 드롭다운
  - BE 경고 메시지 / LSR CPU 바인딩 안내 표시
  - 검토 화면(Step 4), 인스턴스 목록, 인스턴스 상세 페이지에 QoS 표시
  - Standard 클러스터에서는 QoS 선택 숨김

**관련 커밋**: `1120ed6` (PR #311, Closes #302)

---

#### 1-7. E2E 통합 테스트: Standard + GP 혼재 환경 9개 시나리오 (#303 → PR #312)

**구현 완료**: 실제 환경(local-cluster + research 클러스터)에서 Standard + GP 혼재 멀티클러스터 E2E 테스트 스위트 — **9개 테스트 전체 PASS**

**주요 성과**:
- API 레벨 빠른 검증 9개 시나리오 (배포 흐름, 스케줄러 전환, OverridePolicy 정리, 엣지 케이스)
- 실환경 14.9초 내 전체 PASS
- Standard↔GP 타입 전환, 동시 배포, QoS 검증 등 엣지 케이스 커버

**관련 커밋**: `41153e9` (PR #312, Closes #303)

---

### 2. 멀티테넌시 Frontend 마무리: Org/Group UI + Popover 스위처 (#289, #314)

#### 2-1. Org/Group 관리 UI + 리소스 조직 스코핑 (#289 → PR #297)

**구현 완료**: 멀티테넌시 Phase 2-4 최종 완료 — Org/Group 관리 UI, 도메인 기반 멤버십, 리소스 조직 스코핑 전체 적용

**주요 성과**:
- **Frontend**:
  - 조직/그룹 관리 페이지: 생성/수정/삭제, 멤버 관리 (드로어 UI)
  - 도메인 기반 멤버십: 이메일 도메인 설정, 도메인 일치 사용자만 멤버 추가 가능
  - 모델/정책 3종: 빌트인(전역) + 사용자 추가(조직별) 2계층 구분
  - 클러스터: 조직 소유 자원으로 완전 격리
  - 배포 네임스페이스: 현재 그룹의 K8s 네임스페이스 기본값 연동
  - System Admin / Org Admin / Group Admin 3계층 역할 기반 메뉴 가시성 제어

**관련 커밋**: `94258a2` 외 6건 (PR #297, Closes #289)

---

#### 2-2. Org/Group 셀렉터 Popover 스위처 통합 (#314 → PR #315)

**구현 완료**: 사이드바 상단의 별도 Org/Group 드롭다운을 로고 영역에 통합된 Popover 스위처로 교체 — Vercel/Linear/GitHub 스타일의 모던 워크스페이스 스위처 패턴 적용

**주요 성과**:
- **Frontend**:
  - 3줄 → 1줄 압축으로 네비게이션 공간 효율성 대폭 개선
  - 클릭 시 Popover로 조직/그룹 선택 UI 표시
  - 단일 org/group일 경우 인터랙션 비활성화
  - org 선택 시 Popover 즉시 닫힘, group 1개 시 읽기 전용 표시

**관련 커밋**: `5e0490c` (PR #315, Closes #314)

---

### 3. 추론 로그 요청자 API 키 기준 표시 (#320 → PR #321)

**구현 완료**: 추론 로그의 "요청자" 필드를 인증 방식에 따라 구분 표시 — API 키 이름 또는 JWT 사용자명으로 분기

**주요 성과**:
- **Backend**:
  - `inference_logs.api_key_name` 컬럼 추가 (비정규화 저장, 감사 로그 성격 유지)
  - `auth_method` / `api_key_name` 필터 파라미터 지원
- **Frontend**:
  - `RequesterDisplay` 컴포넌트: 인증 방식별 아이콘 분기 (API 키 / JWT)
  - 상세 시트에 인증 방식 Badge, 키 소유자 표시
  - "인증 방식" 드롭다운 + "API 키 이름" 검색 필터 추가

**관련 커밋**: `9e8cf28` (PR #321, Closes #320)

---

## 버그 수정 (Bug Fixes)

### 1. 감사 로그 페이지 브라우저 새로고침 시 React Hook 에러 (#318 → PR #319)

**수정 완료**: `AuditLogsPage`에서 `useCallback` 훅이 조건부 early return 뒤에 위치하여 렌더링 간 훅 호출 수 불일치 발생 → 훅을 조건부 return 위로 이동하여 해결

**관련 커밋**: `900d237` (PR #319, Closes #318)

---

## 리팩토링 (Refactoring)

### 1. schedulerType enum 값 "default" → "none" 변경 (#307 → PR #313)

**완료**: 커스텀 스케줄러 미설치 상태를 `"default"` 대신 `"none"`으로 명확히 표현 — Backend/Frontend/DB 마이그레이션/E2E 테스트/문서/Swagger 전체 22개 파일 일괄 반영

**관련 커밋**: `a03ac42` (PR #313, Closes #307)

---

## 문서화 (Documentation)

### 신규/갱신 기술 문서 (7건)

| 문서 | 내용 | 관련 커밋 |
|------|------|-----------|
| `xxxxxxxxxxx_INSTALL_GUIDE.md` | xxxxxxxxxxx v1.7.0 설치 가이드 (Helm values, 검증 체크리스트, 버전 호환성) | `b9fb25b` (PR #304) |
| `xxxxxxxxxxx_FEATURES.md` §2.3~§2.5 | QoS 클래스 검증 결과 및 H-MAS 기본값 결정 | `b7434f1` (PR #305) |
| `xxxxxxxxxxx_FEATURES.md` §3.4, §5 | Device CRD 구조 분석, GPU 감지 트러블슈팅 | `cb2ade0` (PR #309) |
| `SCHEDULER_CONFIGURATION.md` | schedulerType 정의, 흐름도, DB 스키마, 변경 이력 | `a03ac42` (PR #313) |
| xxxxxxxxxxx 통합 문서 v0.7 구현 완료 상태 | #219 구현 상태 반영 | `85e1c20` |
| xxxxxxx 고아 Work GC 로드맵 및 상태 관리 문서 | #316, #317 이슈 로드맵 반영 | `6608d7d` |
| 기술 문서 전체 최신화 | Core 3 런타임, hmas-agent v0.9 이동, 구현 상태 갱신 | `6084a91` |

---

## 기타 작업 (Chores)

- Ollama, TGI 런타임 코드 완전 제거 — Core 런타임 vLLM + llama.cpp + TEI 3종 집중 전략 정비, Backend 30개 파일 + Frontend 정리, DB 마이그레이션 000036 (PR #299, Closes #298)
- xxxxxxxxxxx v0.7 설계 결정 및 6월 실행 계획 문서화 (`e1a7b5c`)
- v0.7 구현 이슈 번호 반영 (#300~#303) (`de7246b`)
- 런타임 전략 재정비 문서 — Core 3종 집중 (`281164c`)
- 전체 문서 최신화 — v0.7 FP 선행 완료 항목 반영 (`59f4118`)
- 모델 배포 전략(A/B 테스트, 카나리 배포) 문서 보강 (`dbc61ec`)

---

## 이전 Iteration 계획 달성도

Iteration 24에서 계획한 4개 항목 중 **1개 부분 달성, 3개 미착수** — xxxxxxxxxxx v0.7 통합을 일정에 맞춰 전면 착수하면서 기존 계획을 조정:

| 계획 | 이슈 | 상태 | 비고 |
|------|------|------|------|
| 멀티테넌시 안정화 및 7월 데모 시나리오 검증 | #289 | 부분 달성 | Frontend 마무리(#289) + Popover UX(#314) 완료 |
| 추론 프록시 Phase 2: 멀티클러스터 라우팅 | #192 | 미착수 | xxxxxxxxxxx 통합 우선 |
| hmas-agent 핵심 컴포넌트 구현 착수 | #190 | 미착수 | 동일 사유 |
| JWT 사용자별 커스텀 Rate Limit | #267 | 미착수 | 동일 사유 |

**대신 달성**: xxxxxxxxxxx 전 계층 통합(#216, #217, #218, #300~#303), 멀티테넌시 Frontend 완성(#289, #314), 런타임 정비(#298), 추론 로그 개선(#320), 감사 로그 버그 수정(#318), schedulerType 리팩토링(#307), 스케줄러 어노테이션 장기 과제 완료(#66) — 총 14건 해결.

---

## 미해결 이슈 (Open Issues)

### xxxxxxxxxxx 후속 이슈

| 이슈 | 제목 | 상태 |
|------|------|------|
| #216 | xxxxxxxxxxx 실 클러스터 설치 및 환경 호환성 검증 | **Iteration 25 완료** |
| #217 | xxxxxxxxxxx QoS 클래스 및 리소스 오버커밋 동작 검증 | **Iteration 25 완료** |
| #218 | L3 디바이스 검증 — GPU 토폴로지 인식 확인 | **Iteration 25 완료** |
| #300 | 클러스터 스케줄러 타입 변경 API 구현 | **Iteration 25 완료** |
| #301 | 배포 시 xxxxxxxxxxx OverridePolicy 자동 생성/삭제 | **Iteration 25 완료** |
| #302 | 배포 위자드 QoS 클래스 선택 | **Iteration 25 완료** |
| #303 | Standard + GP 혼재 환경 E2E 통합 테스트 | **Iteration 25 완료** |
| #308 | 멀티 GPU 추론 지원 — Gang Scheduling, GPU 토폴로지 실검증 (v0.9) | 미착수 |

### xxxxxxx 상태 관리 (신규)

| 이슈 | 제목 | 상태 |
|------|------|------|
| #316 | xxxxxxx 고아 Work 객체 감지 및 자동 정리 (Orphan Work GC) | 미착수 |
| #317 | 프론트엔드: xxxxxxx 고아 리소스 감지 알림 및 정리 이력 UI | 미착수 (선행: #316) |

### 멀티테넌시 후속 이슈 체인 (Phase 2b / Phase 3)

| 이슈 | 제목 | 상태 |
|------|------|------|
| #290 | Phase 2b-1: GPU 쿼터 혼합 모드 (Dedicated + Org 공유 풀) | 미착수 |
| #291 | Phase 2b-2: Org 프리셋 분리 + Cross-Group 가시성 + Group 삭제 전략 | 미착수 |
| #292 | Phase 3-Early: Org별 클러스터 격리 + NetworkPolicy + PP org-access | 미착수 |
| #293 | Phase 3-Late: 미터링/과금 파이프라인 (Prometheus + PostgreSQL) | 미착수 |

### 인증/인가 후속 이슈 체인

| 이슈 | 제목 | 상태 |
|------|------|------|
| #267 | JWT 사용자별 커스텀 Rate Limit | 미착수 |
| #270 | TPM 토큰 사전 추정 리서치 | 미착수 |

### 추론 프록시 이슈 체인

| 이슈 | 제목 | 상태 |
|------|------|------|
| #192 | Phase 2: 멀티클러스터 라우팅 (agent 터널 통합) | 미착수 |

---

## 이번 주 통계

| 항목 | 수치 |
|------|------|
| 총 커밋 수 | 22개 |
| 머지된 PR | 13개 (#297, #299, #304, #305, #306, #309, #310, #311, #312, #313, #315, #319, #321) |
| 생성된 이슈 | 12개 (#298, #300, #301, #302, #303, #307, #308, #314, #316, #317, #318, #320) |
| 해결된 이슈 | 14개 (#66, #216, #217, #218, #289, #298, #300, #301, #302, #303, #307, #314, #318, #320) |
| 미해결 이슈 | 29개 |
| 신규 기술 문서 | 7건 |
| 코드 변경량 | +8,398줄 / -1,557줄 |

---

## 결론

이번 주는 **xxxxxxxxxxx v0.7 통합을 L1 설치부터 E2E 테스트까지 전 계층을 1주 만에 구현 완료하고, 멀티테넌시 Frontend를 마무리하며, 런타임 전략을 Core 3종으로 정비한 기간**이었습니다. xxxxxxxxxxx 통합(#216→#303)으로 GP 클러스터에서 QoS 기반 서빙이 가능해졌으며, 7월 데모에서 Standard + GP 클러스터 혼재 환경의 차별화된 스케줄링을 시연할 수 있게 되었습니다. 멀티테넌시 Frontend(#289, #314)로 Phase 2a UI를 완전히 마무리하고 모던 UX 패턴을 적용했으며, Ollama/TGI 제거(#298)로 Core 런타임에 집중하는 전략을 확정했습니다. 장기 과제인 #66(스케줄러 어노테이션 변환)도 xxxxxxxxxxx 통합을 통해 최종 해결되었습니다.

**핵심 성과**:
1. **xxxxxxxxxxx v0.7 전 계층 통합 완료**: 설치(#216) → QoS 검증(#217) → 디바이스 검증(#218) → API(#300) → OverridePolicy(#301) → Frontend(#302) → E2E(#303) — 7개 이슈를 의존 순서대로 연속 구현하여 GP 클러스터 서빙 달성
2. **멀티테넌시 Frontend 완성**: Phase 2-4 UI(#289) + Popover 스위처(#314) — 7월 데모 대비 완전한 멀티테넌시 UI 확보
3. **런타임 전략 정비**: Ollama/TGI 제거(#298) — Core 3종(vLLM, llama.cpp, TEI) 집중 전략 확정

**다음 주 계획**:
- xxxxxxx 고아 Work 자동 정리 구현 (#316)
- 추론 프록시 Phase 2: 멀티클러스터 라우팅 (#192)
- hmas-agent 핵심 컴포넌트 구현 착수 (#190)
- JWT 사용자별 커스텀 Rate Limit (#267)

---

**문서 작성일**: 2026년 6월 7일
