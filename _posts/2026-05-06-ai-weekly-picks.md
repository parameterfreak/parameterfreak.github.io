---
title: 'AI Weekly Picks(19주차)'
date: 2026-05-06
permalink: /posts/2026/05/ai-weekly-picks-202619/
categories:
  - Misc
tags:
  - AI
  - News
  - Reference
---

# AI Daily Picks(20260506)

* [KV Cache Optimization: Serve 10x More Users on the Same GPU (2026) - Spheron Blog](https://www.spheron.network/blog/kv-cache-optimization-guide/)
  * LLM 추론 시 긴 컨텍스트에서 발생하는 가장 큰 GPU 메모리 병목인 KV 캐시를 최적화하는 5가지 핵심 기법을 소개합니다.
  * PagedAttention(가상 메모리처럼 필요시 할당), FP8/NVFP4 양자화(메모리 사용량 절감), CPU 오프로딩(트래픽 스파이크 대응), LMCache 및 프리픽스 캐싱(TTFT 단축), GQA/Flash Attention 같은 아키텍처 수준의 최적화 기법들을 다룹니다.
  * 이러한 기법들을 복합적으로 적용하여 동일한 GPU에서 수용 가능한 동시 사용자를 늘리고, 워크로드에 맞게 최적화된 GPU VRAM 예산을 산정할 수 있습니다.
* [LLM Inference Optimization: Quantization, KV Cache, and Serving at Scale - Perivitta Rajendran](https://pr-peri.github.io/blogpost/2026/03/25/blogpost-llm-quantization-kv-cache.html)
  * 프로덕션 환경에서 LLM 추론 시 발생하는 메모리 및 연산 병목을 해결하기 위한 최적화 기법들을 심도 있게 다룹니다.
  * 주요 기법으로 가중치와 KV 캐시의 메모리 사용량을 4~8배 줄이는 양자화(GPTQ, AWQ), 처리량을 10~20배 높이는 연속 배치(Continuous Batching), 지연 시간을 2~3배 단축하는 추측 해독(Speculative Decoding)을 설명합니다.
  * 이러한 최적화 기술들이 내장된 vLLM, TGI와 같은 최신 서빙 프레임워크와 Flash Attention, 텐서 병렬 처리(Tensor Parallelism)를 활용한 효율적인 대규모 인퍼런스 구축 방법을 제공합니다.
