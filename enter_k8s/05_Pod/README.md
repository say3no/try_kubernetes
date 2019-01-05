# 05_Pod

Podとはくじらなどの群れのこと。k8sクラスタ上においては、一つのコンテナの群であり、デプロイの最小単位でもある。1podの中に含まれるコンテナは、全て同じマシン上に配置される。
**Pod内の各コンテナはそれぞれのcgroups上で動作するが、namespaceの多くを共有する**。難しいことはない。個人でdocker-composeを書いていた頃と同じだ。（たぶん）
>同じPod内のアプリケーションは、同じIPとPortを使用し、同じホスト名を持ち、 System VIPCやPOSIXメッセージキューを経由したネイティブなプロセス官通信チャネルを使って通信できます。

ん、どういうことだ？ PodはPodとしかしゃべらないということか？Podの中のコンテナが他のPodのコンテナと喋りたい時はかならずPodを経由するのだから当然か？

Podこそが1枚の巨大なdocker-composeとk8sを分ける考え方なのかもしれない。

Podの分け方は、密接にスケールする必要がある場合は同居させる、という感じなのかな。

Podの設定は、Podマニフェストに記述する。これはk8sオブジェクトをテキストであり、宣言的に表現したもの。

## Podの作成/表示/削除

```sh
$ kubectl run kuard --image=gcr.io/kuar-demo/kuard-amd64:1
deployment.apps "kuard" created
$ kubectl get pods
NAME                    READY     STATUS    RESTARTS   AGE
kuard-b75468d67-vcxpm   1/1       Running   0          1m
$ kubectl delete deployments/kuard
deployment.extensions "kuard" deleted
$ kubectl get pods
No resources found.
```

## Podマニフェストの作成

`yml` or `json`だそうだ。

```sh
docker run -d --name kuard --publish 8080:8080 gcr.io/kuar-demo/kuard-amd64:1
```

上の`docker run`をPodマニフェストで記述するとこうなるそうだ。

```yml
apiVersion: v1
kind: Pod
metadata:
    name: kuard
spec:
    containers:
        - image: gcr.io/kuar-demo/kuard-amd64:1
          name: kuard
          ports:
              - containerPort: 8080
              name: http
              protocol: TCP
```
