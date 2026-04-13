---
title: 'AI Weekly Picks(16주차)'
date: 2026-04-13
permalink: /posts/2026/04/ai-weekly-picks-202616/
categories:
  - Misc
tags:
  - AI
  - News
  - Reference
---

# AI Daily Picks(20260413)

* [Google's TurboQuant: 6x Less Memory for LLM Inference (2026) - Nerd Level Tech](https://nerdleveltech.com/google-turboquant-kv-cache-compression-llm-inference)
  * 구글 연구진이 발표한 TurboQuant 알고리즘은 LLM의 주요 메모리 병목인 KV 캐시를 정확도 손실 없이 3비트로 압축하여 메모리 사용량을 최대 6배 감소시킴.
  * 무작위 직교 회전(Random Orthogonal Rotation)으로 좌표 구성을 균일하게 한 후 Lloyd-Max 최적 양자화를 적용하며, QJL을 통해 에러를 교정하는 과정으로 데이터나 아키텍처 의존성 없이 압축 효율성을 달성.
  * 평가 벤치마크(LongBench 등)에서 기준점과 동등한 점수를 기록하였으며, H100 환경에서 어텐션 로짓 계산 속도를 8배 향상시켜 저렴한 환경에서 긴 컨텍스트 모델 구동을 가능하게 함.
* [Microsoft Open-Sources Industry-Leading Embedding Model](https://blogs.bing.com/search/April-2026/Microsoft-Open-Sources-Industry-Leading-Embedding-Model)
  * 마이크로소프트가 검색 품질 향상 및 차세대 AI 에이전트를 위한 강력한 다국어 텍스트 임베딩 모델 'Harrier(Harrier-OSS-v1)' 제품군을 오픈소스로 공개함.
  * 100개 이상의 언어 지원과 32k 토큰의 긴 컨텍스트 윈도우 처리가 가능하며, Multilingual MTEB v2 벤치마크 평가에서 기존 주요 모델(Gemini Embedding 2 등)을 크게 능가하는 SOTA 성능을 기록함.
  * GPT-5를 활용한 합성 데이터 생성과 20억 쌍의 사전 학습을 거쳤으며, 최고 성능의 27B 모델로부터 소형 모델(0.6B, 270M)로 지식 증류(Knowledge Distillation)를 진행해 매개변수 대비 극대화된 효율성을 확보함.


