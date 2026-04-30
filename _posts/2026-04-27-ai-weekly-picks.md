---
title: 'AI Weekly Picks(18주차)'
date: 2026-04-27
permalink: /posts/2026/04/ai-weekly-picks-202618/
categories:
  - Misc
tags:
  - AI
  - News
  - Reference
---

# AI Daily Picks(20260430)

* [GitHub - gauravfs-14/awesome-jepa](https://github.com/gauravfs-14/awesome-jepa)
  * Yann LeCun과 Meta AI가 제안한 자가지도 학습(Self-supervised learning) 패러다임인 JEPA(Joint Embedding Predictive Architecture) 관련 자료를 모아둔 큐레이션 저장소입니다.
  * 픽셀이나 토큰을 재구성하지 않고 미래의 추상적 표현을 예측하여 학습하는 JEPA의 핵심 개념을 기반으로 한 도구, 라이브러리, 연구 논문, 튜토리얼 등을 종합적으로 제공합니다.
  * 비전, 언어, 의료, 물리, 시계열 등 다양한 도메인에 적용된 최신 연구와 응용 사례들을 꾸준히 업데이트하여 연구자들에게 유용한 지식 허브 역할을 합니다.

* [KAME: Tandem Architecture for Enhancing Knowledge in Real-Time Speech-to-Speech Conversational AI](https://pub.sakana.ai/kame/)
  * 실시간 Speech-to-Speech(S2S) 모델의 빠른 응답성과 백엔드 LLM의 풍부한 지식을 결합한 KAME 아키텍처를 제안했습니다.
  * S2S 모델이 즉각적으로 응답을 시작하는 동시에 백엔드 LLM이 병렬로 실행되며 지식 신호를 비동기적으로 주입하는 "말하면서 생각하기" 패러다임을 구현했습니다.
  * 작업에 따라 백엔드 LLM(gpt-4.1, claude-opus-4-1 등)을 유연하게 교체할 수 있어, 낮은 지연 시간을 유지하면서도 전문적이고 풍부한 응답을 제공합니다.

* [Updating 1T parameters in seconds - P2P weight transfer in Large Scale Distributed RL](https://www.lmsys.org/blog/2026-04-29-p2p-update/)
  * 대규모 분산 RL(강화학습) 환경에서 SGLang을 위한 RDMA 기반의 P2P(Peer-to-Peer) 가중치 업데이트 메커니즘을 도입하여 기존 NCCL 브로드캐스트의 한계를 극복했습니다.
  * 소스 측 CPU 엔진 복제본과 Mooncake TransferEngine을 활용하여 1T 파라미터(Kimi-K2) 모델의 가중치 전송 시간을 53초에서 7.2초로 7배 단축했습니다.
  * 기존의 중앙 집중식 브로드캐스트에서 벗어나 트레이너와 추론 랭크 간의 독립적이고 병렬적인 P2P 매핑 및 Zero-Copy 전송을 구현하여 네트워크 중복을 줄이고 대역폭 활용도를 크게 높였습니다.

# AI Daily Picks(20260428)

* [What's new in Foundry Labs - April 2026 - Microsoft Community Hub](https://techcommunity.microsoft.com/blog/azure-ai-foundry-blog/whats-new-in-foundry-labs---april-2026/4509714)
  * **MAI-Transcribe-1, Voice-1, Image-2**: 마이크로소프트의 자체 개발 AI 스택으로, 높은 정확도의 음성 인식/생성 및 텍스트-이미지 생성 모델 공개.
  * **Harrier-oss-v1**: 94개 언어를 지원하며 지시어 기반(instruction-tuned) 맞춤 설정이 가능한 오픈소스 다국어 텍스트 임베딩 모델(270M, 0.6B, 27B).
  * **Phi-4-Reasoning-Vision-15B**: 15B의 컴팩트한 크기에도 차트, 수식, GUI 화면 등을 이해하고 구조화된 추론을 할 수 있는 소형 비전 추론 모델.
  * **VibeVoice ASR & GigaTIME**: 최대 60분 오디오의 다중 화자 음성 인식과 화자 분리를 한 번에 처리하는 VibeVoice ASR, 그리고 저비용 병리 슬라이드를 고해상도 이미지로 변환해 종양 면역 미세환경 분석을 돕는 GigaTIME 모델 소개.

# AI Daily Picks(20260427)

* [Microsoft Open-Sources Harrier, A New Embedding Blockbuster Model For The Agentic Web](https://msftnewsnow.com/microsoft-open-sources-harrier-embedding-model)
  * 마이크로소프트가 다국어 MTEB v2 벤치마크에서 최고 성능을 달성한 새로운 임베딩 모델 제품군인 Harrier-OSS-v1을 오픈소스로 공개했습니다.
  * 기존 BERT 방식이 아닌 디코더 전용 트랜스포머 아키텍처를 채택하여 최대 32,768 토큰의 긴 문맥을 청킹 없이 네이티브로 처리할 수 있습니다.
  * 27B 플래그십 모델부터 0.6B, 270M 파라미터의 경량 모델까지 제공되며, RAG 시스템 및 다국어 지원 에이전트 구축에 최적화되어 있습니다.

