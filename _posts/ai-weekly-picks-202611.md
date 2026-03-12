---
title: 'AI Weekly Picks(11주차)'
date: 2026-03-11
permalink: /posts/2026/03/ai-weekly-picks-202611/
categories:
  - Misc
tags:
  - AI
  - News
  - Reference
---

# AI Daily Picks(20260312)

# AI Daily Picks(20260311)

* [Top embedding models on the MTEB leaderboard](https://modal.com/blog/mteb-leaderboard-article) : MTEB(Massive Text Embedding Benchmark) 리더보드를 바탕으로 목적에 맞는 임베딩 모델 선택 기준과 현재 상위권에 있는 주요 모델(Qwen3, BGE-M3 등)들의 특징을 소개하는 가이드
* [Introducing RTEB: A New Gold Standard for Evaluating Retrieval Models 2025 : An Exclusive Report - CerebalAiCerebral Ai - Corrected Header](https://thecerebralai.com/introducing-rteb/) : 오픈소스 생태계와 RAG, 시맨틱 검색을 위한 새로운 기준점이 될 AI 검색 모델 평가를 위한 새로운 표준 벤치마크인 RTEB의 도입배경과 특징을 소개하는 글
* [Disaggregated Serving in TensorRT LLM — TensorRT LLM](https://nvidia.github.io/TensorRT-LLM/1.2.0rc1/blogs/tech_blog/blog5_Disaggregated_Serving_in_TensorRT-LLM.html) : LLM 추론의 Context(Prefill) 처리와 Generation(Decode) 생성을 서로 다른 GPU 풀로 분리(Disaggregated Serving)해 간섭을 줄이고 응답시간(TTFT, TPOT)을 최적화하는 TensorRT-LLM의 구조와 성능 분석을 담은 기술 블로그
* [AI Is a Five-Layer Cake](https://x.com/nvidia/status/2031311890752704790) : 컴퓨팅 환경의 근본적인 변화에 맞춰 컴퓨팅 스택을 5단계 계층(에너지, 칩, 인프라, 모델, 애플리케이션)으로 나누어 구조를 설명하는 NVIDIA의 포스트
* [Production LLM serving on Kubernetes with KServe and llm-d](https://x.com/RedHat_AI/status/2031405209310683585) : KServe(Control Plane)와 llm-d(Data Plane)를 결합하여 Kubernetes 환경에서 vLLM 기반 대규모 언어 모델(LLM)을 분산 추론 및 프로덕션 수준으로 서빙하는 아키텍처와 패턴을 소개하는 포스트
* [Gemini Embedding 2: Our first natively multimodal embedding model](https://blog.google/innovation-and-ai/models-and-research/gemini-models/gemini-embedding-2/) : 텍스트, 이미지, 오디오, 비디오, 문서를 단일 공간으로 매핑하는 최초의 네이티브 멀티모달 임베딩 모델인 Gemini Embedding 2 소개
* [pplx-embed: State-of-the-Art Embedding Models for Web-Scale Retrieval](https://research.perplexity.ai/articles/pplx-embed-state-of-the-art-embedding-models-for-web-scale-retrieval) : 실제 웹 스케일 검색을 위해 구축된 뛰어난 성능의 텍스트 임베딩 모델 pplx-embed-v1 및 pplx-embed-context-v1을 소개하는 글
* [Introducing Olmo Hybrid: Combining transformers and linear RNNs for superior scaling](https://allenai.org/blog/olmohybrid) : 트랜스포머의 어텐션 메커니즘과 선형 RNN을 결합하여 컨텍스트 처리 속도와 프리트레이닝 데이터 효율성을 크게 향상시킨 7B 규모의 오픈 모델 'Olmo Hybrid' 소개
* [Andrej Karpathy 가 전하는 'AutoResearch' 프로젝트 결과](https://x.com/karpathy/status/2031135152349524125) : 에이전트 무리를 활용해 약 700번의 자율 실험을 거쳐 실제 11%의 모델 성능 개선을 달성한 안드레 카파시의 프로젝트 성과 경험
* [AutoKernel 오픈소스 공개](https://x.com/Akashi203/status/2031533857082646769) : PyTorch 모델의 병목을 분석하고 Triton 코드를 자동 생성하여 수십 차례 벤치마크와 실험을 수행, GPU 커널을 자율 최적화하는 도구의 오픈소스 배포 소식
* [분산형 검색 엔진 Autosearcher 소개와 AI 에이전트의 검색 랭킹 최적화](https://x.com/varun_mathur/status/2031550020101480507) : 에이전트들이 자율적인 실험과 P2P 교차 수분을 통해 분산형 검색 엔진의 랭킹 모델(ListNet 등)을 스스로 진화시키는 'Autosearcher' 프로젝트 소개
