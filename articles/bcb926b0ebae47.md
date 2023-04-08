---
title: "AWS、React、GolangによるWebアプリ開発~ローカル開発環境構築編~"
emoji: "🌎"
type: "tech"
topics:
  - "aws"
  - "docker"
  - "go"
  - "react"
  - "ecr"
published: true
published_at: "2022-12-29 19:41"
---

こんにちは、@nerusanです。

みなさんは、Dockerを利用したことはありますか？
いまや、コンテナ技術であるDockerはWeb開発には欠かせない技術になっているのではないでしょうか。
本番運用だけでなく、本番環境と同等の環境をローカル上で簡単に構築できます。なので、開発環境では動いていたのに本番では動かないって言う不具合が減ります。
また、試したい技術（MySQL, Golangなど）があれば、ローカル環境に直接インストールすることなくすぐに簡単に試せすこともでき、不要になったらすぐに削除できます。何よりチーム開発におけて、メンバー間のローカル環境差異による環境構築問題を大幅に改善することが魅力的です。

また、本番運用に欠かせないのが、AWSなどのクラウドです。
例えば、AWS Fargateはコンテナ向けサーバーレスオンピューティングエンジンです。コンテナが実際に動作する環境です。
Amazon ECSは、フルマネージドなコンテナオーケストレータです。つまり、コンテナを管理します。
ECSとFargateはセットで使われ、コンテナ環境を本番運用できます。

今回はコンテナ環境で運用できるECS/Fargateを利用する機会があったので、学んだことを紹介しようと思います！

はじめは一つの記事にローカル環境構築と、本番環境デプロイについて記述しようとしましたが、
記事が長くなりすぎるので、分割することにしました。
本記事では主にローカル開発環境構築について述べます。

:::message
本記事では、詳細な構築方法を述べるのではなく、要所要所で要点を述べていけたらと思います。
なので、この記事を参考に環境構築を実装される際は、適宜調べていただく必要があると思うのでご参考までに。
:::


# 今回作るもの

今回はユーザ認証機能を備えたWebアプリケーションを作ることを想定したアーキテクチャを考えます。
フロントはReact、バックエンドはGoを利用したREST　APIによるアプリケーションです。
例えば、以下のようなことができることを想定します。

1. ユーザ名、メールアドレス、パスワードをもとにメールアドレス確認を伴ったサインアップ機能
	* ユーザ名、メールアドレス、パスワードを入力し、仮登録を行う
	* 指定メールアドレスに確認コードを送る
	* 確認コードを入力してもらい正しければ、本登録を行う
	* 保存先はRDB(MySQL)を利用する
2. JWT(JSON Web Token)によるユーザ認証
	* JWTだけでは、任意のタイミングでログアウトができないためミドルウェアとしてキャッシュサーバであるRedisを利用する
	* サインイン機能、サインアウト機能をする
3. サインインユーザのみユーザ一覧を表示
	* MySQLに保存されたユーザを返す
	* サインインユーザ以外は見れない



JWT(JSON Web Token)のによるユーザ認証はまた別の記事にできたらと思います！
https://jwt.io/
# 全体図
## 本番環境
全体の構造としては、バックエンドはAWS ECS/Fargateを利用、フロントエンドは、AWS Amplifyによる構築を行いました。

全体の図は以下のようになります。

![](https://storage.googleapis.com/zenn-user-upload/c9c5ee18d7af-20221227.png)

ポイントとしては、以下の8点です。
1. コンテナ技術にDockerを利用
2. ユーザ認証を行うため、キャッシュサーバーである AWS ElastiCache for Redisを利用する
3. バックエンドは、オーケストレーションにAWS ECS(Elastic Container Service)、コンテナ用サーバーにAWS Fargateを利用することで、可用性を高める
4. メール送信はAmazon　SES(Simple Email Service)を利用
5. DBにはAWS Aurora(MySQL)を利用
6. 管理用サーバーを構築し、そこでのみ本番DBをアクセスを許可することで、セキュリティを高めるかつ、作業者環境差異による不具合をなくす
7. GitHub Actionでイメージ作成し、ECRでイメージを管理する
8. フロントは、AWS Amplifyを利用し、最速デプロイ環境構築を行う

これらのローカルでの開発環境はdocker composeを利用します！
## 開発環境

Swaggerを利用したスキーマ駆動開発を前提とします。
流れとしては以下です。
1. バックエンド側が必要なAPIの洗い出しとインターフェース（OpenAPIの定義）の叩きを作成
2. フロントエンド側は作成したインターフェースをレビュー
3. レビューが通れば、マージし、モックの生成
4. フロント側はSwaggerおよびモックもとに実装、バック側はSwaggerをもとに実装

スキーマ駆動開発を行うことで、ドキュメントを残すこともできる上、一方が使いにくいAPIになりにくいし、同時に作業を進めることが可能になります。
結果的にアプリケーション全体としての品質が高くなります。

# 環境構築用ファイルの作成

ここから実際にインフラ構築のための`Dockerfile`、`docker-compose.yml`の作成を実施します。

## バックエンド
まずは、バックエンド側からです。
`docker-compose.yml`、`Dockerfile`を作成していきます。

### Dockerfile
今回は、バックエンド側は、golangを利用します。以下`Dockerfile`を記述します。

```Dockerfile:Dockerfile
# ビルド用環境
# ----------------------------------------------
FROM golang:1.19.2-bullseye AS deploy-builder

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN go build -trimpath -ldflags "-w -s" -o app

# 本番環境
# ----------------------------------------------
FROM debian:bullseye-slim AS deploy

# X509: Certificate Signed by Unknown Authorityエラーを回避する
RUN apt-get update \
 && apt-get install -y --force-yes --no-install-recommends apt-transport-https curl ca-certificates \
 && apt-get clean \
 && apt-get autoremove \
 && rm -rf /var/lib/apt/lists/*

COPY --from=deploy-builder /app/app .

CMD ["./app"]


# 開発環境
# ----------------------------------------------
FROM golang:1.19.2-alpine3.16 AS dev

WORKDIR /app

RUN apk update && apk add alpine-sdk && apk add jq

RUN go install github.com/cosmtrek/air@latest 
RUN go install github.com/k0kubun/sqldef/cmd/mysqldef@latest
```

ポイントとして、２点あります。

１点目は、**マルチステージビルド**利用によるDockerfileの構築です。
マルチステージビルドを利用することで、イメージのスリム化、Dockerfileの削減につながります。詳しくは、使い方メリットは以下のURLをご参考ください。

https://matsuand.github.io/docs.docker.jp.onthefly/develop/develop-images/multistage-build/

上述のDockerfileでは、ビルド環境、本番運用環境、ローカルでの開発環境の3つの環境を1つのDockerfileに記述しています。

それぞれの環境の選択は、`--target`オプションをつけることで可能になります。

```sh
# deploy環境でビルド
$ docker build --no-cache --target deploy ./
```

2点目は、本番環境運用イメージにはalpine linuxを利用していない点です。以下の記事にもあるように、軽量化されて良いとされておりましたが、パフォーマンス面で劣ったりする可能性があります。

https://blog.inductor.me/entry/alpine-not-recommended

なので、ベースイメージとして`gcr.io/distroless/base-debian10`, `debian:bullseye-slim`が選択肢に上がるかなって思っており、今回は`debian:bullseye-slim`を採用しました。

まず、golangがインストールされているbullseye環境でgoファイルをビルドし、実行ファイルを作成します。次にgolangが入っていないbullseye環境で先ほど作成した実行ファイルを配置し、実行します。
こうすると、本番環境には、golangが入っていないので、軽量になりますね！

開発環境はローカルで動かすため、軽量であるalpine linuxで行います。
できれば、環境を揃えるためbullseyeの方がいいかもしれませんが、現状動作に問題がないため、alpineで開発するとします。

また、開発効率を上げるためairというモジュールを導入しています。これは、ライブリロードを行ってくれるライブラリです。つまり、変更のたびに、サーバーを起動し直さなくて良くなります。フロントエンド開発では、これが当たり前になってきているので、これがないと不変だと思い、導入してます。

https://github.com/cosmtrek/air

また、マイグレーションツールにはsqldefを利用しています。sqldef を利用すると、変更適用先 DB の現在の状況と新規作成 DDL 文の差分を sqldef が自動で判別し、`ALTER TABLE`文などの差分適用 DDL 文を生成/適用してくれます。差分適用 DDL 文を自動で生成してくれるので、新規作成用DDL分のみを管理するだけでいいので、かなり便利です。発行されるDDLが本当に適用しても大丈夫なのかを確認することが必要になりますが、小さいなアプリケーションであれば、全然使えると思うので、今回採用しました。

https://github.com/k0kubun/sqldef

### docker-compose.yml
次に、`docker-compose.yml`の作成です。`docker-compose.yml`では、ローカルでの開発環境構築のための記述をします。

```yaml:docker-compose.yml
services:
  app:
    container_name: app
    restart: always
    build:
      args:
        - target=dev # Dockerfileに対応する名前をしている
    volumes:
      - ./:/app
    environment:
      GO_ENV: development
      DB_HOST: db
      DB_PORT: 3306
      DB_USER: admin
      DB_PASSWORD: password
      DB_NAME: point_app
      REDIS_HOST: point-app-redis
      REDIS_PORT: 6379
      AWS_ENDPOINT: http://aws:4566
      AWS_ACCESS_KEY_ID: accesskey
      AWS_SECRET_KEY: secretkey
      AWS_REGION: ap-northeast-1
      SENDER_MAIL_ADDRESS: example@example.com
      FRONT_ENDPOINT: http://localhost:3000
    tty: true
    ports:
      - 8081:80
  doc:
    image: swaggerapi/swagger-ui:latest
    container_name: doc
    restart: always
    volumes:
      - ./docs/openapi.yml:/usr/share/nginx/html/openapi.yml
    environment:
      - URL=http://localhost/openapi.yml
      - SWAGGER_JSON=./docs/openapi.yml
    ports:
      - 80:8080
  db:
    image: mysql:8.0.31
    platform: linux/amd64
    container_name: point-app-db
    environment:
      MYSQL_ALLOW_EMPTY_PASSWORD: "yes"
      MYSQL_USER: admin
      MYSQL_PASSWORD: password
      MYSQL_DATABASE: point_app
    volumes:
      - point-app-db-data:/var/lib/mysql　# データはdocker上で管理
      - $PWD/_tools/mysql/conf.d:/etc/mysql/conf.d:cached
    ports:
      - "33306:3306"
  cache:
    image: "redis:latest"
    container_name: point-app-redis
    ports:
      - "36379:6379"
    volumes:
      - point-app-redis-data:/data
  panel:
    image: "adminer:latest"
    restart: always
    ports:
      - 8082:8080
  aws:
    image: localstack/localstack
    ports:
      - 4566:4566
      - 4510-4559:4510-4559
    tty: true
    environment:
      - SERVICES=ses
      - DEFAULT_REGION=ap-northeast-1
volumes: # dockerでデータを管理
  point-app-db-data:
  point-app-redis-data:
```
サービスが6つあります。
一つ一つ見ていきましょう。

#### appサービス

まず、`app`サービスです。
こちらは、Goの開発環境です。
イメージはDockerfileに記述されているステージ名`dev`を指定します。
そうすると、alpine linuxのgoの環境を作成することができます。

環境変数は他のサービスに接続のするため適宜指定してください。
注意点として、`docker-compose.yml`上で起動した他のコンテナに接続する際は、ホスト名はservice名または、コンテナ名にする必要があります。例えば、MySQLへの接続は`db`(`point-app-db`)、Redisへの接続は`cache`(`point-app-cache`)、AWSへの接続は`aws`がホスト名になります。

#### docサービス

次に`doc`サービスです。こちらは、swaggerの環境となります。RESTAPI開発においてswaggerを利用することで、開発をスムーズに進めることができます。エンドポイント、リクエスト、レスポンスをなどを記述できるOpenAPIを定義し、それをswaggerに読み込ませることで、APIのインターフェースを簡単に把握することができます。

![](https://storage.googleapis.com/zenn-user-upload/412300f9a458-20221227.png)

APIのインターフェース部分になりますので、swaggerを使えば、フロントエンド、バックエンドそれぞれ作業を平行に進めることができます。つまり、バックエンドの実装を待たなくてもフロントエンドは作り始めることができるのです。

また、openapiよりモックレスポンスを作ってくれるツールもあり、フロントエンド開発もかなり楽になります。（フロントエンド開発環境構築に記載）

#### dbサービス
次に、`db`サービスです。今回は、MySQLを利用するので、ベースイメージはMySQLの公式イメージを利用します。

https://hub.docker.com/_/mysql

ここでのポイントは、MySQLのデータは、dockerで管理することです。そうすると、コンテナを削除しても、保存データが消えないようになります。以下のように記述することで、docker上で管理できます。せっかくユーザのデータ作っていたのに、dbコンテナを削除するたびに消えて落ち込む経験は、何度もしたことがあるので、是非管理するようにしましょう！

```yaml:docker-compose.yml
  db:
    image: mysql:8.0.31
    # 略
    volumes:
      - point-app-db-data:/var/lib/mysql　　# docker上で管理したデータをコンテナに同期
# 略
# 管理
volumes:　 # dockerでデータを管理
  point-app-db-data:
```

#### cacheサービス
次にcacheサービスです。こちらは、Redisの環境になります。
こちらも、ベースイメージはRedisの公式イメージを利用します。

https://hub.docker.com/_/redis

また、dbサービスと同様に、データはdocker上で管理するようにしています。

#### panelサービス
次に、panelサービスです。こちらはRDBをGUIベースで操作するためのツールAdminerの環境です。
SQLを実行することはもちろん、手動で、データを挿入したり削除したり、テーブルを変更したりするのが簡単に行うことができます。

![](https://storage.googleapis.com/zenn-user-upload/6c85358d3e9b-20221227.png)

#### awsサービス
最後にawsサービスです。こちらは、ローカル上で動作するAWSエミュレーション環境です。
今回はメール送信するにはAWS SESを利用しますが、開発の時は、実際にメールを飛ばしたくありません。誤送信のリスクなどもありますし、何よりお金がかかります。そこで、ローカル上にAWSのエミュレーション環境を構築し、そちらを利用することで、誤送信の心配もないし、何より料金がかかりません。

今回は、その環境を構築するためにlocalstackを利用します。

https://localstack.cloud/

こちらは、有料版（Pro版）と無料版（Community版）があり、無料版には制限があったりするので、そこは調べて利用する必要があります。
Amazon SES API v2は、有料版のみでの対応になりますので、今回は、Amazon SES API v1を利用することとします。

- ses v1の利用可能メソッド
https://docs.localstack.cloud/references/coverage/#ses

- ses v2の利用可能なメソッド
https://docs.localstack.cloud/references/coverage/#sesv2



### 実行コマンド

以上で、ファイル作成は終わりました。
次に、実行コマンドを紹介します。
以下2つのコマンドでローカルでの環境が起動します。

```sh
# イメージビルドおよびコンテナ起動
$ docker compose up -d --build
# サーバー起動
$ docker compose exec app air
  __    _   ___  
 / /\  | | | |_) 
/_/--\ |_| |_| \_ , built with Go 

watching .
!exclude _tools
watching auth
watching auth/certificate
watching config
...
``` 

airによるライブリロードでサーバーを起動するため`air`コマンドを叩きます。

サーバー起動コマンドを以下のように`command`に含めてることで`docker compose up -d --build`の１コマンドでサーバ起動を行うことができます。

```yaml:docker-compose.yml
services:
  app:
    command: air # コンテナを起動したら実行するコマンド
```

ただ、自分は起動コマンドを分けた方が柔軟に対応できると感じたため、今回は、含めていません。

## フロントエンド
フロントエンドはdockerhub上に上がっているものベースイメージのみを利用するため、`docker-compose.yml`のみを作成します。

:::message
初回のpackage.jsonの作成手順は、省いています。
:::

### docker-compose.yml

```yml:docker-compose.yml
version: "3.8"
services:
  app:
    working_dir: /app
    image: node:18-alpine
    volumes:
      - ./:/app
    tty: true
    ports:
      - "3000:3000"
    command: yarn dev
  api:
    image: stoplight/prism:latest
    container_name: "api"
    ports:
      - "3001:4010"
    command: mock  -h 0.0.0.0 https://{openAPIのデプロイ先ホスト名}/openapi.yml
```

#### appサービス

こちらは、reactのアプリケーションを動かすnodeの環境になります。
こちらもalpine型を利用しております。


#### apiサービス

こちらは、openapiよりAPIモックを作成してくれる環境です。
バックエンド側で作成した`openapi.yml`(または、`openapi.json`)を利用して、モックのレスポンスを返してくれます。

なので、openapi.ymlを定義し、デプロイするだけで、swaggerを通して、インターフェースの確認もできると同時に、モックAPIも作ってくれるので、一石二鳥になります！

フロント側は、[MSW](https://mswjs.io/)などを利用して、モックデータを作成する必要がなく、とりあえず開発できるので、便利です。アクセスする際は、APIのエンドポイント先をhttp://localhost:3001/ に変更するだけでいけます。

ただ、エラーレスポンス(403, 500など)は出来なさそうなので、そう言った時は、MSWなどを利用して、定義確かめる必要があると思います。

詳しい使い方は、また別の記事にできたらと思います。


### 実行コマンド

以上で、ファイル作成は終わりました。
次に、実行コマンドを紹介します。
以下2つのコマンドでローカルでの環境が起動します。

```sh
# パッケージの導入
$ docker compose run --rm app yarn --frozen-lockfile 
# コンテナの起動およびサーバーの起動
$ docker-compose up -d
```

1つ目のコマンドで、パッケージ導入用コンテナを起動し、nodeのパッケージを導入します。`--rm`オプションをつけることで、終了時にコンテナを削除してくれます。
このコマンド打つことで、ホスト側にも`node_modules`ディレクトリが同期されるので、ホスト側で作業する際も、型補完の恩恵を受けながら作業をすることができます。

2つ目のコマンドでサーバー起動用コンテナ作成し、サーバーを起動します。
フロント側では`comand`として`yarn dev`と指定しているため、コンテナ起動し終わった最後に`yarn dev`が実行されて、サーバーが起動します。`http://localhost:3000/`でアクセスできます。

# まとめ

以上で、ローカル環境構築を紹介しました！

Dockerは今はどこの会社でも使われている技術なので、フロントエンドエンジニア、バックエンドエンジニア間毛なく環境構築ができるようになりたいですね！

他にもMakefileなど、開発の効率を上げる方法があるので、別の記事にしたいと思います。

この記事を参考になれば幸いです。
