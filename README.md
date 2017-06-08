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

Qitaより参照（わかりやすいので利用させて頂きます）: `http://qiita.com/Arturias/items/62499b961b5d7375f608`
```
いくつかのコンテナをグループ化したもの。Kubernets は Pod 単位で作成、開始、停止、削除といった操作を行う(コンテナ単位では操作しない)。 そのため、１つのコンテナを作成したいときも、「コンテナが１つ含まれるPod」を作成することになる。

他に以下の特徴がある。

Pod 内のコンテナは、同一ホスト上に配備される
Pod 内のコンテナは、仮想NICやプロセステーブルを共有する
→ つまり、同じIPを使えたり、互いのプロセスが見えたりする。

```
