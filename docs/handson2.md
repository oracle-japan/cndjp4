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
    deployment "bootcamp-v1" created
    deployment "bootcamp-v2" created

version 1のPodが2つ、varsion 2のPodが1つデプロイされていることを確認してください。

    $ kubectl get pods
    NAME                           READY     STATUS    RESTARTS   AGE
    bootcamp-v1-68cb78f87c-l8szm   1/1       Running   0          42s
    bootcamp-v1-68cb78f87c-zlmqh   1/1       Running   0          42s
    bootcamp-v2-569574d79f-w4p6g   1/1       Running   0          41s

次に、アプリケーションにクラスター外からアクセスするためのService、およびIngressオブジェクトを作成します。ここでは、Ingress CotrollerとしてIstioに付属のIstio Ingress Cotrollerを利用します。

    $ kubectl create -f ./manifests/bootcamp-service.yaml
    service "bootcamp" created
    $ kubectl create -f ./manifests/bootcamp-ingress.yaml
    ingress "bootcamp" created

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
それでは、Istioを使ったトラフィック量の制御を実際に行ってみます。

### 3.1. bootcampアプリケーションのPodをEnvoyを注入したものに置き換える
ここまでの手順でデプロイしたアプリケーションは、Kubernetesの通常の方法でデプロイしているため、PodにEnvoyプロキシを含んでいません。まずは、これをEnvoyを含むものに置き換えます。

以下のコマンドで、一度bootcampアプリケーションを削除します。

    $ kubectl delete -f ./manifests/bootcamp.yaml

次に、istioctlを使って、bootcampアプリケーションのmanifestファイルにEnvoyの記述を追加します。

    $ istioctl kube-inject --debug -f ./manifests/bootcamp.yaml -o ./manifests/bootcamp-istio.yaml

新たに作成されたmanifestファイルの内容を参照してみます。
    
    $ cat ./manifests/bootcamp-istio.yaml

以下は、v1のbootcampアプリケーションのDeployment部分を抜粋したものです。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  name: bootcamp-v1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: bootcamp
  strategy: {}
  template:
    metadata:
      annotations:
        sidecar.istio.io/status: '{"version":"3710def1fd6b417282ccad9d233d1d103bf78de57c31d10d2d7f3555934a4629","initContainers":["istio-init","enable-core-dump"],"containers":["istio-proxy"],"volumes":["istio-envoy","istio-certs"]}'
      creationTimestamp: null
      labels:
        app: bootcamp
        version: v1
    spec:
      containers:
      - image: docker.io/jocatalin/kubernetes-bootcamp:v1
        name: bootcamp
        ports:
        - containerPort: 8080
        resources: {}
      - args:
        - proxy
        - sidecar
        - --configPath
        - /etc/istio/proxy
        - --binaryPath
        - /usr/local/bin/envoy
        - --serviceCluster
        - bootcamp
        - --drainDuration
        - 45s
        - --parentShutdownDuration
        - 1m0s
        - --discoveryAddress
        - istio-pilot.istio-system:15003
        - --discoveryRefreshDelay
        - 1s
        - --zipkinAddress
        - zipkin.istio-system:9411
        - --connectTimeout
        - 10s
        - --statsdUdpAddress
        - istio-mixer.istio-system:9125
        - --proxyAdminPort
        - "15000"
        - --controlPlaneAuthPolicy
        - NONE
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: INSTANCE_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        image: docker.io/istio/proxy_debug:0.6.0
        imagePullPolicy: IfNotPresent
        name: istio-proxy
        resources: {}
        securityContext:
          privileged: true
          readOnlyRootFilesystem: false
          runAsUser: 1337
        volumeMounts:
        - mountPath: /etc/istio/proxy
          name: istio-envoy
        - mountPath: /etc/certs/
          name: istio-certs
          readOnly: true
      initContainers:
      - args:
        - -p
        - "15001"
        - -u
        - "1337"
        image: docker.io/istio/proxy_init:0.6.0
        imagePullPolicy: IfNotPresent
        name: istio-init
        resources: {}
        securityContext:
          capabilities:
            add:
            - NET_ADMIN
          privileged: true
      - args:
        - -c
        - sysctl -w kernel.core_pattern=/etc/istio/proxy/core.%e.%p.%t && ulimit -c
          unlimited
        command:
        - /bin/sh
        image: alpine
        imagePullPolicy: IfNotPresent
        name: enable-core-dump
        resources: {}
        securityContext:
          privileged: true
      volumes:
      - emptyDir:
          medium: Memory
        name: istio-envoy
      - name: istio-certs
        secret:
          optional: true
          secretName: istio.default
status: {}
```
いくつかポイントを上げると、以下の3点があります。

- Envoyプロキシが利用するための環境変数がPodに追加されている
- istio-init, enable-core-dumpという名前のinitContainerが追加されている
- istio-proxyという名前のコンテナが追加されている

initContainerは、Podのデプロイ時に前処理を行うために利用するコンテナです（これはKubernetesの機能です）。

それでは、このmanifestを使って、bootcampアプリケーションをデプロイします。

    $ kubectl create -f ./manifests/bootcamp-istio.yaml
    deployment "bootcamp-v1" created
    deployment "bootcamp-v2" created
    

### 3.2. トラフィックの流量制御を行う
それでは、実際にトラフィックの流量制御を行います。

まずは、全てのトラフィックをv1に振り分けてみます。トラフィックのルールはIstio用のmanifestファイルとして記述することができます。実際のmanifestファイルの内容は以下のようになっています。

    $ cat ./manifests/bootcamp-route-rule-all-v1.yaml

```yaml
apiVersion: config.istio.io/v1alpha2
kind: RouteRule
metadata:
  name: bootcamp-default
spec:
  destination:
    name: bootcamp
  precedence: 1
  route:
  - labels:
      version: v1
    weight: 100
```

``{.spec.route}``に具体的なルーティングのルールを記述しています。この場合、"bootcamp"というServiceがルーティング対象とするPodで、"version:v1"というlabelが設定されているものに、全てのトラフィックを送るようにしています。

このルールを適用するには、以下のコマンドを実行します。

    $ istioctl create -n istio-bootcamp -f ./manifests/bootcamp-route-rule-all-v1.yaml
    Created config route-rule/istio-bootcamp/bootcamp-default at revision 1760

改めて、アプリケーションにリクエストを送信します。繰り返し実行してみて、全ての応答がv1から返却されることを確認してください。

##### Linux/Mac

    $ curl http://$GATEWAY_URL/bootcamp

##### Windows

    $ Invoke-RestMethod -Uri "http://${GATEWAY_URL}/bootcamp"

次に、v1,v2にトラフィックを均等に送るようにルールを変更します。
まずは、新たにデプロイするルールのmanifestファイルを参照してみます。

    $ cat ./manifests/bootcamp-route-rule-50-v2.yaml

```yaml
apiVersion: config.istio.io/v1alpha2
kind: RouteRule
metadata:
  name: bootcamp-default
spec:
  destination:
    name: bootcamp
  precedence: 1
  route:
  - labels:
      version: v1
    weight: 50
  - labels:
      version: v2
    weight: 50
```

このmanifestでは、v1,v2それぞれに"weight: 50"を記述することで、それぞれのバージョンに均等にトラフィックが送信されるようにしています。

それでは、このルールをデプロイします（既存のルールを削除した上で、新しいルールをデプロイします）。

    $ istioctl delete -n istio-bootcamp -f ./manifests/bootcamp-route-rule-all-v1.yaml
    Deleted config: route-rule/istio-bootcamp/bootcamp-default
    $ istioctl create -n istio-bootcamp -f ./manifests/bootcamp-route-rule-50-v2.yaml
    Created config route-rule/istio-bootcamp/bootcamp-default at revision 1835

__参考）__<br>
本来であれば、以下のような``istioctl replace``コマンドをを使ってルールの置き換えを行えるはずです。

    $ istioctl replace -f ./manifests/bootcamp/bootcamp-route-rule-50-v2.yaml

しかしながら、今回利用しているバージョンのIstioには、[これが動作しない不具合](https://github.com/istio/istio/issues/4215)があるようです。

再度リクエストの送信を繰り返して、v1とv2からの応答がおおよそ同じ割合で返ってくることを確認してください。

最後に、v2に全てのトラフィックをルーティングするように、ルールを変更します。

    $ istioctl delete -n istio-bootcamp -f ./manifests/bootcamp-route-rule-50-v2.yaml
    Deleted config: route-rule/istio-bootcamp/bootcamp-default
    $ istioctl create -n istio-bootcamp -f ./manifests/bootcamp-route-rule-all-v2.yaml
    Created config route-rule/istio-bootcamp/bootcamp-default at revision 2145

今度は、応答が全てv2からのものになっていることを確認してください。


4 . クリーンアップ
------------------
お疲れ様でした。以上でハンズオンのPart2は終了です。

これまで作ってきたオブジェクトをクリーンアップしたい場合は、以下のコマンドを実行して、作成したオブジェクトを削除してください。

    $ istioctl delete -f ./manifests/bootcamp-route-rule-all-v2.yaml
    $ kubectl delete namespace istio-bootcamp


5 . Istioの公式チュートリアルのご紹介
-------------------------------------
Isitoの公式ドキュメントには、bookinfoというもう少し本格的なアプリケーションを利用して、Isitoの様々な機能を試すことができるチュートリアルが用意されています。ご興味ある方はぜひ触ってみてください。

- [トラフィックの管理](https://istio.io/docs/tasks/traffic-management/)
- [ポリシーの適用](https://istio.io/docs/tasks/policy-enforcement/)
- [メトリック、ログ、トレース情報の収集](https://istio.io/docs/tasks/telemetry/)
- [セキュリティ](https://istio.io/docs/tasks/security/)

以上。
