# 3-2 ハンズオン：ConfigMap / Secret / Volume ～設定と実行を切り離す～

---

想定環境：KillerCoda（Kubernetes クラスタ起動済み）
以降の操作はすべて kubectl を使用します。

---

## 0. 前提確認

前のハンズオンで作成した Deployment が存在していることを確認します。
```bash
kubectl get deployment
kubectl get pod
```

Pod が稼働していれば OK です。

## 1. ConfigMap を作成する

― 設定を Pod の外に置く ―

#### ConfigMap を定義する
```bash
cat <<EOF > configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  MESSAGE: "Hello from ConfigMap"
EOF
```

#### ConfigMap を作成します。
```bash
kubectl apply -f configmap.yaml
```

#### 確認します。
```bash
kubectl get configmap
```

## 2. ConfigMap を Pod から参照する
#### Deployment を修正する。ConfigMap を環境変数として読み込むようにします。
```bash
cat <<EOF > deployment-config.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sample
  template:
    metadata:
      labels:
        app: sample
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        env:
        - name: MESSAGE
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: MESSAGE
        ports:
        - containerPort: 80
EOF
```

#### 適用します。
```bash
kubectl apply -f deployment-config.yaml
```

#### Pod を確認します。
```bash
kubectl get pod
```

#### 環境変数を確認する
```bash
kubectl exec -it $(kubectl get pod -l app=sample -o jsonpath='{.items[0].metadata.name}') -- env | grep MESSAGE
```

#### 🔍 観察ポイント
- 設定は Pod の中に見える
- しかし定義は Pod の外にある


## 3. Pod を削除する
― 設定は残るか？ ―

#### Pod を削除します。
```bash
kubectl delete pod -l app=sample
```

#### 再度 Pod を確認します。
```bash
kubectl get pod
```

#### 新しい Pod が起動したら、もう一度環境変数を確認します。
```bash
kubectl exec -it $(kubectl get pod -l app=sample -o jsonpath='{.items[0].metadata.name}') -- env | grep MESSAGE
```

#### 🔍 観察ポイント
- Pod は入れ替わった
- 設定はそのまま使われている

Pod は設定を「所有していない」


## 4. ConfigMap を変更する
― 設定はどこに効くか ―

#### ConfigMap を変更します。
```bash
kubectl edit configmap app-config
```

#### MESSAGE の内容を変更して保存してください。
```bash
MESSAGE: "Hello updated ConfigMap"
```

#### Pod を再起動します。
```bash
kubectl delete pod -l app=sample
```

#### 再度環境変数を確認します。
```bash
kubectl exec -it $(kubectl get pod -l app=sample -o jsonpath='{.items[0].metadata.name}') -- env | grep MESSAGE
```

#### 🔍 観察▼ポイント
- 設定変更は Pod 再作成時に反映される
- 設定と実行が分離されている


## 5. Volume を使う
― データの置き場所を考える ―

#### Volume を使う Deployment を作成する
```bash
cat <<EOF > deployment-volume.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: volume-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: volume-sample
  template:
    metadata:
      labels:
        app: volume-sample
    spec:
      containers:
      - name: app
        image: busybox
        command: ["/bin/sh", "-c"]
        args: ["echo hello > /data/message.txt && sleep 3600"]
        volumeMounts:
        - name: data-volume
          mountPath: /data
      volumes:
      - name: data-volume
        emptyDir: {}
EOF
```

#### 適用します。
```bash
kubectl apply -f deployment-volume.yaml
```

#### データを確認する
```bash
kubectl exec -it $(kubectl get pod -l app=volume-sample -o jsonpath='{.items[0].metadata.name}') -- cat /data/message.txt
```

## 6. Pod を削除する
― データは残るか？ ―
```bash
kubectl delete pod -l app=volume-sample
```

#### 新しい Pod が起動したら、再度確認します。
```bash
kubectl exec -it $(kubectl get pod -l app=volume-sample -o jsonpath='{.items[0].metadata.name}') -- ls /data
```

#### 🔍 観察ポイント
- データは消えている
- emptyDir は Pod と運命を共にする


## 7. Secret について（ここでは体験しない）
Secret は、概念は ConfigMap と同じ

ただし用途が異なるため、ここでは 操作は行いません。

重要なのは、「秘匿されているか」より「Pod の外にあるか」という点です。

まとめ：このハンズオンで確認したこと
| 要素 　　　| 役割                     | 
|-----------|-------------------------|
|ConfigMap　|設定を Pod の外に出す|
|Secret　　|秘密情報を Pod の外に出す|
|Volume	　　|状態の寿命を明確にする|


このハンズオンで確認したかったのは、Pod は実行単位であり、状態や設定の保管場所ではないという前提です。

この前提があるからこそ、Pod を捨てられる、自動復旧できる、スケールできるという設計が成立しています。
