CNDJP #4 ハンズオンチュートリアル Part 1
========================================

これは、cndjp第4回勉強会のハンズオン Part 1のチュートリアルです。

このチュートリアルでは、xxxします。


前提条件
--------
このチュートリアルはKubernetes 1.7以降の任意のクラスターを利用可能です。connpassのエントリーで予めご案内したもののうち、ご準備いただいたものをご利用ください。

以降の手順は、ほとんどの操作をコマンドラインツールから実行します。Mac/Linuxの場合はターミナル、Windowsの場合はWindows PowerShell利用するものとして、手順を記述します。

（当日の現地参加者向けのメッセージ）
Windows環境で、[cndjp第1回勉強会のハンズオンチュートリアル](https://github.com/oracle-japan/cndjp1/blob/master/handson/handson1.md)を利用して構築したクラスターは、スクリプトの不具合と思われる問題により、Part2を実施することができません。該当する方は、この時点でminikubeを使ってクラスターを作成してから作業を開始することをおすすめします。


0 . 準備作業
------------

### 0-1. Kubernetesクラスターの起動
Part2では、KubernetesクラスターのRole Based Access Controll(RBAC)が有効になっている必要があります。以下の通り、環境に合わせた方法で起動してください。

##### [cndjp第1回勉強会のハンズオンチュートリアル](https://github.com/oracle-japan/cndjp1/blob/master/handson/handson1.md)を利用して構築した場合
``vagrant up``を実行する際に、AUTHORIZATION\_MODE=RBACオプションを指定して、RBACを有効にします。

    > AUTHORIZATION_MODE=RBAC vagrant up

##### minikubeを利用する場合
特別な設定は不要です。通常の手順でminikubeを起動してください。

    > minikube start

### 0-2. Kubernetesクラスターへのアクセスの確認
クラスターを起動したら、以下のコマンドを実行してクラスターへの疎通を確認します。

##### クラスター情報の取得

    > kubectl cluster-info
    Kubernetes master is running at https://172.17.8.101
    KubeDNS is running at https://172.17.8.101/api/v1/namespaces/kube-system/services/kube-dns/proxy

    To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

##### ノードの一覧の取得

    > kubectl get nodes
    NAME           STATUS                     AGE       VERSION
    172.17.8.101   Ready,SchedulingDisabled   12h       v1.7.11
    172.17.8.102   Ready                      12h       v1.7.11
    172.17.8.103   Ready                      12h       v1.7.11

表示されるノード数は、環境によって異なります。例えば、minkubeの場合は1ノードのみ表示されます。

### 0-3. チュートリアルのファイル一式を取得する
利用するファイル一式が、このチュートリアルをホストするリポジトリに保存されています。これを適当なディレクトリにcloneして下さい。

以下は、コマンドラインツールのgitを使う例です。

    > git clone https://github.com/oracle-japan/cndjp4.git


1 . Pod Network
---------------
まずは、クラスター内のPod間で直接通信する、Pod Networkの挙動を試してみます。

### Namespaceを作成する
最初に、Part1の作業を通して利用するNamespaceとして"k8snet"を作成しておきます。

    $ kubectl create namespace k8snet

kubectlのデフォルトnamespaceを"k8snet"に変更しておきます。

・Mac/Linux

    $ kubectl config set-context $(kubectl config current-context) --namespace=k8snet

・Windows

    $ $CURRENT_CONTEXT=kubectl config current-context
    $ kubectl config set-context $CURRENT_CONTEXT --namespace=k8snet


### 最初のPodのデプロイ
nginxを可動させる、簡単なサンプルPodをKubernetesにデプロイしてみます。
といってもPodを直接デプロイするわけではなく、Deploymentオブジェクトを作成した上で、それを経由して複数のPodをデプロイします。

    $ kubectl create -f ./my-nginx.yaml
    deployment "my-nginx" created

今回の例では、2つのサンプルPodを可動させていますので、Podの一覧を取得すると以下のような結果となります。

    $ kubectl get pods
    NAME                       READY     STATUS    RESTARTS   AGE
    my-nginx-9d5677d94-sl4cs   1/1       Running   0          3333s
    my-nginx-9d5677d94-2n8dc   1/1       Running   0          33s
    
サンプルPodのIPアドレスを取得しておきます。まずは、Podに割り当てられた、クラスター内で一意のIPアドレスです。

    $ kubectl get pod -o 'jsonpath={.items[0].status.podIP}'
    127.17.0.4
    $ kubectl get pod -o 'jsonpath={.items[1].status.podIP}'
    127.17.0.5

これ以降の作業で利用するため、どちらか一方のIPアドレスをテキストエディタ等にコピーしておいてください。

続いて、Podが可動しているノードのIPアドレスです。

    $ kubectl get pod -o 'jsonpath={.items[0].status.hostIP}'
    192.168.99.100
    $ kubectl get pod -o 'jsonpath={.items[1].status.hostIP}'
    192.168.99.100

こちらも同様に、どちらか一方のIPアドレスをテキストエディタ等にコピーしておいてください。

### クラスター内からPodにアクセスしてみる
クラスター内のさらに別のPodから、ひとつ前の手順で作成したサンプルPodにアクセスできるかどうか、確認してみます。<br>
以下のコマンドを実行すると、クラスター内に新たにPodを立ち上げた上で、そのPodのコンソールにアクセスすることができます。

    $ kubectl run inspector --image=radial/busyboxplus:curl -i --tty --rm

ここから、サンプルPodに通信を投げることによって、クラスター内のPod同士の通信の挙動を確認することができます。<br>
まず、pingを試してみます。以下のコマンドでは、IPアドレスの部分を、先にメモしたPodのIPアドレスに置き換えてください。

    [ root@{inspector}:/ ]$ ping -c 4 127.17.0.4
    PING 127.17.0.4 (127.17.0.4): 56 data bytes
    64 bytes from 127.17.0.4: seq=0 ttl=64 time=0.036 ms
    64 bytes from 127.17.0.4: seq=1 ttl=64 time=0.032 ms
    64 bytes from 127.17.0.4: seq=2 ttl=64 time=0.035 ms
    64 bytes from 127.17.0.4: seq=3 ttl=64 time=0.035 ms

サンプルPodへのpingは成功しますので、ネットワークの疎通には問題ないようです。<br>
続いて、curlコマンドを試してみます。

    [ root@{inspector}:/ ]$ curl http://127.17.0.4/
    curl: (7) Failed to connect to 127.17.0.4 port 80: Connection refused

こちらは失敗します。この場合のようにPod間で直接通信するケースでは、少なくともアプリケーション層まで到達することはできないようです。

最後に、nslookupで名前解決ができるかどうかを確認してみます。

    [ root@{inspector}:/ ]$ nslookup 127.17.0.4
    Server:    10.96.0.10
    Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local
    
    Name:      172.17.0.4
    Address 1: 172.17.0.4

この例では、kube-dnsがKubernetesクラスター内のDNSサービスとして可動していることが見て取れます。KubernetesではDNSサービスはアドオンという位置づけのため、環境によっては異なるDNSサービスが可動している場合もあります。

名前解決の結果としては、IPアドレスが取得されているだけですので、まだPodに対してFQDNが割あたっている状態ではありません。

最後に、``exit``コマンドでこのPodのターミナルを抜けておきます。

    [ root@{inspector}:/ ]$ exit


2 . Service Network - (1)
-------------------------
ここでは、Serviceオブジェクトを経由してPodにアクセスする、Service Networkの挙動を確認してみます。

### Serviceオブジェクトの作成


    $ kubectl create -f ./my-nginx-service.yaml
    service "my-nginx" created

    $ kubectl get svc
    NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
    my-nginx     ClusterIP   10.102.34.153   <none>        80/TCP    7s

    $ kubectl describe svc my-nginx
    Name:              my-nginx
    Namespace:         default
    Labels:            run=my-nginx
    Annotations:       <none>
    Selector:          run=my-nginx
    Type:              ClusterIP
    IP:                10.102.34.153
    Port:              <unset>  80/TCP
    TargetPort:        80/TCP
    Endpoints:         172.17.0.4:80
    Session Affinity:  None
    Events:            <none>

    $ kubectl get endpoints
    NAME         ENDPOINTS        AGE
    kubernetes   10.0.2.15:8443   24m
    my-nginx     172.17.0.4:80    1m

    $ kubectl describe endpoints my-nginx
    Name:         my-nginx
    Namespace:    default
    Labels:       run=my-nginx
    Annotations:  <none>
    Subsets:
      Addresses:          172.17.0.4
      NotReadyAddresses:  <none>
      Ports:
        Name     Port  Protocol
        ----     ----  --------
        <unset>  80    TCP

    Events:  <none>
    
    [ root@{inspector}:/ ]$ curl 127.17.0.4
    curl: (7) Failed to connect to 127.17.0.4 port 80: Connection refused

    [ root@{inspector}:/ ]$ curl 10.102.34.153
    <!DOCTYPE html>
    <html>
    <head>
    <title>Welcome to nginx!</title>
    <style>
        body {
            width: 35em;
            margin: 0 auto;
            font-family: Tahoma, Verdana, Arial, sans-serif;
        }
    </style>
    </head>
    <body>
    <h1>Welcome to nginx!</h1>
    <p>If you see this page, the nginx web server is successfully installed and
    working. Further configuration is required.</p>
    
    <p>For online documentation and support please refer to
    <a href="http://nginx.org/">nginx.org</a>.<br/>
    Commercial support is available at
    <a href="http://nginx.com/">nginx.com</a>.</p>
    
    <p><em>Thank you for using nginx.</em></p>
    </body>
    </html>

    [ root@{inspector}:/ ]$ nslookup 10.102.34.153
    Server:    10.96.0.10
    Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

    Name:      10.102.34.153
    Address 1: 10.102.34.153 my-nginx.default.svc.cluster.local

    [ root@{inspector}:/ ]$ curl http://my-nginx.default.svc.cluster.local/
    
    [ root@{inspector}:/ ]$ nslookup my-nginx.default.svc.cluster.local
    [ root@{inspector}:/ ]$ nslookup my-nginx.default
    [ root@{inspector}:/ ]$ nslookup my-nginx


3 . Service Network - (2)
-------------------------

### Headless Serviceを使ったPodへのアクセス

    $ kubectl create -f ./my-nginx-service-headless.yml
    service "my-nginx-headless" created

    $ kubectl get services
    NAME                TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
    kubernetes          ClusterIP   10.96.0.1    <none>        443/TCP   1h
    my-nginx-headless   ClusterIP   None         <none>        80/TCP    18s

    $ kubectl get endpoints
    NAME                ENDPOINTS        AGE
    kubernetes          10.0.2.15:8443   1h
    my-nginx-headless   172.17.0.4:80    6s

    $ kubectl run inspector --image=radial/busyboxplus:curl -i --tty --rm

    [ root@{inspector}:/ ]$ nslookup my-nginx-headless
    Server:    10.96.0.10
    Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

    Name:      my-nginx-headless
    Address 1: 172.17.0.4

    [ root@{inspector}:/ ]$ nslookup my-nginx-headless.default

  (curlは文章で) 

    $ kubectl delete -f ./my-nginx-service-headless.yaml


4 . クラスター外への公開
------------------------

### NodePort

    $ kubectl create -f .\my-nginx-service-nodePort.yaml                      
    service "my-nginx" created                                                 

    $ kubectl get svc                                                         
    NAME         TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE  
    kubernetes   ClusterIP   10.96.0.1     <none>        443/TCP          1h   
    my-nginx     NodePort    10.98.22.75   <none>        8080:32461/TCP   12s  

    $ kubectl get endpoints                                                   
    NAME         ENDPOINTS        AGE                                          
    kubernetes   10.0.2.15:8443   1h                                           
    my-nginx     172.17.0.4:80    29s                                          

    $ kubectl get pods                                                        
    NAME                       READY     STATUS    RESTARTS   AGE              
    my-nginx-9d5677d94-9tzsh   1/1       Running   0          58m              

    $ kubectl get pod my-nginx-9d5677d94-9tzsh -o 'jsonpath={.status.hostIP}' 
    192.168.99.100                                                             

    $ curl 192.168.99.100:32461                                               


### Ingress
        Nginxの実態がどこにあるかも確認しておく
            Ingressの参照先。Podの実態

3-1. IngressとNGINX Ingress Controllerのデプロイ
Ingressを使うと、外部アクセスのロードバランシングやSSL/TLSの終端といった、Webフロントの機能を、クラスター内に構成することができます（IngressはKubernetes 1.8の時点ではBetaの機能です。現時点では、クラスター外にLBを構成する方法が主流となっています）。

Ingressオブジェクトは、Webフロントとしての構成情報を設定するオブジェクトで、実際の機能を担う実態は、別途ReplicationControllerとしてデプロイする必要があります。 今回はNGINXを利用した、NGiNX Ingress Controllerを使います。

まずはじめに、外部からのリクエストに対してルーティング先が見つからなかったときのレスポンスを返すものとして、デフォルトのバックエンドを構成しておきます。

> kubectl create -f deployment/web-default-backend.yaml
replicationcontroller "default-http-backend" created

> kubectl expose rc default-http-backend --port=80 --target-port=8080 --name=default-http-backend
service "default-http-backend" exposed
今回のNGINX Ingress Controllerのmanifestでは、role=frontというLabelのついたNodeにデプロイされるように設定しています。以下のようにNodeにLabelを設定して、デプロイ先のNodeが固定されるようにします。

> kubectl label nodes 172.17.8.102 role=front
NGINX Ingress Controllerをデプロイします。

> kubectl create -f deployment/web-rc-default.yaml
replicationcontroller "nginx-ingress-controller" created
最後にIngressオブジェクトを構成します。

> kubectl create -f deployment/web-ingress.yaml




以上で、ハンズオンのPart1は終了です。

Part2へのリンク





---

ハンズオンチュートリアルのPart2では環境の設定に一定の条件がありますので、もしKubernetesクラスターをまだ起動していないのであれば、先に







Part2ではRole Based Access Control(RBAC)が有効になっているKubernetesクラスターを利用しますので、


    > kubectl run -i --tty --image busybox dns-test --restart=Never --rm /bin/sh

