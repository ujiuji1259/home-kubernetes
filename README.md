## 概要
おうちkubernetesのコアとなるリソースの構築用

## 前提
[クラスタ](https://kubernetes.io/ja/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)の構築まで終わっているものとする。

## 構築
### secret
[1password](https://developer.1password.com/docs/k8s/k8s-operator/)でsecret基盤を構築する。このあとのgiteaの構築とかで使う。

* [developer tool](https://my.1password.com/developer-tools/active)からアクセストークンを取得
* v_id, item_idはitemのプライベートリンクから取得できる

次のようなitemを作っておいて

```
apiVersion: onepassword.com/v1
kind: OnePasswordItem
metadata:
  name: user-token
spec:
  itemPath: "vaults/v_id/items/item_id" 
```

こんな感じで使える
```
- name: GITEA_ADMIN_USERNAME
    valueFrom:
    secretKeyRef:
        name: user-token
        key: username
```

keyは`kubectl describe secret {secret-name}`とかで取れる


### MetalLB
service type: LoadBalancerを作る君。external-IPを振ってまわってくれる。

これを忘れがちなので忘れず実行。
```
kubectl get configmap kube-proxy -n kube-system -o yaml | \
sed -e "s/strictARP: false/strictARP: true/" | \
kubectl apply -f - -n kube-system
```

* https://metallb.io/installation/
* `sudo ip link set wlan0 promisc on`しないとIPの広報が永続化しないので注意
* `kubectl label node <ノード名> node.kubernetes.io/exclude-from-external-load-balancers-`しないとL2 Advertisementが無視されちゃうので忘れず

### nginx-ingress-controller
ingress作る君。後述のexternal-dnsと合わせて、ドメインで参照できるようにする

* https://kubernetes.github.io/ingress-nginx/deploy/baremetal/

### coredns + external-dns
dnsサーバー + ingressのhostnameをdnsに自動で登録する君

* externalIPsは外部からIP指定して参照できるようにするためにmasterの固定IPを直で参照する
* クラスタ内部でも名前解決できるように各ノードにもdnsサーバーとして登録しておく
    * https://freefielder.jp/blog/2021/06/ubuntu-20-04-dns-settings.html


### gitea
gitのhostサービス。軽い

* https://gitea.com/gitea/helm-chart
* 基本helmでいいが、以下の点に注意
    * 1.22.3は動かないので1.23にする
    * rootlessじゃないと動かない
    * persistent volume claimはdirectpv用に設定する必要あり
    * user/passwordはよしなにやってね
    * mail addressは初期はgitea@hogeとかになってるので変える
* [ミラー](https://docs.gitea.com/usage/repo-mirror)の設定をしておくと吉

#### actions
docker buildするときはdocker.sockとdaemon.jsonをmountしておく必要がある。

daemon.jsonはprivate repoにpushする際にhttpを許容するための設定を入れている。

```
name: ci

on:
  push:

jobs:
  docker:
    runs-on: ubuntu-docker
    container:
      volumes:
        - /var/run/docker.sock:/var/run/docker.sock
        - /etc/docker/daemon.json:/etc/docker/daemon.json
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
        with:
          buildkitd-config-inline: |
            [registry."192.168.1.249:5000"]
              http = true

      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          context: 
          push: true
          tags: 192.168.1.249:5000/sample:latest
```
 
### argocd
gitops。
基本install.yaml通りでいいが、家に閉じるのでhttpsを無効化するために--insecureフラグを立てる

* https://argo-cd.readthedocs.io/en/stable/getting_started/
* giteaに立てたhome-kubernetes-app/argocd_appsにApplicationを配置すると、自動的にargocdのApplicationに登録されるようになる（argocd_apps.yaml）
