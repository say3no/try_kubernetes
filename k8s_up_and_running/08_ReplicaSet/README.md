# 08 ReplicaSet

これまでの章で独立したコンテナをPodとして動かす方法を見てきたが、これらのPodは実際おには一回限りしか使えないものだった（どういう意味だ？）。しかし、以下の理由によって多くの場合は一度にコンテナのレプリカを複数動かしたいはず。

- 冗長性
- スケール
  - インスタンスを複数動かせば、多くのreqに対応できる。
- [シャーディング](http://www.utakata.work/entry/2017/12/04/104639)
  - 異なるコンテナのレプリカを動かすと、異なる種類の処理を同時に受け付けられる。

もちろん、だいたい同じのPodマニフェストを作ってPodのコピーを手動で作るのも可能だが、退屈だし、間違いを起こしやすくなる。Podのレプリカを作るなら、そのレプリカ群は一つのまとまりとして考えて管理するのが普通だろう。

**ReplicaSet**は、指定したテンプレートにしたがった正しい数のPodが常に動いているようにする、クラスタ全体のPodマネージャだ。

- Podのレプリカを管理する仕組みは、**調整ループ(reconciliation loop)**の一例である、**k8sのデザインと実装のために重要な仕組み**である。

## 調整ループ

望ましい状態(desired state)と現在の状態(observed state または current state)の2つの概念によって実現される。`ゴール駆動形`で、`自己回復システム`であり、しかも **殆どの場合数行のコードで実現で表現できる**

## Label

kuard-rs.yaml

```yml
apiVersion: extensions/v1beta1
kind: ReplicaSet
metadata:
  name: kuard-rs
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: kuard
        version: "2"
    spec:
      containers:
        - name: kuard
          image: "gcr.io/kuar-demo/kuard-amd64:2"
```

`ReplicaSet`はPodのLabelをつかってクラスタの状態を監視している。`spec.template.metadata.labels`だ。

`kubectl apply -f kuard-rs.yaml`してみる。

```sh
$ kubectl apply -f kuard-rs.yaml

$ kubectl get pods
NAME          READY     STATUS    RESTARTS   AGE
kuard-j22w6   1/1       Running   0          1m
```

describeしてみると、確かに`rs`のtemplate通りのpodが一つ作成されている。作成されたpodはどの`rs`によって作成されたのかは、`Annotation`を見るとわかる。`kubectl get pods kuard-j22w6 -o yaml | peco`で確認してみよう。

## Scale

`kubectl edit rs/kurad-rs`でもできるけど、`kubectl scale`が手っ取り早い。

```sh
$ kubectl get rs -o yaml | grep replicas:
    replicas: 4
    replicas: 4

$ kubectl scale replicasets kuard-rs --replicas=2
replicaset.extensions "kuard-rs" scaled

$ kubectl get rs -o yaml | grep replicas:
    replicas: 2
    replicas: 2
```

ただし、もちろんこういった命令はあくまで一時的に使うべきで、すぐに手元のk8s用リポジトリに設定変更をコミット・pushすべき。
その上で、`kubectl apply hogehoge`すべきだろう。

## ReplicaSetのオートスケール

具体的な数値ではなく、「十分な数」のレプリカをスケールさせたいときどうすればよいだろう？ それはCPUだったりメモリだったりでスケールするサイズは変わってくるわけだが。こういった何らかのメトリクスに対応してスケールする仕組みは、**水平Podオートスケーリング(horizontal pod autoscaling, HPA)**で実現できる。ただしこいつを使うためには、`heapster`というPodがクラスタの中に存在している必要がある。

```sh
$kubectl get pods --namespace=kube-system
NAME                                                     READY     STATUS    RESTARTS   AGE
event-exporter-v0.2.3-54f94754f4-k4nqc                   2/2       Running   0          5d
fluentd-gcp-scaler-6d7bbc67c5-46cf6                      1/1       Running   0          5d
fluentd-gcp-v3.1.0-9m5qk                                 2/2       Running   0          5d
fluentd-gcp-v3.1.0-kz4rg                                 2/2       Running   0          5d
fluentd-gcp-v3.1.0-rl978                                 2/2       Running   0          5d
heapster-v1.5.3-666b78cfb7-tfrnw                         3/3       Running   0          5d # おるやんけ
kube-dns-788979dc8f-fxt9h                                4/4       Running   0          5d
kube-dns-788979dc8f-r4x46                                4/4       Running   0          5d
kube-dns-autoscaler-79b4b844b9-tr7dx                     1/1       Running   0          5d
kube-proxy-gke-kuar-cluster-default-pool-6de07dc9-1nwq   1/1       Running   0          5d
kube-proxy-gke-kuar-cluster-default-pool-6de07dc9-25k0   1/1       Running   0          5d
kube-proxy-gke-kuar-cluster-default-pool-6de07dc9-t8t2   1/1       Running   0          5d
l7-default-backend-5d5b9874d5-2s2f5                      1/1       Running   0          5d
metrics-server-v0.2.1-7486f5bd67-xfvv2                   2/2       Running   0          5d
```

ちなみに、Podの数を増やしていくのを **水平スケール**といい、リソースを増やしていくのを **[垂直スケール](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)**という。

### CPU使用率を元にしたオートスケール

`kubectl autoscale rs kuard --min=2 --max=5 --cpu-percent=80`で文字通りのhpaの設定ができる。`kubectl get hpa`で確認できる。

## ReplicaSetの削除

`kubectl delete rs kuard-rs --cascade=fale`で、RSのみ削除。podは残る。

(memo)Podのマニフェストいらなくね？RSだけでよくね？