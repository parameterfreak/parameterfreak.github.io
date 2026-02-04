---
title: 'AI Weekly Picks(20251217)'
date: 2025-12-17
permalink: /posts/2025/12/ai-weekly-picks-2025-12-17/
categories:
  - Misc
tags:
  - AI
  - News
  - Reference
---

# AI Weekly Picks(20251217)

# Towards Understanding the Importance of Shortcut Connections in Residual Networks
- [https://arxiv.org/pdf/1909.04653](https://arxiv.org/pdf/1909.04653)
- ResNet에서 shrotcut connections이 훈련에 미치는 영향에 대한 연구
    - CNN에서 30개의 이상의 계층을 갖고 있는 모델이 오히려 얕은 모델보다 성능이 나빠지는 경우가 발생
        - 네트워크가 깊어질수록 역전파 과정에서 기울기가 점점 0에 가까워지는 현상이 나타남
    - ResNet에서는 기울기가 준선형적으로 감소
- 숏컷 연결이 단순히 더하기 연산을 통해 정보 전달하는 것을 넘어 학습 알고리즘이 잘못된 최적점이라는 함정에 빠지지 않도록 안전한 경로를 제공

# **GLM-4.6V: Open Source Multimodal Models with Native Tool Use**
- [https://z.ai/blog/glm-4.6v](https://z.ai/blog/glm-4.6v)
- GLM-4.6V 시리즈(GLM-4.6V(106B) 및 GLM-4.6V-Flash(9B) 포함)는 네이티브 함수 호출 기능을 갖춘 오픈 소스 멀티모달 대규모 언어 모델입니다. 이 모델은 시각적 이해 및 추론에 탁월하며, 멀티모달 입력 및 출력을 지원하고, 리치 텍스트 콘텐츠 생성, 시각적 웹 검색, 프런트엔드 복제와 같은 복잡한 작업을 가능하게 합니다. 이 모델은 다양한 벤치마크에서 최첨단 성능을 달성하며, 긴 시퀀스 모델링, 세계 지식 강화, 에이전트 데이터 합성 등의 기술을 통합합니다.

# vLLM Router: A High-Performance and Prefill/Decode Aware Load Balancer for Large-scale Serving
- [https://blog.vllm.ai/2025/12/13/vllm-router-release.html](https://blog.vllm.ai/2025/12/13/vllm-router-release.html)
- vLLM Router는 대규모 vLLM 배포를 위해 Rust로 작성한 새로운 고성능 prefill/decode 인식 로드 밸런서입니다. 이는 일관된 해싱 및 prefill/decode 분리(disaggregation)에 대한 기본 지원과 같은 지능형 로드 밸런싱 전략을 제공하여 표준 로드 밸런서의 한계를 해결합니다.
- 이 Router는 또한 쿠버네티스 서비스 디스커버리, 내결함성 및 관찰 가능성 기능을 통해 엔터프라이즈급 복원력을 제공합니다. 벤치마크에 따르면 vLLM Router는 throughput 및 TTFT 면에서 llm-d 및 vLLM-native와 같은 대안보다 훨씬 뛰어난 성능을 보입니다.

# Olmo 3: Charting a path through the model flow to lead open-source AI
<img src="/images/post/weekly-picks/251217-1.png" alt="benchmark" width="450"/>
- [https://allenai.org/blog/olmo3](https://allenai.org/blog/olmo3)
- [https://github.com/allenai/OLMo-core](https://github.com/allenai/OLMo-core)
- [https://github.com/allenai/dolma](https://github.com/allenai/dolma)
- Allen AI의 Olmo 3는 데이터부터 훈련, 배포에 이르는 전체 "모델 흐름"과 투명성을 강조하는 오픈 소스 AI 이니셔티브입니다. 이 글은 추론 및 지시 따르기 능력에서 상당한 개선을 보인 Olmo 3.1 Think 32B 및 Olmo 3.1 Instruct 32B를 포함한 Olmo 3.1의 출시를 강조합니다. Olmo 3는 프로그래밍, 수학, 채팅과 같은 다양한 작업을 위해 설계된 소형 고밀도 모델(70억 및 320억 매개변수) 제품군입니다. 이 프로젝트는 최종 모델뿐만 아니라 연구원과 개발자가 모델을 이해하고 수정하며 기반을 구축할 수 있도록 데이터, 코드 및 체크포인트를 포함한 전체 개발 파이프라인을 제공합니다. 엄격한 데이터 큐레이션(Dolma 3 및 Dolci 데이터셋)과 효율적인 훈련 인프라를 통해 다양한 벤치마크에서 강력한 성능을 자랑합니다.

# NVIDIA Acquires Open-Source Workload Management Provider SchedMD
- [https://blogs.nvidia.com/blog/nvidia-acquires-schedmd/](https://blogs.nvidia.com/blog/nvidia-acquires-schedmd/)
- NVIDIA가 SchedMD(slurm 개발사)를 인수

