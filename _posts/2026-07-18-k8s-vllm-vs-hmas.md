---
title: '사내 GPU로 챗봇 만들기: vanilla Kubernetes + vLLM 직접 배포 vs 같은 일을 H-MAS로'
date: 2026-07-18 12:00:00 +0900
permalink: /posts/2026/07/k8s-vllm-vs-hmas/
categories:
  - H-MAS
tags:
  - MLOps
  - GPU
  - K8s
  - Inference
  - vLLM
---

> 이 글의 모든 명령·출력·수치는 2026-07-18, 동일한 클러스터에서 두 경로를 실제로 실행하며 기록한 것입니다.

---

## 1. 들어가며

"사내 GPU 서버에 오픈소스 LLM을 올려서, 팀원들이 쓸 수 있는 챗봇을 만들자."

많은 팀이 여기서 출발합니다. 이 글에서는 같은 목표를 두 가지 방법으로 각각 처음부터 끝까지 해봅니다.

1. 1부: vanilla Kubernetes에 vLLM을 직접 배포
2. 2부: 같은 일을 H-MAS 웹 콘솔로

공정한 비교를 위해 조건을 통일했습니다. 두 경로 모두 같은 클러스터, 같은 모델, 같은 서빙 이미지를 사용했고, 실제 파드에 뜬 이미지가 동일함을 kubectl로 확인했습니다(실측 명령·출력은 3.1절에 수록).

| 항목 | 내용 |
|------|------|
| 클러스터 | research (단일 노드 microk8s) |
| GPU | NVIDIA RTX 5060 Ti 16GB × 1 (Blackwell, sm_120) |
| 드라이버 / CUDA | 580.159.03 / CUDA 13.0 |
| 서빙 이미지 | `vllm/vllm-openai:v0.19.1-cu130-ubuntu2404` (양쪽 동일, 실측 확인) |
| 모델 | Qwen/Qwen2.5-1.5B-Instruct (Apache 2.0, HF gated 아님) |

클라이언트 쪽도 통일했습니다. 뒤에서 소개할 콘솔 챗봇 스크립트 하나를 양쪽에 그대로 사용합니다.

---

## 2. 1부: vanilla Kubernetes + vLLM 직접 배포

### 2.1 GPU 노드 준비 상태 확인

먼저 클러스터에 GPU가 스케줄 가능한 상태인지 확인합니다. 이 클러스터에는 NVIDIA GPU Operator가 이미 설치되어 있어, 노드의 `nvidia.com/gpu` capacity/allocatable과 device plugin 파드 상태를 확인하는 것으로 충분했습니다.

```bash
kubectl get node research -o jsonpath='{"capacity: "}{.status.capacity.nvidia\.com/gpu}{"\nallocatable: "}{.status.allocatable.nvidia\.com/gpu}{"\n"}'
kubectl get pods -n gpu-operator-resources -o wide
```

```text
capacity: 1
allocatable: 1
NAME                                                         READY   STATUS      RESTARTS   AGE
gpu-feature-discovery-k6b4j                                  1/1     Running     92         111d
gpu-operator-666bbffcd-m4cts                                 1/1     Running     126        111d
...(생략)...
nvidia-device-plugin-daemonset-jh8fv                         1/1     Running     37         44d
nvidia-operator-validator-9ccr2                              1/1     Running     92         111d
```

실측에서는 GPU를 점유 중인 파드가 없는 것도 별도 스크립트로 함께 확인했습니다(`GPU-occupying pods: NONE`). GPU 1장이 비어 있고 device plugin이 정상입니다. 참고로 GPU Operator가 없는 클러스터라면 이 단계 전에 드라이버·container toolkit·device plugin 설치부터 해야 합니다.

한 가지 미리 짚어둘 점. 이번에 쓰는 Qwen2.5-1.5B-Instruct는 gated 모델이 아니라서 그냥 받아지지만, Llama 계열처럼 gated 모델을 쓴다면 HF 토큰을 담은 Secret을 만들어 파드에 주입하는 단계가 하나 더 필요합니다. 이번 실측 범위에는 포함되지 않았습니다.

### 2.2 매니페스트 작성

직접 쓰면 Namespace, 모델 캐시용 PVC, Deployment, Service 정도가 최소 구성입니다. 실제 사용한 매니페스트 전문입니다.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: blog-vanilla
---
# 모델 캐시 — 재시작 시 재다운로드를 피하기 위한 PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: model-cache
  namespace: blog-vanilla
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 20Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm-qwen
  namespace: blog-vanilla
spec:
  replicas: 1
  selector:
    matchLabels:
      app: vllm-qwen
  template:
    metadata:
      labels:
        app: vllm-qwen
    spec:
      containers:
      - name: vllm
        image: vllm/vllm-openai:v0.19.1-cu130-ubuntu2404
        args:
        - --model
        - Qwen/Qwen2.5-1.5B-Instruct
        - --max-model-len
        - "8192"
        ports:
        - containerPort: 8000
        resources:
          limits:
            nvidia.com/gpu: 1
        volumeMounts:
        - name: model-cache
          mountPath: /root/.cache/huggingface
        - name: shm
          mountPath: /dev/shm
        readinessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
      volumes:
      - name: model-cache
        persistentVolumeClaim:
          claimName: model-cache
      - name: shm
        emptyDir:
          medium: Memory
          sizeLimit: 2Gi
---
apiVersion: v1
kind: Service
metadata:
  name: vllm-qwen
  namespace: blog-vanilla
spec:
  type: NodePort
  selector:
    app: vllm-qwen
  ports:
  - port: 8000
    targetPort: 8000
    nodePort: 30800
```

78줄입니다. 짧아 보이지만 `/dev/shm` 마운트, 모델 캐시 PVC, readiness probe 경로 같은 것들은 한 번씩 겪어보고 나서야 넣게 되는 항목들입니다.

### 2.3 배포와 기동 대기

```bash
kubectl apply -f vllm-qwen.yaml
```

```text
namespace/blog-vanilla created
persistentvolumeclaim/model-cache created
deployment.apps/vllm-qwen created
service/vllm-qwen created
```

rollout이 끝나기를 기다립니다.

```bash
kubectl rollout status deploy/vllm-qwen -n blog-vanilla --timeout=600s
kubectl get pods -n blog-vanilla -o wide
```

```text
Waiting for deployment "vllm-qwen" rollout to finish: 0 of 1 updated replicas are available...
deployment "vllm-qwen" successfully rolled out
NAME                         READY   STATUS    RESTARTS   AGE     IP            NODE
vllm-qwen-6fbd46c88c-btqgq   1/1     Running   0          6m30s   10.1.38.249   research
```

파드가 Ready가 되기까지 약 6분 30초가 걸렸습니다. 시간이 어디에 쓰였는지 기동 로그를 보면 답이 나옵니다.

```text
(APIServer pid=1) INFO 07-18 01:51:14 [utils.py:299]        █     █     █▄   ▄█
(APIServer pid=1) INFO 07-18 01:51:14 [utils.py:299]  ▄▄ ▄█ █     █     █ ▀▄▀ █  version 0.19.1
(APIServer pid=1) INFO 07-18 01:51:14 [utils.py:299]   █▄█▀ █     █     █     █  model   Qwen/Qwen2.5-1.5B-Instruct
...(생략)...
(APIServer pid=1) Warning: You are sending unauthenticated requests to the HF Hub. Please set a HF_TOKEN to enable higher rate limits and faster downloads.
...(생략)...
(EngineCore pid=73) INFO 07-18 01:55:42 [weight_utils.py:581] Time spent downloading weights for Qwen/Qwen2.5-1.5B-Instruct: 249.323250 seconds
...(생략)...
(EngineCore pid=73) INFO 07-18 01:57:08 [core.py:283] init engine (profile, create kv cache, warmup model) took 84.57 seconds
...(생략)...
(APIServer pid=1) INFO 07-18 01:57:15 [api_server.py:596] Starting vLLM server on http://0.0.0.0:8000
(APIServer pid=1) INFO:     Application startup complete.
```

6분 30초의 대부분은 모델 가중치 다운로드(249.3초)였고, 엔진 초기화(프로파일링·KV 캐시·워밍업)에 84.57초가 더 쓰였습니다. 이미지가 노드에 미리 받아져 있어도 가중치는 따로 다운로드해야 합니다. 로그의 HF 미인증 경고는 토큰 없이 받으면 rate limit이 낮아 다운로드가 더 느려질 수 있다는 안내입니다.

### 2.4 API 확인

접근 경로는 두 가지입니다. 매니페스트에 NodePort 30800을 넣어두었으니 클러스터 네트워크 상황에 따라 그쪽을 써도 되고, 이번 실측은 `kubectl port-forward`로 서비스의 8000 포트를 로컬에 연결해 진행했습니다.

```bash
kubectl port-forward -n blog-vanilla svc/vllm-qwen 8000:8000
```

port-forward는 포그라운드로 계속 떠 있으므로, 이후 명령은 별도 터미널에서 실행합니다. 이제 OpenAI 호환 API를 호출해 봅니다.

```bash
curl -s http://localhost:8000/v1/models | python3 -m json.tool
```

```json
{
    "object": "list",
    "data": [
        {
            "id": "Qwen/Qwen2.5-1.5B-Instruct",
            "object": "model",
            "created": 1784339970,
            "owned_by": "vllm",
            "max_model_len": 8192,
            ...(생략)...
        }
    ]
}
```

```bash
curl -s http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen2.5-1.5B-Instruct",
    "messages": [{"role": "user", "content": "안녕하세요! 자기소개 한 문장 부탁해요."}]
  }' | python3 -m json.tool
```

```json
{
    "id": "chatcmpl-b6d82e88c01cdc75",
    "object": "chat.completion",
    "model": "Qwen/Qwen2.5-1.5B-Instruct",
    "choices": [
        {
            "index": 0,
            "message": {
                "role": "assistant",
                "content": "물론입니다! 저는 AI 언어 모델 Qwen을 사용한 인공지능助手로, 여러분의 질문에 최선을 다하여 답변하고 도움을 드리고 있습니다. 무엇이든 물어보세요!"
            },
            "finish_reason": "stop"
        }
    ],
    "usage": {
        "prompt_tokens": 44,
        "total_tokens": 95,
        "completion_tokens": 51
    },
    ...(생략)...
}
```

OpenAI 호환 API가 잘 동작합니다. (1.5B 소형 모델이라 답변에 한자·일본어가 가끔 섞이는 것은 모델 특성입니다.)

### 2.5 콘솔 챗봇 연결

이제 목표였던 챗봇입니다. 의존성은 OpenAI SDK 하나뿐입니다.

```bash
pip install openai
```

약 50줄짜리 콘솔 챗봇 스크립트 전문입니다. 이 스크립트를 2부에서도 한 글자도 고치지 않고 다시 씁니다.

```python
#!/usr/bin/env python3
"""콘솔 챗봇 — OpenAI 호환 엔드포인트에 연결하는 채팅 루프.

환경 변수:
  CHAT_BASE_URL  OpenAI 호환 엔드포인트 (기본: http://localhost:8000/v1)
  CHAT_API_KEY   API 키 (vanilla vLLM은 불필요 — 아무 값이나 가능)
  CHAT_MODEL     모델 이름
"""
import os

from openai import OpenAI

BASE_URL = os.environ.get("CHAT_BASE_URL", "http://localhost:8000/v1")
API_KEY = os.environ.get("CHAT_API_KEY", "not-needed")
MODEL = os.environ.get("CHAT_MODEL", "Qwen/Qwen2.5-1.5B-Instruct")

client = OpenAI(base_url=BASE_URL, api_key=API_KEY)
history = [{"role": "system", "content": "당신은 친절한 사내 어시스턴트입니다. 한국어로 간결하게 답하세요."}]

print(f"모델: {MODEL}")
print(f"엔드포인트: {BASE_URL}")
print("대화를 시작하세요. 종료: /quit")

while True:
    try:
        user_input = input("\n나> ").strip()
    except (EOFError, KeyboardInterrupt):
        break
    if not user_input:
        continue
    if user_input == "/quit":
        break
    history.append({"role": "user", "content": user_input})
    try:
        stream = client.chat.completions.create(model=MODEL, messages=history, stream=True)
        print("봇> ", end="", flush=True)
        chunks = []
        for chunk in stream:
            if not chunk.choices:
                continue
            delta = chunk.choices[0].delta.content or ""
            chunks.append(delta)
            print(delta, end="", flush=True)
        print()
    except Exception as e:
        print(f"\n[오류] 요청 실패: {e}")
        history.pop()
        continue
    history.append({"role": "assistant", "content": "".join(chunks)})
```

port-forward가 열려 있으면 환경 변수 없이 그대로 실행됩니다.

```bash
python chatbot.py
```

실제 대화 기록입니다. (실측 시 질문을 stdin 파이프로 넣어 원 로그에는 질문이 에코되지 않았기에, 아래는 고정 질문을 표기상 복원한 것입니다. 봇의 답변은 로그 그대로입니다.)

```text
모델: Qwen/Qwen2.5-1.5B-Instruct
엔드포인트: http://localhost:8000/v1
대화를 시작하세요. 종료: /quit

나> 안녕하세요! 사내 GPU에서 서빙 중인 모델이에요. 자기소개 부탁해요.
봇> 안녕하세요, 저는 사내 GPU에서 작업 중인 AI 어시스턴트입니다. 언제든지 도움을 드리겠습니다.

나> Kubernetes가 뭔지 두 문장으로 설명해줘.
봇> Kubernetes는 자동화된 클러스터 관리 시스템으로, 애플리케이션을 쉽게 배포하고 운영할 수 있게 해줍니다.
```

챗봇 완성입니다. 여기까지 매니페스트 78줄 + 스크립트 하나로 목표를 달성했습니다.

### 2.6 여기서 멈추면 남는 것들

그런데 이걸 "팀원들이 쓰는 서비스"라고 부르려는 순간, 몇 가지가 걸립니다. 실제로 확인해 봤습니다.

**① 인증이 없다.** 인증 헤더 없이 chat completions를 호출해 봤습니다.

```bash
curl -s -o /dev/null -w "no-auth chat: %{http_code}\n" http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model": "Qwen/Qwen2.5-1.5B-Instruct", "messages": [{"role": "user", "content": "hi"}]}'
```

```text
no-auth chat: 200
```

vLLM 기본 배포에는 인증 개념이 없습니다. 엔드포인트를 아는 사람은 누구나 GPU를 쓸 수 있습니다.

**② 관측은 raw /metrics뿐.** `/metrics`는 Prometheus 텍스트를 뱉지만, 수집기도 대시보드도 없고 요청자 구분도 없습니다.

```bash
curl -s http://localhost:8000/metrics | grep -E "^vllm:" | head -10
```

```text
...(생략)...
vllm:num_requests_running{engine="0",model_name="Qwen/Qwen2.5-1.5B-Instruct"} 0.0
vllm:num_requests_waiting{engine="0",model_name="Qwen/Qwen2.5-1.5B-Instruct"} 0.0
vllm:engine_sleep_state{engine="0",model_name="Qwen/Qwen2.5-1.5B-Instruct",sleep_state="awake"} 1.0
...(생략)...
```

**③ 누가 얼마나 썼는지 알 수 없다.** 사용자/키별 사용량 추적은 해당 개념 자체가 없습니다. 요청자 구분이 불가능합니다.

**④ 노출 관리도 숙제.** NodePort를 열면 클러스터 네트워크 정책·방화벽을 직접 챙겨야 하고, port-forward는 데모용이지 서비스용이 아닙니다.

이것들을 vanilla로 해결하려면 인증 프록시, rate limiter, Prometheus + Grafana, 로깅 파이프라인을 각각 추가로 구축해야 합니다. 전부 가능한 일이지만, "챗봇 하나 띄우기"에서 시작한 일 치고는 스코프가 꽤 커집니다.

실험을 마치고 네임스페이스를 지워 GPU를 반납했습니다.

```text
namespace "blog-vanilla" deleted
GPU-occupying pods: 0
```

---

## 3. 2부: 같은 일을 H-MAS로

이번엔 같은 클러스터, 같은 모델, 같은 이미지를 H-MAS로 배포합니다. H-MAS 설치 자체는 이 글의 범위가 아니므로(이번 실측에서도 이미 설치된 환경을 사용했습니다) 설치 절차는 기존 문서를 참고해 주세요. 웹 콘솔은 `http://research.shrinklabs.com:9110`입니다.

### 3.1 웹 콘솔에서 배포

H-MAS에서는 YAML 대신 배포 위자드에서 모델·런타임(vLLM)·대상 클러스터·리소스를 폼으로 선택합니다.

![H-MAS 배포 위자드](/images/blog-514/deploy-form.png)

*배포 위자드 1단계: Qwen 2.5 1.5B와 vLLM 런타임 선택 (이번 실측 환경에서 캡처)*

배포는 웹 콘솔로도, 같은 기능의 REST API로도 만들 수 있습니다. 이번 실측은 출력을 그대로 남기려고 API로 진행했습니다. 배포 생성 직후 응답입니다.

```json
{
    "id": "3",
    "name": "blog-qwen",
    "status": "pending",
    "message": "Deployment created. Model 'Qwen/Qwen2.5-1.5B-Instruct' will be served on research using vLLM."
}
```

상태를 폴링하며 기다립니다.

```text
[1] status=pending
[2] status=pending
...(생략)...
[24] status=pending
[25] status=running
RUNNING reached at iteration 25 (~375s ≈ 6.25min from creation)
```

생성부터 running까지 375초. 1부와 거의 같은 시간인데, 당연한 결과입니다. 콜드 스타트를 지배하는 모델 가중치 다운로드는 어느 경로로 배포하든 똑같이 발생하기 때문입니다. H-MAS가 줄여주는 것은 이 대기 시간이 아니라, YAML을 쓰고 접근 경로와 인증을 붙이는 사람의 작업입니다.

"같은 이미지" 주장도 여기서 실측으로 확인해 둡니다. H-MAS가 띄운 파드의 실제 컨테이너 이미지를 조회해 보면 1부에서 직접 쓴 이미지와 동일합니다.

```bash
kubectl get pods -n hmas-serving -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].image}{"\n"}{end}'
```

```text
hmas-blog-qwen-5f9486b5c6-mvh7s	vllm/vllm-openai:v0.19.1-cu130-ubuntu2404
```

### 3.2 API 키 발급과 RPM 지정

콘솔의 API 키 메뉴에서 키를 발급합니다. 이때 분당 요청 수(RPM) 제한을 함께 지정할 수 있습니다. 실험용으로 일부러 빡빡하게 RPM 5로 만들었습니다.

```json
{
  "id": 4,
  "name": "blog-demo",
  "key": "sk-hmas-d858...(마스킹)",
  "keyPrefix": "sk-hmas-d858",
  "username": "admin",
  "rateLimitRpm": 5,
  "createdAt": "2026-07-18T02:17:25Z"
}
```

![H-MAS API 키 관리](/images/blog-514/api-keys.png)

*API 키 관리 화면: 키 목록과 RPM 표시 (이번 실측 환경에서 캡처)*

배포된 모델은 OpenAI 호환 추론 프록시 `http://research.shrinklabs.com:9110/api/inference/blog-qwen/v1`로 노출됩니다. 프록시는 발급받은 키를 `Authorization: Bearer` 헤더로 받습니다. curl로 확인해 봅니다. (`$HMAS_KEY`에는 발급받은 키 원문을 넣습니다.)

```bash
HMAS=http://research.shrinklabs.com:9110

curl -s -H "Authorization: Bearer $HMAS_KEY" "$HMAS/api/inference/blog-qwen/v1/models" | python3 -m json.tool
```

```json
{
    "object": "list",
    "data": [
        {
            "id": "Qwen/Qwen2.5-1.5B-Instruct",
            "object": "model",
            "owned_by": "vllm",
            "max_model_len": 32768,
            ...(생략)...
        }
    ]
}
```

```bash
curl -s -H "Authorization: Bearer $HMAS_KEY" -H "Content-Type: application/json" \
  "$HMAS/api/inference/blog-qwen/v1/chat/completions" \
  -d '{
    "model": "Qwen/Qwen2.5-1.5B-Instruct",
    "messages": [{"role": "user", "content": "안녕하세요! 자기소개 한 문장 부탁해요."}]
  }' | python3 -m json.tool
```

```json
{
    "id": "chatcmpl-a5b201f1ef7a7624",
    "object": "chat.completion",
    "model": "Qwen/Qwen2.5-1.5B-Instruct",
    ...(생략)...
    "usage": {
        "prompt_tokens": 44,
        "total_tokens": 70,
        "completion_tokens": 26
    },
    ...(생략)...
}
```

1부와 같은 vLLM이 같은 이미지로 응답합니다. (`max_model_len`이 32768인 것은 1부 매니페스트에서는 8192로 명시 제한했고 여기서는 런타임 기본값이 적용됐기 때문입니다.)

### 3.3 같은 챗봇, 환경 변수 2개만 교체

2.5절의 챗봇 스크립트를 코드 수정 없이 그대로 씁니다. 바꾸는 것은 엔드포인트와 API 키, 환경 변수 두 개뿐입니다.

```bash
CHAT_BASE_URL=http://research.shrinklabs.com:9110/api/inference/blog-qwen/v1 \
CHAT_API_KEY=sk-hmas-d858...(발급받은 키) \
python chatbot.py
```

실제 대화 기록입니다. (1부와 마찬가지로 질문은 표기상 복원, 답변은 로그 그대로입니다.)

```text
모델: Qwen/Qwen2.5-1.5B-Instruct
엔드포인트: http://research.shrinklabs.com:9110/api/inference/blog-qwen/v1
대화를 시작하세요. 종료: /quit

나> 안녕하세요! 이번엔 H-MAS 추론 프록시를 통해 접속했어요. 자기소개 부탁해요.
봇> 안녕하세요, 저는 H-MAS 추론 프록시의 사내 어시스턴트입니다. 무엇을 도와드릴까요?

나> Kubernetes가 뭔지 두 문장으로 설명해줘.
봇> Kubernetes는 개발자들이 쉽게 배포하고 관리할 수 있는 클라우드 컴퓨팅 환경입니다.
```

OpenAI 호환 프록시이기 때문에 기존 OpenAI SDK 기반 코드가 그대로 붙습니다.

### 3.4 1부의 미해결 항목, 다시 실측

2.6절에서 남았던 항목들을 같은 방식으로 다시 확인해 봤습니다.

**① 무인증 호출 → 차단.** 2.6절과 같은 형태의 curl에서 `Authorization` 헤더만 뺀 요청입니다.

```bash
curl -s -o /dev/null -w "no-auth chat: %{http_code}\n" \
  -H "Content-Type: application/json" \
  "$HMAS/api/inference/blog-qwen/v1/chat/completions" \
  -d '{"model": "Qwen/Qwen2.5-1.5B-Instruct", "messages": [{"role": "user", "content": "hi"}]}'
```

```text
no-auth chat: 401
```

1부에서 200이었던 무인증 호출이 여기서는 401로 거부됩니다.

**② RPM 제한이 실제로 동작.** RPM 5짜리 키로 연속 8회 호출해 봤습니다.

```bash
for i in $(seq 1 8); do
  curl -s -o /dev/null -w "req $i: %{http_code}\n" \
    -H "Authorization: Bearer $HMAS_KEY" -H "Content-Type: application/json" \
    "$HMAS/api/inference/blog-qwen/v1/chat/completions" \
    -d '{"model": "Qwen/Qwen2.5-1.5B-Instruct", "messages": [{"role": "user", "content": "hi"}], "max_tokens": 5}'
done
```

```text
req 1: 200
req 2: 200
req 3: 200
req 4: 200
req 5: 200
req 6: 429
req 7: 429
req 8: 429
```

5회까지 통과, 6회째부터 429로 차단됩니다.

**③ 요청자별 추론 로그.** 프록시를 지난 모든 성공 요청이 누가·어떤 키로·어떤 배포에·몇 토큰을 썼는지와 함께 기록됩니다.

```json
{
    "id": 48,
    "requestId": "dd6b6007-ed06-47c1-b4be-7879a941086c",
    "username": "admin",
    "authMethod": "api_key",
    "apiKeyId": 4,
    "apiKeyName": "blog-demo",
    "deploymentName": "blog-qwen",
    "modelName": "Qwen/Qwen2.5-1.5B-Instruct",
    "clusterName": "research",
    "requestedAt": "2026-07-18T02:19:36Z",
    "respondedAt": "2026-07-18T02:19:36Z",
    "ttfbMs": 69,
    "durationMs": 69,
    "statusCode": 200,
    "isStreaming": false,
    "promptTokens": 30,
    "completionTokens": 5,
    "method": "POST",
    "path": "/v1/chat/completions"
}
```

실측에서 확인한 동작 특성이 하나 있습니다. 429로 거부된 요청은 추론 로그에 남지 않고, 백엔드까지 도달한 요청만 기록됩니다. 거부량까지 집계해야 한다면 알아둘 부분입니다.

![H-MAS 추론 로그](/images/blog-514/inference-logs.png)

*추론 로그 화면: 이번 실측에서 blog-demo 키로 보낸 요청들의 실제 기록 (상태 코드·TTFB·토큰 표시)*

화면에 502 한 건이 보이는데, 배포 직후 콜드 스타트 시점에 30초를 기다리다 실패한 요청입니다. 이런 실패 기록도 상태 코드와 함께 로그에 그대로 남아서, 어느 시점에 무슨 요청이 왜 오래 걸렸는지 추적할 수 있습니다.

**④ 모니터링 대시보드.** raw /metrics 대신 콘솔에 모니터링 화면이 기본 제공됩니다.

![H-MAS 모니터링](/images/blog-514/monitoring.png)

*모니터링 화면: blog-qwen 배포에 추론 트래픽을 흘리며 캡처 (TPS·지연·GPU 온도/전력·메트릭 추이·Pod 로그)*

---

## 4. 3부: 비교와 판단

### 4.1 단계별 비교

| 단계 | vanilla K8s + vLLM (직접 구축) | H-MAS (기본 제공) |
|------|------|------|
| 배포 정의 | YAML 78줄 직접 작성 (PVC·shm·probe 포함) | 배포 위자드 폼 입력 |
| 콜드 스타트 | 약 6.5분 (다운로드 249초 + 엔진 초기화 84.6초) | 375초 (다운로드 비용은 동일) |
| 접근 경로 | port-forward 또는 NodePort 직접 관리 | OpenAI 호환 추론 프록시 URL |
| 인증 | 없음 (무인증 호출 200 실측) | API 키 (무인증 호출 401 실측) |
| 속도 제한 | 없음 (직접 구축 필요) | 키별 RPM (5회 통과 후 429 실측) |
| 요청 기록 | raw /metrics뿐, 요청자 구분 불가 | 요청자·키·토큰·TTFB 단위 추론 로그 |
| 모니터링 | Prometheus/Grafana 직접 구축 필요 | 콘솔 대시보드 기본 제공 |
| 클라이언트 코드 | OpenAI SDK 그대로 | OpenAI SDK 그대로 (환경 변수 2개 교체) |

### 4.2 공정하게 말하면

**모델 1개, 클러스터 1개, 개발·실험 용도라면 vanilla로 충분합니다.** 1부에서 봤듯 매니페스트 하나로 챗봇까지 도달할 수 있고, 그 과정에서 배우는 것도 많습니다. 실제로 이 글의 1부도 목표를 달성했습니다.

갈림길은 그다음에 옵니다. 모델이 여러 개가 되고, 사용자가 여러 명이 되고, 클러스터가 여러 개가 되는 지점부터 "직접 만든 YAML + 무인증 엔드포인트"는 인증 프록시, rate limiter, 사용량 집계, 대시보드를 하나씩 손수 붙여야 하는 프로젝트로 변합니다. H-MAS는 그 부속들을 처음부터 플랫폼에 넣어둔 것입니다.

### 4.3 이 글에서 다루지 않은 것

이번 실측은 단일 노드·단일 GPU 환경이라 H-MAS의 나머지 축인 멀티 클러스터 통합 관리(여러 클러스터를 하나의 컨트롤 플레인에서 관리하고 배치 정책으로 스케줄링)와 하드웨어 인지 스케줄링(GPU 토폴로지를 인식한 배치)은 다루지 않았습니다. 이 부분은 실측과 함께 다른 글에서 다루겠습니다.

---

## 5. 맺음말

같은 클러스터에서 같은 모델을 두 번 배포해 보며 확인한 것은 단순합니다. 챗봇을 "띄우는" 비용은 두 경로가 비슷하지만, 챗봇을 "여럿이 안전하게 쓰게 만드는" 비용은 크게 다르다는 것입니다.

GPU 서버를 보유하고 있고 AI 모델 서빙에 관심이 있는 조직이라면, 요청 주시는 분께 온라인/오프라인 미팅 또는 제품 소개 자료를 통해 H-MAS를 자세히 안내해 드립니다. 아래 연락처로 편하게 문의해 주세요.

**연락처**: [contact@parameterfreak.com](mailto:contact@parameterfreak.com)
