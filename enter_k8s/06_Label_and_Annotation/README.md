# 06_Label and Annotation

**LabelはPodやReplicaSetといったk8sオブジェクトに付与できる。** K,Vからなり、オブジェクトをグループ化するための基盤となる機能。

**AnnotationはLabelと似たストレージの仕組み**。Label同様にK,Vからなるが、ツールやライブラリを便利に使用するために必要になる、オブジェクトを特定しない情報を入れられる。

## Labelの適用

突然だがDeploymentという表現がある。Podの集まりを作る方法だそうです。はい。`alpaca`と`bandicoot`というアプリを作りそれを別の環境へ配置する、とする。

- `alpaca-prod`

```sh
kubectl run alpaca-prod \
--image=gcr.io/kuar-demo/kuard-amd64:1 \
--replicas=2 \
--labels="ver=1,app=alpaca,env=prod"
```

- `alpaca-test`

```sh
kubectl run alpaca-test \
--image=gcr.io/kuar-demo/kuard-amd64:2 \
--replicas=1 \
--labels="ver=2,app=alpaca,env=test"
```

- `bandicoot-prod`

```sh
kubectl run bandicoot-prod \
--image=gcr.io/kuar-demo/kuard-amd64:2 \
--replicas=2 \
--labels="ver=2,app=bandicoot,env=prod"
```

- `bandicoot-staging`

```sh
kubectl run bandicoot-staging \
--image=gcr.io/kuar-demo/kuard-amd64:2 \
--replicas=1 \
--labels="ver=2,app=bandicoot,env=staging"
```

deploymentを確認する。確かにlabelがはられている。（そりゃそうだ）

```sh
$ kubectl get deployments --show-labels
NAME                DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE       LABELS
alpaca-prod         2         2         2            2           2m        app=alpaca,env=prod,ver=1
alpaca-test         1         1         1            1           1m        app=alpaca,env=test,ver=2
bandicoot-prod      2         2         2            2           1m        app=bandicoot,env=prod,ver=2
bandicoot-staging   1         1         1            1           1m        app=bandicoot,env=staging,ver=2
```

### Labelの変更

```sh
$ kubectl label deployments alpaca-test "canary=true"
deployment.extensions "alpaca-test" labeled

$ kubectl get deployments --show-labels | grep canary
alpaca-test         1         1         1            1           5m        app=alpaca,canary=true,env=test,ver=2
```

keyがなければ追加し、あれば`--overwrite`で更新。

```sh
$ kubectl get deployments -L canary -L ver -L app -L env
NAME                DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE       CANARY    VER       APP         ENV
alpaca-prod         2         2         2            2           11m                 1         alpaca      prod
alpaca-test         1         1         1            1           11m       false     2         alpaca      test
bandicoot-prod      2         2         2            2           10m                 2         bandicoot   prod
bandicoot-staging   1         1         1            1           10m                 2         bandicoot   staging
```

### Label Selector

Labelをつかってk8sオブジェクトのフィルタリングができる。便利。

```sh
$ kubectl get pods --show-labels
NAME                                 READY     STATUS    RESTARTS   AGE       LABELS
alpaca-prod-65587bf567-d8gq9         1/1       Running   0          13m       app=alpaca,env=prod,pod-template-hash=2114369123,ver=1
alpaca-prod-65587bf567-x5jlp         1/1       Running   0          13m       app=alpaca,env=prod,pod-template-hash=2114369123,ver=1
alpaca-test-6658d779cc-g57v9         1/1       Running   0          13m       app=alpaca,env=test,pod-template-hash=2214833577,ver=2
bandicoot-prod-7bddc557cc-hg5m8      1/1       Running   0          13m       app=bandicoot,env=prod,pod-template-hash=3688711377,ver=2
bandicoot-prod-7bddc557cc-hw475      1/1       Running   0          13m       app=bandicoot,env=prod,pod-template-hash=3688711377,ver=2
bandicoot-staging-7f4788b6df-pmvlm   1/1       Running   0          13m       app=bandicoot,env=staging,pod-template-hash=3903446289,ver=2
kuard                                1/1       Running   0          1h        <none>

$ kubectl get pods --selector="ver=2,app=bandicoot" # and 検索
NAME                                 READY     STATUS    RESTARTS   AGE
bandicoot-prod-7bddc557cc-hg5m8      1/1       Running   0          21m
bandicoot-prod-7bddc557cc-hw475      1/1       Running   0          21m
bandicoot-staging-7f4788b6df-pmvlm   1/1       Running   0          21m

$ kubectl get pods --selector="app in (alpaca,bandicoot)" # or 検索
NAME                                 READY     STATUS    RESTARTS   AGE
alpaca-prod-65587bf567-d8gq9         1/1       Running   0          22m
alpaca-prod-65587bf567-x5jlp         1/1       Running   0          22m
alpaca-test-6658d779cc-g57v9         1/1       Running   0          22m
bandicoot-prod-7bddc557cc-hg5m8      1/1       Running   0          22m
bandicoot-prod-7bddc557cc-hw475      1/1       Running   0          22m
bandicoot-staging-7f4788b6df-pmvlm   1/1       Running   0          22m

$ kubectl get deployments --selector="canary" # keyのみを指定すると、該当keyが存在するものだけを拾ってくる
NAME          DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
alpaca-test   1         1         1            1           26m
```

| operator                | desc            |
| :---------------------- | :-------------- |
| key=value               | みたまま        |
| key!=value              | みたまま        |
| key in ( val1, val2)    | or              |
| key notin ( val1, val2) | どちらでもない  |
| key                     | keyが存在する   |
| !key                    | keyが存在しない |

(memo) `pod-template-hash`というkeyがいつの間にか存在している。文字通りだがこいつの元となったtemplateを参照するためのもの。12章でやる。

### おかたずけ

 `kubectl delete deployments --all`

## Annotation

deploymentsでよく使うらしいのだが一旦とばす。
