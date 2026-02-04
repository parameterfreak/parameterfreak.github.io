---
title: 'dataproc-weekly-release-notes-2025-11-09-11-15'
date: 2025-11-16
permalink: /posts/2025/11/weekly-release-notes-2025-11-09-11-15/
categories:
  - DataProc
tags:
  - DataProc
  - RAG
  - LLM
---

# DataProc 주간 릴리즈 노트
## 데이터셋 관리 시스템 전면 구현 - 2025년 11월 9일 ~ 11월 15일

### 주요 개발 내용 요약

DataProc 프로젝트는 **데이터셋 레지스트리 및 관리 시스템의 전면 구축**에 집중하였습니다. 11월 9일부터 11월 15일까지 총 **13개의 PR**이 병합되었으며, **데이터셋 CRUD 시스템**, **파일 업로드 및 버전 관리**, **검색 및 필터링 기능**, **코드 품질 자동화**가 완성되었습니다. 이번 주는 백엔드 API부터 프론트엔드 UI까지 전체 스택을 아우르는 대규모 구현이 진행되었습니다.

---

## 새로운 기능 (New Features)

### 1. 데이터셋 기본 모델 및 CRUD API 구현 (#75, #76, #77)
- **데이터베이스 모델 설계 및 구현**:
  - `Dataset` 모델 생성 (models/dataset.py)
  - 필수 필드: id, user_id, name, version, dataset_type, description, language, size, license, tags, file_format, split_info
  - User와의 관계 설정 (back_populates)
  - Unique constraint: (user_id, name, version) - 사용자별 이름/버전 중복 방지
  - Alembic 마이그레이션 생성 및 적용
- **Pydantic 스키마 정의**:
  - DatasetBase, DatasetCreate, DatasetUpdate, DatasetResponse 스키마
- **CRUD API 엔드포인트 (3개)**:
  - `POST /dataset/`: 데이터셋 생성
  - `GET /dataset/`: 현재 사용자 데이터셋 목록 조회 (생성일 내림차순)
  - `GET /dataset/{dataset_id}`: 단일 데이터셋 조회 (권한 검증)
  - `PUT /dataset/{dataset_id}`: 데이터셋 부분 수정 (`exclude_unset=True`)
  - `DELETE /dataset/{dataset_id}`: 데이터셋 삭제
- **권한 관리**:
  - JWT 인증 기반 사용자 검증
  - 본인 소유 데이터셋만 접근/수정/삭제 가능
  - 404/401 에러 처리
- **주요 커밋**: PR #75, #76, #77

### 2. 데이터셋 파일 업로드 및 관리 시스템 (#81, #82)
- **Backend 구현**:
  - `DatasetFile` 모델 생성 (models/dataset_file.py)
    - 파일 메타데이터 저장: 파일명, 경로, 크기, MIME 타입
    - Dataset과 일대다 관계 (`cascade="all, delete-orphan"`)
  - Alembic 마이그레이션 (dataset_files 테이블 생성)
- **API 엔드포인트 (3개)**:
  - `POST /dataset/{dataset_id}/files/upload`: 파일 업로드 (UUID 기반 파일명, 충돌 방지)
  - `GET /dataset/{dataset_id}/files`: 파일 목록 조회 (생성일 내림차순)
  - `DELETE /dataset/files/{file_id}`: 개별 파일 삭제 (디스크+DB)
  - `DELETE /dataset/{dataset_id}`: 데이터셋 삭제 시 연관 파일 및 디렉토리 전체 삭제
- **파일 저장 구조**:
  - `uploads/datasets/{dataset_id}/` 디렉토리에 파일 저장
  - UUID 기반 파일명으로 충돌 방지
  - 트랜잭션 안전성: 업로드 실패 시 자동 롤백 및 파일 정리
- **Frontend 구현**:
  - `DatasetFileUploadModal` 컴포넌트
    - 파일 선택 input 및 미리보기 (이름, 크기)
    - 업로드 진행 상태 표시 (로딩 스피너)
    - 에러 핸들링 및 성공 시 목록 자동 갱신
  - `DatasetFileList` 컴포넌트
    - 파일 목록 표시 (파일명, 크기, 업로드일)
    - 파일별 삭제 버튼 (확인 UI 포함)
    - 삭제 중 로딩 상태
  - `DatasetDetailModal`에 "파일 관리" 섹션 통합
  - `formatFileSize()` 유틸리티 (B/KB/MB 포맷팅)
- **주요 커밋**: PR #81, #82

### 3. 데이터셋 검색 및 필터링 시스템 (#83, #84)
- **Backend 검색 API**:
  - `GET /dataset/search` 엔드포인트 구현
  - `PaginatedDatasetResponse` 스키마 (items, total, page, page_size, total_pages)
  - `paginate_query()` 헬퍼 함수 (재사용 가능한 페이징)
- **검색 파라미터 (8개)**:
  - `q`: 검색어 (name, description에서 ILIKE 검색)
  - `dataset_type`: 데이터셋 유형 필터 (정확 일치)
  - `language`: 언어 필터 (정확 일치)
  - `tags`: 태그 필터 (쉼표 구분, JSON 배열 포함 검사)
  - `sort_by`: 정렬 필드 (created_at, name, version)
  - `sort_order`: 정렬 순서 (asc, desc)
  - `page`: 페이지 번호 (기본값: 1)
  - `page_size`: 페이지 크기 (기본값: 20, 최대: 100)
- **태그 필터링 구현**:
  - PostgreSQL JSON 타입 연산 활용
  - `cast(Dataset.tags, String).contains()` 방식
  - 여러 태그 동시 필터링 지원 (AND 조건)
- **Frontend 검색 UI**:
  - `DatasetSearchParams` 타입 정의
  - `datasetService.searchDatasets()` 메서드 추가
  - 검색 입력 필드 (데이터셋 이름/설명)
  - 유형 드롭다운 (텍스트 분류, NER, QA, 번역, 요약, 커스텀)
  - 언어 드롭다운 (한국어, 영어, 일본어, 중국어, 다국어)
  - 태그 입력 필드 (쉼표 구분)
  - 검색/초기화 버튼
- **페이징 UI**:
  - 총 데이터셋 개수 표시
  - 현재 페이지 / 전체 페이지 표시
  - 이전/다음 페이지 네비게이션
  - 페이지 변경 시 자동 검색 (useEffect)
- **주요 커밋**: PR #83, #84

### 4. 데이터셋 출처 URL 필드 추가 (#86)
- **Backend**:
  - `Dataset` 모델에 `source_url` 필드 추가 (String(500), nullable=True)
  - `DatasetBase`/`DatasetUpdate` 스키마에 URL 검증 로직 (http:// 또는 https://)
  - `create_dataset` 엔드포인트에 source_url 저장 로직
  - Alembic 마이그레이션
- **Frontend**:
  - TypeScript 타입 정의에 `source_url` 추가
  - 데이터셋 생성/수정 폼에 URL 입력 필드 (HTML5 URL 검증)
  - 데이터셋 상세 정보 모달에 출처 URL 링크 표시 (외부 링크 아이콘)
  - 데이터셋 목록에 출처 URL 표시 (외부 링크 아이콘)
- **효과**:
  - 데이터의 신뢰성 및 추적성 향상
  - 새 탭에서 출처 확인 가능 (target="_blank", rel="noopener noreferrer")
- **주요 커밋**: PR #86

### 5. 데이터셋 파일 버전 관리 시스템 (Backend) (#89)
- **데이터베이스 스키마**:
  - `DatasetVersion` 모델 추가: 버전 메타데이터 관리
  - `DatasetFile` 모델 확장:
    - `version_id`: 버전 참조 (NULL이면 staging)
    - `file_hash`: SHA256 해시로 파일 무결성 검증
    - `is_staged`: 임시 영역 여부 플래그
  - Alembic 마이그레이션으로 기존 데이터 자동 v1 변환
- **핵심 기능 (7개)**:
  1. **Staging 영역**: 파일을 임시로 추가/삭제하며 자유롭게 작업
  2. **버전 생성**: 임시 영역의 파일을 새 버전으로 확정
  3. **버전 활성화**: 특정 버전을 활성 버전으로 지정
  4. **버전 삭제**: 불필요한 버전 전체 삭제 (마지막 버전 제외)
  5. **버전 비교**: 두 버전 간 파일 변경사항 확인
  6. **파일 무결성**: SHA256 해시로 파일 검증 및 중복 방지
  7. **저장 공간 최적화**: 하드링크로 동일 파일 중복 저장 방지
- **API 엔드포인트 (8개)**:
  - `POST /dataset/{id}/versions/create`: 버전 생성
  - `GET /dataset/{id}/versions`: 버전 목록
  - `GET /dataset/{id}/versions/{version_id}`: 버전 상세
  - `GET /dataset/{id}/versions/{version_id}/files`: 버전 파일 목록
  - `GET /dataset/{id}/staged-files`: 임시 파일 목록
  - `POST /dataset/{id}/versions/{version_id}/activate`: 버전 활성화
  - `DELETE /dataset/{id}/versions/{version_id}`: 버전 삭제
  - `GET /dataset/{id}/versions/compare/{v1}/{v2}`: 버전 비교
- **디렉토리 구조**:
  ```
  uploads/datasets/{dataset_id}/
  ├── staging/          # 임시 영역 (버전 미확정)
  │   ├── {uuid}.csv
  │   └── {uuid}.json
  ├── v1/               # 버전 1
  │   ├── {uuid}.csv
  │   └── {uuid}.json
  └── v2/               # 버전 2
      ├── {uuid}.json
      └── {uuid}.txt
  ```
- **주요 커밋**: PR #89

### 6. pre-commit 도입으로 코드 품질 자동화 (#74)
- **pre-commit 프레임워크 설정**:
  - `.pre-commit-config.yaml` 생성
  - 커밋 전 자동 코드 품질 검사
- **Backend (Python) Hooks**:
  - Black: 코드 자동 포맷팅 (PEP 8)
  - isort: import 문 자동 정렬
  - Ruff: 빠른 Python 린터 (Flake8 대체)
- **Frontend (TypeScript/React) Hooks**:
  - Prettier: 코드 자동 포맷팅
- **공통 Hooks**:
  - 파일 검사 (trailing-whitespace, end-of-file-fixer, mixed-line-ending)
  - YAML/JSON 검증 (check-yaml, check-json, check-toml)
  - 대용량 파일 방지 (1MB 제한)
  - 민감 정보 감지 (detect-private-key, detect-aws-credentials)
  - 커밋 메시지 검증 (Conventional Commits)
  - YAML 포맷팅 (pretty-format-yaml)
- **문서화**:
  - `docs/pre-commit-guide.md`: 상세 사용 가이드 (한국어)
  - `README.md`: Pre-commit 설정 방법 추가
- **코드베이스 전체 적용**:
  - 139개 파일에 자동 포맷팅 적용
  - trailing whitespace 및 EOF 문제 수정
  - Python 코드 스타일 통일
- **효과**:
  - 커밋 전 로컬에서 즉각적인 코드 품질 피드백
  - 팀원 간 일관된 코드 스타일 유지
  - 민감한 정보 및 실수 방지
  - 수동 검토 부담 감소
- **주요 커밋**: PR #74

---

## 주요 개선사항 (Improvements)

### 1. 데이터셋 서비스 레이어 및 UI 구현 (#78, #79, #80)
- **서비스 레이어 구축**:
  - `frontend/src/services/datasetService.ts` 생성
  - `getDatasets()`: 데이터셋 목록 조회
  - `getDataset(id)`: 단일 데이터셋 조회
  - `createDataset()`: 새 데이터셋 생성
  - `updateDataset()`: 데이터셋 수정
  - `deleteDataset()`: 데이터셋 삭제
  - 인증 헤더 자동 추가 및 401 에러 시 로그인 리다이렉트
- **Workspace 탭 추가**:
  - "datasets" 탭 추가 (프로젝트 관리 탭 다음)
  - activeTab 타입 정의에 "datasets" 추가
- **데이터셋 목록 테이블**:
  - 표시 항목: 이름(설명 포함), 버전, 유형, 크기, 언어, 생성일
  - 로딩 상태 및 에러 메시지 표시
  - 데이터 없을 때 안내 메시지
  - 행 클릭 시 상세 모달 표시
- **DatasetCreateModal 컴포넌트**:
  - 새 데이터셋 생성 폼
  - 필수 필드: name, version, dataset_type
  - 선택 필드: description, language, size, license, tags, file_format, split_info
  - JSON 입력 유효성 검사 (tags, split_info)
  - 폼 유효성 검사 및 에러 핸들링
- **DatasetDetailModal 컴포넌트**:
  - 데이터셋 상세 정보 보기/수정/삭제 기능
  - 읽기 전용 모드와 편집 모드 전환
  - JSON 필드 (tags, split_info) 포맷팅 및 syntax highlighting
  - 생성일/수정일 한국 시간 형식으로 표시 (`toLocaleString("ko-KR")`)
  - 삭제 확인 모달 포함 (danger 스타일)
  - "편집" / "삭제" / "닫기" 버튼
- **TypeScript 타입 개선**:
  - any 타입을 구체적인 타입으로 변경
  - tags: `Record<string, string | number | boolean>`
  - split_info: `Record<string, string | number>`
- **UX 개선**:
  - 생성/수정/삭제 후 목록 자동 새로고침
  - 작업 중 로딩 상태 표시 ("처리 중...")
  - 에러 처리 (alert로 사용자 알림)
  - 버튼 비활성화 (로딩 중)
- **주요 커밋**: PR #78, #79, #80

### 2. TypeScript 타입 시스템 강화
- `frontend/src/types/dataset.ts` 생성
- Dataset, DatasetCreate, DatasetFile, DatasetSearchParams, PaginatedDatasets 인터페이스 정의
- 타입 안전성 향상 및 IDE 자동완성 지원

### 3. 파일 유틸리티 추가
- `frontend/src/utils/fileUtils.ts` 생성
- `formatFileSize()` 함수: 파일 크기를 B/KB/MB 단위로 포맷팅
- 재사용 가능한 유틸리티 함수 구조

### 4. API 테스트 스크립트 추가
- `scripts/tests/test_dataset_search.py`: 검색/필터링/페이징 테스트
- API 엔드포인트 자동 테스트로 품질 보증

---

## 성과 지표 (Metrics)

- **총 PR 수**: 13개 (11/9 ~ 11/15)
- **병합된 PR**: 13개 모두 병합 완료
- **새로운 기능**: 6개 주요 기능
  1. 데이터셋 CRUD 시스템
  2. 파일 업로드 및 관리
  3. 검색 및 필터링 시스템
  4. 출처 URL 추적
  5. 파일 버전 관리 (백엔드)
  6. pre-commit 자동화
- **주요 개선사항**: 4개 시스템 최적화
  1. 서비스 레이어 구축
  2. TypeScript 타입 시스템 강화
  3. 파일 유틸리티 추가
  4. API 테스트 스크립트
- **생성된 DB 테이블**: 3개 (datasets, dataset_files, dataset_versions)
- **구현된 API 엔드포인트**: 23개
  - 데이터셋 CRUD: 5개
  - 파일 관리: 3개
  - 검색/필터링: 2개
  - 버전 관리: 8개
  - 출처 URL: 1개 (기존 엔드포인트 확장)
- **프론트엔드 컴포넌트**: 5개
  - DatasetCreateModal, DatasetDetailModal, DatasetFileUploadModal, DatasetFileList, 검색 UI
- **문서화**: 2개 주요 문서 작성
  - pre-commit 가이드
  - README.md 업데이트

### 주요 PR 목록 (병합 순서)
1. **#74**: pre-commit 도입으로 코드 품질 자동화 개선 (11/12)
2. **#75**: 데이터셋 모델 및 마이그레이션 추가 (11/13)
3. **#76**: 데이터셋 기본 CRUD API 구현 (11/14)
4. **#77**: 데이터셋 수정/삭제 API 엔드포인트 구현 (11/14)
5. **#78**: 데이터셋 서비스 레이어 및 목록 UI 구현 (11/14)
6. **#79**: 데이터셋 생성/수정 UI 구현 (11/14)
7. **#80**: 데이터셋 삭제 및 상세보기 개선 (11/14)
8. **#81**: 데이터셋 파일 업로드 및 관리 API 구현 (11/15)
9. **#82**: 데이터셋 파일 업로드 UI 구현 (11/15)
10. **#83**: 데이터셋 검색 및 필터링 API 구현 (11/15)
11. **#84**: 데이터셋 검색 및 필터 UI 구현 (11/15)
12. **#86**: 데이터셋 등록 시 출처 URL 필드 추가 (11/15)
13. **#89**: 데이터셋 파일 버전 관리 시스템 구현 (백엔드) (11/15)

### 핵심 개선 영역
- **데이터셋 관리 시스템**: 완전한 CRUD, 파일 관리, 버전 관리
- **검색 및 필터링**: 고급 검색, 페이징, 태그 필터링, 정렬
- **사용자 경험**: 직관적인 UI, 로딩 상태, 에러 핸들링, 삭제 확인
- **코드 품질**: pre-commit 자동화, 일관된 스타일, 타입 안전성
- **권한 관리**: JWT 인증, 사용자별 데이터 격리, 권한 검증
- **성능 최적화**: 페이징, 하드링크, 트랜잭션 안전성

### 수정 파일 통계
- **Backend**: 11개 파일
  - Models: 3개 (dataset.py, dataset_file.py, user.py)
  - Routers: 1개 (dataset.py)
  - Schemas: 1개 (dataset.py)
  - Migrations: 5개 (Alembic 마이그레이션)
  - Main: 1개 (main.py)
- **Frontend**: 8개 파일
  - Components: 4개 (DatasetCreateModal, DatasetDetailModal, DatasetFileUploadModal, DatasetFileList)
  - Services: 1개 (datasetService.ts)
  - Types: 1개 (dataset.ts)
  - Utils: 1개 (fileUtils.ts)
  - Pages: 1개 (workspace/page.tsx)
- **Configuration**: 6개 파일
  - .pre-commit-config.yaml
  - backend/pyproject.toml
  - backend/requirements-dev.txt
  - frontend/.prettierrc
  - frontend/.prettierignore
  - .markdownlint.yaml
- **Documentation**: 2개 파일
  - docs/pre-commit-guide.md
  - README.md

---

## 향후 계획 (Future Plans)

### 1. 데이터셋 버전 관리 UI 구현 (Issue #88)
- 버전 히스토리 타임라인 UI
- 버전 생성 모달
- 버전 비교 뷰 (파일 변경사항 시각화)
- Staging 영역 표시 및 관리 UI
- 버전 활성화/삭제 UI

### 2. 데이터셋 파일 고급 기능
- 파일 다운로드 기능 (개별 및 일괄)
- 파일 미리보기 (CSV, JSON, TXT 등)
- 파일 업로드 진행률 표시 (대용량 파일)
- 드래그 앤 드롭 업로드
- 파일 유효성 검사 (형식, 크기 제한)

### 3. 데이터셋 협업 및 공유
- 데이터셋 공개/비공개 설정
- 팀원과의 데이터셋 공유
- 데이터셋 복제 (Fork) 기능
- 데이터셋 즐겨찾기 및 북마크
- 데이터셋 사용 통계 (조회수, 다운로드 수)

### 4. 고급 검색 및 추천
- 전문 검색 (Full-text search) 엔진 도입 (Elasticsearch)
- 유사 데이터셋 추천
- 태그 자동 추천 (AI 기반)
- 검색 결과 하이라이팅
- 최근 검색 이력 및 저장된 검색

### 5. 데이터셋 품질 관리
- 데이터셋 검증 파이프라인
- 통계 정보 자동 생성 (행 수, 컬럼 수, 결측치 등)
- 데이터 프로파일링 (분포, 이상치 감지)
- 데이터셋 품질 점수
- 레이블 품질 검증

### 6. 멀티모달 데이터셋 지원
- 이미지 데이터셋 업로드 및 관리
- 오디오 데이터셋 지원
- 비디오 데이터셋 지원
- 멀티모달 데이터셋 (텍스트+이미지+오디오)
- 데이터 증강 (Augmentation) 기능

### 7. 데이터셋 워크플로우 통합
- 데이터셋 → 프로젝트 연결
- 데이터셋 → 모델 학습 파이프라인 연동
- 데이터셋 버전 → 실험 추적 (Experiment tracking)
- 데이터셋 → RAG 시스템 연동
- 데이터셋 → 평가 벤치마크 통합

### 8. 성능 최적화
- 대용량 파일 청크 업로드
- 파일 압축 및 압축 해제
- S3/MinIO 등 객체 스토리지 연동
- 파일 캐싱 전략
- 데이터셋 로딩 병렬화

---

## 결론

DataProc 프로젝트는 이번 주 **데이터셋 관리 시스템의 전면 구축**을 통해 AI/ML 워크플로우의 핵심 인프라를 완성했습니다. 총 **13개의 PR**을 병합하며 **데이터셋 CRUD**, **파일 업로드 및 버전 관리**, **고급 검색 및 필터링**, **코드 품질 자동화**를 구현했습니다.

**데이터셋 레지스트리 시스템**은 User 인증 기반의 권한 관리, Unique constraint를 통한 중복 방지, 그리고 CASCADE 삭제로 데이터 무결성을 보장합니다. **파일 관리 시스템**은 UUID 기반 파일명, 트랜잭션 안전성, SHA256 해시 검증을 통해 안정적인 파일 저장을 제공합니다.

**버전 관리 시스템**은 Staging 영역, 버전 생성/활성화/삭제/비교 기능, 하드링크 기반 저장 공간 최적화를 통해 데이터셋의 변화를 체계적으로 추적합니다. **검색 및 필터링 시스템**은 검색어, 유형, 언어, 태그 기반 필터링, 정렬, 페이징을 지원하며, PostgreSQL JSON 타입 연산을 활용한 고급 태그 검색을 제공합니다.

**pre-commit 자동화** 도입으로 Black, isort, Ruff, Prettier, Conventional Commits 검증이 커밋 전 자동 실행되어 코드 품질과 팀 협업 효율성이 크게 향상되었습니다. 139개 파일에 대한 자동 포맷팅으로 코드베이스 전체의 일관성을 확보했습니다.

**프론트엔드 UI**는 직관적인 모달 컴포넌트(생성/수정/삭제/파일 업로드), 반응형 테이블, 검색/필터 UI, 페이징 네비게이션을 제공하며, TypeScript 타입 시스템으로 타입 안전성을 보장합니다. 사용자 경험 측면에서 로딩 상태, 에러 핸들링, 삭제 확인 모달, 자동 새로고침 등의 세심한 배려가 돋보입니다.

이번 개선사항은 DataProc을 단순한 문서 처리 도구에서 **AI/ML 데이터셋 레지스트리 플랫폼**으로 진화시켰으며, 향후 버전 관리 UI, 협업 기능, 고급 검색, 데이터 품질 관리, 멀티모달 지원을 통해 더욱 강력한 데이터 관리 생태계로 발전할 예정입니다.

---

**문서 작성일**: 2025년 11월 16일  
**버전**: v0.6.0 - Dataset Registry & Management System




