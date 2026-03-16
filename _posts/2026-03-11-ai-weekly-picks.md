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

# AI Daily Picks(20260315)
* [OpenAI gpt-oss-safeguard · Ollama Blog](https://ollama.com/blog/gpt-oss-safeguard)
  * Ollama가 OpenAI 및 ROOST와 협력하여 안전성 분류 작업을 위한 20B/120B 파라미터 규모의 추론(Reasoning) 모델인 'gpt-oss-safeguard'를 Apache 2.0 라이선스로 공개함
  * 자체 정책 지침(Bring your own policy)을 해석하도록 설계되어 있어, 기존 LLM 입출력 필터링이나 콘텐츠 분류 등 다양한 신뢰 및 안전(Trust and Safety) 작업에 최소한의 엔지니어링으로 적용 가능함
  * 단순한 점수가 아닌 '의사결정에 대한 전체 추론 과정'을 제공하여 디버깅을 돕고, 시스템 목적 및 지연 시간에 따라 추론 노력(low, medium, high)을 조절할 수 있음

* [에이전틱 커머스란? AI가 대신 쇼핑하는 시대, 이커머스 브랜드의 생존 전략 - Datarize](https://www.datarize.ai/ko/blog/agentic)
  * 고객이 의도와 조건만 설정하면 AI 에이전트가 탐색부터 결제까지 전 과정을 대행하는 '에이전틱 커머스(제로 클릭 커머스)' 시대가 다가오고 있음
  * 앞으로 검색 최적화 트렌드는 기존의 키워드 중심 SEO에서 벗어나 질문에 대한 답변(AEO) 및 생성형 엔진 최적화(GEO) 방향으로 진화하게 될 것임
  * AI 에이전트에게 지속적으로 선택받기 위해서는 퀄리티 높은 상품 데이터뿐만 아니라, 고객과의 장기적 관계를 구축하는 '데이터 기반 CRM' 역량과 브랜드 신뢰도가 무엇보다 중요함

* [SGLang Destroys vLLM: 3x Faster + 40% Cheaper (2025 H800 Benchmarks) : LLM Practical Experience Hub](https://langcopilot.com/posts/2025-07-07-sglang-disaggregated-llm-inference-architecture-pe)
  * SGLang는 기존 vLLM 대비 3배 빠른 추론(Inference) 속도와 40%의 GPU 비용 절감을 달성한 차세대 대형 언어 모델 추론 아키텍처임
  * 연산 중심의 Prefill 단계와 메모리 중심의 Decode 단계를 분리(Disaggregated Inference)하여, 각 단계의 병목 현상을 해소하고 자원 활용 효율을 극대화함
  * 대규모 KV 캐시 전송으로 인한 지연 시간을 해결하기 위해 Mooncake 엔진 기반의 RDMA(Remote Direct Memory Access) 통신 기술과 상태 머신 스케줄링을 적극 도입함

* [Inference stacks compared: vLLM, TGI, TensorRT-LLM, llama.cpp, and SGLang : Maniac](https://www.maniac.ai/blog/inference-stacks-vllm-tgi-tensorrt)
  * 다양한 LLM 추론 스택(vLLM, TGI, TensorRT-LLM, llama.cpp, SGLang)을 목표 지연 시간(Latency) 및 운영 환경에 맞춰 선택하기 위한 가이드를 제공함
  * vLLM/TGI는 생태계 호환성과 처리량(Throughput)이 중요할 때, TensorRT-LLM은 최고 수준의 성능 최적화가 필요할 때, llama.cpp는 경량화/엣지 배포에, SGLang는 복잡한 프롬프트나 에이전틱 시스템 처리에 강점을 보임
  * 성공적인 프로덕션 도입을 위해서는 오프라인 처리량뿐 아니라 실제 동시성 하에서의 p95 응답 지연 시간, 긴 컨텍스트에서의 메모리 동작, 그리고 장애 대응(Observability) 역량을 전반적으로 평가해야 함

* [NV-Embed: Improved Techniques for Training LLMs as Generalist Embedding Models - 논문 리뷰](https://yoonschallenge.tistory.com/997)
  * 디코더 전용(Decoder-only) 대형 언어 모델(LLM)을 텍스트 임베딩 모델로 최적화하여 기존 양방향(BERT, T5) 모델들의 성능을 뛰어넘는 'NV-Embed' 모델을 제안함
  * 평균 풀링(Mean pooling)이나 토큰 임베딩 대신 '잠재 주의(Latent attention)' 레이어를 도입하여 시퀀스 내 중요 토큰 정보를 잃지 않고 더 풍부한 표현력을 가진 시퀀스 임베딩을 생성함
  * 인배치(In-batch) 네거티브 샘플링을 활용한 검색 중심의 1단계 대조적 훈련과, 네거티브 샘플 없이 인스트럭션 튜닝(Instruction-Tuning) 기반으로 비검색 작업(분류, 클러스터링 등)을 결합하는 2단계 훈련 방식을 통해 범용적인 임베딩 성능을 극대화함

# AI Daily Picks(20260314)
* [X 쓰레드 정리: 심층 연구(Deep Research) 시스템 평가의 한계와 새로운 기준](https://x.com/allen_ai/status/2032189759985258812)
  * 심층 연구 에이전트의 주요 평가 방식인 '쌍대 선호도(pairwise preferences)'는 시스템 순위 산정에는 유용하지만 세부 지표 평가에는 신뢰하기 어렵다는 점을 지적함
  * 이를 개선하기 위해 지표별 특화된 인간의 어노테이션을 설계하고, 점수 일치율뿐 아니라 불일치 원인을 분석할 것을 권장
  * AI 연구 평가 프레임워크는 단일 종합 점수 최적화가 아닌, 전문가들의 다양한 배경과 기대치를 반영하여 다각적으로 모델링되어야 함을 강조
  * 관련 논문("Deep Research, Shallow Evaluation")과 시스템 평가 루브릭을 자동 생성하는 파이프라인 코드(ai2-scholarqa-eval)가 함께 공개됨

* [X 쓰레드 정리: 10x 변호사와 법률 시장의 재편 (The 10x Lawyer)](https://x.com/zackbshapiro/status/2031717962948690355)
  * AI 발전은 시니어 변호사가 여러 어소시에이트 변호사들의 업무를 대체하게 하여, 대형 로펌 수익의 핵심인 '피라미드 팽창 모델'을 붕괴시킴
  * 이를 통해 생산성 병목이 해소되며 결국 개별 실무자의 '탁월한 판단력'이 시장의 핵심 경쟁 요소로 떠오름
  * 향후 법률 시장은 소수의 뛰어난 '10x 변호사'와 저비용으로 평준화된 '상품화된 AI 서비스'로 양극화(바벨 현상)될 것임

* [Modular Manifolds - Thinking Machines Lab](https://thinkingmachines.ai/blog/modular-manifolds/)
  * 신경망 파라미터(예: 트랜스포머의 행렬)를 특정 기하학적 표면(다양체, Manifold)에 제약하여 최적화기를 설계하는 프레임워크를 제안함
  * 가중치 업데이트의 최대 한계치를 제어하는 스펙트럼 정규화(Spectral Norm)와 특이값을 1로 유지하는 스티펠(Stiefel) 다양체를 결합하여 'Manifold Muon' 최적화 알고리즘을 도출함
  * 이러한 제약을 전체 네트워크로 확장하는 '모듈형 다양체(Modular Manifolds)' 추상화를 통해, 립시츠(Lipschitz) 민감도를 추적하여 레이어 간의 학습률(Learning Rate)을 체계적으로 할당하는 방법을 제시함

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
