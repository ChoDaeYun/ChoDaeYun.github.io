---
layout: post
title:  "CI/CD 파이프라인 구축 - GitHub Actions + ArgoCD"
description: GitHub Actions으로 CI, ArgoCD로 CD 구성 및 Grafana 모니터링 설정
date:   2025-06-15 00:00:00 +000
categories: CICD GitHubActions ArgoCD Kubernetes Grafana
---

## 개요
GitHub Actions를 활용한 CI(Continuous Integration)와 ArgoCD를 활용한 CD(Continuous Deployment) 파이프라인 구축<br>
그리고 Grafana를 통한 모니터링 환경 설정

## CI/CD 아키텍처

```
코드 푸시 (GitHub)
    ↓
GitHub Actions (CI)
    ↓
Docker 이미지 빌드 → Container Registry
    ↓
Manifest 업데이트 (Git Repository)
    ↓
ArgoCD (CD)
    ↓
Kubernetes 클러스터 배포
    ↓
Grafana 모니터링
```

## GitHub Actions (CI) 설정

### 1. Workflow 파일 생성
```yaml
# .github/workflows/ci.yml
name: CI Pipeline

on:
  push:
    branches:
      - main
      - develop
  pull_request:
    branches:
      - main

env:
  DOCKER_IMAGE: ghcr.io/${{ github.repository }}
  VERSION: ${{ github.sha }}

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v3

    - name: Set up JDK 17
      uses: actions/setup-java@v3
      with:
        java-version: '17'
        distribution: 'temurin'

    - name: Build with Gradle
      run: |
        chmod +x gradlew
        ./gradlew clean build

    - name: Run tests
      run: ./gradlew test

    - name: Login to GitHub Container Registry
      uses: docker/login-action@v2
      with:
        registry: ghcr.io
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}

    - name: Build and Push Docker image
      run: |
        docker build -t $DOCKER_IMAGE:$VERSION .
        docker tag $DOCKER_IMAGE:$VERSION $DOCKER_IMAGE:latest
        docker push $DOCKER_IMAGE:$VERSION
        docker push $DOCKER_IMAGE:latest

    - name: Update Kubernetes manifests
      run: |
        git config --global user.name "GitHub Actions"
        git config --global user.email "actions@github.com"
        git clone https://${{ secrets.GIT_TOKEN }}@github.com/your-org/k8s-manifests.git
        cd k8s-manifests
        sed -i "s|image:.*|image: $DOCKER_IMAGE:$VERSION|g" deployment.yaml
        git add .
        git commit -m "Update image to $VERSION"
        git push
```

### 2. Dockerfile 작성
```dockerfile
# Dockerfile
FROM openjdk:17-jdk-slim

WORKDIR /app

COPY build/libs/*.jar app.jar

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 3. GitHub Secrets 설정
GitHub Repository > Settings > Secrets and variables > Actions

```
GIT_TOKEN: GitHub Personal Access Token (k8s-manifests 저장소 접근용)
```

## ArgoCD (CD) 설정

### 1. ArgoCD 설치
```bash
# ArgoCD namespace 생성
kubectl create namespace argocd

# ArgoCD 설치
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# ArgoCD Server 서비스 타입 변경 (LoadBalancer)
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

# ArgoCD 초기 비밀번호 확인
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

### 2. ArgoCD CLI 설치
```bash
# macOS
brew install argocd

# Linux
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd
sudo mv argocd /usr/local/bin/
```

### 3. ArgoCD 로그인
```bash
# ArgoCD 서버 IP 확인
kubectl get svc argocd-server -n argocd

# ArgoCD 로그인
argocd login <ARGOCD_SERVER_IP>

# 비밀번호 변경
argocd account update-password
```

### 4. Application 생성
```yaml
# argocd-application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://github.com/your-org/k8s-manifests.git
    targetRevision: HEAD
    path: .

  destination:
    server: https://kubernetes.default.svc
    namespace: my-app

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
    - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

```bash
# Application 배포
kubectl apply -f argocd-application.yaml

# Application 상태 확인
argocd app get my-app

# 수동 동기화 (필요시)
argocd app sync my-app
```

### 5. ArgoCD 대시보드 접속
```bash
# LoadBalancer IP 확인
kubectl get svc argocd-server -n argocd

# 브라우저에서 https://<LOADBALANCER_IP> 접속
# ID: admin
# PW: 위에서 확인한 초기 비밀번호
```

## Grafana 모니터링 설정

### 1. Prometheus + Grafana 설치 (Helm)
```bash
# Helm 설치
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Prometheus 설치
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

kubectl create namespace monitoring

helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.prometheusSpec.retention=30d \
  --set grafana.service.type=LoadBalancer
```

### 2. Grafana 접속
```bash
# Grafana 서비스 확인
kubectl get svc -n monitoring | grep grafana

# 기본 계정 정보
# ID: admin
# PW 확인:
kubectl get secret -n monitoring prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 -d
```

### 3. Grafana 대시보드 설정

**Kubernetes 클러스터 모니터링 대시보드 추가:**
1. Grafana 접속 후 왼쪽 메뉴 > Dashboards > Import
2. 대시보드 ID 입력:
   - **6417**: Kubernetes Cluster Monitoring
   - **8588**: Kubernetes Deployment Metrics
   - **7249**: Kubernetes Pod Metrics

3. Data Source: Prometheus 선택
4. Import 클릭

### 4. 애플리케이션 로그 수집 (Loki)
```bash
# Loki 설치
helm install loki grafana/loki-stack \
  --namespace monitoring \
  --set grafana.enabled=false \
  --set promtail.enabled=true
```

**Grafana에서 Loki 데이터 소스 추가:**
1. Configuration > Data Sources > Add data source
2. Loki 선택
3. URL: `http://loki.monitoring.svc.cluster.local:3100`
4. Save & Test

### 5. 로그 조회
Grafana > Explore 메뉴에서 Loki 선택 후 쿼리:

```logql
# 특정 namespace 로그
{namespace="my-app"}

# 특정 Pod 로그
{namespace="my-app", pod=~"my-app-.*"}

# 에러 로그만 필터링
{namespace="my-app"} |= "ERROR"

# 최근 1시간 로그
{namespace="my-app"} [1h]
```

## 알림 설정 (Grafana Alert)

### Slack 알림 설정
```yaml
# Alerting > Contact points > New contact point
Name: slack-alerts
Type: Slack
Webhook URL: https://hooks.slack.com/services/YOUR/WEBHOOK/URL
```

### Alert Rule 생성
```
Alerting > Alert rules > New alert rule

조건 예시:
- Pod Restart 횟수가 5회 이상
- CPU 사용률 80% 이상
- Memory 사용률 90% 이상
- Pod Pending 상태 5분 이상
```

## 전체 배포 플로우

### 1. 코드 푸시
```bash
git add .
git commit -m "feat: 새로운 기능 추가"
git push origin main
```

### 2. GitHub Actions 자동 실행
- 빌드 및 테스트
- Docker 이미지 생성 및 푸시
- Kubernetes manifest 업데이트

### 3. ArgoCD 자동 동기화
- Git Repository 변경 감지
- Kubernetes 클러스터에 자동 배포
- Health Check 및 Sync Status 확인

### 4. Grafana 모니터링
- 실시간 메트릭 확인
- 로그 조회 및 분석
- 알림 수신 (이상 발생 시)

## 유용한 명령어

```bash
# ArgoCD 애플리케이션 목록
argocd app list

# ArgoCD 애플리케이션 상세 정보
argocd app get my-app

# ArgoCD 배포 히스토리
argocd app history my-app

# ArgoCD Rollback
argocd app rollback my-app <REVISION>

# Grafana Pod 로그 확인
kubectl logs -f -n monitoring <grafana-pod-name>

# Prometheus 타겟 상태 확인
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090
# 브라우저에서 http://localhost:9090/targets 접속
```

## 모니터링 주요 메트릭

### 클러스터 레벨
- Node CPU/Memory 사용률
- Pod 개수 및 상태
- Persistent Volume 사용량

### 애플리케이션 레벨
- Request per Second (RPS)
- Response Time
- Error Rate
- JVM Heap Memory (Java 애플리케이션)

### 배포 메트릭
- 배포 성공/실패 횟수
- 배포 소요 시간
- Rollback 횟수

## 주의사항

### 보안
- ArgoCD 비밀번호 변경 필수
- GitHub Token은 최소 권한만 부여
- Grafana는 인증 설정 필수

### 리소스 관리
- Prometheus retention 기간 적절히 설정 (디스크 사용량)
- Loki 로그 보관 기간 설정
- 불필요한 메트릭 수집 비활성화

### 백업
```bash
# ArgoCD 애플리케이션 백업
argocd app get my-app -o yaml > my-app-backup.yaml

# Grafana 대시보드 백업
# Grafana UI에서 Dashboard > Settings > JSON Model 복사
```

## 마무리
GitHub Actions와 ArgoCD를 활용하면 코드 커밋부터 배포까지 완전 자동화된 파이프라인을 구축할 수 있습니다.<br>
Grafana를 통해 실시간 모니터링과 로그 분석이 가능하여 안정적인 서비스 운영이 가능합니다.

