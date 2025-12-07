# Kubernetes (k8s) 学習ガイド

このドキュメントは、Kubernetesの基礎から本番運用まで段階的に学習した記録です。

---

## 環境構築

### 必要なツールのインストール

```bash
# minikubeのインストール (macOS ARM64)
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-darwin-arm64
sudo install minikube-darwin-arm64 /usr/local/bin/minikube

# kubectlのインストール
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/arm64/kubectl"
sudo install -o root -g wheel -m 0755 kubectl /usr/local/bin/kubectl

# Helmのインストール（Phase 4で使用）
brew install helm

# minikube起動
minikube start

# 確認
kubectl cluster-info
kubectl get nodes
```

---

## Phase 1: k8s基礎

### Step 1: Pod のデプロイ

#### Podとは
- Kubernetesの最小単位
- 1つ以上のコンテナをまとめたもの
- 一時的な存在（削除されたら復活しない）

#### nginx-pod.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-first-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
```

```bash
kubectl apply -f nginx-pod.yaml
kubectl get pods
kubectl exec -it my-first-pod -- bash
```

---

### Step 2: Service のデプロイ（L4ロードバランス）

#### Serviceとは
- Podへの入り口
- 複数のPodに負荷分散

#### nginx-service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
```

---

### Step 3: Deployment のデプロイ

#### Deploymentとは
- Podの管理者
- オートヒーリング（自動復旧）
- ローリングアップデート

#### nginx-deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
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
        image: nginx:latest
        ports:
        - containerPort: 80
```

---

### Step 4-6: ConfigMap / Secret

#### app-configmap.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: "production"
  LOG_LEVEL: "info"
```

#### app-secret.yaml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  DB_PASSWORD: cGFzc3dvcmQxMjM=  # base64エンコード
```

---

## Phase 2: k8s応用

### Step 7: Ingress のデプロイ（L7ロードバランス）

#### L4 vs L7

| 項目 | L4 (Service) | L7 (Ingress) |
|------|--------------|--------------|
| 見るもの | ポート番号 | URLパス |
| 例 | `:30080` → Pod | `/api` → API Pod |

#### Ingress Controller有効化

```bash
minikube addons enable ingress
```

#### my-ingress.yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /v1
        pathType: Prefix
        backend:
          service:
            name: app-v1-service
            port:
              number: 80
      - path: /v2
        pathType: Prefix
        backend:
          service:
            name: app-v2-service
            port:
              number: 80
```

```bash
minikube tunnel  # 別ターミナルで
curl http://127.0.0.1/v1  # → Hello from App V1!
curl http://127.0.0.1/v2  # → Hello from App V2!
```

---

### Step 8: PV / PVC のデプロイ（永続化ストレージ）

#### PV と PVC

- **PV** = 駐車場（管理者が用意する実際のスペース）
- **PVC** = 駐車券（開発者の「スペースください」という申請）

#### my-pvc.yaml

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

#### PVCをマウントしたPod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod-pvc
spec:
  containers:
  - name: nginx
    image: nginx:latest
    volumeMounts:
    - name: my-storage
      mountPath: /data
  volumes:
  - name: my-storage
    persistentVolumeClaim:
      claimName: my-pvc
```

---

## Phase 3: CI/CD

### Step 9: GitHub Actions でDockerイメージ自動ビルド

#### myapp/Dockerfile

```dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
```

#### .github/workflows/docker-build.yml

```yaml
name: Build and Push Docker Image

on:
  push:
    branches:
      - main
    paths:
      - 'myapp/**'

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Login to DockerHub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and Push
        uses: docker/build-push-action@v5
        with:
          context: ./myapp
          push: true
          tags: <username>/myapp:latest
```

---

### Step 10: ArgoCD でGitOps自動デプロイ

#### ArgoCDとは

GitHubのYAMLを監視 → 変更があったら自動でk8sにデプロイ

#### インストール

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

#### UIにアクセス

```bash
# パスワード取得
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d ; echo

# ポートフォワード
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

https://localhost:8080 でログイン（admin / パスワード）

#### manifests/myapp-deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: <username>/myapp:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  type: NodePort
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30000
```

---

## Phase 4: モニタリング

### Step 11: Prometheus + Grafana

#### 全体像

```
Node/Pod → メトリクス収集 → Prometheus → Grafana で可視化
              ↑
         node-exporter
```

- **Prometheus** = メトリクスを集めて保存するDB
- **Grafana** = グラフで見せるダッシュボード
- **node-exporter** = サーバー情報を収集するエージェント

#### インストール（Helm使用）

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace
```

#### Grafana にアクセス

```bash
# パスワード取得
kubectl --namespace monitoring get secrets monitoring-grafana \
  -o jsonpath="{.data.admin-password}" | base64 -d ; echo

# ポートフォワード
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80
```

http://localhost:3000 でログイン（admin / パスワード）

#### おすすめダッシュボード

- **Node Exporter / Nodes** - サーバーのCPU、メモリ、ディスク
- **Kubernetes / Compute Resources / Cluster** - クラスタ全体
- **Kubernetes / Compute Resources / Pod** - Pod ごとのリソース

---

## 全体構成図

```
開発者
  ↓ git push
GitHub ─────────────────────────────────────┐
  │                                          │
  ├─→ GitHub Actions                         │
  │     ↓ docker build & push                │
  │   DockerHub                              │
  │     ↓ イメージ取得                        │
  │                                          │
  └─→ ArgoCD (マニフェスト監視) ←────────────┘
        ↓ kubectl apply
      k8s Cluster
        ├─ Ingress（L7ルーティング）
        ├─ Service（L4負荷分散）
        ├─ Deployment（Pod管理）
        │    └─ Pod → コンテナ
        ├─ ConfigMap / Secret（設定）
        ├─ PVC → PV（永続ストレージ）
        └─ Prometheus + Grafana（監視）
```

---

## CI/CD の流れ

```
コード変更 → git push
    ↓
GitHub Actions: docker build → DockerHubにpush
    ↓
ArgoCD: マニフェスト監視 → k8s自動更新
    ↓
k8s: 新しいPodに入れ替え
    ↓
Grafana: メトリクスを監視
```

---

## 学習ロードマップ

### Phase 1: k8s基礎 ✅ 完了
- [x] Pod, Deployment, Service
- [x] ConfigMap, Secret
- [x] オートヒーリング

### Phase 2: k8s応用 ✅ 完了
- [x] Ingress（L7ロードバランス）
- [x] PV, PVC（永続化ストレージ）

### Phase 3: CI/CD ✅ 完了
- [x] GitHub Actions → DockerHub
- [x] ArgoCD（GitOps）

### Phase 4: モニタリング ✅ 完了
- [x] Prometheus + Grafana

---

## 作成したファイル一覧

### Phase 1: k8s基礎
| ファイル | 内容 |
|----------|------|
| nginx-pod.yaml | Pod |
| nginx-service.yaml | Service |
| nginx-deployment.yaml | Deployment |
| app-configmap.yaml | ConfigMap |
| app-secret.yaml | Secret |
| app-deployment.yaml | 環境変数を使うDeployment |

### Phase 2: k8s応用
| ファイル | 内容 |
|----------|------|
| app-v1.yaml | テスト用アプリV1 |
| app-v2.yaml | テスト用アプリV2 |
| my-ingress.yaml | Ingress |
| my-pv.yaml | PersistentVolume |
| my-pvc.yaml | PersistentVolumeClaim |
| test-pod-with-pvc.yaml | PVCマウントPod |

### Phase 3: CI/CD
| ファイル | 内容 |
|----------|------|
| myapp/index.html | Webページ |
| myapp/Dockerfile | Docker定義 |
| .github/workflows/docker-build.yml | GitHub Actions |
| manifests/myapp-deployment.yaml | ArgoCD用マニフェスト |

---

## Q&A

### Q1: イメージとコンテナの違い？
イメージ=設計図（クッキーの型）、コンテナ=実行中のプロセス（実際のクッキー）

### Q2: L4とL7の違い？
L4=ポート番号で振り分け、L7=URLパスで振り分け

### Q3: PVとPVCの違い？
PV=実際のストレージ（駐車場）、PVC=ストレージの要求（駐車券）

### Q4: PVCで確保できなかったら？
PVCとPodがPending状態になる

### Q5: ArgoCDの競合は？
Flux、Jenkins X、Spinnaker。総称は「GitOpsツール」

### Q6: Vercel/Cloudflareはこれを抽象化してる？
**Yes！** git pushだけで裏で全部やってくれる

### Q8: AWSで対応するサービスは？
| k8s | AWS |
|-----|-----|
| Ingress | ALB |
| Service | NLB |
| Deployment | ECS |
| k8s全体 | EKS |

### Q9: Cloudflare Tunnelの価格は？
**無料！** 自宅サーバーやminikubeを公開できる

### Q10: クラウドk8sの価格は？
- AWS EKS: 月$100〜
- GCP GKE: 月$50〜
- DigitalOcean: 月$12〜

---

## 参考リンク

- [Kubernetes公式](https://kubernetes.io/ja/docs/home/)
- [minikube公式](https://minikube.sigs.k8s.io/docs/)
- [ArgoCD公式](https://argo-cd.readthedocs.io/)
- [Prometheus公式](https://prometheus.io/docs/)
- [Grafana公式](https://grafana.com/docs/)
- [Helm公式](https://helm.sh/docs/)