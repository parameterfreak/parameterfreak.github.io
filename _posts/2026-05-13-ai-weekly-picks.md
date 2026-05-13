---
title: 'AI Weekly Picks(20주차)'
date: 2026-05-06
permalink: /posts/2026/05/ai-weekly-picks-202620/
categories:
  - Misc
tags:
  - AI
  - News
  - Reference
---

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
