---
title: 'AI Weekly Picks(23주차)'
date: 2026-06-01
permalink: /posts/2026/06/ai-weekly-picks-202623/
categories:
  - Misc
tags:
  - AI
  - News
  - Reference
---

# AI Daily Picks(20260602)

* [WeKnora — 문서를 살아있는 지식으로: RAG · Agent 추론 · 자동 Wiki 통합 LLM 지식 프레임워크](https://github.com/Tencent/WeKnora/blob/main/README_KO.md)
  * 엔터프라이즈급 문서 이해, 시맨틱 검색, 자율 추론을 위해 설계된 텐센트의 오픈소스 LLM 기반 지식 프레임워크입니다.
  * RAG 기반 Q&A, 복잡한 다단계 작업을 처리하는 ReAct Agent 추론, 원본 문서에서 마크다운 지식베이스와 인터랙티브 지식 그래프를 자동 생성하는 Wiki 모드를 핵심 기능으로 제공합니다.
  * 다양한 플랫폼(Feishu, Notion 등) 데이터 연동, 20개 이상의 주요 LLM 지원, 세분화된 엔터프라이즈 테넌트 권한 관리(RBAC) 등 실무 환경에 필요한 통합 환경을 제공합니다.

* [LLM Routing](https://outcomeschool.com/blog/llm-routing)
  * LLM 라우팅은 사용자의 쿼리를 분석하여 비용, 지연 시간, 품질을 기준으로 가장 적합한 LLM 모델에 요청을 전달하는 시스템입니다.
  * 단순한 쿼리는 작고 빠른 저비용 LLM에, 복잡한 쿼리는 크고 강력한 고비용 LLM(Frontier LLM)에 할당하여 품질 저하 없이 비용을 크게 절감할 수 있습니다.
  * 라우팅 전략으로는 규칙 기반(Rule-based), 분류기 기반(Classifier-based), 임베딩 기반(Embedding-based), LLM 기반(LLM-as-Router), 캐스케이드(Cascade) 라우팅 등이 있으며, 이 중 캐스케이드 방식이 가장 비용 효율적입니다.

* [mKernel: Fast Multi-GPU, Multi-Node Fused Kernels](https://uccl-project.github.io/posts/mkernel/)
  * mKernel은 노드 내 NVLink 통신, 노드 간 RDMA, 그리고 연산을 단일 영구 커널(persistent kernel) 내에 융합(fuse)한 빠르고 효율적인 멀티 GPU, 멀티 노드 커널 모음입니다.
  * 기존의 호스트(CPU) 주도형 통신은 통제 오버헤드로 인해 최신 AI 작업에서 파이프라인 버블을 발생시키며 병목 현상의 주요 원인이 되고 있습니다.
  * 이를 해결하기 위해 mKernel은 GPU 주도형(GPU-driven) 통신 방식을 채택하여, GPU가 직접 세밀한 데이터 전송을 제어하고 통신과 연산을 커널 단위에서 융합함으로써 병렬 처리 효율을 극대화합니다.

# AI Daily Picks(20260601)

* [State of AI Agent Memory 2026: Benchmarks, Architectures & Production Gaps](https://mem0.ai/blog/state-of-ai-agent-memory-2026)
  * AI 에이전트 메모리가 단순한 컨텍스트 확장을 넘어 LoCoMo, LongMemEval, BEAM 같은 표준 벤치마크를 갖춘 독립적인 핵심 아키텍처 요소로 자리 잡았습니다.
  * 최신 알고리즘은 단일 패스 추출과 다중 신호 검색(시맨틱, 키워드, 엔티티 매칭 결합)을 통해 메모리 성능을 높였으며, 특히 시간적 추론과 멀티홉 추론에서 큰 향상을 보였습니다.
  * 생태계 내 다양한 프레임워크와 벡터 저장소 연동이 크게 성장했지만, 대규모 시계열 추상화, 교차 세션 신원 확인, 메모리 최신성 유지 등의 해결해야 할 운영 과제들이 여전히 남아 있습니다.
