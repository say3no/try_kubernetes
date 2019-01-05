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

上の`docker run`をPodマニフェストで記述するとこうなるそうだ。(`kuard-pod.yaml`)

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

この`pod manufest`を元にpodを作成するにはこう。

```sh
$ kubectl apply -f kuard-pod.yaml
pod "kuard" created
$ kubectl get pods
NAME      READY     STATUS    RESTARTS   AGE
kuard     1/1       Running   0          7s
$ kubectl get pods -o wide # 情報量がちょい増える
NAME      READY     STATUS    RESTARTS   AGE       IP          NODE
kuard     1/1       Running   0          4m        10.32.1.8   gke-kuar-cluster-default-pool-6de07dc9-25k0
$ kubectl describe pods kuard # 詳細を求める時はdescribe
// いっぱいでる
$ kubectl delete pods/kuard
```

- (memo) Podを吹き飛ばすと、Podの上に乗っているものは跡形も残らない。永続化させたいものがある場合は、 `PersistentVolume`を使用する。

### ログからの詳細情報の取得やら

```sh
$ kubectl logs kuard
$ kubectl exec kuard date # docker ライクで助かる
$ kubectl exec -it kuard /bin/sh # docker ライクで助かるぞい
$ kubectl cp hogehoge.txt <Pod>:/tmp/hogehoge.txt # docker ライクで助かるぞい
```

コンテナはイミュータブルであるべきだからあとから `cp`するというのはアンチパターンだけど、障害復旧の手段としてやっぱりあると便利。

### helth check

プロセスヘルスチェックがk8sにはある。プロセスが **動いているか**を監視して、動いていない場合はk8sがプロセスを再起動してくれる。(動いているってなんだ？)

が、実際にユーザーが求めているのは、生きているか死んでいるかではなく健康かどうかだ。k8sは `livenesss`にたいするヘルスチェックが可能で、 `liveness probe`によって正常に応答するかどうかをチェックする。`liveness probe`はここのアプリ固有の設定なので、都度人間がPodマニフェストに定義を書く。

### livness probe

```yml
      livenessProbe:
        httpGet:
          path: /healthy
          port: 8080
        initialDelaySeconds: 5
        timeoutSeconds: 1
        periodSeconds: 10
        failureThreshold: 3
```

直感的にわかる。いいね。こいつは1秒でタイムアウトする。こいつは起動されてから5秒まつ。こいつは10秒おきに叩く。こいつは3回まで失敗を許す。失敗というのは、200<= ステコ < 400 **以外**のときだ。この時、コンテナは障害を起こしていると判断され、再起動される。

`kubectl port-forward kuard 8080:8080`で遊んでみよう。

### readiness probe

k8sすごい。 k8sは`liveness`と`readiness`を区別している！ `liveness`は生死テスト、`readiness`は機能テストだ。ユーザからのリクエストを処理できることを確認する。詳細は7章で説明するとのこと。

### ほかのヘルスチェック

- tcpSocketヘルスチェック
  - HTTPを使わないアプリの監視に便利
- exec監視も可能。オリジナルの監視系スクリプトの返り値が0なら成功、0以外なら失敗…など。

### リソース管理

- こういった仕組みが、イメージのパッケージングや信頼性の高いデプロイを劇的に改善してくれるという点がよく挙げられます。
- インフラへの投資の効率性をあげるには、それぞれのマシンを最大限に使用することが求められます。

リソース要求(resource req)とリソース制限(resourece lim)ができる。

### Volumeを使ったデータの永続化

`spec.bolumes`と`volumeMou$$nts`.

実際にやる時は外部のネットワーク・ストレージサービスをマウントしとく。

PersistentVolumeは奥が深いので13章でやるそうです。