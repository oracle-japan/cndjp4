handson part2
-------------
Istioのバイナリのダウンロード

Mac/Linux

    > curl -L https://git.io/getLatestIstio | sh -
    > unzip 

Windows

    > [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
    > Invoke-WebRequest -Uri https://github.com/istio/istio/releases/download/0.6.0/istio_0.6.0_win.zip -OutFile istio_0.6.0_win.zip
    > Expand-Archive -Path .\istio_0.6.0_win.zip

PATHを通す
    
    > cd istio-0.6
    > export PATH=$PWD/bin:$PATH

    > istioctl version
    Version: 0.6.0
    GitRevision: 2cb09cdf040a8573330a127947b11e5082619895
    User: root@a28f609ab931
    Hub: docker.io/istio
    GolangVersion: go1.9
    BuildStatus: Clean

クラスターを起動

Mac/Linux
    > AUTHORIZATION_MODE=RBAC vagrant up

Windows

    > set-item env:AUTHORIZATION_MODE -value RBAC
    > vagrant up

インストール

    $ cd ./istio_0.6.0_win/istio-0.6.0/

    > kubectl apply -f install/kubernetes/istio.yaml

確認

    > kubectl get svc -n istio-system
    NAME            TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                                                            AGE
    istio-ingress   LoadBalancer   10.100.151.26    <pending>     80:31826/TCP,443:31716/TCP                                         1m
    istio-mixer     ClusterIP      10.100.149.13    <none>        9091/TCP,15004/TCP,9093/TCP,9094/TCP,9102/TCP,9125/UDP,42422/TCP   1m
    istio-pilot     ClusterIP      10.100.118.141   <none>        15003/TCP,8080/TCP,9093/TCP,443/TCP                                1m

    > kubectl get pods -n istio-system
    NAME                             READY     STATUS    RESTARTS   AGE
    istio-ca-418645489-gg8sh         1/1       Running   0          2m
    istio-ingress-1688831668-t24gh   1/1       Running   0          2m
    istio-mixer-3303323913-s8c2l     3/3       Running   0          2m
    istio-pilot-3625647026-0rv6z     2/2       Running   0          2m



---

### Namespaceを作成する
最初に、Part2の作業を通して利用するNamespaceとして"istio-bootcamp"を作成しておきます。

    $ kubectl create namespace istio-bootcamp

kubectlのデフォルトnamespaceを"istio-bootcamp"に変更しておきます。

・Mac/Linux

    $ kubectl config set-context $(kubectl config current-context) --namespace=istio-bootcamp

・Windows

    $ $CURRENT_CONTEXT=kubectl config current-context
    $ kubectl config set-context $CURRENT_CONTEXT --namespace=istio-bootcamp

### bootcamp アプリケーションのデプロイ

    $ cd ./istio-bootcamp/manifests

    $ kubectl create -f ./bootcamp.yaml
    $ kubectl create -f ./bootcamp-service.yaml
    $ kubectl create -f ./bootcamp-ingress.yaml

    $ kubectl describe deployments bootcamp-v1

アクセスするためのURLを確認する

Linux/Mac

    > export GATEWAY_URL=$(kubectl get po -l istio=ingress -n istio-system -o 'jsonpath={.items[0].status.hostIP}'):$(kubectl get svc istio-ingress -n istio-system -o 'jsonpath={.spec.ports[0].nodePort}')

Windows

    > $GATEWAY_HOSTIP=kubectl get po -l istio=ingress -n istio-system -o 'jsonpath={.items[0].status.hostIP}'
    > $GATEWAY_NODEPORT=kubectl get svc istio-ingress -n istio-system -o 'jsonpath={.spec.ports[0].nodePort}'
    > $GATEWAY_URL="$GATEWAY_HOSTIP" + ":" + "$GATEWAY_NODEPORT"

curlを実行

    $ curl http://$GATEWAY_URL/bootcamp

    $ Invoke-RestMethod -Uri "http://${GATEWAY_URL}/bootcamp"




### Envoyを注入したPodに置き換える

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



---
Bookinfoアプリケーションのインストール

namespaceの作成とistio-injectionを有効化

    > kubectl create namespace bookinfo
    > kubectl label namespace bookinfo istio-injection=enabled

Mac/Linux

    > kubectl config set-context $(kubectl config current-context) --namespace=bootcamp

Windows
    > kubectl config current-context
    default-cluster
    > kubectl config set-context default-cluster --namespace=bootcamp
    Context "default-cluster" modified.

bookinfoをインストール

    > kubectl apply -f samples/bookinfo/kube/bookinfo.yaml
    service "details" created
    deployment "details-v1" created
    service "ratings" created
    deployment "ratings-v1" created
    service "reviews" created
    deployment "reviews-v1" created
    deployment "reviews-v2" created
    deployment "reviews-v3" created
    service "productpage" created
    deployment "productpage-v1" created
    ingress "gateway" created

    > kubectl get services
    NAME          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
    details       ClusterIP   10.100.36.203    <none>        9080/TCP   43s
    kubernetes    ClusterIP   10.100.0.1       <none>        443/TCP    1h
    productpage   ClusterIP   10.100.125.154   <none>        9080/TCP   40s
    ratings       ClusterIP   10.100.76.84     <none>        9080/TCP   43s
    reviews       ClusterIP   10.100.64.118    <none>        9080/TCP   42s

    > kubectl get pods
    details-v1-4276665575-bxhvj       1/1       Running   0          3m
    productpage-v1-1223458169-z9sz8   1/1       Running   0          3m
    ratings-v1-1635894359-ltzvr       1/1       Running   0          3m
    reviews-v1-2901386904-dgdkd       1/1       Running   0          3m
    reviews-v2-3191357612-70tl0       1/1       Running   0          3m
    reviews-v3-1009570596-pt69z       1/1       Running   0          3m

