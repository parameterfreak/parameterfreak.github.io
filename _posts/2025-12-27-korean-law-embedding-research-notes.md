---
title: 'korean-law-embedding-research-notes-2025-12-14-12-27'
date: 2025-12-27
permalink: /posts/2025/12/weekly-research-notes-2025-12-14-12-27/
tags:
  - embedding
  - fine-tuning
  - LLM
---

# 법령 특화 임베딩 모델 연구 진행 보고서 (2025-12-14 ~ 2025-12-27)

## 개요

**프로젝트명**: 법령 특화 임베딩 모델 연구  
**연구 기간**: 2025년 12월 14일 ~ 2025년 12월 27일 (14일)  
**연구 목표**: 한국 법령 도메인에 특화된 임베딩 모델을 파인튜닝하여 법령 검색 성능 향상

## 요약

본 주간 보고서는 **Phase 4~5 완료 및 Phase 6 진행 중**인 상황을 다룹니다. 이번 2주간 전처리 파이프라인 구축을 완료하고, Contrastive Learning 기반 학습 데이터 생성 전략을 수립했으며, 첫 번째 파인튜닝 실험 환경을 구축했습니다. 특히 **전처리 파이프라인의 완성**, **학습 데이터 생성 파이프라인 구축**, **파인튜닝 환경 설정 및 학습 설정 이해**가 주요 성과입니다. Phase 4의 청킹 전략 선택 및 전처리 시스템이 완료되었고, Phase 5에서는 4가지 학습 데이터 생성 전략을 비교 분석하여 **하이브리드 접근법**을 선택하고 실제 학습 데이터 생성 파이프라인을 구축했습니다. Phase 6에서는 파인튜닝 환경 설정과 Kaggle 기반 실험 환경을 완성했습니다.

## 연구 진행 현황

### 완료된 Phase

#### Phase 4: 청킹 전략 연구 및 선택 (2025-12-11 ~ 12-15, 완료)

**목표**: 법령 문서에 최적화된 청킹 전략 선택 및 전처리 파이프라인 구축

##### 4.3 최적 청킹 전략 선택 및 전처리 파이프라인 구현 (이슈 #10, 12/15 완료)

**완료 항목**:
- **청킹 전략 결정 문서화**: `docs/chunking_decision.md`
  - 4가지 전략 비교 분석 결과 정리
  - 전략 C (항 단위 청킹) 선택 근거 문서화
  - 법률 구조 유지 + 검색 정밀도 향상 균형점

- **전처리기 구현**: `src/data/preprocessor.py`
  - **TextPreprocessor**: 조문 레벨 전처리
    - 텍스트 정제: 공백/특수문자/줄바꿈 정규화
    - 품질 필터링: 길이, 한글 비율 검증
    - 메타데이터 검증
  - **ChunkPreprocessor**: 청크 레벨 검증
    - 토큰 수 제한 확인 (5~4096 토큰)
    - 필수 메타데이터 필드 검증
    - 유효하지 않은 청크 필터링

- **전처리 스크립트**: `scripts/02_preprocess_data.py`
  - End-to-end 전처리 파이프라인
  - 5단계 처리: 로드 → 전처리 → 청킹 → 검증 → 저장
  - 4가지 청킹 전략 지원 (article, fixed, paragraph, hybrid)
  - 상세 통계 리포트 생성

- **학습용 노트북**: `notebooks/08_preprocessing_pipeline.ipynb`
  - Part 1: 전처리의 필요성 이해
  - Part 2: TextPreprocessor 이해하기
  - Part 3: Phase 4-2 실험 결과와 전략 C 선택 근거
  - Part 4: 전처리 파이프라인 실행 및 통계 분석
    - **통계 지표 해석 가이드** (평균, 중앙값, 표준편차)
    - **시각화 해석 가이드** (히스토그램, 막대 그래프)
  - Part 5: 전처리 스크립트 사용하기
  - Part 6: 다음 단계 준비

**테스트 결과**:

| 지표 | 결과 |
|------|------|
| 입력 조항 수 | 10개 |
| 출력 청크 수 | 20개 |
| 조문 전처리 통과율 | 100% (10/10) |
| 청크 유효율 | 100% (20/20) |
| 평균 토큰 수 | 88.2 |
| 토큰 범위 | 34~207 (모두 512 이하) |
| 법률 구조 보존 | 완벽 |

**선택된 전략: C (항 단위 청킹)**

장점:
- 법률 구조(조-항-호) 유지
- 의미 단위로 정확한 분할
- 검색 정밀도 향상 (관련 항만 반환)
- 조문 제목으로 맥락 제공
- 샘플 데이터에서 모든 항이 512 토큰 이하

**사용 예시**:
```bash
# 기본 사용 (항 단위 청킹)
python scripts/02_preprocess_data.py \
  --input data/raw/samples/tax_articles_phase4.jsonl \
  --output data/processed/chunks/tax_chunks.jsonl \
  --strategy paragraph

# 하이브리드 전략 (토큰 제한 엄격)
python scripts/02_preprocess_data.py \
  --input data/raw/samples/tax_articles_phase4.jsonl \
  --output data/processed/chunks/tax_chunks_hybrid.jsonl \
  --strategy hybrid \
  --max-tokens 512 \
  --split-threshold 256
```

**학습 포인트**:

**1. 통계 지표를 통한 데이터 품질 평가**
- **평균 vs 중앙값**: 이상치 감지
- **표준편차**: 일관성 확인
- **최소/최대값**: 극단값 위험도 평가
- 512 토큰 초과만 확인하는 것이 아니라 **전체 분포 품질** 평가

**2. 시각화를 통한 패턴 발견**
- **히스토그램**: 분포 형태 (정규분포, 꼬리 긴 분포, 이봉 분포)
- **막대 그래프**: 조문 복잡도 비교, 일관성 확인
- 숫자는 요약, 시각화는 전체 그림

**3. 의사결정 프로세스**
- 통계 + 시각화 + 도메인 지식을 종합한 판단
- 복잡도와 성능의 균형점 찾기
- 의사결정 이력 문서화의 중요성

**관련 PR**: [#36] (12/15 병합)

---

### 완료된 Phase (계속)

#### Phase 5: 학습 데이터 생성 (2025-12-17 ~ 12-23, 완료)

**목표**: Contrastive learning을 위한 학습 데이터 생성

##### 5.1 Contrastive Learning 이론 학습 및 데이터 형식 설계 (이슈 #11, 12/21 완료)

**완료 항목**:
- **노트북 작성**: `notebooks/09_training_data_generation.ipynb`
  - Part 1-2: Contrastive Learning 기초 및 핵심 아이디어
    - Query-Positive-Negative 구조 이해
    - Hard negative의 중요성
  - Part 2: Loss Function 비교
    - Triplet Loss, Multiple Negatives Ranking Loss, InfoNCE 비교
    - MNR Loss 선택 (효율성, 검증된 성능)
  - Part 3: 학습 데이터 형식 설계
    - JSONL 형식 + Query-Positive-Negative triplets
    - Chunk ID 참조 방식
    - 풍부한 메타데이터 구조
  - Part 4: Negative Sampling 전략
    - Hard (60%): 같은 조문, 다른 항
    - Medium (30%): 다른 조문, 같은 법
    - Easy (10%): 다른 법
  - Part 5: Query 생성 방법 비교
    - Manual, GPT-4, Template 방식
  - Part 6: 샘플 데이터 생성 실습
    - 10개 pilot 샘플 생성
    - 품질 검증 구현

- **학습 데이터 형식 문서**: `docs/training_data_format.md`
  - JSONL 스키마 정의
  - Negative sampling 전략 상세 설명
  - Query 생성 방법론
  - 품질 검증 기준

**학습 데이터 스키마**:
```json
{
  "query": "1세대 1주택 양도소득세 비과세 요건은?",
  "positive_chunk_id": "소득세법_제89조_1",
  "negative_chunk_ids": [
    "소득세법_제89조_2",  // Hard negative
    "소득세법_제94조_1",  // Medium negative
    "법인세법_제18조_1"   // Easy negative
  ],
  "metadata": {
    "query_type": "natural",
    "difficulty": "medium",
    "created_by": "manual"
  }
}
```

**핵심 의사결정**:

1. **학습 데이터 형식**: Query-Positive-Negative Triplets
   - Explicit hard negatives 사용 (Phase 6 pilot)
   - Chunk ID 참조로 데이터 중복 최소화

2. **Loss Function**: Multiple Negatives Ranking Loss
   - In-batch negatives 활용으로 효율성 확보
   - Sentence-Transformers 검증됨

3. **Negative Mix**: Hard 60% + Medium 30% + Easy 10%
   - Fine-grained distinction 학습 우선

**Hard Negative의 중요성**:

Easy negative만 사용하면 모델이 큰 주제 구분만 학습:
```
"소득세법" vs "법인세법" 구분 (너무 쉬움)
```

Hard negative를 사용하면 미묘한 차이까지 학습:
```
같은 조문 내에서도 "보유요건" vs "거주요건" 구분 (정밀)
```

**관련 PR**: [#37] (12/21 병합)

---

##### 5.2 학습 데이터 생성 전략 비교 및 선택 (이슈 #12, 12/21 완료)

**완료 항목**:
- **노트북 작성**: `notebooks/10_training_data_strategy_comparison.ipynb`
  - Part 1: 전략 개요
    - 전략 A: 같은 문서 내 섹션 매칭 (hard negative 확보)
    - 전략 B: 관련 법령 간 매칭 (법률-시행령-예규)
    - 전략 C: LLM 질문 생성 (GPT-4로 자연스러운 query)
    - 전략 D: 하이브리드 접근 (A+C 조합)
  - Part 2-5: 전략별 구현 및 실습
    - 각 전략으로 10-20개 샘플 생성
    - 실제 코드로 구현하여 실행 가능
  - Part 6: 품질 평가
    - 자동 품질 검증 프레임워크
    - 수동 검증 가이드라인
    - 전략별 품질 비교
  - Part 7: 최종 전략 선택

- **의사결정 문서**: `docs/training_data_strategy.md`
  - 전략 비교 분석 (장단점, 적용 시기)
  - 최종 결정 및 근거
  - 트레이드오프 분석

**4가지 전략 비교**:

**전략 A: 같은 문서 내 섹션 매칭**

| 장점 | 단점 |
|------|------|
| Hard negative 자동 확보 | Query 품질 제한적 |
| 법률 구조 활용 | 자연스러운 질문 부족 |
| 무료, 자동화 가능 | 템플릿 반복 가능 |
| Fine-grained learning | |

예시:
```python
# Positive
query = "1세대 1주택 양도소득세 비과세의 보유요건은?"
positive = "소득세법 제89조 ① 보유기간 2년..."

# Hard Negative (같은 조문, 다른 항)
hard_neg = "소득세법 제89조 ② 거주기간 2년..."
```

---

**전략 B: 관련 법령 간 매칭**

| 장점 | 단점 |
|------|------|
| 계층 구조 학습 | 법령-시행령 관계 파악 어려움 |
| 교차 참조 활용 | 시행령 데이터 수집 필요 |
| 실무 시나리오 반영 | Phase 3 미완료로 불가 |

**적용 시기**: Phase 9 전체 데이터 수집 후

---

**전략 C: LLM 질문 생성**

| 장점 | 단점 |
|------|------|
| 자연스러운 질문 | API 키 필요 (OpenAI 사용 시) |
| 다양한 표현 | 품질 검증 필요 |
| 사용자 의도 반영 | 로컬 LLM은 품질 변동 가능 |

예시:
```
Template:
"1세대 1주택의 양도소득세 비과세의 요건은 무엇인가요?"
→ 반복적이고 형식적

LLM 생성:
"1세대 1주택을 양도할 때 비과세를 받으려면 얼마나 거주해야 하나요?"
→ 자연스럽고 구체적
```

**LLM 활용 효과**:
- 품질 향상: 자연스러움
- 다양한 표현 생성으로 학습 효과 증대

---

**전략 D: 하이브리드**

전략 A (섹션 매칭) + 전략 C (LLM query)의 장점 결합

| 장점 | 단점 |
|------|------|
| Hard negative + 자연스러운 질문 | 구현 복잡도 높음 |
| 품질과 효율의 균형 | 두 시스템 관리 필요 |
| 확장 가능 | |

---

**최종 전략 선택: 하이브리드 접근법**

```
전략 A (섹션 매칭):     40개 (40%)
전략 D (하이브리드):    30개 (30%)
Manual:                20개 (20%)
Template Baseline:     10개 (10%)
────────────────────────────────
Total:                100개
```

**선택 근거**:

1. **Hard negative 70% 확보** (전략 A 40% + 전략 D 30%)
   - Fine-grained learning 우선
   - 같은 조문 내 미묘한 차이 학습

2. **LLM으로 자연스러운 query 50% 생성** (전략 D 30% + Manual 20%)
   - 사용자 의도 반영
   - 다양한 표현 학습

3. **품질과 효율의 균형**
   - LLM 활용으로 자연스러운 표현 확보
   - Manual 20개로 품질 보증

4. **확장 가능성**
   - Phase 9에서 비율 조정 가능
   - 전략 B (관련 법령) 추가 가능

**관련 PR**: [#38] (병합 대기 중, 12/22 예상)

---

##### 5.3 학습 데이터 생성 파이프라인 구축 (이슈 #13, 12/23 완료)

**완료 항목**:
- **노트북 작성**: `notebooks/11_training_data_pipeline.ipynb`
  - 100개 학습 데이터 생성 파이프라인 구축
  - OpenAI / LM Studio / Ollama 3가지 LLM 옵션 지원
  - 전략별 샘플 생성 구현 (A, D, Template, Manual)
  - 법률 구조 기반 Hard Negative Mining
  - 품질 검증 프레임워크
  - Train/Valid split (80/20)

**3가지 LLM 통합**:

| LLM | 특징 | 추천 모델 |
|-----|------|-----------|
| OpenAI API | 최고 품질, 클라우드 | GPT-4o/GPT-4o-mini |
| LM Studio | 로컬 실행, OpenAI 호환 | Qwen2.5-14B-Instruct |
| Ollama | CLI 기반, 설치 간편 | qwen2.5:14b, llama3.1:8b |

**통합 LLMClient 클래스**:
```python
# 3가지 provider 지원
client = LLMClient(provider="openai", model="gpt-4o")
client = LLMClient(provider="lmstudio")
client = LLMClient(provider="ollama", model="qwen2.5:14b")
```

**구현된 기능**:

1. **전략별 샘플 생성**
   - 전략 A: 섹션 매칭 + Template query (40개)
   - 전략 D: Hybrid (섹션 매칭 + LLM query) (30개)
   - Template Baseline (10개)
   - Manual 작성 가이드 (20개)

2. **Hard Negative Mining**
   - 법률 구조 기반 (같은 조문 내 다른 항 선택)
   - Hard/Medium/Easy 3단계 지원

3. **품질 검증 프레임워크**
   - 자동 검증: 길이, 언어 비율, 메타데이터 확인
   - 수동 검증: 50개 샘플링, 1-5점 척도
   - 통계 생성 및 분석

4. **Train/Valid Split**
   - 80/20 분할
   - 전략별 분포 확인
   - JSONL 형식 저장

**학습 목표**:
1. LLM 통합 및 활용 방법 이해
2. 전략별 샘플 생성 자동화
3. 법률 구조 기반 Hard Negative Mining
4. 품질 검증 프로세스 학습
5. 재사용 가능한 파이프라인 구축

**관련 PR**: [#39] (12/23 병합)

---

### 진행 중인 Phase

#### Phase 6: 첫 번째 파인튜닝 (2025-12-24 ~ 진행 중)

**목표**: 파인튜닝 환경 설정 및 첫 실험 수행

##### 6.1 파인튜닝 환경 설정 및 학습 설정 이해 (이슈 #14, 12/26 완료)

**완료 항목**:
- **노트북 작성**: `notebooks/12_first_finetuning_setup.ipynb`
  - Google Colab/Kaggle GPU 환경 구축 가이드
  - Sentence Transformers 설치 및 설정
  - 10개 파일럿 샘플 데이터 로드

**파일럿 샘플 데이터**:
- `data/training/pilot_finetuning_samples.jsonl`: 10개 샘플
- 다양한 세법 영역 커버 (소득세, 법인세, 부가가치세, 상속세)

**하이퍼파라미터 이해 (Section 4)**:

| 파라미터 | 값 | 설명 |
|----------|-----|------|
| Learning Rate | 2e-5 | BERT 논문 권장값 (5e-5, 3e-5, 2e-5 중 하나) |
| Batch Size | 8 | In-batch negatives 전략 (8개 샘플 → 7개 negative) |
| Epochs | 3 | Overfitting 관찰 목적 |
| Warmup Steps | 0 | 파일럿 단계에서는 미사용 |
| Temperature (τ) | 0.05 | Contrastive learning 스케일 조정 |

**In-batch Negatives 전략 이해**:
- Negative를 명시적으로 제공하지 않고 배치 내 다른 샘플의 positive를 활용
- Batch size = N → 각 query당 (N-1)개의 negative 자동 생성
- 데이터 효율적이고 GPU 효율적인 학습 방법

예시:
```python
# Batch size = 4
Query 1: "양도소득세 비과세 요건은?"  → Positive: 소득세법 제89조
Query 2: "법인세 세율은?"           → Positive: 법인세법 제55조
Query 3: "부가세 면세 범위는?"       → Positive: 부가세법 제26조
Query 4: "상속세 계산 방법은?"       → Positive: 상속세법 제13조

# Query 1의 negatives (자동 생성)
- 법인세법 제55조 (Query 2의 positive)
- 부가세법 제26조 (Query 3의 positive)
- 상속세법 제13조 (Query 4의 positive)
```

**학습 실행 및 검증 (Section 5-6)**:
- Multiple Negatives Ranking Loss 설정
- 10개 샘플로 실제 학습 실행
- 학습 전후 임베딩 변화 검증
- Positive vs Negative similarity 확인

**검증 기준**:
- L2 distance > 0.01: 경험적 휴리스틱 (정상 fine-tuning 범위 0.01~0.1)
- Cosine similarity > 0.95: SimCSE 논문 기준 (의미 보존 확인)
- 한계 인식: 단일 쿼리 기준, Phase 7에서 validation set 기반 평가 필요

**학습 관찰 및 분석 프레임워크 (Section 7)**:

4가지 핵심 실험 설계:
- **실험 1**: Loss 수렴 분석 (overfitting 체크)
- **실험 2**: 임베딩 공간 변화 (catastrophic forgetting 방지)
- **실험 3**: Retrieval 성능 (before/after 비교)
- **실험 4**: Hard negative 분석 (난이도 평가)

**Phase 6-2 준비**:
- `experiments_analysis_cells.md` 생성
- 100개 샘플 단계에서 사용할 상세 분석 코드 템플릿
- 하이퍼파라미터 sweep 실험 계획

**관련 PR**: [#40] (12/25 병합)

---

##### 6.2 첫 번째 파인튜닝 실험 환경 구축 (이슈 #15 일부, 12/27 완료)

**완료 항목**:
- **학습 데이터 생성 스크립트**: `scripts/generate_finetuning_data.py`
  - 세법 데이터로부터 학습용 쿼리-positive-negative 쌍 생성
  - In-batch negatives 전략 활용 (MNRL Loss와 함께 사용)
  - 약 1,800개의 학습 샘플 생성

- **Kaggle 노트북**: `notebooks/13_first_finetuning_experiment.ipynb`
  - Sentence Transformers 기반 파인튜닝 파이프라인 구축
  - Kaggle 환경 최적화 (한글 폰트, WandB 비활성화)
  - 상세한 학습 프로세스 설명 포함
  - MultipleNegativesRankingLoss 개념 설명

- **문서화**:
  - `data/training/README.md`: 학습 데이터 형식 및 Kaggle 업로드 가이드
  - `docs/first_finetuning_observations.md`: 실험 관찰 및 분석 문서

**기술적 특징**:

| 항목 | 설정 |
|------|------|
| 손실 함수 | MultipleNegativesRankingLoss (in-batch negatives) |
| 학습 전략 | 2 에포크, batch size 16, warmup 10% |
| 평가 | Cosine similarity 기반 검색 성능 측정 |
| 환경 | Kaggle T4 GPU (무료 티어) |

**Kaggle 환경 최적화**:

1. **한글 폰트 지원**:
   ```python
   # 환경별 폰트 자동 선택 (macOS/Linux/Windows)
   # Kaggle에서 matplotlib 한글 폰트 경고 무시
   # UserWarning 필터링으로 깔끔한 출력
   ```

2. **WandB 비활성화**:
   ```python
   # Kaggle에서 학습이 멈추는 문제 해결
   os.environ['WANDB_DISABLED'] = 'true'
   os.environ['WANDB_MODE'] = 'disabled'
   ```

3. **Progress bar 처리**:
   ```python
   # Kaggle에서 progress bar widget 문제 해결
   # show_progress_bar=False로 학습 hang 방지
   # logging으로 진행상황 표시
   ```

4. **GPU 확인 및 할당**:
   ```python
   # 명시적 GPU 디바이스 설정
   # GPU 메모리 모니터링
   # 모델 파라미터 CUDA 확인
   ```

**MultipleNegativesRankingLoss 심층 설명**:

노트북에 상세한 이론 설명 추가:
- MNRL 메커니즘 및 이론
- In-Batch Negatives 전략 구체적 예시
- Triplet Loss, Contrastive Loss와 비교
- Batch size 4 예시로 시뮬레이션
- Epoch별 학습 효과 시뮬레이션
- Batch size, temperature, 데이터 품질 실무 팁
- 실험 설정과의 연결
- 예상 성능 향상 및 지표

**베이스라인 결과 해석**:
- Perfect Recall (1.0) 현상 분석
- Recall vs Ranking 품질 차이 이해 (nDCG/MRR)
- 파인튜닝 목표 재정의: Recall 향상 → Ranking 향상
- 데이터셋 크기별 성능 추세 분석
- 구체적 목표 지표 및 다음 단계

**테스트 완료**:
- 학습 데이터 생성 스크립트 실행 확인
- Kaggle 환경에서 노트북 실행 가능 확인
- 한글 폰트 정상 렌더링 확인

**다음 단계**:
- [ ] Kaggle에서 전체 파인튜닝 실험 실행 (100/300/500 샘플)
- [ ] 베이스라인 대비 성능 향상 측정
- [ ] 결과 분석 및 Phase 7 평가 프레임워크 설계

**관련 PR**: [#41] (12/27 병합)

---

## 통계

- **Pull Requests**: 6개 병합 (이번 2주)
  - [#41] - Phase 6-2: 첫 번째 파인튜닝 실험 환경 구축 (12/27 병합)
  - [#40] - Phase 6-1: 파인튜닝 환경 설정 및 학습 설정 이해 (12/25 병합)
  - [#39] - Phase 5-3: 학습 데이터 생성 파이프라인 구축 (학습 자료) (12/23 병합)
  - [#38] - Phase 5-2: 학습 데이터 생성 전략 비교 및 선택 (12/22 병합)
  - [#37] - Phase 5-1: Contrastive Learning 이론 학습 및 데이터 형식 설계 (12/21 병합)
  - [#36] - Phase 4-3: 최적 청킹 전략 선택 및 전처리 파이프라인 구현 (12/15 병합)

- **Issues**:
  - 완료: 5개 (#10, #11, #12, #13, #14)
  - 진행 중: 1개 (#15)
  - 대기 중: 12개 (#16~#27)

- **Commits**: 42개 (이번 2주)
  - 기능 추가 (feat): 4개
  - 문서화 (docs): 18개
  - 버그 수정 (fix): 11개
  - 병합 (merge): 6개
  - 스타일 (style): 1개
  - Kaggle 노트북: 2개

- **연구 산출물** (이번 2주):
  - 노트북: 6개 (08, 09, 10, 11, 12, 13)
  - 연구 문서: 4개
    - chunking_decision.md
    - training_data_strategy.md
    - training_data_format.md
    - first_finetuning_observations.md
  - 구현 코드: 3개
    - preprocessor.py
    - 02_preprocess_data.py
    - generate_finetuning_data.py
  - 파일럿 데이터: 2개
    - pilot_finetuning_samples.jsonl (10개)
    - 학습 데이터 생성 스크립트 (약 1,800개 샘플)
  - 기타: experiments_analysis_cells.md, data/training/README.md

---

## 기술적 하이라이트

### 1. Kaggle 환경 최적화 (Phase 6-2)

**Kaggle T4 GPU 무료 티어 활용**:
- Google Colab 대비 안정적인 GPU 할당
- 주당 30시간 무료 GPU 사용
- 한글 폰트, WandB, Progress bar 이슈 모두 해결

**해결한 Kaggle 환경 문제**:

| 문제 | 해결 방법 | 효과 |
|------|-----------|------|
| 한글 폰트 경고 | 환경별 폰트 자동 선택 + UserWarning 필터링 | 깔끔한 출력 |
| WandB로 학습 멈춤 | `WANDB_DISABLED=true` 환경변수 설정 | 정상 학습 |
| Progress bar hang | `show_progress_bar=False` + logging | 진행상황 확인 가능 |
| CPU 기본 학습 | 명시적 GPU 디바이스 할당 | 학습 속도 10배 향상 |

**환경 감지 로직**:
```python
# Kaggle vs Colab vs Local 자동 감지
if 'KAGGLE_KERNEL_RUN_TYPE' in os.environ:
    print("Kaggle 환경")
elif 'COLAB_GPU' in os.environ:
    print("Colab 환경")
else:
    print("로컬 환경")
```

---

### 2. 전처리 파이프라인 설계 (Phase 4-3)

**PreprocessorConfig 구조**:
```python
@dataclass
class PreprocessorConfig:
    # 텍스트 정제
    normalize_whitespace: bool = True
    normalize_special_chars: bool = True
    normalize_newlines: bool = True

    # 품질 필터링
    min_length: int = 10
    max_length: int = 10000
    min_korean_ratio: float = 0.5

    # 메타데이터 검증
    required_fields: List[str] = field(default_factory=lambda: [
        'id', 'law', 'article', 'title', 'content'
    ])
```

**5단계 파이프라인**:
1. 조문 데이터 로드
2. 텍스트 전처리 및 품질 검증
3. 청킹 (4가지 전략 지원)
4. 청크 품질 검증
5. 결과 저장 (JSONL) + 통계 리포트

---

### 3. LLM 통합 학습 데이터 생성 파이프라인 (Phase 5-3)

**3가지 LLM 옵션 지원**:
```python
class LLMClient:
    def __init__(self, provider="openai", model=None):
        if provider == "openai":
            self.client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))
            self.model = model or "gpt-4o"
        elif provider == "lmstudio":
            self.client = OpenAI(base_url="http://localhost:1234/v1", api_key="lm-studio")
            self.model = "local-model"
        elif provider == "ollama":
            # Ollama API 호출
            self.model = model or "qwen2.5:14b"
```

**하이브리드 전략 구현**:
```python
# 전략 A: 섹션 매칭 (40개)
def generate_strategy_a_samples(chunks, count=40):
    for chunk in chunks:
        # 같은 조문 내 다른 항 찾기 (Hard Negative)
        hard_neg = find_same_article_different_paragraph(chunk)
        # Template query 생성
        query = f"{chunk['article_title']}의 요건은?"
        yield (query, chunk['id'], [hard_neg['id']])

# 전략 D: Hybrid (30개)
def generate_strategy_d_samples(chunks, llm_client, count=30):
    for chunk in chunks:
        # LLM으로 자연스러운 query 생성
        query = llm_client.generate_query(chunk['text'])
        # 법률 구조 기반 Hard Negative
        hard_neg = find_same_article_different_paragraph(chunk)
        yield (query, chunk['id'], [hard_neg['id']])
```

**품질 검증**:
```python
def validate_sample(sample):
    # 자동 검증
    assert 10 <= len(sample['query']) <= 200
    assert sample['query'].count('?') >= 1
    assert len(sample['negative_chunk_ids']) >= 1

    # 수동 검증 (50개 샘플링)
    # 1-5점 척도: 1=매우 나쁨, 5=매우 좋음
```

---

### 4. Contrastive Learning 데이터 구조 (Phase 5-1)

**Query-Positive-Negative Triplets**:
```python
{
  "query": "자연스러운 질문",
  "positive_chunk_id": "정답 청크 ID",
  "negative_chunk_ids": [
    "hard_negative",    # 60%
    "medium_negative",  # 30%
    "easy_negative"     # 10%
  ],
  "metadata": {
    "query_type": "natural|technical",
    "difficulty": "easy|medium|hard",
    "created_by": "manual|gpt4|template"
  }
}
```

**Multiple Negatives Ranking Loss**:
```python
# In-batch negatives 활용
# Batch size = 16 → 각 query당 15개 negative 자동 생성
# + Explicit hard negatives 1-3개 추가

loss = -log(
    exp(sim(q, p+) / τ) /
    (Σ exp(sim(q, p-) / τ) + exp(sim(q, p+) / τ))
)
```

---

### 3. 학습 데이터 생성 전략

**하이브리드 접근법**:

| 전략 | 개수 | 비율 | 목적 |
|------|------|------|------|
| A (섹션 매칭) | 40 | 40% | Hard negative 확보 |
| D (A+LLM) | 30 | 30% | Hard negative + 자연스러운 query |
| Manual | 20 | 20% | 품질 보증 |
| Template | 10 | 10% | Baseline |

**Hard Negative Mining (법률 구조 기반)**:
```python
def get_hard_negative(positive_chunk):
    # 같은 조문, 다른 항 선택
    same_article = chunks.filter(
        law=positive_chunk.law,
        article=positive_chunk.article,
        paragraph != positive_chunk.paragraph
    )
    return random.choice(same_article)
```

---

## 주요 연구 내용

### 전처리 파이프라인 설계

**1. 텍스트 정규화**
- 공백 정규화: 연속된 공백 → 단일 공백
- 특수문자 정규화: 전각 → 반각
- 줄바꿈 정규화: 연속된 줄바꿈 제거

**2. 품질 필터링**
- 길이 검증: 10~10,000자
- 한글 비율: 최소 50%
- 메타데이터 완전성 확인

**3. 청크 검증**
- 토큰 수 제한: 5~4,096 토큰
- 메타데이터 필수 필드 확인
- 유효하지 않은 청크 자동 제거

---

### Contrastive Learning 이론

**1. Query-Positive-Negative 구조**
- Query: 사용자 질문
- Positive: 정답 문서
- Negative: 오답 문서 (유사도 낮춰야 함)

**2. Hard Negative의 역할**
- 모델이 미묘한 차이를 학습
- 검색 정밀도 향상
- Overfitting 방지

**3. Loss Function 비교**

| Loss | 특징 | 선택 이유 |
|------|------|-----------|
| Triplet Loss | (q, p+, p-) 쌍 필요 | 구현 복잡 |
| **MNR Loss** | **In-batch negatives 활용** | **효율적, 검증됨** |
| InfoNCE | MNR과 유사 | MNR로 충분 |

---

### 학습 데이터 생성 전략 비교

**1. 품질 vs 효율**
- Manual: 최고 품질, 시간 소요 ↑
- Template: 빠름, 품질 제한적
- LLM: 품질 우수, 설정 필요
- **하이브리드**: 균형점

**2. Hard Negative vs Easy Negative**
- Easy: 큰 주제 구분 (법인세 vs 소득세)
- Medium: 중간 난이도 (같은 법, 다른 조)
- Hard: 미묘한 차이 (같은 조, 다른 항)
- **Mix (60:30:10)**: 점진적 학습

**3. 확장성 vs 품질**
- 자동화 (전략 A): 확장 쉬움, 품질 제한적
- Manual: 확장 어려움, 품질 보장
- **하이브리드**: 80% 자동 + 20% Manual

---

## 과제 및 해결 방안

### 1. LLM 선택 및 활용

**과제**: LLM 통합 및 품질 관리

**해결**:
- 3가지 LLM 옵션 제공 (OpenAI/LM Studio/Ollama)
- 로컬 LLM 활용으로 자유로운 실험 가능
- 효과 검증 후 Phase 9에서 확대

---

### 2. Manual 샘플 작성 시간

**문제**: 20개 Manual 작성에 2-3시간 소요 예상

**해결**:
- 고품질 validation set으로 활용
- 다양한 쿼리 유형 커버
- 향후 자동 생성 품질 기준으로 사용

---

### 3. Hard Negative 확보 어려움

**문제**: 같은 조문 내 항이 적은 경우

**해결 (Phase 5-3)**:
- 법률 구조 기반 자동 선택
- 같은 장(章) 내 다른 조문 활용
- 필요시 BM25 기반 mining 추가

---

## 다음 단계 (Phase 5-3 및 Phase 6)

### Phase 5-3: 학습 데이터 생성 파이프라인 구축 (100-500개)

**목표**: 파일럿 학습 데이터 생성

**작업 계획**:
1. `src/data/pair_generator.py` 구현
   - 전략 A (섹션 매칭) 자동화
   - 전략 D (하이브리드) 통합
   - Hard negative mining
2. `scripts/03_create_training_data.py` 작성
3. Manual 20개 직접 작성
4. 품질 검증 (50개 샘플링)
5. Train/Valid split (80/20)
6. 통계 생성

**예상 완료**: 12월 하순

---

### Phase 6: 첫 번째 파인튜닝

**목표**: 100개 데이터로 파일럿 파인튜닝

**연구 질문**:
- 100개 데이터로 학습 효과가 있는가?
- Overfitting이 발생하는가?
- 어떤 하이퍼파라미터가 적절한가?

**작업 계획**:
1. 파인튜닝 환경 설정 (#14)
   - Google Colab/Kaggle 환경
   - Sentence Transformers 설치
   - 하이퍼파라미터 이해
2. 첫 번째 파인튜닝 실험 (#15)
   - 100개 데이터로 학습
   - Loss 변화 관찰
   - 검색 성능 비교
3. 학습 결과 분석 (#16)
   - 개선된 점, 부족한 점
   - Error analysis
   - 개선 방향 도출

**예상 시작**: 12월 하순

---

## 결론

**Phase 4~5 완료**로 전처리 시스템과 학습 데이터 생성 파이프라인을 성공적으로 구축했습니다. **Phase 6 진행 중**으로 파인튜닝 환경 설정을 완료하고 첫 실험을 준비했습니다.

**이번 2주 핵심 성과**:
1. 전처리 파이프라인 완성 (Phase 4-3 완료, 12/15)
   - 항 단위 청킹 전략 (Strategy C) 최종 선택
   - 10개 조항 → 20개 청크 (100% 성공률)

2. Contrastive Learning 완전 학습 (Phase 5-1 완료, 12/21)
   - Query-Positive-Negative triplets 형식 설계
   - MNR Loss 선택, Hard/Medium/Easy negative 비율 결정

3. 학습 데이터 생성 전략 수립 (Phase 5-2 완료, 12/22)
   - 4가지 전략 비교 및 하이브리드 접근법 선택
   - 전략 A 40% + 전략 D 30% + Manual 20% + Template 10%

4. 학습 데이터 생성 파이프라인 구축 (Phase 5-3 완료, 12/23)
   - OpenAI/LM Studio/Ollama 3가지 LLM 통합
   - 법률 구조 기반 Hard Negative Mining
   - 품질 검증 프레임워크

5. 파인튜닝 환경 설정 (Phase 6-1 완료, 12/26)
   - 하이퍼파라미터 이해 (Learning Rate, Batch Size, Temperature)
   - In-batch Negatives 전략 학습
   - 과학적 실험 방법론 수립

6. Kaggle 파인튜닝 환경 구축 (Phase 6-2 완료, 12/27)
   - 약 1,800개 학습 샘플 생성 스크립트
   - Kaggle 환경 최적화 (한글 폰트, WandB, Progress bar 해결)
   - MultipleNegativesRankingLoss 심층 설명

**현재 상태**:
- Phase 1~5 완료
- Phase 6 진행 중 (6-1, 6-2 완료, 실험 준비 완료)

**다음 마일스톤**:
- Phase 6 실험 실행 (100/300/500 샘플로 파인튜닝)
- 베이스라인 대비 성능 향상 측정
- Phase 7 평가 프레임워크 설계

**기술적 성취**:
- 법률 구조 기반 청킹 시스템
- 3가지 LLM 통합 데이터 생성 파이프라인
- Kaggle 무료 GPU 환경 완전 활용
- 교육적 학습 자료 6개 노트북

**예상 타임라인**:
- Phase 6 완료: 2026년 1월 초
- Phase 7 (평가 프레임워크): 2026년 1월
- Phase 8~9: 2026년 1월~2월
- Phase 10: 2026년 2월 (최종 결과 공유)

---

**문서 작성일**: 2025년 12월 27일  
**연구 진행 상태**: Phase 5 완료, Phase 6 진행 중 (6-1, 6-2 완료, 실험 준비 완료)