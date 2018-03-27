CNDJP #4 ハンズオンチュートリアル Part 2
========================================

これは、cndjp第4回勉強会のハンズオン Part 2のチュートリアルです。

このチュートリアルでは、Kubernetesクラスターにサービスメッシュ"Istio"をインストールしします。更に、サンプルアプリケーションを使って、アプリに送られるトラフィックの流量を制御するところまでを試してみます。


前提条件
--------
このチュートリアルは任意のKubernetes 1.7以降の任意のクラスターを利用可能ですが、Role Based Access Controll(RBAC)が有効になっている必要があります。
connpassのエントリーで予めご案内したもののうち、ご準備いただいたものをご利用いただきますが、「0. 準備作業」でご案内したRBACの有効化手順を必ず実施してください。

以降の手順は、ほとんどの操作をコマンドラインツールから実行します。Mac/Linuxの場合はターミナル、Windowsの場合はWindows PowerShell利用するものとして、手順を記述します。


0 . 準備作業
------------
（Part1の準備作業を実施済みの場合、「1. Istioのインストール」に進んでください）

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


1 . Istioのインストール
-----------------------
KubernetesにIstioをインストールします。

### 1-1. Istioのダウンロードとistioctlのセットアップ
Istioの最新版をダウンロードします。

Isitoは、Kubernetesにデプロイするためのmanifest、コマンドラインツール(istioctl)、サンプルアプリ等の一式をzipアーカイブにまとめたものとして配布されています。<br>
このチュートリアルの作成時点（2018/03/26）では、0.6.0が最新バージョンです。

まず、カレントディレクトリを``cndjp4/istio-bootcamp``に変更しておきます。

    $ cd cndjp4/istio-bootcamp

続いて、Istioをダウンロードします。Mac/Linuxの場合は、最新バージョンを取得するためのスクリプトツールを利用することができます。

##### Mac/Linux

    $ curl -L https://git.io/getLatestIstio | sh -

##### Windows

    $ [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
    $ Invoke-WebRequest -Uri https://github.com/istio/istio/releases/download/0.6.0/istio_0.6.0_win.zip -OutFile istio_0.6.0_win.zip
    $ Expand-Archive -Path .\istio_0.6.0_win.zip

Istioのコマンドラインツール(istioctl)は、ダウンロードしてできたディレクトリ配下の``istio-0.6/bin``に配置されています。各プラットフォームに合わせて、istioctlをPATHに追加してください。
    
##### Mac/Linux

    $ export PATH=$PWD/istio-0.6/bin:$PATH

##### Windows

[コンピューターのプロパティ] > [システムの詳細設定]から、istioctlがあるフォルダをPATHに追加します。
具体的な手順は、[こちら](http://hogepy.blog.fc2.com/blog-entry-107.html)を参考にしてください。

istioctlのバージョン情報を表示して、istioctlが使えることを確認します。``istioctl version``を実行して、以下のような結果となることを確認してください。

    $ istioctl version
    Version: 0.6.0
    GitRevision: 2cb09cdf040a8573330a127947b11e5082619895
    User: root@a28f609ab931
    Hub: docker.io/istio
    GolangVersion: go1.9
    BuildStatus: Clean

### 1-2. IstioをKubernetesクラスターにインストールする
IstioをKubernetesクラスターにインストールします。必要なものは全てKubernetesのmanifestとして記述されているため、これをデプロイするだけでOKです。

（以下のコマンドは、カレント・ディレクトリが``cndjp4/istio-bootcamp``であるものとします）

##### Mac/Linux

    $ cd ./istio-0.6.0/
    $ kubectl apply -f ./install/kubernetes/istio.yaml

##### Windows
    
    $ cd ./istio_0.6.0_win/istio-0.6.0/
    $ kubectl apply -f ./install/kubernetes/istio.yaml

istio-systemというNamespace配下に必要な構成要素が配備されます。ServiceとPodにどのようなオブジェクトが配備されるか、確認してみてください。

##### Service

    $ kubectl get services -n istio-system
    NAME            TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                                                            AGE
    istio-ingress   LoadBalancer   10.100.151.26    <pending>     80:31826/TCP,443:31716/TCP                                         1m
    istio-mixer     ClusterIP      10.100.149.13    <none>        9091/TCP,15004/TCP,9093/TCP,9094/TCP,9102/TCP,9125/UDP,42422/TCP   1m
    istio-pilot     ClusterIP      10.100.118.141   <none>        15003/TCP,8080/TCP,9093/TCP,443/TCP                                1m

##### Pod

    $ kubectl get pods -n istio-system
    NAME                             READY     STATUS    RESTARTS   AGE
    istio-ca-418645489-gg8sh         1/1       Running   0          2m
    istio-ingress-1688831668-t24gh   1/1       Running   0          2m
    istio-mixer-3303323913-s8c2l     3/3       Running   0          2m
    istio-pilot-3625647026-0rv6z     2/2       Running   0          2m

上のように、全てのPodのSTATUSがRunningになればインストール完了です（およそ1-2分ほどかかります）。


2 . サンプルアプリケーションを配備する
--------------------------------------
このチュートリアルでは、サンプルアプリケーションとして、kubernetes-bootcampアプリケーションを使います。複数のバージョンのbootcampアプリをデプロイして、後でおこなう、Istioによるトラフィック制御に利用します。

### 2.1. Namespaceを作成する
以降のチュートリアルの手順で利用するNamespaceとして、"istio-bootcamp"を作成しておきます。

    $ kubectl create namespace istio-bootcamp

kubectlのデフォルトnamespaceを"istio-bootcamp"に変更しておきます。

##### Mac/Linux

    $ kubectl config set-context $(kubectl config current-context) --namespace=istio-bootcamp

##### Windows

    $ $CURRENT_CONTEXT=kubectl config current-context
    $ kubectl config set-context $CURRENT_CONTEXT --namespace=istio-bootcamp

### 2.2. bootcampアプリケーションのデプロイ
現在は"istio-0.6.0"というディレクトリがカレントディレクトリになっていますので、これを"cndjp4/istio-bootcamp/"に戻します。

##### Mac/Linux

    $ cd ../

##### Windwos

    $ cd ../../

続いて、bootcampアプリケーションをデプロイします。

    $ kubectl create -f ./manifests/bootcamp.yaml

version 1のPodが2つ、varsion 2のPodが1つデプロイされていることを確認してください。

    $ kubectl get pods

次に、アプリケーションにクラスター外からアクセスするためのService、およびIngressオブジェクトを作成します。ここでは、Ingress CotrollerとしてIstioに付属のIstio Ingress Cotrollerを利用します。

    $ kubectl create -f ./manifests/bootcamp-service.yaml

    $ kubectl create -f ./manifests/bootcamp-ingress.yaml

### 2.3. bootcampアプリケーションへのアクセスの確認
Ingress Controllerが配備されているNodeと公開されているPort番号を取得して、アプリケーションにアクセスするためのURLを取得します。

##### Linux/Mac

    $ export GATEWAY_URL=$(kubectl get po -l istio=ingress -n istio-system -o 'jsonpath={.items[0].status.hostIP}'):$(kubectl get svc istio-ingress -n istio-system -o 'jsonpath={.spec.ports[0].nodePort}')
    $ echo $GATEWAY_URL

##### Windows

    $ $GATEWAY_HOSTIP=kubectl get po -l istio=ingress -n istio-system -o 'jsonpath={.items[0].status.hostIP}'
    $ $GATEWAY_NODEPORT=kubectl get svc istio-ingress -n istio-system -o 'jsonpath={.spec.ports[0].nodePort}'
    $ $GATEWAY_URL="$GATEWAY_HOSTIP" + ":" + "$GATEWAY_NODEPORT"
    $ echo $GATEWAY_URL

取得したURLの末尾に"/bootcamp"を追加して、その宛先に対してリクエストを送信します。Pod名とアプリケーションのバージョン番号が返却されることを確認してください。

##### Linux/Mac

    $ curl http://$GATEWAY_URL/bootcamp

##### Windows

    $ Invoke-RestMethod -Uri "http://${GATEWAY_URL}/bootcamp"

この時点では、トラフィックの制御を何も行っていないため、3つのPodに対して均等の割合でルーティングされます。リクエストの送信を繰り返し実行すると、おおよそ2/3がv1、1/3がv2からの応答となります。

以上で、サンプルアプリケーションの配備は完了です。


3 . Istioによるトラフィックの制御を行う
---------------------------------------

### 3.1. Envoyを注入したPodに置き換える

    $ kubectl delete -f ./bootcamp.yaml

    $ istioctl kube-inject --debug -f ./bootcamp.yaml -o ./bootcamp-istio.yaml

    $ kubectl create -f ./bootcamp-istio.yaml

    $ kubectl describe deployments bootcamp-v1

<!--
    $ kubectl create -f ./manifests/bootcamp/bootcamp-service.yaml
    $ kubectl create -f ./manifests/bootcamp/bootcamp-ingress.yaml
-->

    $ istioctl create -n istio-bootcamp -f ./bootcamp-route-rule-all-v1.yaml

<!--
    $ istioctl replace -f ./manifests/bootcamp/bootcamp-route-rule-50-v2.yaml
-->

参考）
    https://github.com/istio/istio/issues/4215

    $ istioctl delete -n istio-bootcamp -f ./bootcamp-route-rule-all-v1.yaml
    $ istioctl create -n istio-bootcamp -f ./bootcamp-route-rule-50-v2.yaml


    $ curl http://$GATEWAY_URL/bootcamp
    $ Invoke-RestMethod -Uri "http://${GATEWAY_URL}/bootcamp"

    $ istioctl delete -n istio-bootcamp -f ./bootcamp-route-rule-50-v2.yaml
    $ istioctl create -n istio-bootcamp -f ./bootcamp-route-rule-all-v2.yaml


たくさんサンプルがあることを書いておく



