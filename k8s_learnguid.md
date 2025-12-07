# Kubernetes (k8s) 学習ガイド

このドキュメントは、Kubernetesの基礎を段階的に学習した記録です。

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

# minikube起動
minikube start

# 確認
kubectl cluster-info
kubectl get nodes
```

### Dockerのディスク容量不足への対処

```bash
# 不要なDockerリソースを削除
docker system prune

# または minikube内のDockerをクリーンアップ
minikube ssh -- docker system prune -a
```

---

## Step 1: Pod のデプロイ

### Podとは
- Kubernetesの最小単位
- 1つ以上のコンテナをまとめたもの
- 一時的な存在（削除されたら復活しない）

### nginx-pod.yaml

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

### デプロイと確認

```bash
kubectl apply -f nginx-pod.yaml
kubectl get pods
kubectl describe pod my-first-pod
kubectl logs my-first-pod

# Podの中に入る
kubectl exec -it my-first-pod -- bash
curl localhost:80
exit
```

### イメージとコンテナの違い

- **イメージ** = 設計図、テンプレート（読み取り専用）
  - 例: クッキーの型
- **コンテナ** = イメージから作られた実行中のプロセス
  - 例: その型から作られた実際のクッキー

---

## Step 2: Service のデプロイ（L4ロードバランス）

### Serviceとは
- Podへの入り口
- 外部とPodを繋ぐ橋渡し
- 複数のPodに負荷分散（ロードバランス）する

### nginx-service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app: nginx  # このラベルを持つPodに転送
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
```

### デプロイと確認

```bash
kubectl apply -f nginx-service.yaml
kubectl get services
kubectl describe service nginx-service

# ブラウザでアクセス
minikube service nginx-service --url
```

### L4ロードバランスとは

- レイヤー4（トランスポート層）で動作
- IPアドレスとポート番号だけを見て転送
- HTTPの中身（パス、ヘッダーなど）は見ない
- シンプルで高速

---

## Step 3: Deployment のデプロイ

### Podを直接使う問題点

```bash
# Podを削除すると...
kubectl delete pod my-first-pod
kubectl get pods
# → 復活しない！
```

### Deploymentとは

- Podの管理者
- 指定した数のPodを維持する
- オートヒーリング（自動復旧）
- スケーリング（台数の増減）
- ローリングアップデート

### nginx-deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3  # Podを3つ維持
  selector:
    matchLabels:
      app: nginx
  template:  # ここから下がPodの設定
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

### デプロイと確認

```bash
kubectl apply -f nginx-deployment.yaml
kubectl get deployments
kubectl get pods

# オートヒーリングを試す
kubectl delete pod nginx-deployment-xxxxx-xxxxx
kubectl get pods  # すぐに新しいPodが作られる！
```

### 仕組み

```
Service (nginx-service)
  ↓ app: nginx のラベルを持つPod全部に転送
  ├→ Pod1 (nginx-deployment-xxxxx-aaaaa)
  ├→ Pod2 (nginx-deployment-xxxxx-bbbbb)
  └→ Pod3 (nginx-deployment-xxxxx-ccccc)
```

---

## Step 4: ConfigMap のデプロイ

### ConfigMapとは

- アプリケーションの設定を保存
- イメージとは分離して管理
- 環境ごとに設定を変えられる

### app-configmap.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: "production"
  LOG_LEVEL: "info"
  DATABASE_HOST: "db.example.com"
```

### デプロイと確認

```bash
kubectl apply -f app-configmap.yaml
kubectl get configmap
kubectl describe configmap app-config
```

---

## Step 5: Secret のデプロイ

### Secretとは

- パスワードやAPIキーなどの機密情報を保存
- base64エンコードして保存（暗号化ではない）
- `kubectl describe` では値が見えない

### base64エンコードの方法

```bash
echo -n "password123" | base64
# 結果: cGFzc3dvcmQxMjM=

echo -n "mysecretkey" | base64
# 結果: bXlzZWNyZXRrZXk=
```

### app-secret.yaml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  DB_PASSWORD: cGFzc3dvcmQxMjM=
  API_KEY: bXlzZWNyZXRrZXk=
```

### デプロイと確認

```bash
kubectl apply -f app-secret.yaml
kubectl get secrets
kubectl describe secret app-secret  # 値は見えない
```

---

## Step 6: ConfigMapとSecretを使う

### app-deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: nginx:latest
        env:
        # ConfigMapから環境変数を読み込む
        - name: APP_ENV
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: APP_ENV
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: LOG_LEVEL
        # Secretから環境変数を読み込む
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: DB_PASSWORD
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: API_KEY
```

### 確認

```bash
kubectl apply -f app-deployment.yaml
kubectl get pods

# Pod内で環境変数を確認
kubectl exec -it app-deployment-xxxxx -- bash
echo $APP_ENV       # production
echo $LOG_LEVEL     # info
echo $DB_PASSWORD   # password123（自動でデコードされる）
echo $API_KEY       # mysecretkey
exit
```

---

## Step 7: Ingress のデプロイ（L7ロードバランス）

### Ingressとは

L4（Service）との違い：

| 項目 | L4 (Service NodePort) | L7 (Ingress) |
|------|----------------------|--------------|
| 動作レイヤー | トランスポート層 | アプリケーション層 |
| 見るもの | IPアドレス、ポート番号 | HTTPパス、ホスト名、ヘッダー |
| ルーティング | ポート番号ベース | パス/ホストベース |
| 例 | `:30080` → nginx Pod | `/api` → API Pod, `/web` → Web Pod |

```
【Serviceだけの構成】
ブラウザ → :30080 → Service → Pod (nginx)
ブラウザ → :30081 → Service → Pod (api)
↑ ポートがどんどん増える...

【Ingressを使った構成】
ブラウザ → Ingress (:80)
              ├─ /v1  → Service → Pod (app-v1)
              └─ /v2  → Service → Pod (app-v2)
↑ 1つの入り口でパスごとに振り分け！
```

### 7-1. Ingress Controllerの有効化

```bash
minikube addons enable ingress
kubectl get pods -n ingress-nginx
# ingress-nginx-controller-xxxxx が Running になるまで待つ
```

### 7-2. テスト用アプリを2つ作成

#### app-v1.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app-v1
  template:
    metadata:
      labels:
        app: app-v1
    spec:
      containers:
      - name: app
        image: hashicorp/http-echo
        args:
          - "-text=Hello from App V1!"
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: app-v1-service
spec:
  selector:
    app: app-v1
  ports:
  - port: 80
    targetPort: 5678
```

#### app-v2.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app-v2
  template:
    metadata:
      labels:
        app: app-v2
    spec:
      containers:
      - name: app
        image: hashicorp/http-echo
        args:
          - "-text=Hello from App V2!"
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: app-v2-service
spec:
  selector:
    app: app-v2
  ports:
  - port: 80
    targetPort: 5678
```

```bash
kubectl apply -f app-v1.yaml
kubectl apply -f app-v2.yaml
```

### 7-3. Ingressを作成

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
kubectl apply -f my-ingress.yaml
kubectl get ingress
```

### 7-4. 動作確認

```bash
# 別ターミナルで実行（開いたままにする）
minikube tunnel

# 元のターミナルで確認
curl http://127.0.0.1/v1  # → Hello from App V1!
curl http://127.0.0.1/v2  # → Hello from App V2!
```

---

## Step 8: PV / PVC のデプロイ（永続化ストレージ）

### なぜ必要？

Podは一時的な存在。消えたらデータも消える。

```bash
# 実験：PVCなしのPodでファイルを作成
kubectl apply -f test-pod.yaml
kubectl exec -it test-pod -- sh -c "echo 'hello' > /tmp/test.txt"
kubectl exec -it test-pod -- cat /tmp/test.txt  # → hello

# Podを消して再作成
kubectl delete pod test-pod
kubectl apply -f test-pod.yaml
kubectl exec -it test-pod -- cat /tmp/test.txt  # → No such file or directory
# → データが消えた！
```

### PV と PVC の関係

```
PV  = 駐車場（管理者が用意する実際のスペース）
PVC = 駐車券（開発者が「車1台分ください」と申請）

開発者はどこの駐車場かは気にしない。
「1台分のスペースが欲しい」とだけ言えばOK。
```

### 8-1. テスト用Pod（PVCなし）

#### test-pod.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
  - name: nginx
    image: nginx:latest
```

### 8-2. PersistentVolume を作成

#### my-pv.yaml

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /data/my-pv
```

### 8-3. PersistentVolumeClaim を作成

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

```bash
kubectl apply -f my-pv.yaml
kubectl apply -f my-pvc.yaml
kubectl get pv
kubectl get pvc  # STATUS が Bound になればOK
```

### 8-4. PVCをマウントしたPod

#### test-pod-with-pvc.yaml

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

### 8-5. 永続化の確認

```bash
kubectl apply -f test-pod-with-pvc.yaml

# ファイルを作成
kubectl exec -it test-pod-pvc -- sh -c "echo 'hello persistent!' > /data/test.txt"
kubectl exec -it test-pod-pvc -- cat /data/test.txt  # → hello persistent!

# Podを消して再作成
kubectl delete pod test-pod-pvc
kubectl apply -f test-pod-with-pvc.yaml

# データが残っている！
kubectl exec -it test-pod-pvc -- cat /data/test.txt  # → hello persistent!
```

---

## 現在の構成図

```
外部（ブラウザ）
  ↓
Ingress（/v1, /v2 でルーティング - L7）
  ↓
Service（負荷分散 - L4）
  ↓
Deployment（Pod管理・オートヒーリング）
  ↓
Pod → コンテナ
  ↑       ↑
  |       ConfigMap / Secret（設定・機密情報）
  |
PVC → PV（永続ストレージ）
```

---

## 作成したYAMLファイル一覧

### Phase 1: k8s基礎
| ファイル名 | 内容 |
|-----------|------|
| nginx-pod.yaml | 単体のPod |
| nginx-service.yaml | NodePort Service |
| nginx-deployment.yaml | Deployment（replicas: 3） |
| app-configmap.yaml | ConfigMap |
| app-secret.yaml | Secret |
| app-deployment.yaml | ConfigMap/Secretを使うDeployment |

### Phase 2: k8s応用
| ファイル名 | 内容 |
|-----------|------|
| app-v1.yaml | テスト用アプリV1（Deployment + Service） |
| app-v2.yaml | テスト用アプリV2（Deployment + Service） |
| my-ingress.yaml | Ingress（パスベースルーティング） |
| test-pod.yaml | テスト用Pod（PVCなし） |
| my-pv.yaml | PersistentVolume |
| my-pvc.yaml | PersistentVolumeClaim |
| test-pod-with-pvc.yaml | PVCをマウントしたPod |

---

## 学習ロードマップ

### Phase 1: k8s基礎 ✅ 完了
- [x] Pod のデプロイ
- [x] Deployment のデプロイ
- [x] Service のデプロイ（L4ロードバランス）
- [x] オートヒーリング
- [x] ConfigMap, Secret のデプロイ

### Phase 2: k8s応用 ✅ 完了
- [x] Ingress のデプロイ（L7ロードバランス）
- [x] PV, PVC のデプロイ（永続化ストレージ）

### Phase 3: CI/CD ⬜ 次はここ
- [ ] GitHub Actions でイメージビルド → DockerHub に push
- [ ] ArgoCD でGitOps（自動デプロイ）

### Phase 4: モニタリング
- [ ] Prometheus + Grafana でメトリクス可視化

---

## Q&A（学習中に出た質問と回答）

### Q1: イメージとコンテナの違いは？

**A:** 
- **イメージ** = 設計図、テンプレート（読み取り専用）。クッキーの「型」のようなもの。
- **コンテナ** = イメージから作られた実行中のプロセス。型から作られた「実際のクッキー」。

同じイメージから複数のコンテナを作れる。イメージは変更されないが、コンテナ内部は変更可能（ただし一時的）。

---

### Q2: L4ロードバランスとL7ロードバランスの違いは？

**A:**

| 項目 | L4（Service） | L7（Ingress） |
|------|---------------|---------------|
| 動作レイヤー | トランスポート層 | アプリケーション層 |
| 見るもの | IPアドレス、ポート番号 | HTTPパス、ホスト名、ヘッダー |
| ルーティング | ポート番号ベース | パス/ホストベース |
| 例 | `:30080` → nginx Pod | `/api` → API Pod, `/web` → Web Pod |

L7の方がきめ細かいルーティングが可能。マイクロサービスアーキテクチャで重要。

---

### Q3: PVとPVCの違いは？

**A:**
- **PV（PersistentVolume）** = 管理者が用意する「実際のストレージ」。ディスクそのもの。
- **PVC（PersistentVolumeClaim）** = 開発者が出す「ストレージの要求書」。「10GBのストレージが欲しい」という申請。

分離する理由:
- 開発者はインフラの詳細を知らなくていい
- 管理者はストレージを一元管理できる
- 環境（dev/prod）ごとに異なるストレージを割り当てられる

---

### Q4: なぜPodを直接使わずDeploymentを使う？

**A:**
Podは一時的な存在で、削除されたら復活しない。Deploymentを使うと：
- 指定した数のPodを常に維持（オートヒーリング）
- スケーリングが簡単（`kubectl scale`）
- ローリングアップデートでダウンタイムなし

本番環境では必ずDeploymentを使う。

---

### Q5: ConfigMapとSecretの違いは？

**A:**
- **ConfigMap** = 一般的な設定（環境名、ログレベルなど）
- **Secret** = 機密情報（パスワード、APIキーなど）

Secretはbase64エンコードされ、`kubectl describe`で値が表示されない。ただし、base64は暗号化ではないので、本番では追加のセキュリティ対策（Vault等）を検討。

---

### Q6: labelとselectorはなぜ必要？

**A:**
k8sは多数のリソースを管理するため、「どれとどれが関連しているか」を判断する仕組みが必要。

```yaml
# Podに付けるラベル
labels:
  app: nginx

# Serviceが選ぶ条件
selector:
  app: nginx  # このラベルを持つPodに転送
```

これにより、Serviceは動的にPodを発見できる（Podが増減しても自動で対応）。

---

### Q7: minikubeとDocker Desktop内蔵のk8sどっちがいい？

**A:**
- **Docker Desktop内蔵k8s**: 手軽に始められる。間接的に触るだけなら十分。
- **minikube**: より本番に近い環境。学習目的ならこちらがおすすめ。

今回はminikubeを使用。kubectlも別途インストールが必要だが、本番環境でも同じコマンドが使えるメリットがある。

---

### Q8: hashicorp/http-echo って何？

**A:**
HashiCorp（Terraformの会社）が作った超シンプルなテスト用Webサーバー。起動すると指定したテキストを返すだけ。

```
リクエスト → http-echo → "Hello from App V1!"
```

今回使う理由：
- nginxだと両方同じ画面で区別しにくい
- 自分でアプリ作らなくていい
- どのPodに繋がったか一目でわかる

---

### Q9: `kubectl apply -f` の `-f` オプションは？

**A:**
`-f` は **file** の略。「このファイルを適用して」という意味。

```bash
kubectl apply -f app-v1.yaml     # ファイル指定
kubectl apply -f ./manifests/    # ディレクトリ内の全YAML適用
kubectl apply -f https://...     # URLからも可能
```

`-f` なしだと `error: must specify one of -f and -k` エラーになる。

---

### Q10: AWSでk8sに該当するサービスは？あえてk8sを使うメリットは？

**A:**

| k8sの機能 | AWSのマネージドサービス |
|-----------|------------------------|
| Ingress (L7振り分け) | ALB (Application Load Balancer) |
| Service (L4振り分け) | NLB (Network Load Balancer) |
| Deployment (コンテナ管理) | ECS (Elastic Container Service) |
| Pod (コンテナ実行) | Fargate |
| k8s全体 | EKS (Elastic Kubernetes Service) |

**あえてk8sを使うメリット：**

1. **ベンダーロックイン回避** - k8sのYAMLはAWS/GCP/Azure/オンプレどこでも動く
2. **大規模での柔軟性** - ECSより複雑な構成を表現できる
3. **エコシステム** - ArgoCD, Prometheus, Istio などk8s前提のツールが豊富

**使い分けの目安：**
| 状況 | おすすめ |
|------|----------|
| 小〜中規模、AWS固定 | ECS + Fargate（楽） |
| マルチクラウド、大規模 | EKS (AWS上のk8s) |
| オンプレ必須 | k8s一択 |
| 学習目的 | k8s（どこでも通用する知識） |

---

### Q11: k8sを使ってる会社は多い？

**A:**
めちゃくちゃ多い。大手はほぼ使ってる。

国内: Mercari, LINE, PayPay, SmartNews, Cookpad, Wantedly, サイバーエージェント, Yahoo など
海外: Spotify, Airbnb, Pinterest, Twitter など

「コンテナ使ってる大規模サービス ≒ k8s」くらいの勢い。

---

### Q12: 疎結合とは？なぜIngressで「疎結合」と言われる？

**A:**
フロントエンドで例えると「npmパッケージを分けてる」のサーバー版。

```
// 密結合（1つの巨大なコード）
monolith-app/
  └── src/
      ├── users.js
      ├── orders.js      ← 全部同じプロセスで動く
      └── payments.js

// 疎結合（分離されたサービス）
user-service/      ← 独立してデプロイ可能
order-service/     ← 独立してスケール可能
payment-service/   ← 障害が起きても他に影響しない
```

Ingressを使うと：
```
https://myapp.com/api/users  → user-service
https://myapp.com/api/orders → order-service  
https://myapp.com/           → frontend-service
```

各サービスが独立してデプロイ・スケールできる。これが「疎結合」の実現。

---

## 参考リンク

- [Kubernetes公式ドキュメント](https://kubernetes.io/ja/docs/home/)
- [minikube公式ドキュメント](https://minikube.sigs.k8s.io/docs/)
- [kubectl チートシート](https://kubernetes.io/ja/docs/reference/kubectl/cheatsheet/)