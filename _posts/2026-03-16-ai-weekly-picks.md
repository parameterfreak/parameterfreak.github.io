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

