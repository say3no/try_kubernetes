# 02_create_and_delete_contaienr

kuardは **K**8s **u**p **a** **r**unningの略だそうな。

gcpのgcrは、githubと連携しなくても（当然だけれど）pushができる

```sh
docker build -t gcr.io/say3no-try-gcp/try_k8s/kuard .
docker push gcr.io/say3no-try-gcp/try_k8s/kuad
```

以下はテキストで使用するコンテナ。

```sh
docker run -d --name kuard --publish 8080:8080 gcr.io/kuar-demo/kuard-amd64:1
```