---
title: 'AI Weekly Picks(12주차)'
date: 2026-03-16
permalink: /posts/2026/03/ai-weekly-picks-202612/
categories:
  - Misc
tags:
  - AI
  - News
  - Reference
---

# AI Daily Picks(20260320)

* [Understanding LLMInferenceService - KServe](https://kserve.github.io/website/docs/model-serving/generative-inference/llmisvc/llmisvc-overview)
  * KServe의 `LLMInferenceService`는 생성형 AI 워크로드(대규모 언어 모델 서빙) 관리를 위해 설계된 전용 Kubernetes CRD입니다.
  * 기존 예측형 AI 중심의 `InferenceService`와 분리된 방식(Dual-Track 전략)을 통해, LLM에 더욱 특화된 라우팅 설정과 모델 스케줄링 기능을 제공합니다.
  * 단일 노드부터 다중 노드 텐서/데이터 병렬화, Prefill-Decode 분리 등 고성능 분산 추론을 위한 다양한 배포 패턴을 지원합니다.
  * Model, Workload, Router, Parallelism 사양(spec)을 조합하여, 엔터프라이즈 환경에서 유연하고 최적화된 서빙 아키텍처를 구성할 수 있습니다.

* [TSEmbed: Unlocking Task Scaling in Universal Multimodal Embeddings](https://arxiv.org/abs/2603.04772)
  * 범용 멀티모달 임베딩 모델의 성능 개발을 저해하는 "태스크 충돌(task conflict)" 문제를 해결하기 위한 프레임워크 **TSEmbed**를 제안합니다.
  * MoE(Mixture-of-Experts)와 LoRA를 결합하여 서로 충돌하는 태스크 목표를 명시적으로 분리하고, 전문가 인지 네거티브 샘플링(EANS) 전략을 통해 학습 안정성을 확보합니다.
  - MMEB(Massive Multimodal Embedding Benchmark)와 실제 산업 현장의 프로덕션 데이터셋에서 **SOTA(State-of-the-Art)** 성능을 달성하며 범용 멀티모달 임베딩의 확장 가능성을 입증했습니다.

* [LLM2VEC-GEN: Generative Embeddings from Large Language Models](https://arxiv.org/abs/2603.10913)
  * 입력을 직접 인코딩하는 기존 방식 대신, LLM이 생성할 잠재적 응답을 인코딩하는 새로운 자기지도 학습 패러다임 **LLM2VEC-GEN**을 제안합니다.
  * 학습 가능한 특수 토큰을 활용해 LLM의 응답을 고정 길이 임베딩으로 압축하며, LLM 백본을 동결한 상태에서도 효율적인 학습이 가능합니다.
  * MTEB 벤치마크에서 자기지도 학습 SOTA를 달성했으며, 유해 콘텐츠 검색 성능 감소 및 추론 능력 향상 등 실질적인 임베딩 품질 개선을 입증했습니다.

* [Llama-Embed-Nemotron-8B: A Universal Text Embedding Model for Multilingual and Cross-Lingual Tasks](https://arxiv.org/abs/2511.07025)
  * NVIDIA에서 개발한 오픈 웨이트 텍스트 임베딩 모델 **llama-embed-nemotron-8b**로, MMTEB 리더보드에서 1위를 달성한 고성능 범용 모델입니다.
  * 1,610만 개의 대규모 학습 데이터와 합성 데이터 생성(SDG) 전략을 활용한 2단계 학습 시스템을 통해 뛰어난 다국어 처리 및 크로스링구얼 성능을 확보했습니다.
  * Contrastive loss 실용 가이드와 SDG 전략 평가 등 상세한 ablation study 결과를 공개하여, 지시문(Instruction) 기반 임베딩 성능 향상을 위한 방법론을 제시했습니다.


# AI Daily Picks(20260319)

* [What are embedding models? A complete guide](https://www.openlayer.com/blog/post/what-are-embedding-models-complete-guide)
  * 임베딩 모델은 텍스트, 이미지 등의 비정형 데이터를 숫자로 이루어진 벡터로 변환하여 데이터 간의 의미적 유사성을 계산할 수 있게 해주는 알고리즘입니다.
  * 트랜스포머 아키텍처를 기반으로 토큰화와 어텐션 메커니즘을 사용하여 단어의 문맥적 의미를 정밀하게 파악하고 다차원 벡터 공간에 매핑합니다.
  * RAG(검색 증강 생성) 파이프라인에서 사용자 질문과 가장 관련성이 높은 문서를 벡터 DB에서 빠르게 검색하여 LLM에 컨텍스트로 제공하는 핵심 엔진입니다.
  * 보안성과 속도가 중요한 경우 오픈소스(BGE, MiniLM 등)를, 편의성과 확장성이 중요한 경우 상용 API(OpenAI, Cohere 등)를 선택하는 등 목적에 맞는 모델 선택이 필요합니다.
* [Embedding Models in 2026: Provider Options, Pros, Cons, and Practical Architecture Choices](https://www.stackspend.app/resources/blog/embedding-models-2026-options-pros-cons)
  * OpenAI, Cohere, Google Vertex AI, Amazon Bedrock 등 주요 제공자별 2026년 임베딩 모델의 장단점과 특징을 비교 분석합니다.
  * Anthropic은 자체 임베딩 모델 대신 Voyage AI 등 타사 솔루션을 권장하며, xAI Grok은 컬렉션 기반 임베딩 방식을 제공합니다.
  * Hugging Face를 통한 오픈소스 모델 호스팅 방식은 유연성과 비용 최적화 측면에서 강점이 있지만, 운영 및 평가의 복잡도가 높습니다.
  * 프로덕션 환경의 기본 아키텍처로 관리형 벤더 모델(OpenAI 등)로 시작하여 점진적으로 오픈소스 및 전문 임베딩 모델(Voyage, Jina 등)을 벤치마킹하고, 생성과 임베딩 계층을 분리할 것을 권장합니다.
* [How to Cut LLM Inference Costs with KV Caching](https://blog.purestorage.com/purely-technical/cut-llm-inference-costs-with-kv-caching/)
  * LLM 추론 시 프롬프트 처리(Prefill) 단계의 막대한 연산 비용을 줄이기 위해, 계산된 KV 캐시를 GPU 로컬 메모리(HBM)에만 두지 않고 전역 스토리지 계층(ICMS 등)에 저장하여 재사용하는 아키텍처가 도입되고 있습니다.
  * FlashBlade와 같은 고성능 스토리지를 통한 다이렉트 캐시 주입 기술은 컴퓨팅 헤비한 GPU 연산을 스토리지 I/O 대역폭으로 대체시켜, 첫 토큰 도달 시간(TTFT)을 획기적으로 단축하고 GPU 자원 낭비를 막습니다.
  * 블록 해싱 기반으로 재사용 캐시를 매칭하며, 엔터프라이즈 보안 요구사항에 따라 의도치 않은 프라이빗 데이터 활용 및 해시 충돌을 막기 위해 철저한 테넌트 고립(Isolation) 설계가 병행되어야 합니다.
  * 각각의 인퍼런스 서버간 텐서 병렬화(TP) 설정이 다를 경우 캐시의 호환성이 담보되지 않으므로, 룩업 시 이를 고려하여 트래픽을 영리하게 분배하는 GPU 토폴로지 인지 스케줄러의 도입이 필수적입니다.
* [OpenClaw, 그리고 그 대안들에 대해 알아봅시다](https://turingpost.co.kr/p/openclaw)
  * OpenClaw는 텔레그램, 디스코드 등의 메시징 채널과 연동하여 사용자의 환경과 선호도를 반영한 맞춤형 '개인 AI 비서'를 구축할 수 있게 해주는 오픈소스 프레임워크입니다.
  * 에이전트의 정체성, 프롬프트 규칙, 메모리를 복잡한 코드가 아닌 일반 마크다운 파일(SOUL.md, MEMORY.md 등)로 정의하여 에이전트를 내구성 있고 버전 관리가 가능한 인프라로 취급합니다.
  * 표면적으로는 챗봇 같지만, 중앙 Gateway 프로세스가 세션 라우팅, 툴 호출, 장기 메모리를 조율하며 언어 모델과 메시징 플랫폼 사이의 컨트롤 플레인(Control Plane) 역할을 수행합니다.
  * OpenClaw의 철학에 영감을 받아, 초경량 구동(PicoClaw), 샌드박스 기반의 강력한 보안(IronClaw), 5달러짜리 하드웨어 구동(MimiClaw) 등 다양한 형태의 대안 프로젝트들이 파생되고 있습니다.


# AI Daily Picks(20260318)

* [Octen Series: Optimizing Embedding Models to #1 on RTEB Leaderboard - Octen Blog](https://octen-team.github.io/octen_blog/posts/octen-rteb-first-place/)
  * MTEB가 새롭게 런칭한 RTEB(Retrieval Embedding Benchmark) 리더보드에서 1위를 달성한 Octen 임베딩 모델의 기법을 소개합니다.
  * 법률, 금융, 의료, 코드 등 주요 산업 도메인 특성에 맞춘 합성 데이터를 선별적으로 구축하고 쿼리 다양성을 강화했습니다.
  * Hard Negative 샘플링, Multi-Positive 데이터 활용, False Negative 실시간 필터링을 도입해 데이터 활용도와 판별 정확도를 높였습니다.
  * Qwen3 기반의 LoRA 파인튜닝과 다중 GPU 간의 Negative 공유, 동적 배치 사이즈 조절을 통해 대규모 훈련의 효율성을 극대화했습니다.
* [LLM 추론 최적화 완벽 가이드: vLLM, TensorRT-LLM, Speculative Decoding - Chaos and Order](https://www.youngju.dev/blog/llm/2026-03-14-llm-inference-optimization-vllm-tensorrt-speculative-decoding)
  * LLM 서빙 시 메모리 활용을 높이기 위한 PagedAttention 기반 KV Cache 구조 및 Continuous Batching 등 vLLM의 고성능 추론 엔진 최적화 기술을 소개합니다.
  * 복잡한 빌드 과정을 개선하고 최신 양자화(FP8 등) 및 병렬화 기법을 도입하여 추론 성능을 극대화한 NVIDIA TensorRT-LLM 1.0의 핵심 변화를 설명합니다.
  * 빠르고 작은 드래프트 모델의 초안 생성을 활용해 품질 저하 없이 디코딩 속도를 비약적으로 높여주는 Speculative Decoding의 원리를 다룹니다.
* [GitHub - hemingkx/SpeculativeDecodingPapers: 📰 Must-read papers and blogs on Speculative Decoding ⚡️](https://github.com/hemingkx/SpeculativeDecodingPapers)
  * LLM 추론 속도 향상을 위한 핵심 기술인 Speculative Decoding과 관련된 중요 논문 및 블로그 포스트들을 모아둔 저장소입니다.
  * 기본 개념부터 LLM, 다중 모달 모델, 확산 모델 등에 적용된 연구 사례들을 세부 카테고리별로 꼼꼼하게 분류하고 있습니다.
  * 학술 논문뿐만 아니라 Huggingface, vLLM, TensorRT-LLM 등 실제 프레임워크에서의 최적화 기법과 주요 튜토리얼 링크를 함께 제공합니다.
* [vLLM vs SGLang vs LMDeploy: Fastest LLM Inference Engine in 2026? - Prem AI Blog](https://blog.premai.io/vllm-vs-sglang-vs-lmdeploy-fastest-llm-inference-engine-in-2026/)
  * 안정적인 프로덕션 표준 vLLM, 다중 턴 대화에 강점을 가지는 SGLang, 양자화 모델 서빙에 특화된 LMDeploy의 성능과 특징을 비교 분석합니다.
  * SGLang은 RadixAttention 구조를 도입하여 KV 캐시 히트율을 대폭 개선함으로써 에이전트 워크플로우나 다중 턴 챗봇 환경에서 높은 효율성을 발휘합니다.
  * 반면, LMDeploy는 TurboMind 아키텍처를 기반으로 양자화 모델(Int4 등) 구동 시 최소한의 초기 응답 지연 시간(TTFT)과 최고의 성능을 제공합니다.


# AI Daily Picks(20260317)

* [SGLang v0.4: Zero-Overhead Batch Scheduler, Cache-Aware Load Balancer, Faster Structured Outputs](https://lmsys.org/blog/2024-12-04-sglang-v0-4/)
  * Zero-overhead batch scheduler 도입으로 CPU 스케줄링과 GPU 연산을 중첩 처리하여 처리량(throughput)을 1.1배 향상시켰습니다.
  * Cache-aware load balancer를 통해 워커(worker)의 KV 캐시 적중률을 예측 및 라우팅하여 최대 1.9배 처리량 증가와 3.8배 캐시 적중률 향상을 달성했습니다.
  * DeepSeek 모델을 위한 Data parallelism attention(DP Attention)을 구현하여 중복된 KV 캐시 메모리 사용을 줄이고 디코딩 처리량을 최대 1.9배 높였습니다.
  * XGrammar 백엔드 연동을 통해 JSON 디코딩과 같은 구조화된 출력(Structured Outputs) 생성 속도를 기존 대비 최대 10배 이상 가속화했습니다.

* [Mastering LLM Techniques: Inference Optimization](https://developer.nvidia.com/blog/mastering-llm-techniques-inference-optimization)
  * LLM 추론은 고도의 병렬 처리가 가능한 프리필(prefill) 단계와 메모리 대역폭이 제한되는 자동 회귀적 디코딩(decode) 단계로 나뉩니다.
  * 디코딩 단계 최적화를 위한 KV(Key-Value) 캐싱은 메모리 사용량이 많으나, PagedAttention 기술을 통해 KV 캐시를 비연속적 공간에 저장하여 메모리 낭비를 줄이고 배치 크기(throughput)를 향상시킬 수 있습니다.
  * 파이프라인, 텐서 및 시퀀스 병렬 처리 기법과 MQA, GQA, FlashAttention 등 어텐션 메커니즘 최적화를 통해 GPU 당 메모리 풋프린트를 크게 줄일 수 있습니다.
  * In-flight batching (동시 다중 요청 처리) 및 Speculative inference (추측 기반 병렬 실행)와 같은 향상된 모델 서빙 기법을 이용하면 처리량을 더욱 높일 수 있습니다.

# AI Daily Picks(20260316)

* [What Is Neocloud? Everything You Need to Know](https://phoenixnap.com/blog/neocloud)
  * 네오클라우드(Neocloud)는 AI/ML 학습, HPC(고성능 컴퓨팅), GPU 렌더링 등 컴퓨팅 집약적이고 현대적인 워크로드 처리에 초점을 맞춘 고성능 특수 목적 클라우드 플랫폼임
  * 다양한 범용 서비스 확장에 주력하는 대형 클라우드 제공업체와 달리, 고성능 GPU 인프라, 워크로드 기반 SLA, 컨테이너 네이티브 오케스트레이션 및 투명한 종량제 요금 모델을 통해 높은 비용 효율성 및 성능 최적화를 제공함
  * 생성형 AI 및 엣지 컴퓨팅의 폭발적인 수요 증가 속에서 예측 가능한 성능과 유연한 하드웨어 제어를 요구하는 엔터프라이즈 환경에서 새로운 인프라 대안으로 주목받고 있음

* [2026 Agentic Coding Trends Report: How coding agents are reshaping software development](https://drive.google.com/file/d/13uu-fIhBoB975g3OJVrnSjbP63tTktf8/view)
  * 개발자 역할의 근본적 변화 (구현자에서 오케스트레이터로): 소프트웨어 엔지니어링의 중심이 단순히 코드를 작성하는 것에서 시스템 아키텍처 설계, 에이전트 팀 조율 및 최종 결과물 평가로 이동하고 있습니다.
  * 소프트웨어 개발 생명주기(SDLC)의 극적인 단축: 새로운 코드베이스에 대한 온보딩 시간이 몇 주에서 몇 시간으로 줄어들고, 수개월이 걸리던 대규모 리팩토링이나 기능 개발이 AI의 맥락 이해 능력을 통해 획기적으로 빨라지고 있습니다.
  * 에이전틱 코딩의 민주화와 스택 확장: 전문 개발자뿐만 아니라 보안, 디자인, 데이터 분석 등 다양한 직군의 비전문가들도 코딩 에이전트를 활용해 복잡한 자동화와 문제 해결을 수행할 수 있게 되며, 개발과 비개발 업무 사이의 경계가 허물어지고 있습니다.
  * 협업 에이전트 시스템의 보편화: 단일 에이전트를 넘어 여러 에이전트가 서로 협력하여 복잡한 시스템을 자율적으로 구축하는 에이전트 팀 모델이 등장합니다.

* [X 쓰레드 정리: 기존 트랜스포머의 한계를 극복하는 Kimi 팀의 'Attention Residuals' 아키텍처](https://x.com/_avichawla/status/2033472650836914495)
  * 문제점: 표준 트랜스포머에서는 모든 계층이 동일한 중요도를 가져 깊은 계층의 기여도가 묻히는 "PreNorm 희석" 문제가 발생함
  * 해결책: 고정 가중치 대신 각 계층에 학습된 쿼리 기반 소프트맥스 어텐션을 적용하여 기여도를 동적으로 조절하는 'Attention Residuals' 제안
  * 최적화: 계층들을 블록으로 묶는 'Block Attention Residuals' 방식을 통해 메모리 오버헤드를 $O(Nd)$ 로 대폭 최적화
  * 결과: 베이스라인 대비 훨씬 개선된 벤치마크 점수를 기록하면서도 지연 시간(latency) 오버헤드를 2% 미만으로 유지

