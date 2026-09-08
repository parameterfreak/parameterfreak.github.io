---
title: 'GPU가 남아 있는지와 일하고 있는지는 다른 질문입니다'
date: 2026-09-08 12:00:00 +0900
permalink: /posts/2026/09/hmas-monitoring-architecture/
categories:
  - H-MAS
tags:
  - MLOps
  - GPU
  - K8s
  - Prometheus
  - Monitoring
---

> 본문의 설정 값과 쿼리는 2026-09 기준 H-MAS 코드와 research 데모 환경의 실제 구성입니다.

---

## 1. 들어가며

사내 GPU 서버 몇 대에 LLM을 올려 여러 팀이 나눠 쓰기 시작하면 운영자는 금방 두 가지 질문을 번갈아 받게 됩니다. "지금 GPU 남은 거 있어요?"와 "우리 모델 왜 이렇게 느려요?"입니다.

첫 번째는 수량을 묻는 질문입니다. 클러스터에 GPU가 몇 장 있고 그중 몇 장이 이미 배정됐는지 알면 답할 수 있습니다. 두 번째는 상태를 묻는 질문이라 다릅니다. 배정된 GPU가 실제로 얼마나 바쁜지, 요청이 큐에 얼마나 쌓여 있는지, 첫 토큰이 나오기까지 몇 초가 걸리는지를 알아야 합니다.

두 숫자는 같은 대시보드에 나란히 놓이지만 출처가 완전히 다릅니다. H-MAS의 모니터링은 이 두 출처를 처음부터 분리해서 설계했고, 이 글은 그 구조를 설명합니다. 뒷부분에서는 사무실 밖에 있는 클러스터의 메트릭을 끌어오면서 내린 결정들을 사례로 다룹니다.

Prometheus의 기본 개념(scrape, 레이블, PromQL)은 안다고 가정하고 씁니다. H-MAS 자체가 처음이라면 [vanilla Kubernetes + vLLM vs H-MAS 비교 글](/posts/2026/07/k8s-vllm-vs-hmas/)을 먼저 보시면 배경이 잡힙니다.

---

## 2. 두 종류의 숫자

H-MAS는 여러 클러스터를 하나의 컨트롤 플레인에서 관리합니다. 각 멤버 클러스터의 자원 정보는 두 경로로 들어옵니다.

| 정보 | 출처 A: 클러스터 API 서버 | 출처 B: Prometheus |
|------|------|------|
| GPU 수량 (총 / 배정 / 가용) | 노드의 `nvidia.com/gpu` capacity·allocatable | — |
| GPU 사용률 (%) | — | DCGM Exporter `DCGM_FI_DEV_GPU_UTIL` |
| GPU 메모리 사용량·온도·전력 | — | DCGM Exporter |
| 추론 성능 (TTFT, TPS, 큐 길이) | — | 서빙 런타임 `/metrics` |
| 노드 CPU·메모리·디스크·네트워크 실사용 | — | node-exporter |
| 노드 수와 Ready 상태 | 노드 오브젝트 | — |

출처 A는 Kubernetes가 스케줄링에 쓰는 선언된 자원입니다. GPU 한 장을 Pod에 배정하면 그 Pod가 GPU를 1% 쓰든 99% 쓰든 "배정됨"으로 잡힙니다. 새 모델을 어느 클러스터에 올릴 수 있는지 판단할 때 필요한 숫자가 이것입니다.

출처 B는 하드웨어와 런타임이 지금 이 순간 보고하는 값입니다. 배정된 GPU가 놀고 있는지, 큐가 쌓이고 있는지는 여기서만 알 수 있습니다.

처음에는 출처 A만으로 시작했습니다. 컨트롤 플레인이 이미 각 클러스터의 노드 자원 총량을 알고 있었기 때문에 추가 인프라 없이도 클러스터 카드에 "GPU 3/8" 같은 숫자를 보여줄 수 있었습니다. 다만 "왜 느린지"에는 답을 못 했고, 그래서 출처 B를 붙였습니다. 하나로 다른 하나를 대체하지 않은 이유는 위 표를 보면 드러납니다. 겹치는 칸이 하나도 없습니다.

---

## 3. 전체 구조

메트릭 파이프라인은 세 층으로 나뉩니다. 멤버 클러스터에서 모으고, 컨트롤 플레인에서 저장하고, H-MAS 백엔드가 읽어서 화면에 그립니다.

![H-MAS 메트릭 파이프라인: 멤버 클러스터의 서빙 런타임·DCGM·node-exporter를 Prometheus Agent가 수집해 remote_write로 컨트롤 플레인 Prometheus에 보내고, 백엔드가 PromQL로 읽어 프론트엔드에 그린다](/images/blog-monitoring/monitoring-pipeline.png)

이 구조로 가기 전에 백엔드가 각 서빙 Pod의 `/metrics`를 직접 긁어오는 방식도 검토했습니다. 클러스터가 하나라면 그쪽이 단순합니다. 하지만 클러스터가 둘 이상이 되는 순간 백엔드가 모든 멤버의 Pod 네트워크에 도달할 수 있어야 하고, Pod가 늘수록 백엔드가 스크레이퍼 노릇까지 떠안게 됩니다. Prometheus를 사이에 두면 수집과 저장은 표준 스택에 맡기고 백엔드는 PromQL 클라이언트로만 남습니다. 나중에 클러스터가 늘어 Prometheus 한 대로 버거워지면 수신 측을 Thanos Receiver로 바꾸면 되는데, 그때도 멤버 쪽 설정은 손대지 않아도 됩니다.

---

## 4. 멤버 클러스터에서 모으는 것

### 4.1 세 가지 소스

서빙 런타임이 첫 번째 소스입니다. vLLM과 llama.cpp는 별도 exporter 없이 자체적으로 Prometheus 형식의 `/metrics`를 노출하고, 서빙 API와 같은 포트를 씁니다.

| 런타임 | 포트 | 대표 메트릭 |
|------|------|------|
| vLLM | 8000 | `vllm:time_to_first_token_seconds`, `vllm:num_requests_waiting`, `vllm:kv_cache_usage_perc` |
| llama.cpp | 8080 | `llamacpp:predicted_tokens_seconds`, `llamacpp:requests_processing` |

두 번째는 GPU 하드웨어입니다. NVIDIA DCGM Exporter를 DaemonSet으로 GPU 노드마다 띄우면 사용률, 프레임버퍼 사용량, 온도, 전력이 GPU 단위로 나옵니다. NVIDIA GPU Operator, 최소한 드라이버와 Container Toolkit은 먼저 깔려 있어야 합니다.

세 번째는 호스트입니다. node-exporter가 CPU, 메모리, 디스크, 네트워크 인터페이스 메트릭을 냅니다. GPU 서빙에서 호스트 메트릭을 들여다볼 일이 없을 것 같지만 의외로 자주 옵니다. 모델 로딩 중에 디스크가 꽉 차거나, 모델 다운로드가 NIC 대역폭을 다 먹는 경우가 그렇습니다.

### 4.2 Prometheus Agent 모드

멤버 클러스터의 Prometheus는 Agent 모드로 돕니다. 로컬 저장소도 쿼리 API도 없고, 스크레이프한 것을 remote_write로 내보내는 일만 합니다. GPU 노드에서 모니터링이 차지하는 자원을 줄이려는 선택입니다.

Prometheus 3.x 기준으로 핵심 설정은 이렇습니다.

```yaml
server:
  defaultFlagsOverride:
    - "--agent"                 # 3.x에서는 --enable-feature=agent가 제거됨
    - "--config.file=/etc/config/prometheus.yml"
  persistentVolume:
    enabled: false              # Agent는 로컬 저장 불필요
  global:
    scrape_interval: 15s
serverFiles:
  prometheus.yml:
    rule_files: []              # Agent 모드에서는 rule_files 불허
```

주석을 단 두 줄은 실제로 걸려 넘어진 곳입니다. 2.x 문서를 보고 `--enable-feature=agent`를 넣으면 3.x에서는 아예 뜨지 않고, `rule_files`가 차트 기본값으로 남아 있으면 Agent 모드가 설정 파일을 거부합니다. 둘 다 에러 메시지가 친절한 편은 아니었습니다.

### 4.3 서빙 Pod 자동 발견

새 모델을 배포할 때마다 스크레이프 대상을 손으로 추가하지는 않습니다. H-MAS가 서빙 Deployment를 만들 때 Pod 템플릿에 표준 어노테이션을 붙여 두면 Agent의 `kubernetes-pods` 스크레이프 잡이 알아서 찾습니다.

```yaml
annotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "8000"      # 런타임 프리셋의 포트
  prometheus.io/path: "/metrics"
```

DCGM Exporter의 Service에도 같은 어노테이션이 붙어 있어서 GPU 노드가 추가되면 자동으로 수집 대상이 됩니다.

### 4.4 설치는 컨트롤 플레인이 한다

여기까지 읽으면 클러스터마다 Agent와 DCGM을 설치하고 remote_write 주소를 넣어줘야 하는 건지 궁금해집니다. 그 부분은 컨트롤 플레인이 대신합니다.

H-MAS 컨트롤 플레인은 멤버 클러스터에 워크로드를 자동으로 전파하는 채널을 가지고 있고, 서빙 배포도 이 채널로 나갑니다. 모니터링 컴포넌트도 같은 채널을 탑니다. Prometheus Agent와 DCGM Exporter의 매니페스트는 빌드 시점에 Helm 차트에서 렌더링해 백엔드에 내장해 두고, 백엔드가 기동하면서 이를 컨트롤 플레인에 등록합니다. 어느 클러스터에 무엇을 보낼지는 정책이 정합니다.

| 컴포넌트 | 전파 대상 |
|------|------|
| Prometheus Agent + node-exporter | 등록된 모든 멤버 클러스터 |
| DCGM Exporter | GPU 클러스터로 표시된 멤버만 |

전파할 때 클러스터별로 두 값이 주입됩니다. 하나는 remote_write 목적지 URL로, 컨트롤 플레인 전역 설정 한 곳에서 옵니다. 다른 하나는 `external_labels.cluster`로, 각 클러스터의 등록 이름이 들어갑니다. 이 `cluster` 레이블이 컨트롤 플레인 Prometheus에서 멤버를 구분하는 유일한 키라서, 백엔드의 모든 PromQL 질의에는 이 레이블 필터가 들어갑니다.

운영자가 직접 하는 일은 컨트롤 플레인 values에 URL 하나를 적는 것입니다.

```yaml
monitoring:
  remoteWrite:
    url: "https://<metrics-host>/api/v1/write"
    authSecret: "metrics-ingest-credentials"    # 선택, 6절
```

새 클러스터를 등록하면 몇 분 안에 그 클러스터의 노드 CPU 그래프가 대시보드에 올라옵니다.

---

## 5. 컨트롤 플레인에서 저장하고 읽는 것

### 5.1 Prometheus Server

컨트롤 플레인 Prometheus는 H-MAS Helm 차트의 subchart(`prometheus-community/prometheus`)로 들어갑니다. `helm install h-mas` 한 번이면 같이 설치되고, 이미 Prometheus를 운영 중이면 `prometheus.url`로 외부 인스턴스를 가리킬 수도 있습니다.

```yaml
prometheus:
  enabled: true
  server:
    retention: "15d"
    persistentVolume:
      size: 50Gi
    extraFlags:
      - "web.enable-remote-write-receiver"
```

`web.enable-remote-write-receiver`가 remote_write 수신을 켭니다. 차트에 `enableRemoteWriteReceiver`라는 값이 따로 있는데 3.x 이미지에서는 먹지 않아서 플래그로 직접 넣었습니다. Alertmanager, kube-state-metrics, pushgateway 같은 부속은 꺼 두었습니다. 알림은 H-MAS 자체 알림 시스템이 맡고 있어서 Prometheus는 시계열 저장소 역할만 합니다.

### 5.2 런타임별 이름을 하나로

프론트엔드가 vLLM인지 llama.cpp인지에 따라 다른 메트릭 이름을 알아야 한다면 런타임을 하나 추가할 때마다 화면 코드를 고쳐야 합니다. 그래서 백엔드가 런타임 ID와 통합 메트릭 이름을 받아 PromQL로 바꿔 줍니다.

| 통합 이름 | vLLM | llama.cpp |
|------|------|------|
| `requestRate` | `rate(vllm:request_success_total)` | `rate(llamacpp:n_decode_total)` |
| `ttft` / `ttftP95` / `ttftP99` | `vllm:time_to_first_token_seconds` | — |
| `tps` | `rate(vllm:generation_tokens_total)` | `llamacpp:predicted_tokens_seconds` |
| `queueDepth` | `vllm:num_requests_waiting` | `llamacpp:requests_processing` |
| `activeRequests` | `vllm:num_requests_running` | `requests_processing + requests_deferred` |
| `kvCacheUsage` | `vllm:kv_cache_usage_perc` | — |
| `gpuUtilization` | `DCGM_FI_DEV_GPU_UTIL` | 동일 |
| `gpuMemoryUsed` | `DCGM_FI_DEV_FB_USED` | 동일 |

빈 칸은 그 런타임이 해당 메트릭을 내지 않는다는 뜻이고, API 응답에서도 그 필드가 빠집니다. 화면은 있는 값만 그립니다. 런타임을 하나 추가하는 작업은 이 표에 열 하나를 더하는 것으로 끝납니다.

### 5.3 API와 상한

프론트엔드가 쓰는 엔드포인트는 세 개입니다.

| 엔드포인트 | 용도 |
|------|------|
| `GET /api/deployments/:id/metrics` | 배포 하나의 현재 스냅샷 |
| `GET /api/deployments/:id/metrics/history?range=1h&step=30s` | 배포 하나의 시계열 |
| `GET /api/clusters/:name/gpu-metrics` | 클러스터 GPU·노드 요약과 시계열 |

`history`의 `range`는 24시간, `step`은 5초까지만 받습니다. 이 상한이 없으면 `range=30d&step=1s` 같은 요청 하나가 Prometheus에서 수백만 포인트를 끌어오고 백엔드는 500을 뱉게 됩니다. 지금은 상한을 넘는 요청을 400으로 바로 돌려보냅니다. 어차피 15일 보존인 저장소에서 하루가 넘는 창을 화면에 그려 봐야 읽을 수 있는 그래프가 나오지 않습니다.

### 5.4 실제 하드웨어에서만 보이는 것

노드 네트워크 처리량 쿼리는 처음에 이랬습니다.

```promql
sum by (node) (rate(node_network_receive_bytes_total{cluster="gpu-01"}[5m]))
```

저희 데모 클러스터에서 실측해 보니 값이 실제 NIC 트래픽의 몇 배로 나왔습니다. `veth*`, `cali*`, `flannel*`, `docker*` 같은 가상 인터페이스가 같은 패킷을 여러 번 세고 있었습니다. 지금 쿼리는 이렇게 걸러냅니다. (`gpu-01`은 4.4절에서 주입되는 클러스터 등록 이름의 예시입니다.)

```promql
sum by (node) (
  rate(node_network_receive_bytes_total{
    cluster="gpu-01",
    device!~"veth.*|lo|docker.*|cni.*|flannel.*|cali.*|vxlan.*|br-.*|virbr.*|tunl.*|kube-ipvs0"
  }[5m])
)
```

보기 좋은 목록은 아닙니다. 설계 문서에서 나온 것도 아니고, 실제 클러스터의 인터페이스 목록을 보면서 하나씩 늘린 것입니다. CNI가 다르면 이름 규칙도 달라서 이 목록은 앞으로도 새 환경을 만날 때마다 늘어날 텐데, 그래서 코드 한 곳에 상수로 모아 두었습니다.

---

## 6. 사례: 사무실 밖 클러스터를 붙이기

여기까지의 파이프라인은 멤버와 컨트롤 플레인이 같은 망에 있어서 IP로 직접 닿는다는 전제 위에 있었습니다. 데모 환경에 외부 조직의 클러스터를 붙이기로 하면서 이 전제가 무너졌습니다.

### 6.1 무너진 전제 세 가지

| 전제 | 외부 클러스터에서의 현실 |
|------|------|
| 컨트롤 플레인 IP로 직접 도달 | 사설 IP는 외부에서 도달 불가 |
| IP가 바뀌지 않음 | DHCP 갱신, 노드 교체, 인스턴스 재시작으로 바뀜. 바뀌면 모든 멤버의 메트릭이 원인 표시 없이 동시에 끊김 |
| 내부망이니 평문·무인증 허용 | 공인 노출 시 누구나 가짜 메트릭을 밀어 넣거나 부하를 줄 수 있음 |

두 번째 줄은 실제로 겪은 일입니다. 데모 컨트롤 플레인의 주소가 바뀐 뒤 대시보드의 모든 그래프가 조용히 평평해졌는데, 화면 어디에도 왜 그런지 나오지 않았습니다.

### 6.2 결정

멤버 쪽에는 표준 Prometheus 설정만 남기고, 복잡한 것은 전부 컨트롤 플레인 진입점 쪽으로 몰았습니다.

![외부 클러스터 메트릭 수집 경로: Prometheus Agent가 공인 DNS와 진입점을 거쳐 ingress-nginx에서 TLS 종료·Basic Auth를 통과한 뒤 /api/v1/write 하나로만 Prometheus Server에 도달한다](/images/blog-monitoring/external-ingest-path.png)

| 항목 | 결정 | 이유 |
|------|------|------|
| 주소 | 공인 DNS 호스트네임 | IP가 바뀌어도 DNS 레코드만 고치면 됨. 멤버 재배포 없음 |
| 진입점 | ingress-nginx | TLS, 인증, 경로 제한을 설정만으로 처리 |
| TLS | cert-manager + Let's Encrypt | 공인 CA라 멤버 쪽에 `tls_config`를 추가할 필요가 없음 |
| 인증 | Basic Auth | ingress-nginx가 기본 지원하고 Prometheus `remote_write.basic_auth`도 표준 필드라 따로 짤 코드가 없음 |
| 노출 범위 | `/api/v1/write` 하나 | PromQL 조회 API는 외부에 열지 않음 |

Ingress 리소스는 이 정도입니다.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: metrics-ingest
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: metrics-ingest-auth
spec:
  ingressClassName: nginx
  tls:
    - hosts: ["<metrics-host>"]
      secretName: metrics-ingest-tls
  rules:
    - host: <metrics-host>
      http:
        paths:
          - path: /api/v1/write
            pathType: Exact
            backend:
              service:
                name: hmas-prometheus-server
                port: { number: 9090 }
```

멤버 쪽은 4.4절의 자동 전파가 처리하기 때문에 운영자가 손댈 파일이 없습니다. 자격 증명은 Kubernetes Secret으로 전파되고 Agent는 `password_file`로 읽습니다. values 파일에 비밀번호가 평문으로 들어가는 일은 없습니다.

```yaml
remote_write:
  - url: https://<metrics-host>/api/v1/write
    basic_auth:
      username: metrics-writer
      password_file: /etc/secrets/metrics-ingest/password
```

Basic Auth가 2026년에 좀 촌스러워 보이는 것은 저희도 압니다. 그래도 양쪽이 모두 표준으로 지원하는 인증 방식 중 온보딩 비용이 이보다 낮은 것을 찾지 못했습니다.

### 6.3 기각한 대안

| 대안 | 기각 이유 |
|------|------|
| VPN 오버레이 (WireGuard, Tailscale) | 상대 망에 VPN을 설치해 달라는 요청 자체가 온보딩 장벽 |
| 자체 CA + 자체 서명 인증서 | 모든 멤버에 CA 번들을 배포해야 함 |
| Bearer Token | ingress-nginx에 기본 검증 수단이 없어 외부 auth 서비스가 따로 필요 |
| mTLS | 클라이언트 인증서 발급·회전 인프라(PKI)가 필요. 데모 범위에 과함 |
| Gateway API | 단일 경로 노출에는 과한 계층. Basic Auth 처리도 구현체별 확장에 기댐 |

판단 기준은 하나였습니다. 상대 클러스터에 요구하는 것이 "아웃바운드 HTTPS가 된다" 이상이면 안 된다는 것입니다. 이 기준을 통과한 조합이 공인 DNS, Let's Encrypt, Basic Auth였습니다.

### 6.4 끊겼을 때

컨트롤 플레인이 잠깐 내려가도 멤버 Agent는 WAL에 버퍼링했다가 복구 후 다시 보냅니다. 기본 버퍼가 두 시간쯤이라 DNS 전환이나 인증서 갱신, 컨트롤 플레인 재배포 정도는 메트릭 유실 없이 지나갑니다.

그보다 길게 끊기면 그 구간은 유실입니다. 유실 자체보다 곤란한 것은 유실을 눈치채는 시점이 늦어지는 일이고, 6.1절의 장애가 바로 그 경우였습니다. 그래서 클러스터마다 마지막 수신 시각을 계산해 화면에 배너로 띄우는 진단 API를 추가했습니다.

```promql
max(max_over_time(timestamp(up{cluster="<name>"})[24h:1m]))
```

서브쿼리를 쓰는 데는 이유가 있습니다. `timestamp(up{...})`만 즉시 질의하면 Prometheus의 staleness 처리(기본 5분) 때문에 단절 5분 뒤에는 시계열 자체가 사라져서, "한 번도 받은 적 없음"과 "5분 전에 끊김"을 구분할 수 없습니다. 24시간 창을 두면 마지막 수신 시각이 남고, 그 시각과 현재의 차이가 임계값(기본 5분)을 넘으면 단절로 표시합니다.

---

## 7. 아직 하지 않은 것

아웃바운드 HTTPS조차 막힌 폐쇄망 클러스터는 6절의 방식으로 붙일 수 없습니다. 이쪽은 멤버 쪽 에이전트가 역방향 터널을 여는 방식으로 따로 설계하고 있습니다.

자격 증명도 아직 공용 Basic Auth 하나입니다. 외부 클러스터가 늘면 htpasswd에 엔트리를 여러 개 두어 개별 회수가 되게 하고, 장기적으로는 클러스터 등록 시 발급하는 토큰으로 넘어갈 생각입니다.

대시보드는 매번 raw 시계열에 `rate()`를 걸고 있습니다. 클러스터와 배포가 늘면 Recording Rules로 옮길 지점은 정해 두었습니다. Thanos도 마찬가지로, 3절에 쓴 대로 수신 측만 바꾸면 되도록 경계만 잡아 두고 아직 넣지 않았습니다.

---

## 8. 맺음말

정리하면 이렇습니다. 수량은 클러스터 API 서버에서 받고 상태는 Prometheus에서 받되 둘을 섞지 않습니다. 멤버에는 Agent 모드 Prometheus만 두고, 저장과 조회는 컨트롤 플레인이 맡습니다. 멤버 쪽 설치와 설정은 컨트롤 플레인이 자동으로 전파하므로 운영자는 URL 하나만 적으면 됩니다. 외부 클러스터는 표준 Prometheus 설정만으로 붙고, 복잡한 부분은 진입점이 떠안습니다. 그리고 끊김은 반드시 화면에 드러나게 했습니다.

마지막 항목은 설계 문서에 처음부터 있던 것이 아니라 장애를 한 번 겪고 나서 넣은 것입니다. 모니터링 시스템 자체가 모니터링되지 않으면, 그래프가 평평해진 이유를 결국 로그를 뒤져서 찾게 됩니다.

이 구조나 결정에 대해 의견이 있거나, 여러분의 환경에서는 다른 선택을 했다면 이야기를 듣고 싶습니다.

**연락처**: [contact@parameterfreak.com](mailto:contact@parameterfreak.com)
