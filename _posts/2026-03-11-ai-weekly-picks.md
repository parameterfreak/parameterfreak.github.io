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

* [TensorRT-LLM Speculative Decoding Boosts Inference Throughput by up to 3.6x - NVIDIA Technical Blog](https://developer.nvidia.com/blog/tensorrt-llm-speculative-decoding-boosts-inference-throughput-by-up-to-3-6x)
  * TensorRT-LLM의 Speculative Decoding을 통해 Llama 3.1 405B/70B와 같은 대규모 모델의 추론 처리량(Throughput)을 최대 3.6배까지 향상
  * 작고 빠른 Draft 모델(예: Llama 3.2 1B)이 토큰을 미리 예측하고 크고 느린 Target 모델이 이를 검증하는 방식으로 지연 시간 단축
  * FP8 정밀도를 활용한 단일/다중 H200 GPU 환경에서의 Draft-Target 모델 엔진 빌드 및 Triton 서버 배포 가이트 제공

* [RTEB: The New Gold Standard for Evaluating Retrieval Models](https://news.lavx.hu/article/rteb-the-new-gold-standard-for-evaluating-retrieval-models)
  * 허깅페이스(Hugging Face)가 기존 평가 지표의 한계를 극복하기 위해 공개 및 비공개 데이터셋을 결합한 하이브리드 평가 프레임워크 RTEB를 출시했습니다.
  * 법률, 의료, 금융 등 다양한 산업 분야와 20개 언어를 지원하여, 실제 환경에서의 모델 일반화 능력을 보다 정확하게 측정할 수 있습니다.
  * 평가 모델이 특정 벤치마크에 과적합(Overfitting)되는 것을 방지하여 RAG 및 엔터프라이즈 검색 시스템의 신뢰성을 향상시킵니다.

* [Introducing EmbeddingGemma: The Best-in-Class Open Model for On-Device Embeddings](https://developers.googleblog.com/en/introducing-embeddinggemma/)
  * 구글이 온디바이스 및 모바일 환경에 최적화된 새로운 오픈 텍스트 임베딩 모델인 EmbeddingGemma를 공개했습니다.
  * Gemma 3 아키텍처 기반의 308M 파라미터 모델로, 양자화 시 200MB 이하의 RAM으로도 작동하며 100개 이상의 언어 모델에서 MTEB 기준 동급 최고의 성능을 제공합니다.
  * Matryoshka Representation Learning(MRL)을 적용하여 출력 차원을 유연하게 조절할 수 있으며, 오프라인 환경에서도 안전한 RAG 파이프라인 구축을 지원합니다.

* [월 150만 유저가 사용하는 LLM Inference 인프라 안정적으로 운영하기](https://blog.scatterlab.co.kr/%EC%9B%94-150%EB%A7%8C-%EC%9C%A0%EC%A0%80%EA%B0%80-%EC%82%AC%EC%9A%A9%ED%95%98%EB%8A%94-llm-inference-%EC%9D%B8%ED%94%84%EB%9D%BC-%EC%95%88%EC%A0%95%EC%A0%81%EC%9C%BC%EB%A1%9C-%EC%9A%B4%EC%98%81%ED%95%98%EA%B8%B0-122607)
  * 스캐터랩(이루다)이 월 150만 유저 규모의 LLM 서빙 인프라를 클라우드 기반 Kubernetes 환경에서 vLLM, SGLang 등의 오픈소스 엔진으로 컨테이너화하여 운영하는 경험을 공유합니다.
  * GPU Capacity 확보를 위해 Multi-Region 클러스터를 도입하고, Terraform으로 인프라를 코드화하며, Istio Multicluster로 서비스 디스커버리 및 부하분산 문제를 해결했습니다.
  * Karmada를 활용한 멀티 클러스터 단일 Control Plane 구축과 Blob Storage/ACR을 이용한 멀티 리전 스토리지 관리 전략을 소개합니다.

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
