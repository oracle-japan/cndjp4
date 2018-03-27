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

##### Mac/Linux

    $ kubectl config set-context $(kubectl config current-context) --namespace=k8snet

##### Windows

    $ $CURRENT_CONTEXT=kubectl config current-context
    $ kubectl config set-context $CURRENT_CONTEXT --namespace=k8snet


### 最初のPodのデプロイ
まずは、nginxを可動させる、簡単なサンプルPodをKubernetesにデプロイしてみます。
といってもPodを直接デプロイするわけではなく、Deploymentオブジェクトを作成した上で、それを経由して複数のPodをデプロイします。

最初に``cndjp4/k8snet``をカレントディレクトリにしておきます。

    $ cd ./cndjp4/k8snet

続いてDeploymentオブジェクト作成します。

    $ kubectl create -f ./manifests/my-nginx.yaml
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
以下のコマンドでServiceオブジェクトを作成します。

    $ kubectl create -f ./manifests/my-nginx-service.yaml
    service "my-nginx" created

Serviceの一覧を取得すると以下のような結果となります。ここはClusterIPタイプのService（クラスター外部に公開されない）を作っているため、EXTERNAL-IPは\<none\>となっています。

    $ kubectl get services
    NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
    my-nginx     ClusterIP   10.104.246.87  <none>        80/TCP    7s

後の手順でServiceへのアクセスを思考するため、CLUSTER-IPの値をテキストエディタ等にコピーしておいてください。

Serviceの詳細情報を表示してみます。今回利用するサンプルPodには``run=my-nginx``というLabelが設定されています。このPodにルーティングするため、ServiceではLabelSelectorに同じく``run=my-nginx``を指定しています。

    $ kubectl describe services my-nginx
    Name:              my-nginx
    Namespace:         k8snet
    Labels:            run=my-nginx
    Annotations:       <none>
    Selector:          run=my-nginx
    Type:              ClusterIP
    IP:                10.104.246.87
    Port:              <unset>  80/TCP
    TargetPort:        80/TCP
    Endpoints:         <none>
    Session Affinity:  None
    Events:            <none>

Label/LabelSelectorによってServiceからのルーティング対象の紐づけができていますので、合わせてEndpointsオブジェクトも作成されています。<br>
Endpointの一覧と詳細情報も表示してみましょう。ルーティング対象のPodが2つあるため、ENDPOINTSには各々アドレスが表示されます。

##### 一覧

    $ kubectl get endpoints
    NAME       ENDPOINTS                     AGE
    my-nginx   172.17.0.4:80,172.17.0.5:80   1m

##### 詳細情報

    $ kubectl describe endpoints my-nginx
    Name:         my-nginx
    Namespace:    k8snet
    Labels:       run=my-nginx
    Annotations:  <none>
    Subsets:
      Addresses:          172.17.0.4,172.17.0.5
      NotReadyAddresses:  <none>
      Ports:
        Name     Port  Protocol
        ----     ----  --------
        <unset>  80    TCP

    Events:  <none>

### Serviceへのアクセスの確認

クラスター内からの通信も試してみます。前述と同様、inspectorのPodを立ち上げてターミナルに入ります。

    $ kubectl run inspector --image=radial/busyboxplus:curl -i --tty --rm

curlでServiceのIPアドレスに対してリクエストを送信することで、nginxにアクセスできていることがわかります（下のコマンドで、IPアドレスには先にメモしたServiceのものを指定してください）。

    [ root@{inspector}:/ ]$ curl 10.104.246.87
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

最後に、nslookupで名前解決ができるかどうかを確認してみます（下のコマンドで、IPアドレスには先にメモしたServiceのものを指定してください）。

    [ root@{inspector}:/ ]$ nslookup 10.104.246.87
    Server:    10.96.0.10
    Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

    Name:      10.104.246.87
    Address 1: 10.104.246.87 my-nginx.k8snet.svc.cluster.local

nslookupの結果ServiceのFQDNが解決できています。この前のステップではIPアドレスを直接指定してcurlを実行しましたが、改めてFQDNでアクセスしてみます。
FQDNを使って、Service経由でPodにアクセスできていることがわかります。

    [ root@{inspector}:/ ]$ curl my-nginx.k8snet.svc.cluster.local

FQDNの"svc.cluster.local"の部分は、ServiceオブジェクトのFQDNの場合固定の文字列となり、これは省略することができます。

    [ root@{inspector}:/ ]$ nslookup my-nginx.k8snet

"k8snet"の部分は、対象のServiceが属するNamespace名が設定されます。同じNamespaceに属するPod同士であれば、この部分も省略することができます。inspectorのPodは同じ"k8snet"内にデプロイされていますので、以下のコマンドでもServiceへのアクセスが可能です。

    [ root@{inspector}:/ ]$ curl my-nginx

最後に、``exit``コマンドでinspectorのターミナルを抜けておきます。

    [ root@{inspector}:/ ]$ exit


3 . Service Network - (2)
-------------------------
ここでは、Headless Serviceオブジェクトを作って挙動を確認してみます。<br>
Headless Serviceを使うと、ServiceではなくPodのIPアドレスに対して直接FQDNが設定されます。

### Serviceオブジェクトの作成
Headless ServiceはClusterIPに"None"を指定したSerivceオブジェクトです。Kubernetesにデプロイする前に、manifestファイルの内容を確認してみてください。

    > cat ./manifests/my-nginx-service-headless.yaml

以下のコマンドでHeadless Serviceオブジェクトを作成します。

    $ kubectl create -f ./manifests/my-nginx-service-headless.yml
    service "my-nginx-headless" created

ServiceとEndpointsの一覧を取得すると、Headless ServiceにはClusterIPが割り当てられていないことがわかります。

##### Service一覧

    $ kubectl get services
    NAME                TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
    my-nginx            ClusterIP   10.104.246.87  <none>        80/TCP    9m
    my-nginx-headless   ClusterIP   None           <none>        80/TCP    18s

##### Endpoints一覧

    $ kubectl get endpoints
    NAME                ENDPOINTS                     AGE
    my-nginx            172.17.0.4:80,172.17.0.5:80   23s
    my-nginx-headless   172.17.0.4:80,172.17.0.5:80   16s

### Serviceへのアクセスの確認
inspectorを利用してnslookupしてみます。ここではClusterIPではなく（そもそも割り当てられていない）、Serviceに設定されるFQDNのルールに従って、名前を指定してみます。

    $ kubectl run inspector --image=radial/busyboxplus:curl -i --tty --rm
    [ root@{inspector}:/ ]$ nslookup my-nginx-headless
    Server:    10.96.0.10
    Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

    Name:      my-nginx-headless
    Address 1: 172.17.0.4
    Address 2: 172.17.0.5

PodのIPアドレスに対して、名前が割り当てられいることがわかります。<br>
同じように、curlでPodにアクセスすることも可能です。

    [ root@{inspector}:/ ]$ curl my-nginx-headless

最後に、``exit``コマンドでinspectorのターミナルを抜けておきます。
<!--また、Headless Serviceは以降の手順では使わないので、削除しておきます。-->

    [ root@{inspector}:/ ]$ exit
<!--    $ kubectl delete -f ./my-nginx-service-headless.yaml-->


4 . クラスター外への公開 - (1)
------------------------------
ここでは、クラスター外からのアクセスを受け入れるようにKubenetes Objectを構成してみます。

Kubernetesにはいくつかの公開方法がありますが、ローカルのクラスターでも利用可能なもののうち、ここではServiceのNodePortタイプを試します。<br>
ServiceのNodePortタイプを利用すると、Kubernetesを可動させているNodeの、特定のPortを公開します。公開されたPortにアクセスすると、当該Serviceを通してルーティング対象のPodにリクエストが到達します。

### Serviceオブジェクトの作成

ServiceオブジェクトでNodePortタイプを指定するには、``{.spec.type}``に"NopePort"を記述します。Kubernetesにデプロイする前に、manifestファイルの内容を確認してみてください。

    $ cat ./manifests/my-nginx-service-nodePort.yaml

以下のコマンドでNodePortタイプのServiceオブジェクトを作成します。

    $ kubectl create -f ./manifests/my-nginx-service-nodePort.yaml｀：ｗ
    service "my-nginx-nodeport" created                                                 
Serviceの一覧を表示してみます。

    $ kubectl get services
    NAME                TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
    my-nginx            ClusterIP   10.104.246.87    <none>        80/TCP           44s
    my-nginx-headless   ClusterIP   None             <none>        80/TCP           34s
    my-nginx-nodeport   NodePort    10.111.120.119   <none>        8080:30774/TCP   24s

my-nginx-nodeportを見ると、PORTに"8080:30774/TCP"とあります（30774の部分はランダムに選択されるため、異なる値になっている可能性があります）。これは、Nodeの30774ポートにアクセスしたときに、このServiceのClusterIPの、8080ポートにフォワードされることを意味します。<br>
Serviceに到達すると、後はService Networkと同様の仕組みで、Podにルーティングされます。

続いて、Endpointの一覧です。

    $ kubectl get endpoints                                                   
    NAME                ENDPOINTS                     AGE                                          
    my-nginx            172.17.0.4:80,172.17.0.5:80   1m
    my-nginx-headless   172.17.0.4:80,172.17.0.5:80   1m
    my-nginx-nodeport   172.17.0.4:80,172.17.0.5:80   1m

### Serviceへのアクセスの確認
それではこのServiceによって公開された、Node/Portに実際にアクセスしてみます。
先にメモしておいたNodeのIPアドレスと、公開されているPort番号（この例の場合30774）を指定して、以下のようにコマンドを実行してください。

##### Mac/Linux

    $ curl 192.168.99.100:30774

##### Windows
（レスポンスボディを直接コンソールに表示できないので、一度ファイルに保存してからその内容を表示しています）

    $ Invoke-RestMethod -Uri "http://192.168.99.100:30774/" -o my-nginx-content.html
    $ cat ./my-nginx-content.html

クラスター外から、サンプルPodにアクセスして、応答を得ることができました。


5 . クラスター外への公開 - (2)
------------------------------
ここでは、クラスター外からのアクセスを受け入れるもう一つの手段として、Ingressを試します。

Serviceオブジェクトは、L3/4相当のロードバランシングの機能を提供しますが、Ingressを利用すると、L7ロードバランシングやTLS/SSLの終端等が行えます。

### IngressとNGINX Ingress Controllerのデプロイ
Ingressオブジェクトは、Webフロントとしての設定情報を保持するオブジェクトで、実際の機能を担う実態は、別途ReplicationControllerとしてデプロイする必要があります。<br>
今回は[NGiNX Ingress Controller](https://github.com/kubernetes/ingress-nginx)を利用します。これを使うと、クラスター内にnginxのPodを立ち上げて、それをロードバランサとして利用するように動きます。

まずはじめに、外部からのリクエストに対してルーティング先が見つからなかったときのフォールバック先として、デフォルトのバックエンドを構成しておきます。

    $ kubectl create -f ./manifests/web-default-backend.yaml
    replicationcontroller "default-http-backend" created

ReplicationContrllerが、フォールバック先の実態となるPodを立ち上げてくれています。

    $ kubectl get pods
    NAME                         READY     STATUS    RESTARTS   AGE
    default-http-backend-8ktdr   1/1       Running   0          1m
    my-nginx-9d5677d94-4jpsh     1/1       Running   0          1h
    my-nginx-9d5677d94-648cx     1/1       Running   0          1h

これだけではフォールバック先にアクセスできないので、Serviceを構成してクラスター内に公開しておきます。

    $ kubectl expose replicationcontroller default-http-backend --port=80 --target-port=8080 --name=default-http-backend
    service "default-http-backend" exposed

次にNGINX Ingress Controllerをデプロイします。

    $ kubectl create -f ./manifests/nginx-ingress-controller.yaml
    replicationcontroller "nginx-ingress-controller" created

デフォルト・バックエンドと同じように、Ingress ControllerのPodも立ち上がります。

    $ kubectl get pods
    NAME                             READY     STATUS    RESTARTS   AGE
    default-http-backend-8ktdr       1/1       Running   0          17m
    my-nginx-9d5677d94-4jpsh         1/1       Running   0          1h
    my-nginx-9d5677d94-648cx         1/1       Running   0          1h
    nginx-ingress-controller-gm8xv   1/1       Running   0          46s

最後に、Ingressオブジェクトを構成して、Ingress Controllerを公開します。Ingressオブジェクトのmanifestには、Ingress Controllerに適用する設定情報（ルーティング・ルール等）を記述しておきます。<br>
今回の例では、manifestの内容は以下のようになっています。

    $ cat ./manifests/my-nginx-ingress.yaml
    apiVersion: extensions/v1beta1          
    kind: Ingress                           
    metadata:                               
      name: my-nginx-ingress                
      annotations:                          
        kubernetes.io/ingress.class: "nginx"
    spec:                                   
      rules:                                
      - http:                               
          paths:                            
          - path: /                         
            backend:                        
              serviceName: my-nginx         
              servicePort: 80                

``{.metadata.annotations}``には、ロードバランサの実態として利用するIngress Controllerを指定します。上記の例のように書くとNGiNX Ingress Cotrollerが利用されます。

``{.spec}``配下はルーティングのルールを記述しています。この例では、全てのホスト、パスに対するリクエストが、Serviceオブジェクト"my-nginx"の80ポートにルーティングされます。<br>
このようなルールは複数記述することが、ホストのドメイン名のパターンや、パスのパターン毎に異なるServiceにルーティングするよう設定することも可能です。

最後に、Ingressオブジェクトをデプロイします。

    $ kubectl create -f ./manifests/my-nginx-ingress.yaml

### Ingressへのアクセスの確認
NGiNX Ingress ControllerのPodが可動しているホストを調べるため、以下のコマンドを実行します。

    $ kubectl get pods -l name=nginx-ingress-lb -o 'jsonpath={.items[0].status.hostIP}'
    192.168.99.100

NGiNX Ingress Cotrollerはデフォルトで80番ポート(http)へのアクセスを443番ポート(https)にリダイレクトします。また、443番ポートは自己署名証明書でSSL/TLSが構成されています。

curlを使ってアクセスを確認することもできますが、ここではブラウザで動作を見てみます。以下のURLにアクセスしてみてください（IPアドレスは上で取得したものを指定）。

    http://192.168.99.100/

httpsスキームのURLにリダイレクトされた後、信頼できない証明書が利用されている旨の警告画面が表示されます。さらにアクセスを続行すると、nginxのWelcome画面が表示されます（これは"my-nginx"としてデプロイしたサンプルPodが返しています）。


6 . クリーンアップ
------------------------------
お疲れ様でした。以上でハンズオンのPart1は終了です。

これまで作ってきたオブジェクトをクリーンアップしたい場合は、以下のコマンドを実行して、Namespaceごと削除を行ってください。Part2は比較的リソースを多く消費するため、特にminikubeを使っている場合は削除しておくことをおすすめします。

    $ kubectl delete namespace k8snet


以上。
