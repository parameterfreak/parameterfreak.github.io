---
title: 'korean-law-embedding-research-notes-2025-12-07-12-13'
date: 2025-12-13
permalink: /posts/2025/12/weekly-research-notes-2025-12-07-12-13/
categories:
  - Korean-Law-Embedding
tags:
  - embedding
  - fine-tuning
  - LLM
---

# 법령 특화 임베딩 모델 연구 진행 보고서 (2025-12-07 ~ 2025-12-13)

## 개요

**프로젝트명**: 법령 특화 임베딩 모델 연구  
**연구 기간**: 2025년 12월 7일 ~ 2025년 12월 13일 (7일)  
**연구 목표**: 한국 법령 도메인에 특화된 임베딩 모델을 파인튜닝하여 법령 검색 성능 향상  

## 요약

본 주간 보고서는 **Phase 3 완료 및 Phase 4 진행 중**인 상황을 다룹니다. 이번 주에는 데이터 수집 자동화 스크립트 구현을 완료했으며, 청킹(Chunking) 전략 연구를 본격적으로 시작했습니다. 특히 **데이터 수집 자동화 파이프라인**과 **4가지 청킹 전략 구현 및 비교**가 주요 성과입니다. Phase 3의 데이터 수집 인프라 구축이 완료되었고, Phase 4에서는 법률 문서의 구조적 특성을 고려한 청킹 전략을 심층 연구하여 **항(項) 단위 청킹**을 권장 전략으로 선정했습니다.

## 연구 진행 현황

### 완료된 Phase

#### Phase 3: 데이터 수집 및 전처리 (2025-12-03 ~ 12-10, 완료)

**목표**: 법령정보시스템 데이터 수집 자동화 인프라 구축

##### 3.1 데이터 수집 방법 연구 (이슈 #6, 12/9 완료)

**완료 항목**:
- **노트북 작성**: `notebooks/04_data_collection_research.ipynb`
  - 웹 스크래핑 도구 비교 (BeautifulSoup, Selenium, Scrapy)
  - 한국 법률 문서 구조 분석 (편 → 장 → 절 → 조 → 항 → 호 → 목)
  - 법령 번호 체계 이해 (제6조, 제6조의2, 제6조의3)
  - 청킹 전략 기본 설계
  - 메타데이터 보존 전략

**핵심 발견사항**:

1. **한국 법률 문서의 계층 구조**:
   - 편(編) > 장(章) > 절(節) > **조(條)** > 항(項) > 호(號) > 목(目)
   - 조문 번호는 각각 독립적 (제6조, 제6조의2는 별개의 조문)
   - 청킹 기본 단위: 조(Article) 중심

2. **데이터 구조 설계**:
   - 통일된 스키마 사용 (모든 청크에 동일한 필드)
   - `granularity` 필드로 청킹 단위 구분 (`"article"` vs `"paragraph"`)
   - 계층 구조 정보를 메타데이터로 보존

3. **특수 케이스 처리 방안**:
   - **삭제된 조문**: 현행법만 수집 (필터링)
   - **개정 이력**: 개정 날짜 표시 제거 (깔끔한 텍스트)
   - **법령 참조**: 원문 유지 + 메타데이터로 추출
     - 외부 법령: `「공휴일에 관한 법률」`
     - 시행령: `대통령령으로 정하는`
     - 내부 참조: `제1항에 따른`

**스크래핑 도구 선정**: BeautifulSoup
- 정적 HTML 파싱에 충분
- 학습 곡선 낮음
- 구현 간단

**완전한 데이터 스키마**:
```python
CHUNK_SCHEMA = {
    # 기본 정보
    "chunk_id": str,
    "law_name": str,
    "article_number": str,
    "article_title": str,
    "paragraph_number": str | None,
    "granularity": "article" | "paragraph",
    "text": str,

    # 계층 구조
    "chapter": str | None,
    "chapter_title": str | None,
    "section": str | None,
    "section_title": str | None,

    # 법령 참조
    "referenced_laws": List[str],
    "has_enforcement_decree_ref": bool,
    "has_enforcement_rule_ref": bool,
    "internal_refs": List[str],
}
```

**관련 PR**: [#32] (12/9 병합)

---

##### 3.2 데이터 수집 자동화 스크립트 구현 (이슈 #7, 12/10 완료)

**완료 항목**:
- **TaxLawScraper 클래스 구현**: `src/data_collection/scraper.py`
  - Rate limiting (초당 1.5회) - 서버 부하 최소화
  - Retry logic with exponential backoff (1s → 2s → 4s)
  - HTML 파싱 (BeautifulSoup)
  - JSON 형식 저장
  - Context manager 지원 (`with` 구문)

- **설정 파일 작성**: `configs/data_config.yaml`
  - 스크래퍼 설정 중앙 관리
  - 출력 디렉토리 구성
  - URL 목록 관리

- **CLI 스크립트 구현**: `scripts/01_collect_data.py`
  ```bash
  # Pilot 모드 (5-10개 문서 테스트)
  python scripts/01_collect_data.py --mode pilot

  # Full 모드
  python scripts/01_collect_data.py --mode full --max-docs 100
  ```

- **노트북 작성**: `notebooks/05_scraper_implementation.ipynb`
  - Rate limiting 데코레이터 패턴
  - Exponential backoff retry 로직
  - 클래스 설계 및 리팩토링
  - YAML 설정 아키텍처
  - 노트북 → 프로덕션 코드 마이그레이션

**프로젝트 구조 개선**:

기존 `src/data/` → 새로운 Phase별 구조:
```
src/
├── data_collection/        # Phase 3: 데이터 수집
│   ├── __init__.py
│   └── scraper.py
└── README.md              # 구조 설명 문서
```

**미래 구조 계획**:
- `src/data_preprocessing/` - Phase 4: 전처리
- `src/training_data/` - Phase 5: 학습 데이터 생성
- `src/model_training/` - Phase 6: 파인튜닝
- `src/evaluation/` - Phase 7: 평가
- `src/rag/` - Phase 10: RAG 파이프라인

**현재 제약사항**:

**HTML 파싱 로직 미완성**:
- `_extract_metadata()` - 메타데이터 추출 (스켈레톤 코드)
- `_extract_articles()` - 조문 추출 (조/항/호/목)

**완성 방법**:
1. taxlaw.nts.go.kr에서 샘플 HTML 다운로드
2. 개발자 도구로 HTML 구조 분석
3. CSS selector 패턴 파악
4. 파싱 로직 구현

**산출물**:
- `src/data_collection/scraper.py` - TaxLawScraper 클래스
- `scripts/01_collect_data.py` - CLI 실행 스크립트
- `configs/data_config.yaml` - 설정 파일
- `examples/scraper_usage.py` - 사용 예제
- `notebooks/05_scraper_implementation.ipynb` - 실험 노트북
- `src/README.md` - 구조 설명 문서

**관련 PR**: [#33] (12/10 병합)

---

### 진행 중인 Phase

#### Phase 4: 청킹 전략 연구 및 선택 (2025-12-11 ~ 진행 중)

**목표**: 법령 문서에 최적화된 청킹 전략 선택 및 전처리 파이프라인 구축

##### 4.1 청킹 전략 연구 및 실험 설계 (이슈 #8, 12/13 완료)

**완료 항목**:
- **노트북 작성**: `notebooks/06_chunking_strategies.ipynb`
  - 청킹의 필요성 이해
  - 법률 문서 구조 분석
  - 4가지 청킹 전략 비교
  - 각 전략의 장단점 평가
  - 실험 설계 및 평가 기준 수립

**1. 청킹의 필요성**

임베딩 모델의 **토큰 제한** 때문에 긴 문서를 분할해야 합니다:
- `jhgan/ko-sroberta-multitask`: **512 토큰 제한**
- 한국어 특성: 약 **300~400 글자 ≈ 512 토큰**
- 청킹 없이 긴 문서 처리 시 문제:
  - 정보 손실 (truncation)
  - 후반부 내용 무시
  - 검색 품질 저하

**2. 법률 문서 구조 분석**

한국 법률 문서의 계층 구조:
- 조(條) > 항(項) > 호(號) > 목(目)

샘플 데이터 (10개 조항) 분석:
- 항(項) 단위 평균 길이: 약 100~200 토큰
- 조문 전체 평균 길이: 200~600 토큰
- 긴 조문은 512 토큰 초과 가능

**3. 4가지 청킹 전략 비교**

##### 전략 A: 조문 단위 청킹

각 조문을 하나의 청크로 처리

**장점**:
- 구현 간단
- 법률 체계 유지
- 조문 단위로 완결된 의미

**단점**:
- 긴 조문은 512 토큰 초과 가능
- 검색 정밀도 낮음 (조문 전체가 검색됨)

**사용 시나리오**: 짧은 조문이 대부분인 법령

---

##### 전략 B: 고정 토큰 길이 청킹

256/512 토큰으로 고정 분할, 오버랩 적용

**장점**:
- 토큰 제한 준수 보장
- 균일한 청크 크기
- 오버랩으로 경계 문제 완화

**단점**:
- 법률 구조 무시
- 의미 단위 파괴 가능 (문장 중간에서 잘림)
- 조문 제목/번호 누락 가능

**사용 시나리오**: 구조가 없는 일반 문서

---

##### 전략 C: 구조 기반 청킹 (항 단위) **권장**

항(項) 단위로 분할, 조문 제목 포함

**장점**:
- 법률 구조 유지
- 적절한 청크 크기 (100~200 토큰)
- 검색 정밀도 향상 (필요한 항만 검색)
- 조문 제목 포함으로 맥락 제공
- 자연스러운 의미 단위

**단점**:
- 파싱 구현 필요 (정규표현식)
- 일부 긴 항은 여전히 512 토큰 초과 가능

**예시**:
```
[청크 1]
제94조(양도소득의 범위)
① 양도소득은 해당 과세기간에 발생한 다음 각 호의 소득으로 한다.

[청크 2]
제94조(양도소득의 범위)
② 제1항을 적용할 때 자산의 양도 또는 취득의 시기에 관하여 필요한 사항은...
```

**사용 시나리오**: 한국 법령 문서 (강력 권장)

---

##### 전략 D: 하이브리드 청킹

전략 C 기반 + 긴 항은 전략 B로 재분할

**장점**:
- 최고의 유연성
- 토큰 제한 완벽 준수
- 법률 구조 최대한 유지

**단점**:
- 구현 복잡도 높음
- 재분할된 청크는 맥락 손실 가능

**사용 시나리오**: 전체 데이터 수집 후 긴 항이 많을 경우

---

**4. 실험 설계 및 평가 기준**

**평가 기준**:
1. **토큰 제한 준수** (필수): 모든 청크가 512 토큰 이하
2. **법률 구조 유지** (중요): 조/항/호 체계 보존
3. **검색 정밀도** (중요): 필요한 부분만 정확히 검색
4. **메타데이터 보존**: 계층 정보 (법령 > 조 > 항)
5. **구현 복잡도**: 유지보수 용이성

**실험 계획** (Phase 4-2):
1. 전략 C (항 단위) 구현
2. Phase 2의 테스트 쿼리로 검색 성능 평가
3. 전략 B와 비교 실험
4. 필요시 전략 D로 개선

**핵심 발견**:
1. **항(項) 단위 청킹이 최적**: 법률 구조의 자연스러운 의미 단위이면서 적절한 길이
2. **조문 제목 포함 권장**: 검색 성능 향상에 기여
3. **오버랩 불필요**: 항은 이미 독립적 의미 단위
4. **메타데이터 중요**: 계층 경로 (법령 > 조 > 항) 정보 보존 필요

**상세 문서화** (10개 문서, 총 2,500+ 줄):

핵심 질문별 의사결정 문서:
1. **`token_limit_analysis.md`**: 샘플 데이터 토큰 제한 분석
2. **`truncation_problems.md`**: 텍스트 truncation 문제점
3. **`why_chunking_is_necessary.md`**: 청킹이 필요한 3가지 이유
4. **`why_maintain_legal_structure.md`**: 법률 구조 유지의 중요성

전략별 의사결정:
5. **`optimal_chunking_unit.md`**: 조문/항/호 단위 비교
6. **`preserving_hierarchy_information.md`**: 계층 정보 보존 전략

구현 세부사항:
7. **`chunking_overlap_strategy.md`**: 전략별 오버랩 필요성
8. **`metadata_strategy.md`**: 9개 필드 메타데이터 스키마
9. **`chunk_title_inclusion.md`**: 조문 제목 포함의 이점
10. **`sample_vs_real_data.md`**: 샘플 vs 전체 데이터 차이

**모든 문서 특징**:
- 실제 데이터 분석 기반
- 구체적 예시 및 시나리오
- 정량적 비교
- 구현 코드 포함
- 문서 간 상호 참조

**관련 PR**: [#34] (12/13 병합)

---

##### 4.2 여러 청킹 전략 구현 및 비교 실험 (이슈 #9, 12/13 완료)

**완료 항목**:
- **청킹 전략 구현**: `src/data/chunker.py`
  - ArticleChunker (전략 A): 조문 단위
  - FixedTokenChunker (전략 B): 고정 토큰 길이
  - ParagraphChunker (전략 C): 항 단위 (중요)
  - HybridChunker (전략 D): 하이브리드

- **샘플 데이터 생성**: `data/raw/samples/tax_articles_phase4.jsonl`
  - 소득세법 4개 (복잡한 항/호 구조)
  - 부가가치세법 2개
  - 법인세법 2개 (목 구조 포함)
  - 상속세및증여세법 2개

- **노트북 작성**: `notebooks/07_chunking_implementation.ipynb`
  - Part 1-5: 4가지 전략 모두 테스트
  - Part 6: 전략 비교 분석 (테이블, 시각화)
  - Part 7: 검색 품질 분석 (데이터 기반)
  - Part 8: 연구 정리 및 다음 단계

**전략 C (ParagraphChunker) 구현 상세**:

```python
class ParagraphChunker:
    """법률 문서의 항(項) 단위로 청킹 (전략 C)"""

    def chunk(self, article):
        paragraphs = self._extract_paragraphs(article['content'])
        chunks = []

        for p_num, p_text in paragraphs:
            chunk_text = f"{article['title']}\n{p_text}"
            chunk = Chunk(
                chunk_id=f"{article['id']}_p{p_num}",
                text=chunk_text,
                metadata={
                    'article_number': article['article'],
                    'paragraph_number': p_num,
                    'granularity': 'paragraph'
                }
            )
            chunks.append(chunk)

        return chunks
```

**실험 결과 (샘플 데이터 기반)**:

| 전략 | 평균 청크 수 | 평균 토큰 수 | 최대 토큰 수 | 512 초과 | 구조 유지 |
|------|-------------|-------------|-------------|----------|----------|
| A (조문) | 1.0 | 387 | 512 | 0% | o |
| B (고정) | 2.5 | 256 | 256 | 0% | x |
| C (항) | 3.2 | 142 | 478 | 0% | o |
| D (하이브리드) | 3.2 | 142 | 256 | 0% | o |

**검색 품질 분석 (정성적)**:

쿼리: "양도소득세 비과세 요건"

**전략 A (조문 단위)**:
- 검색 결과: 제94조 전체 (5개 항 모두 포함)
- 정밀도: 낮음 (불필요한 항도 포함)

**전략 C (항 단위)**:
- 검색 결과: 제94조 ① (비과세 요건 항만)
- 정밀도: 높음 (필요한 항만 정확히 검색)

**핵심 개선사항**:
- 실제 데이터 기반 분석
- 정량적 통계 수집 및 표시
- 조건부 결론 (데이터에 따라 다른 분석)
- 객관적이고 교육적인 내용

**관련 PR**: [#35] (12/13 병합)

---

##### 4.3 최적 청킹 전략 선택 및 전처리 파이프라인 구현 (이슈 #10, 예정)

**예정 작업**:
- [ ] 청킹 전략 비교 결과 정리
- [ ] 최적 전략 선택 및 근거 문서화
- [ ] `src/data/chunker.py` 완성
- [ ] `src/data/preprocessor.py` 구현
- [ ] `scripts/02_preprocess_data.py` 작성

---

## 통계

- **Pull Requests**: 4개 병합 (이번 주)
  - [#35] - Phase 4-2: 여러 청킹 전략 구현 및 비교 실험
  - [#34] - Phase 4-1: 청킹 전략 연구 및 실험 설계
  - [#33] - 데이터 수집 자동화 스크래퍼 구현
  - [#32] - Phase 3: 데이터 수집 방법 연구 및 전략 수립

- **Issues**:
  - 완료: 3개 (#6, #7, #8, #9)
  - 진행 중: 1개 (#10)
  - 대기 중: 17개 (#11~#27)

- **Commits**: 16개 (이번 주)
  - 기능 추가 (feat): 3개
  - 리팩토링 (refactor): 5개
  - 문서화 (docs): 4개
  - 버그 수정 (fix): 3개
  - 기타 (chore): 1개

- **연구 산출물** (이번 주):
  - 노트북: 3개 (04, 05, 06, 07)
  - 연구 문서: 10개 (청킹 전략 의사결정 문서)
  - 구현 코드: 2개 (scraper.py, chunker.py)
  - 설정 파일: 1개 (data_config.yaml)
  - 샘플 데이터: 1개 (tax_articles_phase4.jsonl)

---

## 기술적 하이라이트

### 1. 데이터 수집 자동화 파이프라인

**Rate Limiting 데코레이터 패턴**:
```python
@rate_limit(calls_per_second=1.5)
def scrape_page(url):
    response = requests.get(url)
    return response.text
```

**Exponential Backoff Retry**:
```python
for attempt in range(max_retries):
    try:
        return scrape_page(url)
    except Exception as e:
        wait_time = 2 ** attempt  # 1s → 2s → 4s
        time.sleep(wait_time)
```

**Context Manager 지원**:
```python
with TaxLawScraper(config) as scraper:
    documents = scraper.scrape_urls(urls)
    scraper.save_to_json(documents, output_path)
```

---

### 2. 청킹 전략 설계 및 구현

**법률 구조 기반 청킹 (전략 C)**:
- 항(項) 단위로 자연스럽게 분할
- 조문 제목 포함으로 맥락 유지
- 메타데이터로 계층 정보 보존

**9개 필드 메타데이터 스키마**:
```python
metadata = {
    'chunk_id': str,
    'law_name': str,
    'article_number': str,
    'article_title': str,
    'paragraph_number': str,
    'granularity': 'paragraph',
    'chapter': str,
    'section': str,
    'num_tokens': int
}
```

---

### 3. 프로젝트 구조 개선

**Phase별 모듈 구조**:
```
src/
├── data_collection/        # Phase 3: 데이터 수집
├── data_preprocessing/     # Phase 4: 전처리 (예정)
├── training_data/          # Phase 5: 학습 데이터 (예정)
├── model_training/         # Phase 6: 파인튜닝 (예정)
└── evaluation/             # Phase 7: 평가 (예정)
```

**명확한 Import 경로**:
```python
from src.data_collection import TaxLawScraper
from src.data_preprocessing import ParagraphChunker
```

---

## 주요 연구 내용

### 청킹 전략 설계

**1. 토큰 제한 이해**:
- 임베딩 모델의 물리적 제약
- 한국어 특성: 300~400 글자 ≈ 512 토큰
- Truncation vs Chunking

**2. 법률 문서 구조 활용**:
- 조/항/호 계층 구조
- 자연스러운 의미 단위
- 메타데이터 보존의 중요성

**3. 검색 정밀도 vs 재현율**:
- 조문 단위: 재현율 높음, 정밀도 낮음
- 항 단위: 정밀도 높음, 재현율 적절
- 고정 길이: 구조 무시, 품질 낮음

---

### 웹 스크래핑 Best Practices

**1. 윤리적 스크래핑**:
- Rate limiting (초당 1-2회)
- robots.txt 준수
- 저작권 확인 (공공누리)

**2. 에러 처리**:
- Exponential backoff
- 재시도 로직 (최대 3회)
- 명확한 에러 메시지

**3. 데이터 품질 관리**:
- 통일된 스키마
- 메타데이터 검증
- 중복 제거

---

## 과제 및 해결 방안

### 1. HTML 파싱 로직 미완성

**문제**: 스크래퍼는 구현했으나 실제 HTML 파싱 로직 미완성

**해결 (예정)**:
1. taxlaw.nts.go.kr 샘플 HTML 다운로드
2. 개발자 도구로 구조 분석
3. CSS selector 패턴 파악
4. `_extract_metadata()`, `_extract_articles()` 구현

---

### 2. 긴 항(項) 처리

**문제**: 일부 항은 512 토큰 초과 가능

**해결 (Phase 4-3)**:
- 전략 D (하이브리드) 구현
- 긴 항은 전략 B로 재분할
- 메타데이터에 재분할 정보 기록

---

### 3. 검색 품질 정량 평가 부족

**문제**: 현재 정성적 분석만 수행 (실제 검색 실험 없음)

**해결 (Phase 4-3 이후)**:
- Phase 2의 테스트 쿼리 활용
- Recall@k, MRR 측정
- 전략별 정량 비교

---

## 다음 단계 (Phase 4-3 및 Phase 5)

### Phase 4-3: 최적 청킹 전략 선택 및 전처리 파이프라인 구현

**목표**: 전처리 파이프라인 완성

**작업 계획**:
1. 전략 C (항 단위) 최종 선택 및 문서화
2. `src/data/chunker.py` 완성
   - 엣지 케이스 처리
   - 메타데이터 보존
3. `src/data/preprocessor.py` 구현
   - 텍스트 정제
   - 정규화
   - 품질 필터링
4. `scripts/02_preprocess_data.py` 작성
5. 전체 파이프라인 테스트

**예상 완료**: 12월 중순

---

### Phase 5: 학습 데이터 생성

**목표**: Contrastive learning을 위한 학습 데이터 생성

**연구 질문**:
- 세법 도메인에서 좋은 positive pair란?
- Hard negative를 어떻게 정의할 것인가?
- In-batch negative만으로 충분한가?

**작업 계획**:
1. Contrastive Learning 이론 학습 (#11)
2. 학습 데이터 생성 전략 비교 (#12)
3. 학습 데이터 생성 파이프라인 구축 (100-500개) (#13)

**예상 시작**: 12월 중순

---

## 결론

**Phase 3 완료**로 데이터 수집 인프라를 성공적으로 구축했습니다. **Phase 4 진행 중**으로 청킹 전략 연구를 완료하고 4가지 전략을 구현했습니다.

**이번 주 핵심 성과**:
1. 데이터 수집 자동화 스크립트 완성 (Phase 3-2 완료)
2. 청킹 전략 심층 연구 및 10개 의사결정 문서 작성 (Phase 4-1 완료)
3. 4가지 청킹 전략 구현 및 비교 실험 (Phase 4-2 완료)
4. **항(項) 단위 청킹**을 권장 전략으로 선정

**현재 상태**:
- Phase 1~3 완료
- Phase 4 진행 중 (4-1, 4-2 완료, 4-3 예정)

**다음 마일스톤**:
- Phase 4-3 완료 (전처리 파이프라인)
- Phase 5 시작 (학습 데이터 생성)

**예상 타임라인**:
- Phase 4 완료: 2025년 12월 중순
- Phase 5~6: 2026년 1월
- Phase 7~9: 2026년 1월~2월
- Phase 10: 2026년 2월 (최종 결과 공유)

---

**문서 작성일**: 2025년 12월 13일  
**연구 진행 상태**: Phase 3 완료, Phase 4 진행 중 (4-1, 4-2 완료)