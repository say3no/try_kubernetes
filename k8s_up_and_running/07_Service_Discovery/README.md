# 07 Service Discovery

- **Podが間違いなく起動して動き続けるようにする**。そのためにノードへのPodの配置を制御し、必要に応じて別のノードへと割り当ても行う。負荷に応じて、スケールする機能もある。
- k8sはたくさんのサービスを同時に実行可能であるが、 **動いているサービスを見つけるのは難しくなる**
- アニカを見つけるという問題とその解決策を一般に**Service Discovery**と呼ぶ。

k8sにおけるサービスディスカバリは、`Service`オブジェクトを作るところから始まる。`Service`オブジェクトとは、名前のついたLabelセレクタを作る仕組みのこと。

```=sh
kubectl run alpaca-prod \
--image=gcr.io/kuar-demo/kuard-amd64:1 \
--replicas=3 \
--port=8080 \
--labels="ver=1,app=alpaca,env=prod"

kubectl expose deployment alpaca-prod

kubectl run bandicoot-prod \
--image=gcr.io/kuar-demo/kuard-amd64:2 \
--replicas=2 \
--port=8080 \
--labels="ver=2,app=bandicoot,env=prod"

kubectl expose deployment bandicoot-prod

kubectl get services -o wide
```

ちなみに未だにdeploymentっていうのが何なのか分かっていない。いま`kubecctl get all`するとこんなかんじ

```sh
$ kubectl get all
NAME                                  READY     STATUS    RESTARTS   AGE
pod/alpaca-prod-7f94b54866-mvjtv      1/1       Running   0          4m
pod/alpaca-prod-7f94b54866-spctt      1/1       Running   0          4m
pod/alpaca-prod-7f94b54866-wlltl      1/1       Running   0          4m
pod/bandicoot-prod-85ddf4c7dd-k2zmj   1/1       Running   0          57s
pod/bandicoot-prod-85ddf4c7dd-z8l87   1/1       Running   0          57s

NAME                     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/alpaca-prod      ClusterIP   10.35.249.218   <none>        8080/TCP   3m
service/bandicoot-prod   ClusterIP   10.35.242.49    <none>        8080/TCP   56s
service/kubernetes       ClusterIP   10.35.240.1     <none>        443/TCP    5d

NAME                             DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/alpaca-prod      3         3         3            3           4m
deployment.apps/bandicoot-prod   2         2         2            2           57s

NAME                                        DESIRED   CURRENT   READY     AGE
replicaset.apps/alpaca-prod-7f94b54866      3         3         3         4m
replicaset.apps/bandicoot-prod-85ddf4c7dd   2         2         2         57s
try_kubernetes (feature/say3no/primer_k8s_07):
```

(memo) `pod`は最小単位ではあるけれど、実際には`service`の`cluseter-ip`が各サービスやらコンテナがお喋りするためのEPになるのか？

```sh
$ ALPACA_POD=$(kubectl get pods -l app=alpaca -o jsonpath='{.items[0].metadata.name}')
NAME                           READY     STATUS    RESTARTS   AGE
alpaca-prod-7f94b54866-mvjtv   1/1       Running   0          18m
alpaca-prod-7f94b54866-spctt   1/1       Running   0          18m
alpaca-prod-7f94b54866-wlltl   1/1       Running   0          18m
$ kubectl port-forward $ALPACA_POD 48858:8080
```

- `cluster-ip`はServiceオブジェクトを再作成しない限り変更されないので、DNSアドレスとして使用するのにも適している。

## Readiness probe

```sh
 kubectl edit deployment/alpaca-prod
```

(memo)これ、`:wq`だとエラーになるっぽい。`:w`,`:q`でいこう。

```vim
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          periodSeconds: 2
          initialDelaySeconds: 0
          failureThreshold: 3
          successThreshold: 1
```

Podが再作成されるので、再度Podのnameを拾ってきてport-forwardでつないでみる。すると、確かに`Readiness Probe`が動いている！

## EPのWatch

`kubectl get endpoints alpaca-prod --watch`ブラウザでEPの様子が見れる。ぶっちゃけよく分かっていないのだが、`Readiness probe`からfailを3つ以上ダスト、先程の規定どおりに機能不全とみなされてEPが閉塞されることがわかる…のかなあ。わからんけど。

## Type: NodePort

- Podが外部と通信するためには、Serviceを拡張するNodePortと呼ばれる機能を使うのがもっともポータブル。

`kuebctl edit service alpaca-prod`で、spec.typeを`ClusterIP`から`NodePort`に変更してみる。`kubectl get service/alpaca-prod`で、Typeが変わったことが確認できる。

## Type: LoadBalancer

同様にしてspec.typeを`LoadBalancer`に変更してみる。 **しばらくすると、パブリックアドレスを付与される！**

## Endpoints オブジェクト

`kubectl describe endpoints alpaca-prod`

```sh
kubectl get endpoints/alpaca-prod --watch
NAME          ENDPOINTS                                         AGE
alpaca-prod   10.32.0.13:8080,10.32.1.16:8080,10.32.2.14:8080   1h
```

そうか、`alpaca-prod`のreplica setを3に設定したから3つあるのか！ ただし、外部からこいつにアクセスしたいときは、こういったことを知覚しておく必要はないのだ！
外部からは、Seviceオブジェクトが持つIPだけを見ていればいい。そうだろ松！

ためしに`kubectl edit deployments/alpaca-prod`でreplicasetを5にしてみる。すると…

```sh
$ kubectl get endpoints/alpaca-prod
NAME          ENDPOINTS                                                     AGE
alpaca-prod   10.32.0.13:8080,10.32.1.16:8080,10.32.1.17:8080 + 2 more...   1h
```

ほうら！ だんだん分かってきたぞ。