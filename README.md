cndjp4 ハンズオンチュートリアル
===============================
このハンズオンは、Part 1,2の二部構成です。以下のリンクからそれぞれのチュートリアルに進んでください。

- [Part 1](https://github.com/oracle-japan/cndjp4/blob/master/docs/handson1.md)
- [Part 2](https://github.com/oracle-japan/cndjp4/blob/master/docs/handson2.md)


Windows PCでのminikubeのインストール手順
----------------------------------------
Windows PCでローカルクラスターを立てる手段としては、minikubeをご利用ください。<br>

ここでは、chocolatey（Windowsのパッケージマネージャ）を使ったインストール手順を記します。

### chocolateyのインストール
管理者権限でWindows Power Shellを起動し、以下のコマンドを実行してください。

    > Set-ExecutionPolicy Bypass -Scope Process -Force; iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))

プロンプトに表示されるメッセージに従ってインストールを進めます。

以降はchocolateyを使って順次必要なものをインストールしていきます。ご自身の環境にインストールされていないものについて、以下の手順を実施してください。

### Virtual Boxのインストール

    > choco install virtualbox

### kubectlのインストール
kubectlをインストールするには、以下のコマンドを実行します。

    > choco install kubernetes-cli

### minikubeのインストール
minikubeをインストールするには、以下のコマンドを実行します。

    > choco install minikube

以下のコマンドでminikubeがインストールされていることw確認してください。

    > minikube version
    minikube version: v0.25.x


### minikubeクラスターの起動と停止
minikubeクラスターの起動/停止/削除をするには、以下のコマンドを実行します。

##### 起動

    > minikube start

##### 停止

    > minikube stop

##### 削除

    > minikube delete

以上。
