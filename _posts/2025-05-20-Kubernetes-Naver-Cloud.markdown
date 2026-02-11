---
layout: post
title:  "Kubernetes 기본 설정 가이드 - Naver Cloud"
description: 네이버 클라우드 플랫폼에서 Kubernetes 클러스터 구축 및 기본 설정
date:   2025-05-20 00:00:00 +000
categories: Kubernetes NCloud DevOps Container
---

## 개요
네이버 클라우드 플랫폼(NCP)에서 Kubernetes 클러스터를 생성하고 기본 설정하는 방법

## Naver Cloud Platform Kubernetes Service (NKS)

네이버 클라우드는 관리형 Kubernetes 서비스인 NKS(Naver Kubernetes Service)를 제공합니다.<br>
컨트롤 플레인을 자동으로 관리해주기 때문에 사용자는 워커 노드와 애플리케이션에만 집중할 수 있습니다.

## 클러스터 생성

### 1. NCP 콘솔 접속
<a href="https://console.ncloud.com" target="_blank">[네이버 클라우드 콘솔]</a><br>

Services > Container > Kubernetes Service 메뉴로 이동

### 2. 클러스터 생성 설정

**기본 설정:**
```
클러스터 이름: my-k8s-cluster
Kubernetes 버전: 1.27.x (최신 안정화 버전 선택)
VPC: 기존 VPC 선택 또는 신규 생성
Subnet: Private Subnet 선택 (권장)
로그인 키: SSH 접근용 키 선택
```

**노드 풀 설정:**
```
노드 풀 이름: worker-pool-01
노드 수: 2개 (최소 권장)
노드 타입: Standard (2vCPU, 8GB) 이상
스토리지: 50GB 이상
```

## kubectl 설정

### 1. kubectl 설치
```bash
# macOS
brew install kubectl

# Linux
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Windows (chocolatey)
choco install kubernetes-cli
```

### 2. kubeconfig 다운로드
NCP 콘솔에서 생성된 클러스터의 kubeconfig 파일을 다운로드합니다.

```bash
# kubeconfig 파일 위치 설정
export KUBECONFIG=~/Downloads/nks-kubeconfig.yaml

# 또는 기본 위치로 복사
mkdir -p ~/.kube
cp ~/Downloads/nks-kubeconfig.yaml ~/.kube/config
```

### 3. 클러스터 연결 확인
```bash
# 클러스터 정보 확인
kubectl cluster-info

# 노드 목록 확인
kubectl get nodes

# 결과 예시
NAME                          STATUS   ROLES    AGE   VERSION
worker-pool-01-node-1         Ready    <none>   5m    v1.27.3
worker-pool-01-node-2         Ready    <none>   5m    v1.27.3
```

## 기본 리소스 배포

### 1. Namespace 생성
```bash
kubectl create namespace my-app
```

### 2. 샘플 애플리케이션 배포
```yaml
# nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: my-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
```

```bash
kubectl apply -f nginx-deployment.yaml
```

### 3. Service 생성
```yaml
# nginx-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: my-app
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

```bash
kubectl apply -f nginx-service.yaml

# LoadBalancer IP 확인 (NCP에서 자동 할당)
kubectl get svc -n my-app
```

## 모니터링 및 로그 확인

### Pod 상태 확인
```bash
# 전체 Pod 조회
kubectl get pods -n my-app

# Pod 상세 정보
kubectl describe pod <pod-name> -n my-app

# Pod 로그 확인
kubectl logs <pod-name> -n my-app

# 실시간 로그
kubectl logs -f <pod-name> -n my-app
```

### 리소스 사용량 확인
```bash
# 노드 리소스 확인
kubectl top nodes

# Pod 리소스 확인
kubectl top pods -n my-app
```

## LoadBalancer 설정

NCP에서는 Kubernetes Service의 LoadBalancer 타입을 사용하면 자동으로 NCP Load Balancer가 생성됩니다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  annotations:
    # NCP LoadBalancer 설정
    service.beta.kubernetes.io/ncloud-load-balancer-layer-type: "nlb"
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
```

## 스토리지 설정 (Persistent Volume)

```yaml
# pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
  namespace: my-app
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: nks-block-storage
```

NCP에서 제공하는 스토리지 클래스:
- `nks-block-storage`: 블록 스토리지 (기본)
- `nks-nas`: NAS 스토리지

## 주의사항

### 비용 관리
- LoadBalancer 생성 시 별도 비용 발생
- 사용하지 않는 리소스는 즉시 삭제 권장
- 노드 수와 타입은 실제 부하에 맞게 조정

### 보안 설정
- Private Subnet 사용 권장
- ACG(Access Control Group)로 네트워크 접근 제어
- RBAC 설정으로 권한 관리

### 백업
```bash
# 리소스 백업
kubectl get all -n my-app -o yaml > backup.yaml

# 복원
kubectl apply -f backup.yaml
```

## 유용한 명령어 모음

```bash
# 모든 리소스 조회
kubectl get all -n my-app

# 특정 리소스 삭제
kubectl delete deployment nginx-deployment -n my-app

# 컨텍스트 확인
kubectl config get-contexts

# 현재 컨텍스트 설정
kubectl config use-context <context-name>

# Pod 내부 접속
kubectl exec -it <pod-name> -n my-app -- /bin/bash

# Port Forward (로컬 테스트)
kubectl port-forward svc/nginx-service 8080:80 -n my-app
```

## 마무리
네이버 클라우드의 NKS를 사용하면 복잡한 Kubernetes 클러스터 관리 없이 빠르게 컨테이너 기반 애플리케이션을 배포할 수 있습니다.<br>
다음에는 CI/CD 파이프라인 구축 및 Auto Scaling 설정에 대해 다뤄보겠습니다.

