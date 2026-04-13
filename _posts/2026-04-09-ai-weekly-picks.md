---
title: 'AI Weekly Picks(15주차)'
date: 2026-04-09
permalink: /posts/2026/04/ai-weekly-picks-202615/
categories:
  - Misc
tags:
  - AI
  - News
  - Reference
---

# AI Daily Picks(20260410)

* [A Framework for Assessing AI Agent Decisions and Outcomes in AutoML Pipelines](/Users/jaehoonkim/ws/tmp/til/paper/2602.22442v2.pdf)
  * LLM 기반의 Agentic AutoML 파이프라인에서 중간 의사결정 과정을 평가하기 위한 전용 평가 에이전트(Evaluation Agent, EA) 프레임워크를 제안함.
  * 기존 평가는 최종 모델 성능(정확도 등)에만 치중하지만, EA는 의사결정 타당성, 추론 일관성, 정확도 이외의 모델 품질, 그리고 반사실적(counterfactual) 요인 분석 등 4가지 차원에서 시스템을 다각적으로 진단함.
  * 실험 결과, EA는 0.919의 F1 스코어로 잘못된 의사결정을 탐지하고 시스템의 최종 성능에 가려진 추론 오류 및 성능 변동(-4.9% ~ +8.3%)의 근본 원인을 효과적으로 식별함.

# AI Daily Picks(20260409)

* [LLM Inference: Prefill, Decode, KV Cache & Cost Guide (2026) - Morph](https://www.morphllm.com/llm-inference)
  * LLM 추론은 병렬로 입력 토큰을 처리하는 Prefill(연산 집약적) 단계와 토큰을 하나씩 생성하는 Decode(메모리 대역폭 집약적) 단계로 나뉨.
  * KV Cache는 이전 토큰 처리 결과를 저장해 비용을 O(n)으로 줄여주지만 많은 메모리를 소모하므로 PagedAttention, 양자화 등의 기법이 필수적으로 적용됨.
  * 추론 비용과 효율성을 개선하기 위해 Flash Attention 3, 연속 배치(Continuous Batching), 추측 해독(Speculative Decoding) 같은 최적화 기법이 널리 쓰임.
  * 요약하자면 토큰 전달량을 줄이는 컨텍스트 압축이 추론 속도를 높이고 전체 비용을 낮추는 가장 근본적인 최적화 방법임.


