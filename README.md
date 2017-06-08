# Kubernetes


## セットアップ
- 環境
  - Mac(10.12.2)
  - brew
    - `Homebrew 1.2.2`
    - `Homebrew/homebrew-core (git revision 889f1; last commit 2017-06-07)`
  - Vagrantのバージョン
    - `Vagrant 1.9.5`
  - VirtualBoxのバージョン
    - `5.0.16r105871`
　　　　- Dockerのバージョン  
    - `Docker version 1.10.3, build 20f81dd`
  - Vagrant/CoreOSの構成でMacローカルに構築

#### 環境構築について
- Kubernetesインストール
  - `$ brew install kubect`
- CoreOS
  -`$ git clone https://github.com/pires/kubernetes-vagrant-coreos-cluster.git`

- クラスタの構築
`$ cd kubernetes-vagrant-coreos-cluster`
`$ NUM_NODES=3 MASTER_MEM=512 NODE_MEM=512 vagrant up`
→途中で管理者パスワードが問われる（Macのsudoパスワードを入力）


　- `$ kubectl cluster-info`

```
Kubernetes master is running at https://172.17.8.101
KubeDNS is running at https://172.17.8.101/api/v1/proxy/namespaces/kube-system/services/kube-dns

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

`$ kubectl get node`
```
NAME           STATUS                     AGE       VERSION
172.17.8.101   Ready,SchedulingDisabled   22m       v1.6.4
172.17.8.102   Ready                      19m       v1.6.4
172.17.8.103   Ready                      16m       v1.6.4
```


#### Podを作成する

`$ cat pod-nginx.yaml`

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx-container
      image: nginx
      ports:
      - containerPort: 80
```

`$ kubectl create -f pod-nginx.yaml`
```
pod "nginx-pod" created
```

`$ kubectl get pod`
```
NAME        READY     STATUS              RESTARTS   AGE
nginx-pod   0/1       ContainerCreating   0          12s
```

Qitaより参照（わかりやすいので利用させて頂きます）: http://qiita.com/Arturias/items/62499b961b5d7375f608
```
Podとは いくつかのコンテナをグループ化したもの。Kubernets は Pod 単位で作成、開始、停止、削除といった操作を行う(コンテナ単位では操作しない)。 そのため、１つのコンテナを作成したいときも、「コンテナが１つ含まれるPod」を作成することになる。

他に以下の特徴がある。

Pod 内のコンテナは、同一ホスト上に配備される
Pod 内のコンテナは、仮想NICやプロセステーブルを共有する
→ つまり、同じIPを使えたり、互いのプロセスが見えたりする。

```

- Podの詳細確認
```
~/k/test-config ❯❯❯ kubectl describe pods nginx-pod
Name:		nginx-pod
Namespace:	default
Node:		172.17.8.103/172.17.8.103
Start Time:	Thu, 08 Jun 2017 13:52:58 +0900
Labels:		<none>
Annotations:	<none>
Status:		Running
IP:		10.244.70.2
Controllers:	<none>
Containers:
  nginx-container:
    Container ID:	docker://36aa8fb0b0496133a8d5e3ccfadfc632f7856a3a0200652b06d5504d6c7a2675
    Image:		nginx
    Image ID:		docker-pullable://nginx@sha256:41ad9967ea448d7c2b203c699b429abe1ed5af331cd92533900c6d77490e0268
    Port:		80/TCP
    State:		Running
      Started:		Thu, 08 Jun 2017 13:53:41 +0900
    Ready:		True
    Restart Count:	0
    Environment:	<none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-w9x7d (ro)
Conditions:
  Type		Status
  Initialized 	True
  Ready 	True
  PodScheduled 	True
Volumes:
  default-token-w9x7d:
    Type:	Secret (a volume populated by a Secret)
    SecretName:	default-token-w9x7d
    Optional:	false
QoS Class:	BestEffort
Node-Selectors:	<none>
Tolerations:	<none>
Events:
  FirstSeen	LastSeen	Count	From			SubObjectPath				Type		Reason		Message
  ---------	--------	-----	----			-------------				--------	------		-------
  5m		5m		1	default-scheduler						Normal		Scheduled	Successfully assigned nginx-pod to 172.17.8.103
  5m		5m		1	kubelet, 172.17.8.103	spec.containers{nginx-container}	Normal		Pulling		pulling image "nginx"
  4m		4m		1	kubelet, 172.17.8.103	spec.containers{nginx-container}	Normal		Pulled		Successfully pulled image "nginx"
  4m		4m		1	kubelet, 172.17.8.103	spec.containers{nginx-container}	Normal		Created		Created container with id 36aa8fb0b0496133a8d5e3ccfadfc632f7856a3a0200652b06d5504d6c7a2675
  4m		4m		1	kubelet, 172.17.8.103	spec.containers{nginx-container}	Normal		Started		Started container with id 36aa8fb0b0496133a8d5e3ccfadfc632f7856a3a0200652b06d5504d6c7a2675
```

#### それぞれのコンテナOSにログインしてコンテナを確認する

- core-01
```
core@node-01 ~ $ docker ps
CONTAINER ID        IMAGE                                             COMMAND                  CREATED             STATUS              PORTS               NAMES
d625f8663942        gcr.io/google_containers/k8s-dns-dnsmasq-amd64    "/usr/sbin/dnsmasq..."   3 hours ago         Up 3 hours                              k8s_dnsmasq_kube-dns-3612311663-3f5mp_kube-system_6d2a7399-4bff-11e7-a956-080027cc0fcb_0
a7cb1f325e68        gcr.io/google_containers/k8s-dns-kube-dns-amd64   "/kube-dns --domai..."   3 hours ago         Up 3 hours                              k8s_kubedns_kube-dns-3612311663-3f5mp_kube-system_6d2a7399-4bff-11e7-a956-080027cc0fcb_0
1388297ff10e        gcr.io/google_containers/k8s-dns-sidecar-amd64    "/sidecar --v=2 --..."   3 hours ago         Up 3 hours                              k8s_sidecar_kube-dns-3612311663-3f5mp_kube-system_6d2a7399-4bff-11e7-a956-080027cc0fcb_0
2323a150154d        gcr.io/google_containers/pause-amd64:3.0          "/pause"                 3 hours ago         Up 3 hours                              k8s_POD_kube-dns-3612311663-3f5mp_kube-system_6d2a7399-4bff-11e7-a956-080027cc0fcb_0
90d18d654e00        dd615ee13021                                      "/hyperkube proxy ..."   3 hours ago         Up 3 hours                              k8s_kube-proxy_kube-proxy-172.17.8.102_kube-system_1e7461d2d2ed0697101672b948a16ba0_0
c302d3de32f4        gcr.io/google_containers/pause-amd64:3.0          "/pause"                 3 hours ago         Up 3 hours                              k8s_POD_kube-proxy-172.17.8.102_kube-system_1e7461d2d2ed0697101672b948a16ba0_0
55c94dac0af8        gcr.io/google_containers/hyperkube-amd64:v1.6.4   "/hyperkube kubele..."   3 hours ago         Up 3 hours                              kubelet
```
