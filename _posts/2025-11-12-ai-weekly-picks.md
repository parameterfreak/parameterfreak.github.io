---
title: 'AI Weekly Picks(20251112)'
date: 2025-11-12
permalink: /posts/2025/11/ai-weekly-picks-2025-11-12/
categories:
  - Misc
tags:
  - AI
  - News
---

# AI Weekly Picks(20251112)

# **Kosmos: An AI Scientist for Data-Driven Discovery**

- [https://arxiv.org/pdf/2511.02824](https://arxiv.org/pdf/2511.02824)
- **자율적인 과학적 발견을 위한 AI 과학자**인 **Kosmos**라는 시스템
    - 구조화된 월드 모델(structured world model)을 사용하여 병렬로 실행되는 다수의 에이전트들의 출력을 관리
    - 데이터 기반 발견(data-driven discovery)을 자동화
- **추적 가능한 추론 과정**을 보장하기 위해 모든 진술에 코드나 문헌을 인용
- 데이터 분석 에이전트와 문헌 검색 에이전트 간에 정보를 공유하고 문맥을 관리
- 한계 및 앞으로 해결해야할 점
    - **정확성 및 해석의 개선**, **기술적 확장성 문제 해결**, 그리고 **발견 가치 평가 자동화**의 필요성
    - Kosmos는 데이터셋을 약 **5GB**까지만 관리
    - **이미지나 원시 시퀀싱 파일**과 같은 **원시 데이터(raw data)** 분석에는 능숙하지 않음
- Kosmos의 가장 중요한 기여 중 하나는 **대규모의 편향 없는 탐색**을 통해 인간 연구자가 놓쳤던 새로운 메커니즘을 발견한 것

# **Efficient Word Vector Estimation in Vector Space**

- [https://arxiv.org/pdf/1301.3781](https://arxiv.org/pdf/1301.3781)
- 고전(Word2Vec)
- 수십억 개의 단어로 구성된 **방대한 데이터셋**과 수백만 개의 단어로 구성된 어휘집에서 **고품질 단어 벡터(high-quality word vectors)를 학습**하는 데 사용할 수 있는 기술을 도입이 목표
    - **대규모 데이터 세트에서 연속적인 단어 벡터 표현을 계산하기 위한** 두 가지 **모델 아키텍처**를 제안 - CBOW(Continuous Bag-of-Words)와 **Skip-gram** 모델
    - **CBOW 모델**은 비선형 은닉 계층이 제거된 피드포워드 NNLM과 유사하며, 문맥 단어들(이전 및 이후 단어)의 벡터를 공유하고 평균화하여 현재(중앙) 단어를 정확하게 분류하도록 훈련(CBOW 모델이 문맥 정보를 평균화하는 방식)
        - **단어 순서가 투영에 영향을 미치지 않는다**는 특성
    - **Skip-gram 모델**은 현재 단어를 입력으로 사용하여 연속 투영 계층(continuous projection layer)을 가진 로그-선형 분류기를 통해 해당 단어 주변의 문맥 단어들을 예측하려고 시도(**현재(중앙) 단어를 입력**으로 사용하여 **주변 문맥 단어들을 예측)**

# Disaggregated Inference: 18 Months Later

- [https://hao-ai-lab.github.io/blogs/distserve-retro/](https://hao-ai-lab.github.io/blogs/distserve-retro/)
- DistServe에 의해 소개된 LLM 추론을 사전 채우기(prefill) 및 디코딩(decode) 단계로 분리하는 개념이 대규모 LLM 서비스에서 표준 관행이 되었음을 설명합니다. 이 접근 방식은 간섭을 제거하고 독립적인 확장을 가능하게 하여 공동 배치(colocation)의 한계를 해결하며, 지연 시간과 리소스 효율성에서 상당한 개선을 가져왔습니다. 이 기사는 다양한 프로덕션 프레임워크에서의 채택을 강조하고 어텐션-FFN 분리와 같은 미래 방향을 탐구합니다.

# IterResearch: Rethinking Long-Horizon Agents via Markovian State Reconstruction

- [https://arxiv.org/pdf/2511.07327](https://arxiv.org/pdf/2511.07327)
- 모노-컨텍스트 패러다임이 겪는 컨텍스트 질식(context suffocation)과 **노이즈 오염(noise contamination)** 문제를 해결하기 위해 고안된 새로운 **반복적 심층 연구 패러다임**
- IterResearch는 이러한 초기 오류와 노이즈의 연쇄 전파를 방지하도록 설계
    - 주기적인 합성(periodic synthesis)과 **전략적인 망각**을 통해 노이즈를 필터링하는 자연스러운 분기점(breakpoints)을 제공
- 전략적인 작업 공간 재구성을 통해 이러한 신호 희석 문제를 해결
    - 전체 기록을 누적하는 대신, 에이전트가 업데이트하는 진화하는 보고서를 유지
    - **필터링:** 이전 라운드의 모든 중요한 발견을 합성하고 노이즈를 걸러내는 역할
    - **선택적 유지:** 에이전트의 합성 과정은 LLM의 **정보 압축 및 관련성 필터링** 능력을 활용하여, 관련 없는 정보나 오류가 필터링되지 않은 채 미래 결정에 **직접적으로 전파되는 것을 방지**
- EAPO는 새로운 반복적 심층 연구 패러다임의 잠재력을 완전히 실현하기 위해 개발된 강화 학습(RL) 프레임워크
- 주요 결과
    - 전반적인 성능 우위 및 격차 해소
    - **상호작용 확장성 ( 2048회 상호작용 확장**
    - **효과적인 프롬프팅 전략으로서의 역할(model-agnostic: ReAct 대비 우위)**
    - **패러다임 간 지식 전달**