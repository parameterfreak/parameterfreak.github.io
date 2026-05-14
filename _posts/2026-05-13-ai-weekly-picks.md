---
title: 'AI Weekly Picks(20주차)'
date: 2026-05-13
permalink: /posts/2026/05/ai-weekly-picks-202620/
categories:
  - Misc
tags:
  - AI
  - News
  - Reference
---

# AI Daily Picks(20260515)

* [Teaching 'Fundamentals of Machine Learning' - Kyunghyun Cho](https://kyunghyuncho.me/teaching-fundamentals-of-machine-learning/)
  * 코딩 어시스턴트(AI 툴)를 적극 활용하여 'Fundamentals of Machine Learning' 수업 방식을 전면 개편한 교수의 경험담.
  * 기존의 빈칸 채우기 식 실습에서 벗어나, 매주 배운 ML 개념을 바탕으로 밑바닥부터 완전한 웹 앱을 '바이브 코딩(vibe-coding)'하는 방식으로 수업을 진행함.
  * 학생들에게 자유도를 주고 스스로 AI 도구를 활용해 프로젝트를 완성하게 함으로써, 학생들의 창의성과 문제 해결 능력이 크게 향상됨을 관찰함.
* [How to Make LLM Training Faster with Unsloth and NVIDIA](https://unsloth.ai/blog/nvidia-collab)
  * NVIDIA와의 협업을 통해 Unsloth의 LLM 파인튜닝 속도를 정확도 손실 없이 기존 대비 약 25% 추가 향상시킴.
  * 반복되는 Packed Sequence 메타데이터를 매 레이어마다 재생성하지 않고 캐싱하여 재사용함으로써 14.3%의 속도 향상을 얻음.
  * 그래픽 메모리와 시스템 메모리 간의 Activation 복사와 역전파 연산을 중첩(Double Buffered Async Gradient Checkpointing)시켜 8%의 속도 향상을 달성함.
  * MoE 라우팅 시 동적 쿼리(`torch.where`) 대신 `argsort`와 `bincount`를 활용해 연산을 한 번에 묶어서 처리함으로써 GPT-OSS 학습 속도를 15% 단축함.
* [TokenSpeed: A Speed-of-Light LLM Inference Engine for Agentic Workloads](https://lightseek.org/blog/lightseek-tokenspeed.html)
  * 에이전틱(Agentic) 워크로드에 최적화된 새로운 초고속 LLM 추론 엔진인 TokenSpeed 발표 (LightSeek Foundation, NVIDIA, AMD 등 공동 개발).
  * Control Plane(상태 머신, 자원 관리)과 Execution Plane을 분리한 고성능 스케줄러 및 자동화된 모델링 병렬화 메커니즘을 도입함.
  * Tensor Core 활용도를 극대화한 자체 최적화 커널(TokenSpeed MLA 등)을 적용해, 에이전트 환경에서 TensorRT-LLM 대비 더 높은 처리량(TPM)과 더 낮은 지연 시간(TPS)을 달성함.
* [State of Routing in Model Serving - Netflix TechBlog](https://netflixtechblog.com/state-of-routing-in-model-serving-16e22fe18741)
  * Netflix에서 초당 100만 건 이상의 요청을 처리하기 위해 구축한 중앙 집중식 ML 모델 서빙 플랫폼의 아키텍처 진화 과정 소개.
  * 초기에는 'Switchboard'라는 중앙 프록시를 통해 컨텍스트 기반 라우팅과 A/B 테스트를 처리했으나, 단일 장애점(SPOF) 문제와 네트워크 지연이 발생함.
  * 이를 해결하기 위해 라우팅 정보 리졸브(Lightbulb)와 실제 트래픽 라우팅(Envoy proxy) 계층을 분리하는 구조로 개편하여, 안정성과 성능을 개선함.


# AI Daily Picks(20260513)

* [How to achieve truly serverless GPUs](https://modal.com/blog/truly-serverless-gpus)
  * GPU 인퍼런스 환경의 콜드 스타트를 수십 분에서 수십 초 단위로 단축하여 낭비 없는(Serverless) 리소스 할당을 달성한 엔지니어링 4단계 핵심 요소를 설명함.
  * **클라우드 버퍼 (Cloud buffers):** 건강한 유휴 GPU 버퍼를 미리 유지하여 새로운 인스턴스 할당과 헬스 체크에 소요되는 수십 분의 지연 시간을 제거함.
  * **커스텀 파일 시스템 (ImageFS):** libfuse를 기반으로 티어 구조의 캐시를 구현하여 전체 이미지 로드 없이 컨테이너 실행에 필요한 파일만 지연 로딩(Lazy Loading)함으로써 컨테이너 시작 시간을 단축함.
  * **호스트 및 디바이스 메모리 스냅샷 (Checkpoint/Restore):** gVisor(runsc)를 활용해 프로세스 메모리 상태(Host)를 복원하고, Nvidia 드라이버 기능으로 GPU 메모리(Device)까지 체크포인트 및 복원하여 런타임 초기화와 가중치 로드 시간을 획기적으로 줄임.
* [Agentic RAG](https://outcomeschool.com/blog/agentic-rag)
  * 단순 정보 검색 후 생성을 수행하는 기존 RAG의 한계를 극복하기 위해, AI Agent가 검색의 전 과정을 주도하는 Agentic RAG 개념과 워크플로우를 소개함.
  * 기존 RAG는 한 번의 검색만 수행하여 복잡한 다중 홉(multi-hop) 질문이나 모호한 쿼리 처리에 취약하지만, Agentic RAG는 Agent, 도구(Tools), 루프(Loop)를 활용해 목표 달성 시까지 검색과 평가를 반복함.
  * 대표적인 패턴으로 ReAct, Self-RAG, Corrective RAG(CRAG)가 있으며, 복잡한 쿼리와 다중 데이터 소스 환경에서 뛰어난 성능을 보이나 지연 시간과 비용 증가를 고려해야 함.
