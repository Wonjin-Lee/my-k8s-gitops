# 🏗️ Kubernetes Cloud-Native Infrastructure Setup
이 저장소는 Kubernetes, Istio, ArgoCD를 활용한 GitOps 기반의 MSA 환경 구축 프로젝트입니다. 클러스터 초기 구축 시 아래 가이드를 따라 인프라를 설정합니다.

## 📋 사전 요구 사항
 - Kubernetes Cluster (Docker Desktop, minikube, EKS 등)
 - Helm 3.x 이상
 - kubectl

---

### 1. Istio (Service Mesh) 설치
Istio는 Helm을 사용하여 관리하며, 서비스 간 보안 통신(mTLS) 및 트래픽 제어를 담당합니다.
```
# 1-1. Helm 리포지토리 추가 및 업데이트
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update

# 1-2. Istio Base 설치 (커스텀 리소스 정의 - CRD)
kubectl create namespace istio-system
helm install istio-base istio/base -n istio-system

# 1-3. Istiod 설치 (컨트롤 플레인 - 트래픽 관리 및 보안)
helm install istiod istio/istiod -n istio-system --wait

# 1-4. Istio Ingress Gateway 설치 (외부 트래픽 인입점)
kubectl create namespace istio-ingress
helm install istio-ingress istio/gateway -n istio-ingress --wait
```

### 2. 관측성(Observability) 도구 설치
모니터링, 로깅, 시각화를 위해 Istio 공식 애드온을 설치합니다.
```
# Prometheus (메트릭 수집 및 저장)
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/prometheus.yaml

# Grafana (데이터 시각화 대시보드)
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/grafana.yaml

# Kiali (서비스 메시 트래픽 그래프 시각화)
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/kiali.yaml
```

### 3. Sidecar Injection 활성화
애플리케이션이 배포될 때 자동으로 Istio Proxy(Envoy)가 주입되도록 네임스페이스에 라벨을 설정합니다.
```
# default 네임스페이스에 자동 주입 활성화
kubectl label namespace default istio-injection=enabled --overwrite
```

### 4. ArgoCD (GitOps) 설치 및 설정
Git 저장소의 상태를 클러스터와 동기화하기 위한 도구입니다.
```
# ArgoCD 설치
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# ArgoCD API Server 포트 포워딩 (접속용)
# kubectl port-forward svc/argocd-server -n argocd 8080:443
```

---

## 🛠️ 주요 리소스 구조
- Gateway: 클러스터 외부 포트(80) 오픈
- VirtualService: 트래픽 라우팅 규칙 (Canary, Header-based)
- DestinationRule: 서비스 그룹화(v1, v2) 및 트래픽 정책(mTLS, 서킷 브레이커)
- Service: Kubernetes 내부 로드밸런싱 주소