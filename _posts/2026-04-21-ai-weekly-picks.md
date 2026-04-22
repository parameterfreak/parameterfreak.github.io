---
title: 'AI Weekly Picks(17주차)'
date: 2026-04-21
permalink: /posts/2026/04/ai-weekly-picks-202617/
categories:
  - Misc
tags:
  - AI
  - News
  - Reference
---

# AI Daily Picks(20260423)

* [Production-Grade LLM Inference at Scale with KServe, llm-d, and vLLM](https://llm-d.ai/blog/production-grade-llm-inference-at-scale-kserve-llm-d-vllm)
  * 수백 개의 대형 언어 모델(LLM)을 스케일링하고 관리하는 과정에서 기존 vLLM의 단순 배포 방식이 가진 스토리지 지연, 인프라 종속성, 비효율적 로드 밸런싱 문제를 KServe, llm-d, vLLM 조합으로 해결한 사례를 공유합니다.
  * KServe의 LLMInferenceService를 통해 표준 Kubernetes API 수준에서 vLLM 설정을 정밀하게 커스터마이징하여 하드웨어 특성에 맞는 유연한 배포를 가능하게 했습니다.
  * Envoy 및 Gateway API Inference Extension을 활용한 접두어 캐시 인식 라우팅(prefix-cache aware routing)을 적용하여 GPU 효율성을 극대화했으며, 실제 Llama 3.1 70B 모델 배포에서 출력 토큰 생성 속도를 3배, 첫 토큰 응답 시간(TTFT)을 절반으로 단축하는 성과를 얻었습니다.

* [Microsoft Open-Sources Embedding Models to Improve AI Data Retrieval](https://pureai.com/articles/2026/04/09/microsoft-open-sources-embedding-models-to-improve-ai-data-retrieval.aspx)
  * 마이크로소프트가 단순한 키워드 매칭을 넘어 의미론적 연관성을 파악해 검색, 추천 및 데이터 검색 시스템의 기반을 이루는 새로운 AI 임베딩 모델 제품군(Harrier)을 오픈소스로 공개했습니다.
  * 엔터프라이즈 AI가 실험 단계에서 실제 운영 환경으로 전환됨에 따라, 생성된 결과물의 정교함보다 검증된 실제 데이터에 기반하여 환각(오류)을 줄이고 신뢰성을 높이는 검색 증강 생성(RAG) 및 에이전트 시스템의 중요성이 커지고 있습니다.
  * 100개 이상의 언어를 지원하고 다양한 환경에서 구동 가능한 이 고성능 임베딩 모델의 오픈소스화는, 개발자들이 폭넓은 애플리케이션에 이를 쉽게 통합할 수 있게 함으로써 기업의 내부 검색 도구 강화 및 AI 어시스턴트 성능 향상에 실질적으로 기여할 것으로 기대됩니다.


# AI Daily Picks(20260422)

* [From Models to Scientists: Building AI Agents for Scientific Discovery - Kempner Institute](https://kempnerinstitute.harvard.edu/research/deeper-learning/from-models-to-scientists-building-ai-agents-for-scientific-discovery)
  * 하버드 Kempner Institute에서 개발한 **ToolUniverse**는 대규모 언어 모델(LLM)이 600개 이상의 과학 도구(머신러닝 모델, DB, 시뮬레이터 등)를 활용해 연구를 기획하고 수행할 수 있게 해주는 AI 과학자 에이전트 구축 프레임워크입니다.
  * 4가지 핵심 컴포넌트(Manager, Composer, Discover, Optimizer)를 제공하여, 단순한 도구 호출(MCP)을 넘어 다단계 과학적 추론과 워크플로우를 실행하고 새로운 도구를 자율적으로 생성 및 최적화할 수 있도록 지원합니다.
  * 실제 사례로 AI 에이전트가 단백질 타겟 탐색부터 화합물 식별, 구조 최적화 및 특허 검증에 이르는 약물 발견의 전 과정을 시뮬레이션하여 인간 연구자의 강력한 협력자(Co-scientist)로 활용될 수 있음을 보여주었습니다.

# AI Daily Picks(20260421)

* [Microsoft's Harrier Embeds 32K Tokens at Once - AI Beat](https://ai-beat.github.io/news/2026/03/harrier-oss-multilingual-embeddings/)
  * 최대 32,768 토큰(기존 대비 30~60배)의 컨텍스트 윈도우를 지원하는 디코더 전용 다국어 임베딩 모델 제품군(Harrier-OSS-v1: 270M, 0.6B, 27B)이 MIT 라이선스로 오픈소스 공개되었습니다.
  * 27B 모델은 다국어 MTEB v2 벤치마크에서 74.3점으로 최고 성능(SOTA)을 달성했으며, 문서를 청킹하지 않고 전체 논문이나 계약서를 한 번에 임베딩할 수 있어 RAG(검색 증강 생성) 시스템의 설계에 변화를 줄 수 있습니다.
  * 디코더 전용 구조(마지막 토큰 풀링 방식)를 채택하여 강력한 성능을 발휘하지만, 양방향 인코더 방식에 비해 추론 비용(Inference cost)이 증가한다는 단점이 있으며 27B 모델은 전용 GPU 인프라를 요구합니다.
* [AI agents, tech circularity: What’s ahead for platforms in 2026 - MIT Sloan](https://mitsloan.mit.edu/ideas-made-to-matter/ai-agents-tech-circularity-whats-ahead-platforms-2026)
  * 플랫폼 시장 환경은 자율적인 AI 에이전트의 등장에 대비해 에이전트 전용 인터페이스와 통제 규칙을 마련하는 방향으로 변화하고 있습니다.
  * 생성형 AI의 활발한 도입은 코딩 생산성을 높였으나 최신 시스템에서는 관리가 어려운 불완전한 코드가 축적되는 "기술 부채 폭증(turbocharged technical debt)" 리스크를 발생시키고 있습니다.
  * AI 스택의 소수 빅테크 의존 심화 문제에 대응하기 위해 신중한 AI 툴 도입이 필요하며, 나아가 디지털 플랫폼 전략이 하드웨어 재활용 및 재판매를 통한 순환 경제(circular economy) 영역까지 확장되고 있습니다.
* [TurboQuant KV Cache Compression: What Changes for LLM Inference](https://www.kriraai.com/blog/turboquant-kv-cache-compression-changes-llm-inference)
  * 별도의 재학습 없이(training-free) LLM의 KV 캐시 메모리 사용량을 최대 6배 줄여주는 구글의 압축 기술 'TurboQuant'를 분석한 글로, 장문 컨텍스트 추론의 핵심 병목인 VRAM 한계를 획기적으로 극복할 수 있습니다.
  * 극심한 비대칭성 문제를 지닌 캐시 값 분포를 'PolarQuant' 단계의 직교 회전(Orthogonal rotation)으로 균일화하여 양자화 효율을 극대화하며, 실제 구현 시에는 논문상의 QJL 잔차 교정 대신 보편적인 MSE 전용 양자화(MSE-only quantization)를 사용하는 것이 성능 면에서 우수한 것으로 입증되었습니다.
  * 기존 파라미터 양자화(AWQ 등)와 동시 적용이 가능한 상호보완적 기술로서, 향후 프레임워크 지원이 완료되면 다중 접속 및 장문 입력이 필요한 에이전트/RAG 기반 기업용 AI 환경의 인프라 효율성을 극대화할 것입니다.
* [LLM Inference and KV Cache Complete Guide [2026]: How Token Generation Works - QubitTool](https://qubittool.com/blog/llm-inference-kv-cache-guide)
  * LLM 텍스트 생성 시 이전 토큰에 대한 중복 연산을 방지해주는 'KV Cache(Key-Value Cache)' 메커니즘의 원리를 설명한 상세 가이드입니다.
  * 계산 속도를 O(1) 수준으로 높이는 대신 GPU의 VRAM 메모리 소모가 컨텍스트 길이에 비례해 선형적으로 증가하며, 결국 메모리 관리(Memory bottleneck)가 실제 운영 및 배포 단계에서 가장 중요한 과제가 됨을 강조합니다.
  * 메모리 한계 부딪힘을 극복하기 위한 대표적 최적화 기술로 PagedAttention(메모리 파편화 방지), GQA(멀티 쿼리 헤드의 KV 헤드 공유), KV Cache Quantization(데이터 정밀도 축소) 등을 소개하고 활용을 권장합니다.
* [Microsoft Harrier Disrupts the AI Landscape Dethroning Proprietary Embedding Models — ML Hive](https://mlhive.com/2026/04/microsoft-harrier-open-source-embedding-models-rag)
  * 마이크로소프트가 상용 임베딩 모델(OpenAI, Google)의 성능을 능가하는 최대 27B 파라미터 규모의 다국어 오픈소스 임베딩 모델군 'Harrier'를 공개하여 기업용 RAG 구조의 혁신을 주도하고 있습니다.
  * 초거대 파라미터를 기반으로 단어 구분의 혼동(Semantic crowding) 문제를 해결함으로써 컨텍스트의 미세한 차이를 예리하게 분별해내며, 번역 과정 없이도 100개 이상의 언어를 동일한 다국어 벡터 공간으로 일관성 있게 맵핑할 수 있습니다.
  * 최상위 성능의 모델을 오픈소스로 무료 공개한 것은 검색 시스템 구축 문턱을 허물고 엔터프라이즈 AI 애플리케이션의 확산시키며, 나아가 이와 맞물린 마이크로소프트 자체 클라우드(Azure) 인프라 수요를 촉진하기 위한 전략적 선택("commoditizing your complement")으로 평가됩니다.

