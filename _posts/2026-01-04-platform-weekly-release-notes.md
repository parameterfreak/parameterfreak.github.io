---
title: 'platform-weekly-release-notes-2025-12-28-2026-01-03'
date: 2026-01-04
permalink: /posts/2026/01/plotform-weeklyrelease-notes-2025-12-28-2026-01-04/
tags:
  - platform 
---

# Platform 주간 릴리즈 노트
## 2025년 12월 28일 - 2026년 1월 3일

### 주요 개발 내용 요약

이번 주는 **MLflow 완전 배포 및 검증**, **플랫폼 아키텍처 문서화**, **Helm 기반 배포 정책 확립**에 중점을 둔 기간이었습니다. 총 **31개의 커밋**이 완료되었으며, **완전히 작동하는 ML 실험 추적 플랫폼**이 구축되었습니다. 특히 로컬 환경에서 대형 모델(수 GB ~ 수십 GB)을 MLflow에 업로드할 수 있는 완전한 워크플로우가 확립되었습니다.

---

## 새로운 기능 (New Features)

### 1. MLflow 완전 배포 및 검증

**구현 완료**: 로컬 환경에서 전체 MLflow 기능 사용 가능

**주요 성과**:
- **문제 발견 및 해결**:
  - 초기: 로컬에서 artifact 업로드 실패 (MinIO S3 API가 클러스터 내부에만 노출)
  - 해결: MinIO S3 API 외부 Ingress 생성 (`minio-api.**********.com`)
- **대형 모델 지원**:
  - 파일 크기 제한 제거 (`proxy-body-size: "0"`)
  - 타임아웃 1시간 설정 (수십 GB 모델 전송 대응)
  - 버퍼링 비활성화로 메모리 효율성 확보
- **완전한 기능 검증**:
  - 파라미터, 메트릭 로깅
  - Artifact 업로드 (모델, 데이터, 플롯 등)
  - MLflow UI에서 모든 데이터 확인 가능
  - MinIO Console에서 스토리지 검증

**관련 커밋**: `93d47fc`, `03c7b62`, `f6b27a9`, `1671244`, `229b004`, `03c7b62`

---

### 2. MLflow Quickstart 테스트 환경 구축

**구현 완료**: MLflow 사용을 위한 예제 및 테스트 도구

**주요 기능**:
- **로컬 테스트 스크립트** (`test_basic.py`):
  - 파라미터, 메트릭, artifact 업로드 검증
  - 83줄의 완전한 예제 코드
- **클러스터 내부 테스트** (`test_cluster.py`):
  - Pod 내부에서 MLflow 연결 검증
  - 31줄의 간결한 검증 스크립트
- **테스트 Pod 매니페스트** (`test-pod.yaml`):
  - Kubernetes 환경에서 즉시 테스트 가능
- **상세한 사용 가이드** (`README.md`):
  - 로컬 설정부터 클러스터 배포까지 단계별 설명

**파일 위치**: `examples/mlflow-quickstart/`

**관련 커밋**: `edcfca1`

---

### 3. ArgoCD 및 MLflow Helm 배포 자동화

**구현 완료**: GitOps 기반 자동 배포 시스템

**주요 구성**:
- **ArgoCD Helm Chart** (`bootstrap/argocd/`):
  - Helm dependency를 통한 ArgoCD 설치
  - Application CRD를 통한 MLflow 자동 배포
  - HTTP Ingress 설정 (HTTPS는 DDNS 제약으로 보류)
- **MLflow Helm Chart** (`deployments/mlflow/`):
  - Bitnami MLflow Helm Chart 활용
  - PostgreSQL 내장 배포
  - MinIO 연동 설정
  - 메모리 2Gi, Security Middleware 비활성화

**주요 설정**:
```yaml
# MLflow Helm values
image:
  tag: "2.19.0-debian-12-r0"
resources:
  limits:
    memory: 2Gi
tracking:
  extraFlags:
    - "--no-serve-artifacts"
artifactRoot: "s3://mlflow/"
```

**관련 커밋**: `3b8d1c4`, `89178f8`, `2976c18`, `5222184`

---

### 4. MinIO 및 Grafana Ingress 설정

**구현 완료**: 주요 인프라 서비스 외부 접근 설정

**MinIO**:
- **Console Ingress** (`minio..**********..com`): 웹 UI 접근
- **S3 API Ingress** (`minio-api..**********..com`): Artifact 업로드 엔드포인트
  - 대형 파일 지원 설정
  - 타임아웃 최적화

**Grafana**:
- **Dashboard Ingress** (`grafana..**********..com`): 모니터링 대시보드 접근

**DNS 설정**:
```
argocd..**********..com      → **********
mlflow..**********..com      → **********
grafana..**********..com     → **********
minio..**********..com       → **********
minio-api..**********..com   → **********
```

**관련 커밋**: `d8aaf1f`, `48b8f9d`, `93d47fc`

---

## 문서화 (Documentation)

### 1. 포괄적인 플랫폼 문서 작성

**총 9개 문서, 5,616줄 추가**

#### 핵심 가이드 문서 (필독)

**MLflow 배포 및 사용**:
- **`mlflow-deployment-complete.md`** (342줄):
  - MLflow + MinIO 통합 완료 과정 상세 기록
  - Artifact 업로드 문제 발견 및 해결 과정
  - 아키텍처 다이어그램 및 데이터 흐름
  - 대형 모델 지원 설정
  - 문제 해결 가이드
- **`mlflow-usage-guide.md`** (692줄):
  - 로컬 환경 vs 클러스터 환경 사용법
  - Python 클라이언트 설정
  - 실험 추적, 모델 등록, artifact 관리
  - RunPod/Kaggle/Colab 외부 GPU 연동
  - 실전 예제 코드

**플랫폼 아키텍처 및 전략**:
- **`ml-platform-strategy.md`** (953줄, 추천):
  - 외부 GPU 서비스 연동 전략 (RunPod, Colab, Kaggle)
  - Kubeflow vs 대안 도구 비교 분석
  - Airflow vs Argo Workflows 상세 비교
  - 단계별 플랫폼 구성 전략
  - 실제 추천 시스템 재학습 워크플로우 예제
- **`platform-architecture.md`** (750줄, 신규):
  - 통합 플랫폼 포털 아키텍처 설계
  - Frontend (React + Tailwind), Backend (FastAPI) 구조
  - 서비스 카탈로그 및 통합 인증
  - MLflow, Grafana, MinIO 임베딩 방안
- **`platform-implementation-guide.md`** (1,216줄, 신규):
  - Platform Portal 단계별 구현 가이드
  - 개발 환경 구축부터 프로덕션 배포까지
  - 코드 예제 및 베스트 프랙티스

**Kubernetes 기초**:
- **`ingress-routing.md`** (301줄, 신규):
  - Kubernetes Ingress 라우팅 원리 설명
  - Host 기반 vs Path 기반 라우팅
  - Nginx Ingress Controller 동작 원리
  - MLflow/MinIO/Grafana 라우팅 예제

**요약 문서**:
- **`platform-summary.md`** (354줄):
  - 플랫폼 핵심 내용 요약
  - 빠른 참조용 가이드

**관련 커밋**: `30e11e7`, `0c80326`, `10695b1`, `b75a4b2`

---

### 2. README 대규모 개편

**변경 사항**:
- **플랫폼 아키텍처 2계층 구조 명확화**:
  - **연구용 서비스 레이어**: MLflow, MinIO (연구자가 직접 사용)
  - **인프라 레이어**: ArgoCD, Grafana, Prometheus (플랫폼 운영)
- **서비스 URL 레이어별 구분**:
  - 연구용 서비스와 인프라 관리 서비스 분리 표시
- **최근 업데이트 섹션 추가** (2026-01-01):
  - MLflow 완전 검증 과정 요약
  - MinIO S3 API 외부 노출 중요성 강조
- **문서 섹션 재구성**:
  - 아키텍처 & 설계 / 구현 & 배포 / 서비스 가이드로 분류
  - 신규 문서 링크 추가 및 추천 표시

**플랫폼 구조 다이어그램 추가**:
```
┌─────────────────────────────────────────────┐
│         AI 연구 플랫폼                        │
├─────────────────────────────────────────────┤
│  연구용 서비스 레이어                      │
│  ├─ MLflow         (실험 추적)               │
│  └─ MinIO          (모델/데이터 저장)         │
│                                              │
│  → 연구자가 직접 사용하는 도구                │
├─────────────────────────────────────────────┤
│  인프라 레이어                            │
│  ├─ ArgoCD         (배포 자동화)             │
│  ├─ Grafana        (모니터링)                │
│  ├─ Prometheus     (메트릭 수집)             │
│  ├─ Loki           (로그 수집)               │
│  └─ Nginx Ingress  (라우팅)                  │
│                                              │
│  → 플랫폼 운영 및 관리 도구                   │
└─────────────────────────────────────────────┘
```

**관련 커밋**: `7b4a7d7`, `88b6635`, `42c4bee`, `cdb9b49`, `8964505`, `afce69c`

---

### 3. Helm 기반 배포 정책 문서화

**신규 문서**: `DEPLOYMENT-POLICY.md` (277줄)

**핵심 원칙**:
- **Helm Chart만 사용**: Chart.yaml + values.yaml
- **순수 Kubernetes 매니페스트 금지**
- **이유**: 버전 관리, 설정 관리, 재사용성 향상

**정책 내용**:
1. 모든 애플리케이션은 Helm Chart로 관리
2. 외부 Chart는 dependency로 활용
3. ArgoCD Application에서 Helm Chart 참조
4. 예외: Ingress, ConfigMap 등 단순 리소스

**관련 커밋**: `f79ccdd`

---

## 🔧 개선사항 (Improvements)

### 1. MLflow 설정 최적화

**보안 및 성능 개선**:
- **Security Middleware 비활성화**: Gunicorn 호환성 확보
- **메모리 제한 증가**: 1Gi → 2Gi (대형 모델 처리)
- **PostgreSQL 설정 수정**: 내장 PostgreSQL 사용
- **로그 비활성화**: gunicorn-opts 충돌 방지

**설정 변경**:
```yaml
tracking:
  extraFlags:
    - "--no-serve-artifacts"
    - "--gunicorn-opts='--timeout=300'"
extraEnvVars:
  MLFLOW_DISABLE_SECURITY_MIDDLEWARE: "true"
```

**관련 커밋**: `425f444`, `1671244`, `229b004`, `03c7b62`, `f2bf815`, `798afc8`, `39195c8`, `f6b27a9`

---

### 2. 프로젝트 구조 정리

**디렉토리 구조 확립**:
- **`bootstrap/argocd/`**: ArgoCD Helm 배포 및 Application 정의
- **`deployments/`**: 애플리케이션 Helm Chart (MLflow 등)
- **`infrastructure/`**: 인프라 리소스 (MinIO, Grafana Ingress)
- **`examples/`**: 사용 예제 및 테스트 코드
- **`docs/`**: 포괄적인 문서

**불필요한 디렉토리 제거**:
- `argocd-apps/` 삭제 (역할 불명확)
- `apps/README.md` 삭제 (중복 문서)

**관련 커밋**: `43d9069`, `afce69c`

---

### 3. Git 및 문서 관리 개선

**.gitignore 업데이트**:
```
# MLflow uploads
examples/mlflow-quickstart/mlruns/
examples/mlflow-quickstart/model.pkl
examples/mlflow-quickstart/artifacts/
examples/mlflow-quickstart/__pycache__/
```

**문서 구조 업데이트**:
- 현재 배포 상태 반영
- Phase 0-2 완료 표시
- MicroK8s Addon 활성화 현황 업데이트

**관련 커밋**: `edcfca1`, `1367f4f`, `d11b37b`, `8b314c7`

---

## 인프라 및 배포 (Infrastructure & Deployment)

### 1. Kubernetes 클러스터

**클러스터 정보**:
- **노드**: **********
- **도메인**: **********

---

### 2. GitOps 배포 파이프라인

**ArgoCD 기반 자동 배포**:
1. GitHub에 Helm Chart 커밋
2. ArgoCD가 변경 감지
3. 자동으로 Kubernetes에 동기화
4. UI에서 배포 상태 확인

**현재 관리 중인 애플리케이션**:
- ArgoCD 자체 (bootstrap)
- MLflow (Application CRD)

**향후 추가 예정**:
- JupyterHub
- Argo Workflows
- 기타 ML 도구

---

### 3. 서비스 네트워크 아키텍처

**외부 접근 가능한 서비스**:

| 서비스 | URL | 포트 | 용도 |
|--------|-----|------|------|
| ArgoCD | http://argocd..**********..com | 80 | GitOps 배포 관리 |
| MLflow | http://mlflow..**********..com | 80 | 실험 추적 및 모델 레지스트리 |
| Grafana | http://grafana..**********..com | 80 | 모니터링 대시보드 |
| MinIO Console | http://minio..**********..com | 80 | 스토리지 관리 UI |
| MinIO S3 API | http://minio-api..**********..com | 80 | Artifact 업로드 엔드포인트 |

**라우팅 구조**:
```
외부 요청
    ↓
라우터 포트포워딩 (80 → xxx.xxx.x.xx:80)
    ↓
Nginx Ingress Controller
    ↓
Host 기반 라우팅 (mlflow..**********..com → MLflow Service)
```

---

## 성과 지표 (Metrics)

### 커밋 통계

- **총 커밋 수**: 31개
- **커밋 타입 분포**:
  - `docs`: 13개 (42%)
  - `fix`: 8개 (26%)
  - `feat`: 7개 (23%)
  - `refactor`: 2개 (6%)
  - `chore`: 1개 (3%)

### 코드 변경 통계

```
32 files changed
5,616 insertions(+)
261 deletions(-)
```

**주요 변경 파일**:
- 신규 문서: 9개 (5,300줄+)
- Helm Chart: 3개 (Chart.yaml, values.yaml)
- Ingress 설정: 3개 (MLflow, MinIO, Grafana)
- 예제 코드: 4개 (테스트 스크립트, Pod 정의)

### 인프라 현황

**Phase 0-2 완료**:
- MicroK8s 설치 및 Addon 활성화
- ArgoCD GitOps 배포
- MLflow + PostgreSQL + MinIO 통합 완료
- Grafana 모니터링 대시보드

**운영 중인 서비스**: 5개
**작성된 문서**: 13개 (Setup Guide 포함)

---

## 주요 성과

### 1. 완전히 작동하는 ML 실험 플랫폼 구축 

**로컬 노트북에서 가능한 작업**:
1. 모델 학습 및 파라미터 튜닝
2. MLflow로 실험 추적 (파라미터, 메트릭)
3. **학습된 모델을 MinIO에 업로드** (수십 GB 지원)
4. MLflow UI에서 실험 비교 및 시각화
5. 모델 레지스트리에서 버전 관리

**검증 완료**:
- 로컬 Python 스크립트에서 전체 워크플로우 동작
- 대형 모델 업로드 성공 (타임아웃 1시간)
- MLflow UI 및 MinIO Console에서 데이터 확인

---

### 2. 외부 GPU 서비스 연동 전략 확립

**지원하는 환경**:
- **RunPod**: 경제적인 GPU 인스턴스
- **Kaggle**: 무료 GPU (주 30시간)
- **Google Colab**: 무료/유료 GPU

**워크플로우**:
1. 외부 GPU 환경에서 모델 학습
2. MLflow 클라이언트로 메트릭 전송
3. 학습 완료 후 모델을 MinIO에 업로드
4. 로컬에서 MLflow UI로 결과 확인

**장점**:
- 온프레미스 GPU 투자 없이 ML 연구 가능
- 비용 효율적 (사용한 만큼만 지불)
- 세션 독립적 (언제든 재연결 가능)

---

### 3. 확장 가능한 플랫폼 아키텍처 설계

**현재 (Phase 2)**:
- MLflow + MinIO (연구용 최소 구성)

**향후 확장 계획**:

**Phase 3 - 프로덕션**:
- Model Serving (TorchServe, vLLM)
- API Gateway (Kong, Traefik)

**Phase 4 - 자동화 (선택)**:
- Airflow (데이터 ETL, 재학습 스케줄링)
- Argo Workflows (ML 파이프라인 오케스트레이션)

**설계 원칙**:
- 점진적 개선 (필요할 때만 복잡도 추가)
- 적재적소 (각 도구의 강점 활용)
- 과도한 엔지니어링 방지

---

### 4. 포괄적인 문서화 체계 완성

**문서 계층 구조**:

**Tier 1 - 빠른 시작**:
- README.md: 프로젝트 개요 및 현황
- quick-start.md: 빠른 시작 가이드

**Tier 2 - 핵심 가이드** (필독):
- mlflow-usage-guide.md: MLflow 사용법
- mlflow-deployment-complete.md: 배포 완료 과정
- ml-platform-strategy.md: 플랫폼 확장 전략

**Tier 3 - 아키텍처 설계**:
- architecture.md: 전체 아키텍처
- platform-architecture.md: Platform Portal 설계
- ingress-routing.md: Kubernetes 라우팅 원리

**Tier 4 - 구현 가이드**:
- setup-guide.md: 설치 가이드
- implementation-plan.md: 구현 계획
- platform-implementation-guide.md: Platform 구현 가이드

**특징**:
- 이론과 실전의 균형
- 코드 예제 풍부
- 문제 해결 과정 상세 기록
- 한국어로 작성 (접근성 향상)

---

## 다음 단계 (Phase 3+)

### 단기 (1-2주)

**1. MLflow 사용성 개선**:
- [ ] 사용자 인증 및 권한 관리
- [ ] 실험 템플릿 및 베스트 프랙티스 정리
- [ ] 백업 및 복구 전략 수립

**2. 모니터링 강화**:
- [ ] Grafana 대시보드 커스터마이징
- [ ] MLflow 메트릭 시각화
- [ ] MinIO 스토리지 사용량 모니터링

**3. 문서 개선**:
- [ ] 외부 GPU 서비스별 상세 가이드
- [ ] 트러블슈팅 FAQ 작성
- [ ] 비디오 튜토리얼 제작

---

### 중기 (1-2개월)

**1. JupyterHub 배포**:
- 멀티 유저 노트북 환경
- MLflow 사전 설정
- 팀 협업 워크스페이스

**2. Model Serving 추가**:
- TorchServe 또는 vLLM 배포
- 학습된 모델 API 서빙
- A/B 테스트 지원

**3. HTTPS 재시도**:
- Let's Encrypt 인증서
- 자체 SSL 인증서 고려
- DDNS 대안 탐색

---

### 장기 (3-6개월)

**1. 워크플로우 자동화**:
- Airflow 배포 (데이터 파이프라인)
- Argo Workflows (ML 파이프라인)
- CI/CD 통합

**2. Platform Portal 구현**:
- 통합 대시보드 (React + FastAPI)
- 서비스 카탈로그
- 통합 인증 (OAuth2)
- MLflow, Grafana, MinIO 임베딩

**3. GPU 인프라 확장**:
- 온프레미스 GPU 서버 추가 (예산 허용 시)
- NVIDIA GPU Operator 설치
- GPU 리소스 스케줄링

---

## 기술적 하이라이트

### 1. MinIO S3 API Ingress 설정

**핵심 Annotation**:
```yaml
nginx.ingress.kubernetes.io/proxy-body-size: "0"  # 무제한
nginx.ingress.kubernetes.io/proxy-request-buffering: "off"
nginx.ingress.kubernetes.io/proxy-buffering: "off"
nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"  # 1시간
nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
nginx.ingress.kubernetes.io/proxy-connect-timeout: "300"
nginx.ingress.kubernetes.io/client-body-timeout: "3600"
```

**효과**:
- 수십 GB 모델 업로드 가능
- 메모리 효율적 (버퍼링 없음)
- 안정적인 전송 (타임아웃 충분)

---

### 2. MLflow 환경변수 설정

**로컬 환경**:
```python
import os
os.environ['MLFLOW_TRACKING_URI'] = 'http://mlflow..**********..com'
os.environ['MLFLOW_S3_ENDPOINT_URL'] = 'http://minio-api..**********..com'
os.environ['AWS_ACCESS_KEY_ID'] = 'your-access-key'
os.environ['AWS_SECRET_ACCESS_KEY'] = 'your-secret-key'
```

**클러스터 내부**:
```yaml
env:
  - name: MLFLOW_TRACKING_URI
    value: "http://mlflow-tracking.mlflow.svc.cluster.local"
  - name: MLFLOW_S3_ENDPOINT_URL
    value: "http://minio.minio-operator.svc.cluster.local"
```

**차이점**:
- 로컬: 외부 도메인 사용 (`minio-api..**********..com`)
- 클러스터: 내부 서비스명 사용 (`.svc.cluster.local`)

---

### 3. Helm Chart Dependency 관리

**ArgoCD Chart.yaml**:
```yaml
apiVersion: v2
name: argocd
version: 0.1.0
dependencies:
  - name: argo-cd
    version: "7.7.12"
    repository: https://argoproj.github.io/argo-helm
```

**MLflow Chart.yaml**:
```yaml
apiVersion: v2
name: mlflow
version: 0.1.0
dependencies:
  - name: mlflow
    version: "1.6.8"
    repository: https://charts.bitnami.com/bitnami
```

**장점**:
- 외부 Chart 버전 고정
- `helm dependency update`로 자동 다운로드
- ArgoCD와 완벽 통합

---

## 학습 및 인사이트

### 1. Kubernetes Ingress의 중요성

**문제**: MinIO Console은 접근 가능하지만 S3 API는 불가

**원인**: Ingress가 Console용 9090 포트만 노출

**해결**: S3 API용 별도 Ingress 생성 (포트 80 → 9000)

**교훈**:
- 서비스마다 여러 포트를 가질 수 있음
- Ingress는 포트별로 별도 설정 필요
- Host 기반 라우팅으로 명확한 분리

---

### 2. 대형 파일 전송 최적화

**문제**: 수 GB 모델 업로드 시 타임아웃 또는 메모리 부족

**해결**:
1. `proxy-body-size: "0"`: 파일 크기 제한 제거
2. `proxy-buffering: off`: 버퍼링 비활성화 (메모리 절약)
3. `proxy-read-timeout: 3600`: 충분한 타임아웃

**교훈**:
- Nginx 기본 설정은 소형 파일용
- ML 워크로드는 대형 파일 고려 필수
- Annotation으로 세밀한 제어 가능

---

### 3. Helm vs Kubernetes Manifest

**초기 시도**: 순수 YAML 매니페스트로 MLflow 배포

**문제점**:
- 설정 변경마다 모든 YAML 수정
- 버전 관리 어려움
- 재사용 불가능

**Helm 도입 후**:
- `values.yaml` 하나로 모든 설정 관리
- Chart 버전으로 롤백 가능
- ArgoCD와 완벽 통합

**교훈**:
- 간단한 리소스 외에는 Helm 사용 권장
- GitOps에서 Helm은 필수
- 학습 곡선이 있지만 장기적으로 유리

---

### 4. 문서화의 중요성

**초기**: 설정 과정을 기억에 의존

**문제점**:
- 재설치 시 과정 재현 어려움
- 팀원과 공유 불가능
- 문제 해결 시간 낭비

**문서화 후**:
- 모든 단계 재현 가능
- 문제 해결 과정 공유
- 새로운 사용자 온보딩 용이

**교훈**:
- 실시간으로 문서 작성 (나중은 없음)
- 문제 해결 과정도 기록 (트러블슈팅 자산)
- 코드 예제 필수 (이론만으로 부족)

---

## 결론

이번 주는 **Research Platform의 핵심 기능이 완성된 전환점**이었습니다. MLflow + MinIO 통합을 통해 **완전히 작동하는 ML 실험 추적 플랫폼**을 구축했으며, 로컬 환경에서 대형 모델을 업로드할 수 있는 완전한 워크플로우를 확립했습니다.

**핵심 성과**:
1. **MLflow 완전 배포**: 로컬에서 전체 기능 사용 가능
2. **대형 모델 지원**: 수십 GB 모델 업로드 검증
3. **외부 GPU 연동**: RunPod/Kaggle/Colab 활용 전략
4. **포괄적인 문서화**: 9개 문서, 5,600+ 줄

**기술적 달성**:
- GitOps 기반 자동 배포 (ArgoCD)
- Helm Chart 기반 애플리케이션 관리
- Kubernetes Ingress를 통한 서비스 노출
- 대형 파일 전송 최적화

**다음 단계**:
- Phase 3: MLflow 사용성 개선 및 모니터링 강화
- Phase 4+: JupyterHub, Model Serving 추가
- 장기: Platform Portal 구현 및 워크플로우 자동화

이제 연구자는 **개인 노트북에서 실험을 진행하고, 외부 GPU로 대형 모델을 학습하며, 모든 결과를 중앙 플랫폼에서 관리**할 수 있습니다. 이는 소규모 연구팀이 엔터프라이즈급 ML 인프라의 핵심 기능을 활용할 수 있는 토대를 마련했습니다.

---

**문서 작성일**: 2026년 1월 4일  
**Phase 상태**: Phase 2 완료, Phase 3 준비 중
