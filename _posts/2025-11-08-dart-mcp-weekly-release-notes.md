---
title: 'dart-mcp-weekly-release-notes-2025-11-02-11-08'
date: 2025-11-08
permalink: /posts/2025/11/dart-mcp-weekly-release-notes-11-08/
tags:
  - DART-MCP
  - MCP
  - Agent
---

# OpenDart MCP Server 릴리즈 노트
## MCP 서버 초기 구현 및 테스트 환경 구축 - 2025년 11월 2일 ~ 11월 8일

### 주요 개발 내용 요약

OpenDart MCP Server 프로젝트는 **한국 기업의 공시정보를 Claude Desktop에서 자연어로 조회할 수 있도록 하는 Model Context Protocol (MCP) 서버 초기 구현**에 집중하였습니다. 11월 2일부터 11월 8일까지 총 **2개의 커밋**이 진행되었으며, **OpenDart API 통합**, **FastMCP 프레임워크 기반 MCP 서버 구현**, **MCP Inspector 테스트 환경 구축**이 완성되었습니다.

---

## 새로운 기능 (New Features)

### 1. OpenDart MCP 서버 초기 구현
- **구현 완료**: FastMCP 프레임워크 기반 MCP 서버 구축
- **OpenDart API 클라이언트 구현**:
  - REST API 기반 OpenDart 공시정보 조회 클라이언트
  - 기업 코드 목록 로컬 캐싱 메커니즘 (`corp_code_cache.json`)
  - 캐시를 통한 반복 API 호출 최소화
  - 환경 변수를 통한 API 키 관리 (`OPENDART_API_KEY`)
- **4개 MCP 도구 구현**:
  - `search_company`: 기업명으로 고유번호 및 종목코드 검색
  - `get_disclosure_list`: 기간별/기업별 공시정보 목록 조회
  - `get_disclosure_document`: 특정 공시의 문서 뷰어 URL 제공
  - `get_company_info`: 기업 개황 정보 조회 (기업명, 대표자, 업종, 주소, 홈페이지 등)
- **공시 유형 지원**:
  - A=정기공시, B=주요사항보고, C=발행공시, D=지분공시, E=기타공시
  - F=외부감사관련, G=펀드공시, H=자산유동화, I=거래소공시, J=공정위공시
- **프로젝트 구성**:
  - Python 3.11+ 기반 프로젝트 설정 (`pyproject.toml`)
  - uv 패키지 매니저 지원
  - 의존성 관리: `fastmcp`, `requests`, `python-dotenv`
  - Claude Desktop 통합 가이드 (`claude_desktop_config.json` 설정)
- **주요 커밋**: `04e79e5`

### 2. MCP Inspector 테스트 환경 구축
- **구현 완료**: 웹 기반 인터랙티브 디버깅 환경
- **MCP Inspector 통합**:
  - 브라우저 기반 MCP 서버 테스트 도구
  - 실시간 API 응답 확인
  - 4개 MCP 도구에 대한 인터랙티브 파라미터 입력 인터페이스
- **테스트 가이드 문서화**:
  - `.env` 파일을 통한 API 키 설정 가이드
  - `npx @modelcontextprotocol/inspector` 실행 방법
  - 단계별 테스트 예시 제공 (삼성전자 기업 검색, 2024년 공시 목록 조회)
  - 브라우저에서 4개 도구 확인 및 실행 방법
- **개발 모드 실행**:
  - `fastmcp dev` 자동 재시작 지원
  - 코드 변경 시 실시간 반영
  - 개발/테스트 환경 분리 (`.env` vs `claude_desktop_config.json`)
- **uv.lock 파일 추가**:
  - 1,454줄의 종속성 잠금 파일
  - 재현 가능한 빌드 보장
- **주요 커밋**: `0dcad08`

---

## 주요 개선사항 (Improvements)

### 1. Claude Desktop 통합 설정
- **설정 파일 위치 가이드**:
  - macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
  - Windows: `%APPDATA%\Claude\claude_desktop_config.json`
- **환경 변수 설정**:
  - MCP 서버별 독립적인 환경 변수 지원
  - `OPENDART_API_KEY` 보안 관리
- **Python 가상환경 경로 설정**:
  - `which python` / `where python` 명령어로 경로 확인
  - Claude Desktop 재시작으로 설정 적용

### 2. 캐싱 메커니즘
- **기업 코드 목록 로컬 캐싱**:
  - `corp_code_cache.json` 파일로 캐시 저장
  - 반복적인 API 호출 최소화
  - 네트워크 트래픽 및 API 할당량 절약
- **캐시 무효화 방법 제공**:
  - `rm corp_code_cache.json` 명령어로 캐시 삭제

### 3. 에러 핸들링 및 사용자 경험
- **친절한 오류 메시지**:
  - API 키 미설정 시 명확한 안내 메시지
  - 검색 결과 없을 때 안내 메시지
- **결과 제한 및 포맷팅**:
  - 기업 검색 결과 최대 20개 표시
  - 초과 시 더 구체적인 검색어 요청
  - 보기 좋은 텍스트 포맷팅

### 4. 문서화
- **종합 README 작성**:
  - 기능 소개 (4개 아이콘과 함께)
  - 요구사항 및 설치 방법 (uv, pip 모두 지원)
  - OpenDart API 키 발급 가이드
  - Claude Desktop 설정 방법
  - MCP Inspector 테스트 가이드
  - 사용 예시 (자연어 쿼리)
  - 문제 해결 섹션
  - 참고 자료 링크

---

## 성과 지표 (Metrics)

- **총 커밋 수**: 2개 커밋
- **병합된 PR**: 0개 (직접 main 브랜치 커밋)
- **새로운 기능**: 2개 주요 기능 (OpenDart MCP 서버, MCP Inspector 테스트 환경)
- **구현된 MCP 도구**: 4개 (기업 검색, 공시 목록, 공시 문서, 기업 정보)
- **문서화**: 2개 주요 문서 (README, 인라인 코드 문서)
- **지원 공시 유형**: 10개 (정기공시 ~ 공정위공시)
- **캐싱 메커니즘**: 1개 (기업 코드 목록)

### 주요 커밋 목록
- **04e79e5**: OpenDart MCP 서버 초기 구현
- **0dcad08**: MCP Inspector 테스트 가이드 추가

### 핵심 개선 영역
- **MCP 생태계 통합**: FastMCP 프레임워크 기반 Claude Desktop 연동
- **OpenDart API 활용**: 한국 기업 공시정보 자연어 조회 가능
- **개발자 경험**: MCP Inspector 브라우저 기반 테스트 환경
- **문서화**: 종합 가이드 및 문제 해결 섹션

---

## 기술 스택 (Tech Stack)

### Backend
- **Python 3.11+**: 최신 Python 기능 활용
- **FastMCP**: Model Context Protocol 서버 프레임워크
- **Requests**: OpenDart REST API 클라이언트
- **python-dotenv**: 환경 변수 관리

### 개발 도구
- **uv**: 빠른 Python 패키지 매니저
- **MCP Inspector**: 웹 기반 MCP 디버깅 도구
- **setuptools**: Python 패키지 빌드 시스템

### API
- **OpenDart API**: 금융감독원 전자공시시스템 공식 API
- **Model Context Protocol (MCP)**: Anthropic LLM 통합 프로토콜

---

## 향후 계획 (Future Plans)

### 1. 기능 확장
- 추가 OpenDart API 엔드포인트 통합
  - 재무제표 조회 (`fnlttSinglAcntAll`)
  - 지분 변동 현황 조회
  - 배당금 정보 조회
  - 임원 현황 조회
- 고급 검색 필터링 옵션 추가
- 공시 문서 본문 텍스트 추출 및 요약

### 2. 성능 최적화
- Redis 기반 분산 캐싱 시스템
- API 응답 캐싱 레이어 추가
- 비동기 API 호출 (`aiohttp`)
- Rate limiting 구현

### 3. 보안 강화
- API 키 암호화 저장
- Rate limiting 및 API 할당량 관리
- 입력 검증 및 sanitization 강화
- 에러 로깅 및 모니터링

### 4. 테스트 및 품질
- Unit test 작성 (`pytest`)
- Integration test 추가
- Type checking (`mypy`) 설정
- Code formatting (`black`) 자동화
- CI/CD 파이프라인 구축

### 5. 문서화 개선
- API 레퍼런스 문서 생성
- 사용 사례 및 튜토리얼 추가
- 동영상 데모 제작
- 다국어 지원 (영문 README)

### 6. Claude Desktop 사용자 경험
- 더 자연스러운 한국어 응답 포맷팅
- 공시 문서 요약 기능
- 기업 비교 분석 기능
- 시각화 데이터 제공 (차트, 그래프)

---

## 결론

OpenDart MCP Server 프로젝트는 이번 주 **초기 MCP 서버 구현 및 테스트 환경 구축**을 통해 프로토타입을 완성하였습니다. **FastMCP 프레임워크 기반 4개 MCP 도구 구현**, **OpenDart API 통합 및 캐싱 메커니즘**, **MCP Inspector 테스트 환경 구축**을 통해 Claude Desktop에서 한국 기업의 공시정보를 자연어로 조회할 수 있는 기반을 마련하였습니다.

특히 **FastMCP 프레임워크 활용**은 Model Context Protocol 서버 개발을 단순화하고, **캐싱 메커니즘**은 API 호출 최적화를 통해 사용자 경험을 개선하였습니다. **MCP Inspector 통합**을 통해 개발자가 브라우저에서 실시간으로 MCP 도구를 테스트할 수 있는 환경을 제공하였습니다.

또한 **종합 문서화**를 통해 설치부터 설정, 테스트, 문제 해결까지 모든 과정을 상세히 안내하여 다른 개발자들이 쉽게 프로젝트를 시작할 수 있도록 하였습니다.

앞으로 추가 API 엔드포인트 통합, 성능 최적화, 보안 강화, 테스트 코드 작성, 고급 분석 기능 추가를 통해 더욱 강력하고 유용한 OpenDart MCP 서버로 발전할 예정입니다.

---

**문서 작성일**: 2025년 11월 8일
**버전**: v0.1.0 - Initial MCP Server Implementation
