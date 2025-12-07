# Kubernetes (k8s) 学習ガイド

このドキュメントは、Kubernetesの基礎を段階的に学習した記録です。

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

## ここまでのまとめ

### 完了した内容

✅ Pod のデプロイ  
✅ Deployment のデプロイ  
✅ Service のデプロイ（L4ロードバランス）  
✅ オートヒーリング  
✅ ConfigMap, Secret のデプロイ

### k8sの基本構造

```
外部（ブラウザ）
  ↓
Service（入り口・負荷分散）
  ↓
Deployment（Pod管理者）
  ├→ Pod1（コンテナ実行）
  ├→ Pod2（コンテナ実行）
  └→ Pod3（コンテナ実行）
       ↑
ConfigMap / Secret（設定・機密情報）
```

### 重要な概念

1. **ラベルとセレクター**
   - ServiceがどのPodに転送するか判断する仕組み
   - `labels` と `selector` が一致するものを選択

2. **宣言的な管理**
   - YAMLで「こうあるべき」を宣言
   - k8sが自動で現状をその状態に保つ

3. **設定とコードの分離**
   - イメージには変更を加えず、ConfigMap/Secretで設定を変更できる

---

## 次のステップ

- [ ] Ingress のデプロイ（L7ロードバランス）
- [ ] PV, PVC のデプロイ（永続化ストレージ）
- [ ] CI/CD（GitHub Actions + ArgoCD）
- [ ] モニタリング（Prometheus + Grafana）

---

## 参考リンク

- [Kubernetes公式ドキュメント](https://kubernetes.io/ja/docs/home/)
- [minikube公式ドキュメント](https://minikube.sigs.k8s.io/docs/)
- [kubectl チートシート](https://kubernetes.io/ja/docs/reference/kubectl/cheatsheet/)