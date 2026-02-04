---
title: 'AI Weekly Picks(20251119)'
date: 2025-11-19
permalink: /posts/2025/11/ai-weekly-picks-2025-11-19/
categories:
  - Misc
tags:
  - AI
  - News
---

# AI Weekly Picks(20251119)

# Tensorlake: 다양한 문서 및 이미지로부터 구조화된 데이터를 추출하고 처리하는 서버리스 플랫폼

- [https://discuss.pytorch.kr/t/tensorlake/8192](https://discuss.pytorch.kr/t/tensorlake/8192)
- [https://www.tensorlake.ai/](https://www.tensorlake.ai/)
- Tensorlake는 다양한 문서와 이미지로부터 구조화된 데이터를 추출하기 위해 설계된 서버리스 플랫폼입니다. 워크플로우 기반의 서버리스 API를 사용하여 다양한 문서 유형을 구조화된 데이터로 변환할 수 있으며, 로컬 및 클라우드 실행을 모두 지원합니다. Tensorlake는 Python 친화적인 API를 제공하고 JSON Schema 및 Pydantic 모델을 기반으로 구조화된 데이터 추출을 가능하게 함으로써 경쟁사와 차별화됩니다. 예제에서는 Tensorlake를 사용하여 이미지에서 구조화된 데이터를 추출하는 방법을 보여줍니다.

# AI 학습데이터 구축 안내서

- [https://www.aihub.or.kr/aihubnews/notice/view.do?currMenu=131&topMenu=103&nttSn=10476](https://www.aihub.or.kr/aihubnews/notice/view.do?currMenu=131&topMenu=103&nttSn=10476)
- 학습 데이터에 대한 개념 및 구축 절차에 대한 안내서

# Google Colab VS Code Extension

- [https://marketplace.visualstudio.com/items?itemName=Google.colab](https://marketplace.visualstudio.com/items?itemName=Google.colab)

  <img src="/images/post/weekly-picks/251119-1.png" alt="colab extension" width="450"/>

# Understanding the Foundations of Large Language Models: A Comprehensive Guide

- [https://medium.com/@sanaz.jamalzadeh/understanding-the-foundations-of-large-language-models-a-comprehensive-guide-a21e42b44da8](https://medium.com/@sanaz.jamalzadeh/understanding-the-foundations-of-large-language-models-a-comprehensive-guide-a21e42b44da8)
- 대규모 언어 모델(LLM)의 핵심 원리를 다룹니다. 방대한 데이터로 사전 훈련된 LLM이 어떻게 자연어를 이해하고 생성하는지, 그리고 프롬프팅, 검색 증강 생성(RAG), 도구 사용을 통해 능력을 확장하는 방법을 설명합니다. 또한, 인간의 가치에 맞게 모델을 정렬하는 중요성과 추론 최적화 기법에 대해서도 소개합니다.

# 강화학습의 핵심은 ‘정밀도’였다 — BF16에서 FP16으로의 전환이 만든 차이

- [https://turingpost.co.kr/p/bf16-fp16](https://turingpost.co.kr/p/bf16-fp16)
- 강화 학습(RL) 미세 조정에서 BF16에서 FP16 정밀도로 전환하여 안정성과 일관성을 크게 향상시키는 방법을 논의합니다. BF16의 범위가 더 넓음에도 불구하고 FP16의 더 높은 정밀도가 RL에서 "훈련-추론 불일치"를 유발하는 반올림 오류를 줄인다는 점을 설명합니다. 직관에 어긋나는 것처럼 보일 수 있는 이러한 변화는 RL 미세 조정의 지속적인 불안정성을 성공적으로 해결하여 복잡한 알고리즘 솔루션이 불필요하게 만들었습니다.

# Small Language Models: What They’re Really For?

- [https://www.linkedin.com/pulse/small-language-models-what-theyre-really-alex-wang-xqpac/](https://www.linkedin.com/pulse/small-language-models-what-theyre-really-alex-wang-xqpac/)
- "소규모"의 정의: SLM은 컴퓨팅 및 메모리 사용량이 적고, 소비자용 하드웨어에서 효율적으로 실행되며, 학습/미세 조정 비용이 저렴하고, 지연 시간이 짧으면서도 더 빠르고 저렴한 추론을 제공합니다. 또한 컨텍스트 요구 사항이 적고, 4k~32k 토큰 윈도우에서 잘 작동하며, 더 작은 도메인별 데이터 세트를 필요로 합니다. SLM은 특정 기능을 위해 설계된 전문가로, "한 가지 작업을 매우 잘" 수행하는 데 탁월합니다.
- SLM의 부상 이유: LLM은 강력하지만 비용이 많이 들고, 추론 비용이 높고, 지연 시간이 짧으며, 클라우드 의존성이 높고, 개인정보 보호 문제가 있습니다. 대부분의 엔터프라이즈 작업은 규모가 작기 때문에 SLM은 훨씬 적은 비용으로 LLM 성능의 80~90%를 달성할 수 있습니다. 하드웨어 발전과 더 나은 모델 압축/증류 기술 또한 SLM의 성장에 기여하고 있습니다.
- 비즈니스 애플리케이션: SLM은 주권과 개인정보 보호, 추론 비용 절감, 실시간 운영, 엣지 및 기기 네이티브 AI를 제공하며, 내부 폐쇄형 환경 에이전트에 이상적입니다.
- SLM 대 LLM: LLM은 심층 추론 및 복잡한 의사 결정을 위한 반면, SLM은 임베디드 AI, 내부 에이전트, 대용량 작업, 엣지/오프라인 AI, 엄격하게 관리되는 사용 사례와 같은 작업을 위해 제품, 도구 및 회사 내부에서 실행되는 운영 계층입니다.
- "큰 모델은 생각하고, 작은 모델은 작업을 수행합니다."