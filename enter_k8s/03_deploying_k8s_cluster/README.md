# 3章 Kubenetes クラスタのデプロイ。

k8sクラスタとの通信は、k8sの鯖が公開されているAPIと通信してやり取りをするのだが、それらをラッパしたCLI`kubectl`でおしゃべりをする。

```sh
$ kubectl version
Client Version: version.Info{Major:"1", Minor:"10", GitVersion:"v1.10.3", GitCommit:"2bba0127d85d5a46ab4b778548be28623b32d0b0", GitTreeState:"clean", BuildDate:"2018-05-21T09:17:39Z", GoVersion:"go1.9.3", Compiler:"gc", Platform:"darwin/amd64"}
Server Version: version.Info{Major:"1", Minor:"10+", GitVersion:"v1.10.9-gke.5", GitCommit:"d776b4deeb3655fa4b8f4e8e7e4651d00c5f4a98", GitTreeState:"clean", BuildDate:"2018-11-08T20:33:00Z", GoVersion:"go1.9.3b4", Compiler:"gc", Platform:"linux/amd64"}
```

これは安心感がある。困った時は `kubectl help`で助けてくれる。

```sh
$ kubectl get componentstatuses
NAME                 STATUS    MESSAGE              ERROR
etcd-0               Healthy   {"health": "true"}
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-1               Healthy   {"health": "true"}
```

多少なりともopenstackの開発経験があるおかげで、節々の理解を助けてくれている気がする。

k8sクラスタ、k8sによる系とはなにか。雑な理解として「コンテナが乗っている」は正解なのだが、このノードを増やしていって分散させることも可能…なのだろうか。可能なのだと思う。しかし分散と聞くとビザンチン将軍が黙っていないのでは…？　とにかく読み勧めてみよう。

いま、コンポーネントとして, `etcd-0`,`controller-manager`,`scheduler`,`etcd-1`がある。TODO: コンポーネントとはどういったものなのだろう？

さあ、次は。 `kubectl get nodes`　だ。このk8sクラスタは、3ノードからなっていることがわかる。

```sh
$ kubectl get nodes
NAME                                          STATUS    ROLES     AGE       VERSION
gke-kuar-cluster-default-pool-6de07dc9-1nwq   Ready     <none>    20m       v1.10.9-gke.5
gke-kuar-cluster-default-pool-6de07dc9-25k0   Ready     <none>    20m       v1.10.9-gke.5
gke-kuar-cluster-default-pool-6de07dc9-t8t2   Ready     <none>    20m       v1.10.9-gke.5
```

```sh
$ kubectl describe nodes gke-kuar-cluster-default-pool-6de07dc9-1nwq
Name:               gke-kuar-cluster-default-pool-6de07dc9-1nwq
Roles:              <none>
Labels:             beta.kubernetes.io/arch=amd64
```