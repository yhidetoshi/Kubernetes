# Kubernetes

![Alt Text](https://github.com/yhidetoshi/Pictures/raw/master/Docker/kubernetes.png)

## 環境構築(AWS + Centos7のパターン)
- まずはCentOS7で環境を作ります

- 環境
  - AWS
    - master
      - `CentOS7`
      - `10.0.1.251`
    - Minion1
      - `CentOS7`
      - `10.0.1.223`
    - Minion2
      - `CentOS7`
      - `10.0.1.60`

`# kubectl --version`
 > Kubernetes v1.5.2

- 3台のhostsファイル
```
10.0.1.251 master
10.0.1.223 minion1
10.0.1.60  minion2
```

### Masterの設定
- `# yum -y update`
- `# yum -y install docker kubernetes flannel`
- `# systemctl enable docker`


##### kubelet設定
- `/etc/kubernetes/config`を編集
```
KUBE_LOGTOSTDERR="--logtostderr=true"
KUBE_LOG_LEVEL="--v=0"
KUBE_ALLOW_PRIV="--allow-privileged=false"
KUBE_MASTER="--master=http://master:8080"```
```

- `/etc/kubernetes/kubelet`を編集
```
KUBELET_ADDRESS="--address=master"
KUBELET_HOSTNAME="--hostname-override=master"
KUBELET_API_SERVER="--api-servers=http://master:8080"
KUBELET_POD_INFRA_CONTAINER="--pod-infra-container-image=registry.access.redhat.com/rhel7/pod-infrastructure:latest"
KUBELET_ARGS=""
```

- `/etc/kubernetes/apiserver`を編集
```
KUBE_API_ADDRESS="--address=0.0.0.0"
KUBE_API_PORT="--port=8080"
KUBELET_PORT="--kubelet-port=10250"
KUBE_ETCD_SERVERS="--etcd-servers=http://master:2379"
KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=10.254.0.0/16"
KUBE_API_ARGS=""
```

- `/etc/kubernetes/controller-manager`を編集
```
KUBE_CONTROLLER_MANAGER_ARGS="--service_account_private_key_file=/etc/kubernetes/serviceaccount.key"
KUBELET_ADDRESSES="--machines=master,minion1,minion2"
```

- ` /etc/etcd/etcd.conf`を編集
```
# [member]
ETCD_NAME=default
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"

#[cluster]
ETCD_ADVERTISE_CLIENT_URLS="http://0.0.0.0:2379"

#[proxy]

#[security]

#[logging]
```

- `# systemctl start etcd`
- `# etcdctl mkdir /kube-centos/network`
- `#etcdctl mk /kube-centos/network/config "{ \"Network\": \"172.30.0.0/16\", \"SubnetLen\": 24, \"Backend\": { \"Type\": \"vxlan\" } }"`

##### flannelファイルを編集: `/etc/sysconfig/flanneld`
```
FLANNEL_ETCD="http://master:2379"
FLANNEL_ETCD_PREFIX="/kube-centos/network"
```

- それぞれのサービスを起動(下記のシェルスクリプトを実行)
```
#!/sh/bin

for SERVICES in docker etcd kube-apiserver kube-controller-manager kube-scheduler flanneld; do
    systemctl restart $SERVICES
    systemctl enable $SERVICES
    systemctl status $SERVICES
done
```

#### Minionの設定

- `# yum update -y`
- `# yum -y install docker kubernetes flannel`

- `/etc/sysconfig/flanneld`を編集（Masterと同じ内容にする）
- `/etc/kubernetes/kubelet`　
```
KUBELET_ADDRESS="--address=0.0.0.0"
KUBELET_PORT="--port=10250"
KUBELET_HOSTNAME=""
KUBELET_API_SERVER="--api-servers=http://master:8080"
KUBELET_ARGS=""
```

- それぞれのサービスを起動(下記のシェルスクリプトを実行)
```
#!/sh/bin

for SERVICES in kube-proxy kubelet docker flanneld; do 
    systemctl restart $SERVICES
    systemctl enable $SERVICES
    systemctl status $SERVICES 
done
```
#### Masterから動作確認

##### ノード確認
```
# kubectl get nodes
NAME                                            STATUS    AGE
ip-10-0-1-223.ap-northeast-1.compute.internal   Ready     16h
ip-10-0-1-60.ap-northeast-1.compute.internal    Ready     35m
```
→ これでMasterとMinionの環境セットアップが完了。

まずは色々ためすために下記のように適当にディレクトリを作成
```
# tree work/.
.
├── deployment
│   └── deployment-nginx.yaml
├── pod
│   └── pod-nginx.yaml
└── service
    └── service.yaml
```


##### Pod作成

- `pod-nginx.yaml`
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

- `# kubectl create -f pod-nginx.yaml`
```
pod "nginx-pod" created
```

- `# kubectl get pod`
```
NAME        READY     STATUS              RESTARTS   AGE
nginx-pod   0/1       ContainerCreating   0          13s
```

- Podの削除
  - `# kubectl delete pod <POD_NAME>`

- Podの詳細確認
  - `# kubectl describe pods nginx-deployment-4087004473-1px8p`
```
Name:		nginx-deployment-4087004473-1px8p
Namespace:	default
Node:		ip-10-0-1-223.ap-northeast-1.compute.internal/10.0.1.223
Start Time:	Thu, 03 Aug 2017 02:27:25 +0000
Labels:		app=nginx
		pod-template-hash=4087004473
Status:		Running
IP:		172.30.50.2
Controllers:	ReplicaSet/nginx-deployment-4087004473
Containers:
  nginx:
    Container ID:		docker://c926feff159eb273c8df8e238d1e59395867a578ff6ef10360c53fba6f4bd952
    Image:			nginx:1.7.9
    Image ID:			docker-pullable://docker.io/nginx@sha256:e3456c851a152494c3e4ff5fcc26f240206abac0c9d794affb40e0714846c451
    Port:			80/TCP
    State:			Running
      Started:			Thu, 03 Aug 2017 02:27:54 +0000
    Ready:			True
    Restart Count:		0
    Volume Mounts:		<none>
    Environment Variables:	<none>
Conditions:
  Type		Status
  Initialized 	True
  Ready 	True
  PodScheduled 	True
No volumes.
QoS Class:	BestEffort
Tolerations:	<none>
No events.
```

`===============================
`
- deployment

- `deployment-nginx.yaml`
```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```


`# kubectl create -f deployment-nginx.yaml --record`
> deployment "nginx-deployment" created

`# kubectl get deployment`
```
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   2         2         2            0           16s
```

`# kubectl get pods --show-labels`
```
NAME                                READY     STATUS    RESTARTS   AGE       LABELS
nginx-deployment-4087004473-1px8p   1/1       Running   0          3m        app=nginx,pod-template-hash=4087004473
nginx-deployment-4087004473-vc1dm   1/1       Running   0          3m        app=nginx,pod-template-hash=4087004473
```

##### Service作成
- `# kubectl create -f service.yaml`

- `service.yaml` 
```
service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  type: NodePort
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    run: nginx
```

### Deployment検証

**deploymentを削除してノード上のコンテナが削除されるかの検証**

- `# kubectl get deployment` (deploymentの一覧取得)
```
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   2         2         2            2           1d
```

- `# kubectl delete deployment nginx-deployment`(deploymentの一削除)
> deployment "nginx-deployment" deleted

- `# kubectl get deployment`
> No resources found.

**(deploymentを削除して)node側でコンテナが削除されているか確認**


- `# docker ps` (deploymentを削除する前)
```
CONTAINER ID        IMAGE                                      COMMAND                  CREATED             STATUS              PORTS               NAMES
c926feff159e        nginx:1.7.9                                "nginx -g 'daemon off"   27 hours ago        Up 27 hours                             k8s_nginx.b0df00ef_nginx-deployment-4087004473-1px8p_default_49ea4593-77f3-11e7-8a59-06839e14de9c_28cd889d
7f59581a9910        gcr.io/google_containers/pause-amd64:3.0   "/pause"                 27 hours ago        Up 27 hours                             k8s_POD.b2390301_nginx-deployment-4087004473-1px8p_default_49ea4593-77f3-11e7-8a59-06839e14de9c_3b415bb9
```

- `# docker ps` (deploymentを削除した後)
```
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
```



## 環境構築(Mac + Vagrantパターン)
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

- master
```
core@master ~ $ docker ps
CONTAINER ID        IMAGE                                             COMMAND                  CREATED             STATUS              PORTS               NAMES
ccd278462284        gcr.io/google_containers/hyperkube-amd64          "/hyperkube schedu..."   3 hours ago         Up 3 hours                              k8s_kube-scheduler_kube-scheduler-172.17.8.101_kube-system_189fefc4732931d3c7eeaa8553f068dd_0
124eb4421e6c        gcr.io/google_containers/hyperkube-amd64          "/hyperkube proxy ..."   3 hours ago         Up 3 hours                              k8s_kube-proxy_kube-proxy-172.17.8.101_kube-system_876b5c028f7b1c10fde4d3e4a650343a_0
b802a9262089        gcr.io/google_containers/hyperkube-amd64          "/hyperkube apiser..."   3 hours ago         Up 3 hours                              k8s_kube-apiserver_kube-apiserver-172.17.8.101_kube-system_7a6ee5ada1c80522663165ad94314e15_0
33d2661fcfb9        gcr.io/google_containers/hyperkube-amd64          "/hyperkube contro..."   3 hours ago         Up 3 hours                              k8s_kube-controller-manager_kube-controller-manager-172.17.8.101_kube-system_789aa35ccf8d7b4ee999b5c5517294fa_0
b04871b24b58        gcr.io/google_containers/pause-amd64:3.0          "/pause"                 3 hours ago         Up 3 hours                              k8s_POD_kube-apiserver-172.17.8.101_kube-system_7a6ee5ada1c80522663165ad94314e15_0
6980efd8003b        gcr.io/google_containers/pause-amd64:3.0          "/pause"                 3 hours ago         Up 3 hours                              k8s_POD_kube-scheduler-172.17.8.101_kube-system_189fefc4732931d3c7eeaa8553f068dd_0
173dbcaaab26        gcr.io/google_containers/pause-amd64:3.0          "/pause"                 3 hours ago         Up 3 hours                              k8s_POD_kube-proxy-172.17.8.101_kube-system_876b5c028f7b1c10fde4d3e4a650343a_0
ec784a6bf7c9        gcr.io/google_containers/pause-amd64:3.0          "/pause"                 3 hours ago         Up 3 hours                              k8s_POD_kube-controller-manager-172.17.8.101_kube-system_789aa35ccf8d7b4ee999b5c5517294fa_0
9d0cad460b7d        gcr.io/google_containers/hyperkube-amd64:v1.6.4   "/hyperkube kubele..."   3 hours ago         Up 3 hours                              kubelet
```

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

- core-02
```
core@node-02 ~ $ docker ps
CONTAINER ID        IMAGE                                             COMMAND                  CREATED             STATUS              PORTS               NAMES
36aa8fb0b049        nginx                                             "nginx -g 'daemon ..."   2 hours ago         Up 2 hours                              k8s_nginx-container_nginx-pod_default_57f76233-4c06-11e7-a956-080027cc0fcb_0
2b6255a4879c        gcr.io/google_containers/pause-amd64:3.0          "/pause"                 2 hours ago         Up 2 hours                              k8s_POD_nginx-pod_default_57f76233-4c06-11e7-a956-080027cc0fcb_0
0b303d04c726        dd615ee13021                                      "/hyperkube proxy ..."   3 hours ago         Up 3 hours                              k8s_kube-proxy_kube-proxy-172.17.8.103_kube-system_1e7461d2d2ed0697101672b948a16ba0_0
7ce666b91d9b        gcr.io/google_containers/pause-amd64:3.0          "/pause"                 3 hours ago         Up 3 hours                              k8s_POD_kube-proxy-172.17.8.103_kube-system_1e7461d2d2ed0697101672b948a16ba0_0
9baa08d8067a        gcr.io/google_containers/hyperkube-amd64:v1.6.4   "/hyperkube kubele..."   3 hours ago         Up 3 hours                              kubelet
```

- node-02からコンテナとして作ったNginxに接続できるかの確認
  - `core@node-02 ~ $ curl http://10.244.70.2`
  - `IPはkubectl describe podsで確認`


## 環境構築その１(未完成)
- 環境
  - AWS
    - master
      - `CoreOS : CoreOS-alpha-584.0.0-hvm (ami-00958f01)`
      - `10.0.1.154`
      - hostname: master
    - node1
      - `CoreOS : CoreOS-alpha-584.0.0-hvm (ami-00958f01)`
      - `10.0.1.200`
      - hostname: node1
    - node2
      - `CoreOS : CoreOS-alpha-584.0.0-hvm (ami-00958f01)`
      - `10.0.1.83`
      - hostname: node2
 
 #### まずはCoreOS側の設定から行う
 - 3台にhostnameをひけるように設定
   - `/usr/share/oem/cloud-config.yml` 
   - `/etc/hosts`を記述するには書きの方法をとる
```
write_files:
    - path: /run/systemd/system/etcd.service.d/10-oem.conf
      permissions: 0644
      content: |
        [Service]
        Environment=ETCD_PEER_ELECTION_TIMEOUT=1200
+   - path: /etc/hosts
+     content: |
+        127.0.0.1 localhost
+        127.0.0.1 master
+        10.0.1.200 node1
+        10.0.1.83 node2
```
 - cloud-configを反映するコマンド
   - `$ sudo coreos-cloudinit --from-file /usr/share/oem/cloud-config.yml`
- 試しにPing
```
master oem # ping node1
PING node1 (10.0.1.200) 56(84) bytes of data.
64 bytes from node1 (10.0.1.200): icmp_seq=1 ttl=64 time=0.643 ms
^C
--- node1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.643/0.643/0.643/0.000 ms
master oem # ping node2
PING node2 (10.0.1.83) 56(84) bytes of data.
64 bytes from node2 (10.0.1.83): icmp_seq=1 ttl=64 time=0.997 ms
^C
--- node2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.997/0.997/0.997/0.000 ms
master oem # ping master
PING master (127.0.0.1) 56(84) bytes of data.
64 bytes from localhost (127.0.0.1): icmp_seq=1 ttl=64 time=0.021 ms
^C
--- master ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.021/0.021/0.021/0.000 ms
```
- 次に `etcd`, `flannel` を設定していく
