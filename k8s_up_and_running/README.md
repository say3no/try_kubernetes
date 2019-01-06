# Kubenetes: Up and Running

kubernetesのトライアル的なサムシング。

minikubeとかいろいろあるんだけどまずは[入門 Kubernetes][1]をしっかりと読んで、その背景や哲学を学習してからでも遅くはないっていうかそういったコンテキストを得ていないままminikubeやって挫折しました。

## Kubernetesとは

コンテナオーケストレーションソフトウェア。とよく聞くがその実態ははっきりしていなかったのが正直なところ。「人間は考える葦である」とか「我思う故に我あり」が彼らの思想の端的な表現だとしても、その背景や思想を熟知していなければ「は？ｗ」ってなるよね。それとおんなじ。

以下は[本][1]の一章と少しを読んだ僕の感想だ。

- Kubenetersはマシンリソースを抽象的に管理することができる。
- 抽象的であるから、物理的なリソースの範囲内において、それはスケーラブルに管理することができる。
- だから、たぶん、その実態がクラウドかオンプレというのは考慮しない。(IPの変更に過ぎない）
- これらを支えているのは、イミュータブルなインフラであり、つまりコンテナだ。
- 各コンテナ群をPodという単位で扱い、それへのアクセスはLBを経由するAPIという形で実現している。それはServiceという。
- 当然、系を複数立てることで分散システムのマネジメントも可能
- kubernetesはMSAであることが前提になっている

## コンテナの歴史,コンテナランタイム,OCI

k8sの裏側は、コンテナランタイムインターフェース(CRI)を操作している。ではコンテナランタイムとは。
コンテナは大別して`Container Image`と`Contaienr Runtime`の２つからなり、これらを標準化する組織としてOCIがある。そのコアランタイムが昨年末に公開された[containerd](https://containerd.io)

- [Dockerだけではないコンテナのはなし][3]

ところでランタイム（ランタイムライブラリ）とは、文字通りプログラムの実行時(Run time)に**常に同時に動くことが前提となっているライブラリのこと**。RPGツクールRTPとかあったよね。k8sはコンテナオーケストレーションであって、dockerコンテナオーケストレーションではない。

> - [Kubernetesの新たなコンテナランタイム][4]
>   - これまでのDockerとrktは、いずれも**非公開のAPI**を通じて、Kubernetesのソースコードに**密接に統合**されていた。そのため、これら以外のランタイムの追加が困難な上に、安定性の保証されないソースコードの修正を行なう必要があった。Kubernetesにアルファ機能として導入された**Container Runtime Interface(CRI)は、これを変えようというもの**だ。CRIは、コンテナランタイムをKubernetesシステムにプラグインするための**共通インターフェースを公開**する。これによってユーザは、Dockerやrkt以外のインフラストラクチャの編成や拡張が可能になる。runvのようなコンテナベースのハイパーバイザもランタイムとして使用可能だ。

- [Linux コンテナとは][5]
  - OCIメンバであるRedhadによる解説。歴史とOCIについてよくまとまっているように見える。

## コンテナのリソース制御

[本][1]にて初めて知った点。

### メモリリソース

`--memory 200m`,`--memory-swap 1G`でrunするとメモリリソースを制御できる。しらなかった。

CRIから見てみると、マシンリソースが設定どおりに制限されていることが確認できる。

```sh
try_linux (master): docker stats
CONTAINER ID        NAME                CPU %               MEM USAGE / LIMIT   MEM %               NET I/O             BLOCK I/O           PIDS
c7755a75560e        m-max               0.00%               812KiB / 1.952GiB   0.04%               788B / 0B           0B / 0B             1
acc4e21bb72c        m200                0.00%               808KiB / 200MiB     0.39%               968B / 0B           0B / 0B             1
```

ちなみにコンテナの内側から観てみると変化がない。
簡単に検証したところ、どうやらdocker for MacのGUIからコンフィグした値(Advancedのやつ)が参照されているようだ。

- 2Gbのとき

```sh
try_linux (master): docker run -ti --rm --name m200 --memory 200m say3no/try_linux
root@5fca9c7756ec:/# free -tm
              total        used        free      shared  buff/cache   available
Mem:           1998         219         474           1        1304        1626
Swap:          1023           0        1023
Total:         3022         219        1498
root@5fca9c7756ec:/# exit
exit
try_linux (master): docker run -ti --rm --name m-max say3no/try_linux
root@fee59805fd3d:/# free -tm
              total        used        free      shared  buff/cache   available
Mem:           1998         222         470           1        1305        1622
Swap:          1023           0        1023
Total:         3022         222        1494
```

- 1.5Gbのとき

```sh
try_linux (master): docker run -ti --rm --name m200 --memory 200m say3no/try_linux
root@6445991fe146:/# free -tm
              total        used        free      shared  buff/cache   available
Mem:           1494         215          67           0        1211        1130
Swap:          1023           0        1023
Total:         2518         215        1091
```

### CPUリソース

`--cpu-shares` で**使用率**を制限できる。ADみたいに上限1024。50%なら512。helpには記載がないのでやや不親切？

```sh
$ docker -v
Docker version 18.09.0, build 4d60db4
$ dokcer help run | grep '\--cpu-shares'
  -c, --cpu-shares int                 CPU shares (relative weight)
```

[Webの公式doc](https://docs.docker.com/config/containers/resource_constraints/#configure-the-default-cfs-scheduler)はちゃんと`Set this flag to a value greater or less than the default of 1024 to increase or reduce the container’s weight,`ってかいてあるけど…。

## 参照

- [入門 Kubernetes | Kelsey Hightower, Brendan Burns, Joe Beda, 松浦 隼人 |本 | 通販 | Amazon][1]
- [Hello Minikube][2]
  - kubernetes公式のチュートリアル。
- [「入門Kubernetes」入門][6]
  - 入門 Kubernetesの図が少ない問題を補足するスライド

[1]: https://www.amazon.co.jp/入門-Kubernetes-Kelsey-Hightower/dp/4873118409/
[2]: https://kubernetes.io/docs/tutorials/hello-minikube/
[3]: https://www.slideshare.net/potix2_jp/docker-77560696
[4]: https://www.infoq.com/jp/news/2017/06/alternative-kubernetes-runtimes
[5]: https://www.redhat.com/ja/topics/containers/whats-a-linux-container 
[6]: https://speakerdeck.com/doublemarket/20180609-gcpug-hiroshima-number-4?slide=1
