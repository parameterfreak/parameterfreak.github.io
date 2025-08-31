---
title: "PaperOps"
excerpt: "AI기반의 문서 생성 관리<br/><img src='/images/portfolio/thumbnail.png'>"
collection: portfolio
---

## 목적

PaperOps는 AI 기반 지능형 문서 작성 및 연구 자료 분석 플랫폼입니다. RAG(Retrieval-Augmented Generation) 기반의 의미 검색과 다중 LLM 프로바이더 지원을 통해 연구자, 작가, 비즈니스 전문가 등이 다양한 형식의 전문 문서를 효율적으로 작성하고 연구 자료를 분석할 수 있도록 돕습니다. 관계형 데이터베이스와 벡터 데이터베이스를 결합한 하이브리드 구조로 정확한 메타데이터 관리와 강력한 의미 기반 검색을 동시에 지원합니다.

![PaperOps 메인 화면](/images/portfolio/thumbnail.png)

## 주요 기능

### ✨ AI Assistant 
*   **실시간 대화 인터페이스:** 학술 문서 작성 과정에서 AI와 직접 상호작용
*   **다중 LLM 지원:** OpenAI, LM Studio, Ollama, vLLM 프로바이더 지원
*   **연결 상태 모니터링:** 실시간 LLM 서비스 상태 확인
*   **RAG 기반 응답:** 업로드된 연구 자료를 기반으로 한 맥락적 답변

![AI Assistant 인터페이스](/images/portfolio/write.png)

### 📄 지능형 문서 관리
*   **PDF 자동 처리:** 텍스트 추출, 청킹, 임베딩 생성
*   **의미 기반 검색:** 키워드가 아닌 의미로 문서 검색
*   **실시간 처리 상태:** 임베딩 처리 진행상황 실시간 모니터링
*   **하이브리드 저장:** 메타데이터(PostgreSQL) + 벡터(ChromaDB)

![문서 관리 시스템](/images/portfolio/pds.png)

### 🔐 보안 및 권한 관리
*   **JWT 인증:** 안전한 토큰 기반 인증
*   **역할 기반 접근:** 관리자/사용자 권한 분리
*   **데이터 격리:** 사용자별 데이터 접근 제어

### 🎯 사용자 인터페이스
*   **반응형 디자인:** 모던한 Tailwind CSS 기반 UI
*   **실시간 피드백:** 파일 업로드, 처리 상태 실시간 업데이트
*   **직관적 네비게이션:** 관리자/사용자 대시보드 분리

## 사용된 기술

### 핵심 기술 스택
*   **프론트엔드:** Next.js 15, TypeScript, Tailwind CSS
*   **백엔드:** FastAPI (Python), SQLAlchemy ORM
*   **데이터베이스:** PostgreSQL (관계형, Docker), ChromaDB (벡터)
*   **인증:** JWT (JSON 웹 토큰)
*   **임베딩 모델:** BGE-M3 (1024차원 임베딩용)

### LLM 프로바이더
*   **OpenAI:** GPT-4o, Function Calling, Vision API
*   **LM Studio:** 로컬 모델 실행, LoRA 지원
*   **Ollama:** 경량 로컬 LLM 프레임워크
*   **vLLM:** 고성능 추론 서버 (향후 지원)

### 개발 도구
*   **개발 환경:** Docker Compose, 가상환경
*   **API 문서:** FastAPI 자동 문서화 (Swagger UI)
*   **버전 관리:** Git, GitHub
*   **모니터링:** 실시간 상태 확인, 일관성 검사 도구
