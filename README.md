# このチュートリアルについて
AWS初心者がシンプルなJava（Spring Boot）のアプリケーションをEC2にデプロイするためのチュートリアルです。  

AWSコンソールの手順はこのチュートリアルの内容が陳腐化しないようなるだけ公式のものを採用しています。  

最初は読みづらいかもしれませんが、慣れていかなければいけないものなので頑張って読みましょう。  

たとえば、`IAM 作成`というキーワードで調べてもう少し分かりやすい記事と公式ドキュメントを並行して読みすすめるのもよいです。  

# ローカルでの起動手順

ローカルPCにJavaがインストールされていなければ起動できませんが、その場合はローカルでの起動はスキップしても大丈夫です。

このリポジトリをクローンした上で下記コマンドを実行してアプリケーションを起動します。

```sh
$ ./gradlew bootRun
```

下記を実行し、正常にレスポンスが得られることを確認します。  

```sh
% curl -i "http://localhost:8080/hello"
HTTP/1.1 200
Content-Type: application/json
Transfer-Encoding: chunked
Date: Sun, 18 Dec 2022 04:29:23 GMT

{"message":"hello world"}
```

```sh
% curl -i "http://localhost:8080/hello?name=yamada"
HTTP/1.1 200
Content-Type: application/json
Transfer-Encoding: chunked
Date: Sun, 18 Dec 2022 04:29:47 GMT

{"message":"hello yamada"}
```

アプリケーションを停止してください。

# AWSの準備

アカウントは作成済みの前提で進めます。  

# IAMユーザーの作成

IAMユーザーを作成してください。  
こちらの手順を参考にしましょう。  

https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/id_users_create.html#id_users_create_console  
> IAM ユーザーの作成（コンソール）
- IAMユーザー名は任意で決めてOKですが、ご自身の名前の小文字半角アルファベットでよいかと思います
- ポリシーはなんでもできるAdministratorAccessでOKです
- タグの設定は任意に設定してください
- CLIでの操作はしないのでアクセスキーやシークレットアクセスキーは不要です

MFAも設定してください。  
MFAについてはこちらを参考にしてください。  
https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/id_credentials_mfa.html  
> 仮想 MFA デバイス

MFAの割当はこちらの手順を参考にしましょう。  
https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/id_credentials_mfa_enable_virtual.html
> IAM ユーザーの仮想 MFA デバイスの有効化 (コンソール)

筆者はGoogle Authenticatorを使っています。  
Android: https://play.google.com/store/apps/details?id=com.google.android.apps.authenticator2&hl=ja&gl=US&pli=1  
iOS: https://apps.apple.com/jp/app/google-authenticator/id388497605  

ここまででIAMユーザーの作成ができるはずなので、一度サインアウトして作成したIAMユーザーでサインインしてください。  

# 環境の作成

今回作る環境のイメージはこんな感じです。  

<img width="500" alt="diagram" src="https://user-images.githubusercontent.com/62045457/207836925-db1dba99-e52d-46ad-8b33-c67f8433d0d2.png">

やることは下記のとおりです。  

- VPCの作成
  - Internet Gatewayやルートテーブル、サブネットもまとめて作成します
- EC2の作成
  - 赤点線枠のセキュリティグループも作成します

# VPCの作成

まずはRegionを東京にしましょう。  

<img width="500" alt="スクリーンショット 2022-12-15 19 15 06" src="https://user-images.githubusercontent.com/62045457/207833197-fe06cb1d-1481-4e1f-bbe7-c30f96522113.png">

VPCの作成はこちらを参考にしてください。  
https://docs.aws.amazon.com/ja_jp/vpc/latest/userguide/working-with-vpcs.html#create-vpc-and-other-resources  

VPCを作成するときには「VPCなど」を選びましょう。

 <img width="500" alt="スクリーンショット 2022-12-10 12 41 22" src="https://user-images.githubusercontent.com/62045457/206827408-dae9ca1f-1cee-4a84-88c5-e535475970f4.png">

それから、VPCの名前タグはmy-tutorial-vpcなどわかりやすい命名にしましょう

ほかは基本的に何も変更しなくてもいいですが、それぞれの設定値などできる範囲で調べておくのがいいでしょう。  

VPCを作成すると下記のように成功表示がでればOKです。  

<img width="500" alt="スクリーンショット 2022-12-15 19 30 08" src="https://user-images.githubusercontent.com/62045457/207836376-44c23da9-35ab-4b6a-8e9e-5b2895d29dd3.png">

この時点で下記のようなネットワーク構成が作成されます。  

<img width="500" alt="diagram" src="https://user-images.githubusercontent.com/62045457/207837351-e3725f5c-a179-46ed-8633-cba9529d8cd1.png">

# EC2の作成

EC2を作成したVPCのサブネット内に配置しましょう。  

手順はこちらを参考にしてください。  
https://docs.aws.amazon.com/ja_jp/AWSEC2/latest/UserGuide/EC2_GetStarted.html  

ただし、作成する際に以下の点を気をつけてください。  

## インスタンスの名前
名前はmy-tutorial-ec2などわかりやすく命名してください。  


## キーペアの作成と置き場所
ローカルからサーバーにアクセスするため、キーペアを作成してダウンロードしてください。  
`自分で決めたキーペア名.pem`という名前でダウンロードできるはずです。  

pemファイルの置き場所は自分で決めればよいですが、ダウンロード下に置いておくのは望ましくないです。  
たとえば、`~/.ssh`というディレクトリ直下に配置しておくのが一般的です。  

`~（チルダ）`が何を表すのかについてもぜひ調べるとよいです。
`チルダ　意味 ディレクトリ`や`チルダ パス コマンド`で調べるとよいです。  

ダウンロード下から`~/.ssh`への移動はmvコマンドで実施してみましょう。  

## ネットワーク設定

ネットワーク設定はデフォルトのままにしないでください。  
このままではAWSアカウント作成時に自動で提供されているデフォルトVPCが利用されますので、編集をします。  
<img width="500" alt="スクリーンショット 2022-12-15 19 45 10" src="https://user-images.githubusercontent.com/62045457/207839534-c5c5ffe4-5038-4f74-ba68-12d14f9145dd.png">

デフォルトVPCについてはこちらを参考にしてください。  
https://docs.aws.amazon.com/ja_jp/vpc/latest/userguide/default-vpc.html  

VPCは前の手順で作成したものを選びましょう。  
サブネットは「public」と記載されているものを選んでください。  

aとcの2種類ありますが、どちらでも大きな問題にはならないですが、環境のイメージ図にあわせるのであればaを選びましょう。    

<img width="500" alt="スクリーンショット 2022-12-15 19 47 46" src="https://user-images.githubusercontent.com/62045457/207840130-266858fe-87d3-4f78-a242-9c14d6b86c01.png">

<img width="500" alt="スクリーンショット 2022-12-15 19 50 21" src="https://user-images.githubusercontent.com/62045457/207840638-94be38fe-568c-4560-91e5-5e1e71c09f02.png">

セキュリティグループについてですが、`8080`ポートへのアクセスを許可するためにインバウンドルールを独自に設定しましょう。  
<img width="500" alt="スクリーンショット 2022-12-15 19 52 54" src="https://user-images.githubusercontent.com/62045457/207841249-d23f709a-6e8e-4e4f-89a1-81ef2aaf6f21.png">

<img width="500" alt="スクリーンショット 2022-12-15 19 55 54" src="https://user-images.githubusercontent.com/62045457/207841758-025ed9b0-02ea-4a61-a215-50484375057e.png">

# EC2への接続

起動したEC2インスタンスへの接続をしましょう。  

接続方法は下記を参考にしてください。  
接続するにはsshコマンド、chmodコマンドの理解が必要です。  

`EC2 インスタンス 接続`や`sshとは`、`chmodとは`で調べてそれぞれ学習するのをオススメします。  

https://docs.aws.amazon.com/ja_jp/AWSEC2/latest/UserGuide/AccessingInstancesLinux.html
> https://docs.aws.amazon.com/ja_jp/AWSEC2/latest/UserGuide/AccessingInstancesLinux.html#AccessingInstancesLinuxSSHClient

また、sshで任意の場所からのアクセスを許可しているのであればEC2 Instannce Connectを使って接続してしまってもよいです。  
<img width="500" alt="スクリーンショット 2022-12-15 19 59 39" src="https://user-images.githubusercontent.com/62045457/207842633-b05d35ba-b3d4-4ae7-a99c-6992455f67f9.png">

<img width="500" alt="スクリーンショット 2022-12-15 20 00 28" src="https://user-images.githubusercontent.com/62045457/207842755-b08c4045-8a65-403f-9940-1ab2f7e22ea8.png">

このようにEC2のアスキーアートが表示されれば接続できています。  
<img width="500" alt="スクリーンショット 2022-12-15 20 01 10" src="https://user-images.githubusercontent.com/62045457/207843244-a698f636-6fa2-4b4f-bc6f-ac4065edc1f8.png">

# EC2へGitとJavaをインストールする

EC2インスタンスへssh接続した状態で下記を実行してください。  

現在のパスを確認しましょう。  
```sh
$ pwd
/home/ec2-user
```

`/home/ec2-user`下で作業しますが、自身が利用しているサーバーのディレクトリ構成やそれぞれの役割も調べておくとよいです。  

とくにファイルやディレクトリがないことを確認しましょう。  

```sh
$ ls
```

Gitが利用可能でないことを確認してください。  
```sh
$ git version
-bash: git: command not found
```

では、Gitのインストールをします。

まずは、下記コマンドを実行してください。

```sh
$ sudo yum update
``` 

コマンド実行中に`Is this ok [y/d/N]:`と確認が求められます。  

`y`を入力してエンターキーを押すとupdateが実行されます。  

```sh
Complete!  
```

と表示されることを確認してください。  

sudo、yumがそれぞれどういうものかについてはこのタイミングで把握しておきましょう。

参考  
sudoについて  
https://linuxjm.osdn.jp/html/sudo/man8/sudo.8.html  
https://en.wikipedia.org/wiki/Sudo  

yumについて  
https://man7.org/linux/man-pages/man8/yum.8.html  
yumを調べるにあたってパッケージマネージャとはなにか？も理解しておく必要があります。  
https://www.debian.org/doc/manuals/aptitude/pr01s02.en.html  
https://en.wikipedia.org/wiki/Package_manager  

次に、Gitをインストールしてください。  

```sh
$ sudo yum install git
```

同じく、`Is this ok [y/d/N]:`に対しては`y`を入力してエンターキーを押してください。  

```sh
Complete!  
```

と表示されることを確認してください。  

Gitが利用可能になっていることを確認してください。  

```sh
$ git version
git version 2.38.1
```

次にJavaのインストールをします。  

Javaが利用可能でないことを確認します。  

```sh
$ java
-bash: java: command not found
```

Java（JDK17）をインストールします。  
```sh
$ sudo yum install java-17-amazon-corretto-devel
```

同じく、`Is this ok [y/d/N]:`に対しては`y`を入力してエンターキーを押してください。  

下記のように表示されたら問題ないです。  

```sh
Complete!  
```

Javaが利用可能になっていることを確認してください。
※記事作成時点ではJava17が最新のLTS版ですが、このチュートリアル実施時点ではより新しいバージョンがリリースされている可能性があります。  
チュートリアルに使うアプリケーションはJava17を前提にしているのでもし、うまく起動ができなければJava17をインストールするか、起動するアプリケーションのJavaのバージョンを最新のLTS版に変更してください。  
```sh
$ java --version
openjdk 17.0.10 2024-01-16 LTS
OpenJDK Runtime Environment Corretto-17.0.10.8.1 (build 17.0.10+8-LTS)
OpenJDK 64-Bit Server VM Corretto-17.0.10.8.1 (build 17.0.10+8-LTS, mixed mode, sharing)
```

# アプリケーションを起動してアクセスする

EC2上でこのリポジトリをcloneしましょう。

sshではなくhttps通信を利用してcloneします。  
```sh
$ git clone https://github.com/raisetech-for-student/tutorial-aws-deploy-simple-java-app-to-ec2.git
```

cloneが正常に完了するとディレクトリが作成されているはずです。
```sh
$ ls
tutorial-aws-deploy-simple-java-app-to-ec2
```

ディレクトリ内に移動してアプリケーションを起動します。  
```sh
$ cd tutorial-aws-deploy-simple-java-app-to-ec2/
```

```sh
$ ./gradlew bootRun
```

インスタンスのスペックが低いのでローカルよりも起動に時間を要します。  

数分〜十数分待つと起動できるはずです。  

```sh
> Task :bootRun

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v3.0.0)

2022-12-18T04:55:36.197Z  INFO 20967 --- [           main] c.e.helloworld.HelloworldApplication     : Starting HelloworldApplication using Java 17.0.5 with PID 20967 (/home/ec2-user/tutorial-aws-deploy-simple-java-app-to-ec2/build/classes/java/main started by ec2-user in /home/ec2-user/tutorial-aws-deploy-simple-java-app-to-ec2)
2022-12-18T04:55:36.202Z  INFO 20967 --- [           main] c.e.helloworld.HelloworldApplication     : No active profile set, falling back to 1 default profile: "default"
2022-12-18T04:55:37.324Z  INFO 20967 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8080 (http)
2022-12-18T04:55:37.340Z  INFO 20967 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2022-12-18T04:55:37.342Z  INFO 20967 --- [           main] o.apache.catalina.core.StandardEngine    : Starting Servlet engine: [Apache Tomcat/10.1.1]
2022-12-18T04:55:37.597Z  INFO 20967 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2022-12-18T04:55:37.602Z  INFO 20967 --- [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 1329 ms
2022-12-18T04:55:38.279Z  INFO 20967 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
2022-12-18T04:55:38.296Z  INFO 20967 --- [           main] c.e.helloworld.HelloworldApplication     : Started HelloworldApplication in 2.648 seconds (process running for 3.149)
```

起動ができたらアクセスをしてみましょう。  

AWSコンソールのEC2 > インスタンスから、今回作成したインスタンスのインスタンスIDをクリックします。  

パブリックIPv4アドレスをコピーしましょう。  

たとえば、IPアドレスが`52.197.88.4`だとします。  

下記のようなリクエストを送信して、レスポンスが得られることを確認してください。  

```sh
% curl -i "http://52.197.88.4:8080/hello"
HTTP/1.1 200
Content-Type: application/json
Transfer-Encoding: chunked
Date: Sun, 18 Dec 2022 05:02:21 GMT

{"message":"hello world"}
```

```sh
% curl -i "http://52.197.88.4:8080/hello?name=yamada"
HTTP/1.1 200
Content-Type: application/json
Transfer-Encoding: chunked
Date: Sun, 18 Dec 2022 05:02:50 GMT

{"message":"hello yamada"}
```

アプリケーションが起動し、インターネットを経由してアクセスできています。  

# 後片付け

ssh接続した状態で`ctrol + c`を押下してアプリケーションを停止しましょう。  

AWSコンソールからEC2 > インスタンスに遷移し、今回作成したインスタンスを終了してください。  

<img width="500" alt="スクリーンショット 2022-12-18 14 04 46" src="https://user-images.githubusercontent.com/62045457/208282386-bb45c176-03e4-4c2c-a962-a5859ff463f8.png">

終了後、しばらくまつとインスタンスが終了できます。  
インスタンスの終了が完了したら、VPCの削除をします。  

https://ap-northeast-1.console.aws.amazon.com/vpc/home?region=ap-northeast-1#vpcs:にアクセスして、今回作成したVPCを選択して削除してください。  

<img width="1650" alt="スクリーンショット 2022-12-18 14 10 49" src="https://user-images.githubusercontent.com/62045457/208282521-7a6e6144-226a-4354-9a18-5b84a8aefc31.png">
