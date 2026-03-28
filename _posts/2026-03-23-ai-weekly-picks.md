---
title: 'AI Weekly Picks(13주차)'
date: 2026-03-23
permalink: /posts/2026/03/ai-weekly-picks-202613/
categories:
  - Misc
tags:
  - AI
  - News
  - Reference
---

# AI Daily Picks(20260326)

* [KV-Cache Wins You Can See: From Prefix Caching in vLLM to Distributed Scheduling with llm-d](https://llm-d.ai/blog/kvcache-wins-you-can-see)
  * 대화형 AI 및 에이전트 워크플로우 같은 프롬프트(접두사) 비중이 높은 환경에서 vLLM의 캐싱은 핵심적이나, 이를 분산 환경 확장 시 스케줄러가 캐시 위치를 파악하지 못해 비용과 응답 지연 문제가 발생합니다.
  * llm-d는 vLLM 각 포드의 실시간 KVEvents 스트림을 통해 클러스터 전체의 KV-Block 캐시 인덱스를 글로벌 뷰로 관리합니다.
  * 이를 바탕으로 스케줄러 내 지정 기능(Precise Prefix-Cache Scorer)이 개별 포드의 프리픽스 캐시 보유 비율을 계산해 가장 최적화된 포드로 요청을 라우팅합니다.
  * 벤치마크 결과, 근사적 로드 밸런싱 대비 사용자 응답 속도(TTFT)가 57배 빠르며 전체 시스템 처리량을 25% 향상시켰습니다.

* [Understanding KV Cache in LLM Inference](https://jingchaozhang.github.io/understanding-kv-cache/)
  * LLM 추론 시 GPU 메모리를 가장 많이 차지하는 것은 모델 가중치가 아니라 KV 캐시입니다. KV 캐시는 시퀀스 길이에 완벽하게 선형적으로 비례하며, 컨텍스트가 길어지거나 동시 요청(Batch Size)이 증가할수록 메모리 사용량이 급증합니다.
  * Grouped Query Attention(GQA)는 필수적인 최적화 기법으로, 여러 쿼리 헤드가 K/V 헤드를 공유하여 KV 캐시 크기를 획기적으로 줄여줍니다 (예: Qwen2.5-7B는 86% 감소).
  * Prefill 단계는 연산 중심(Compute-bound)인 반면, Decode 단계는 메모리 대역폭 중심(Memory-bandwidth-bound)으로 두 단계의 병목 원인이 완전히 다름을 실험으로 증명했습니다.

# AI Daily Picks(20260325)

* [Distributed KV Cache Management and Systems Architecture for Long-Context LLM Inference](https://uplatz.com/blog/distributed-kv-cache-management-and-systems-architecture-for-long-context-llm-inference/)
  * LLM의 컨텍스트 윈도우가 확장(128k ~ 1,000만+ 토큰)됨에 따라 추론 시스템의 병목 현상이 계산 능력(FLOPS)에서 메모리 용량 및 대역폭(KV Cache)으로 이동했음을 분석합니다.
  * Ring Attention, TokenRing, LMCache, NVIDIA Dynamo와 같은 기술을 통해 KV Cache를 일시적인 버퍼가 아닌, 클러스터 전체에서 관리되는 분산형 자산으로 전환하는 아키텍처 변화를 설명합니다.
  * CXL(Compute Express Link) 기반의 메모리 확장 하드웨어와 Jamba(Hybrid SSM-Transformer), Infini-attention(압축 메모리)과 같은 알고리즘 혁신이 대규모 컨텍스트 처리를 위한 핵심 요소임을 강조합니다.

* [How NVIDIA Dynamo 1.0 Powers Multi-Node Inference at Production Scale - NVIDIA Technical Blog](https://developer.nvidia.com/blog/nvidia-dynamo-1-production-ready/)
  * NVIDIA Dynamo 1.0은 대규모 멀티 노드 AI 배포를 위한 가용성 높은 분산 추론 프레임워크로, 주요 클라우드 플랫폼(AWS, Azure, GCP 등) 및 오픈소스 엔진(SGLang, vLLM 등)과 통합되어 최대 7배의 처리량 향상을 제공합니다.
  * 에이전트 기반 추론 최적화(Priority-based routing, Cache pinning), 멀티모달 가속(분리된 E/P/D 단계, 임베딩 캐시), 그리고 비디오 생성 모델 지원을 통해 복잡한 생성형 AI 워크로드의 효율성을 극대화합니다.
  * 쿠버네티스 네이티브 배포(DGDR), 계층적 결함 탐지 및 요청 마이그레이션 기능을 통한 고가용성 보장, 그리고 다양한 스토리지 계층(S3 등)을 지원하는 유연한 KV 블록 관리 시스템을 갖추고 있습니다.

* [KV-Embedding: KV 재라우팅을 통한 냉동 LLM의 시퀀스 레벨 컨텍스트 활성화](https://arxiv.org/abs/2601.01046)
  * 추가적인 학습 없이 냉동(Frozen) LLM의 KV 상태를 재라우팅하여 정보 비대칭 문제를 해결하고 시퀀스 레벨 컨텍스트 표현을 가능하게 하는 KV-Embedding 프레임워크를 제안합니다.
  * 인과적 어텐션의 한계를 극복하기 위해 각 레이어의 마지막 토큰 KV를 입력 앞에 접두어로 배치하여 모든 토큰이 한 번의 순전파로 전체 문맥에 접근할 수 있도록 설계되었습니다.
  * MTEB 벤치마크에서 기존 비학습 기반 방식 대비 최대 10%의 성능 향상을 보였으며, 내재적 차원 기반의 자동 레이어 선택과 하이브리드 풀링 전략의 효과를 입증했습니다.

# AI Daily Picks(20260324)

* [LMCache: Slash LLM Inference Latency and Multiply Throughput with Enterprise-Grade KV Cache Reuse](https://www.papercodex.com/lmcache-slash-llm-inference-latency-and-multiply-throughput-with-enterprise-grade-kv-cache-reuse)
  * 대규모 LLM 추론 시 발생하는 KV 캐시 메모리 병목 현상을 해결하기 위해 고안된 빠르고 유연한 오픈소스 KV 캐시 계층입니다.
  * KV 캐시를 GPU 메모리에만 두지 않고 GPU, CPU, 로컬 디스크, 원격 스토리지 등 다중 계층 스토리지에 분산 저장하여 재사용성과 확장성을 극대화합니다.
  * 프롬프트 내 단순 접두사 일치뿐만 아니라 반복되는 모든 텍스트 시퀀스를 재사용할 수 있으며, vLLM, SGLang과 원활하게 통합되어 첫 토큰 생성 시간(TTFT)을 단축하고 처리량을 크게 향상시킵니다.

* [KV-Embedding: KV 재라우팅을 통한 냉동 LLM의 시퀀스 레벨 컨텍스트 활성화](https://arxiv.org/abs/2601.01046)
  * 추가적인 학습 없이 냉동(Frozen) LLM의 KV 상태를 재라우팅하여 정보 비대칭 문제를 해결하고 시퀀스 레벨 컨텍스트 표현을 가능하게 하는 KV-Embedding 프레임워크를 제안합니다.
  * 인과적 어텐션의 한계를 극복하기 위해 각 레이어의 마지막 토큰 KV를 입력 앞에 접두어로 배치하여 모든 토큰이 한 번의 순전파로 전체 문맥에 접근할 수 있도록 설계되었습니다.
  * MTEB 벤치마크에서 기존 비학습 기반 방식 대비 최대 10%의 성능 향상을 보였으며, 내재적 차원 기반의 자동 레이어 선택과 하이브리드 풀링 전략의 효과를 입증했습니다.

# AI Daily Picks(20260323)

* [Normal Inference vs KV Cache vs LMCache](https://www.f22labs.com/blogs/normal-inference-vs-kvcache-vs-lmcache/)
  * Normal Inference: LLM의 기본 텍스트 생성 방식으로, 매 단계마다 이전 모든 토큰의 어텐션 상태를 다시 계산하여 시퀀스가 길어질수록 비용이 급증함.
  * KV Cache: 이전 토큰들의 Key-Value 텐서를 저장하여 재사용하는 최적화 기법으로, 새로운 토큰에 대해서만 연산을 수행하여 효율성을 높임.
  * LMCache: KV Cache를 계층적 저장소(GPU, CPU, Disk 등)와 청크 단위 관리로 확장하여, RAG나 멀티턴 대화 등에서 중복 프롬프트의 재사용성을 극대화함.

* [Rethinking LLM Inference Economics with llm-d, LMCache, and IBM Storage Scale](https://community.ibm.com/community/user/blogs/anthony-hsu/2026/02/06/rethinking-llm-inference-economics)
  * LLM 추론에서 컨텍스트가 길어질수록 발생하는 중복 계산(prefill) 문제를 해결하기 위해, KV Cache의 재사용성을 극대화하는 아키텍처를 제안함.
  * `llm-d`(오케스트레이터), `LMCache`(캐시 관리), `IBM Storage Scale`(고성능 파일 시스템)을 결합하여 KV Cache를 여러 요청과 노드 간에 공유하고 지속시킴.
  * 이를 통해 TTFT(첫 토큰 생성 시간)를 획기적으로 단축하고 토큰당 비용을 10배 이상 절감하며, DRAM 수준의 성능을 경제적으로 구현할 수 있음을 입증함.

* [KV-Cache Wins You Can See: From Prefix Caching in vLLM to Distributed Scheduling with llm-d](https://llm-d.ai/blog/kvcache-wins-you-can-see)
  * LLM 추론 시 KV-cache 적중률은 지연 시간(Latency)과 비용을 결정하는 가장 핵심적인 지표로, 단일 인스턴스 환경에서 vLLM의 Prefix Caching은 효율적이지만 분산 환경에서는 캐시가 파편화되어 성능이 저하되는 한계가 있음.
  * llm-d는 KVEvents를 통해 클러스터 전체의 KV-cache 상태를 실시간으로 파악하는 '정밀 접두사 캐시 인식 스케줄링(precise prefix-cache aware scheduling)'을 도입하여 캐시가 있는 파드로 요청을 정확히 라우팅함.
  * 벤치마크 결과, 해당 스케줄링 방식은 기존 또는 캐시를 고려하지 않는 방식보다 첫 토큰 생성 시간(TTFT)을 57배 단축하고 전체 시스템 처리량을 2배 이상 높여 안정적인 성능을 입증함.


