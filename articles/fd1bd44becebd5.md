---
title: "【Go】RedisとJWTでログイン認証システムを作ってみた話"
emoji: "🗝️"
type: "tech"
topics:
  - "golang"
  - "jwt"
  - "gin"
  - "認証"
  - "認可"
published: true
published_at: "2023-02-19 14:36"
---

こんにちは、@nerusanです。
昨今のWebアプリケーションにおいて認証機能は、必須です。

それを実現する手軽な方法として[auth0](https://auth0.com/jp?utm_content=japanjpbrandauth0-generic-&utm_source=google&utm_campaign=apac_japan_jpn_all_ciam-all_dg-ao_auth0_search_google_text_kw_utm2&utm_medium=cpc&utm_id=aNK4z0000004IZLGA2&utm_term=auth0-c&gclid=CjwKCAiA5sieBhBnEiwAR9oh2qwm7X3KjCSjckkBHH_1d03vXpSIYlGnwNlFWCFYT6QgrtH4qUR4uBoCPNoQAvD_BwE)や[Firebase Authentication](https://firebase.google.com/products/auth?hl=ja)などのIDaasがあります。

Firebase Autheticationでは、メールアドレスとパスワードの組み合わせの他にも、電話認証、Google、Twitter、Facebook、GitHub のログインなどに対応しており、かなり便利です。


今回は、そちらの使わず、Go、Redis、JWTを利用した認証機能を作る機会があったためそちらの共有ができたらと思います。


# 環境

- golang v1.19
- gin v1.8.1
- jwx/v2 v2.0.8


# 要件定義
まずは要件定義を述べたいと思います。

大まかな要件定義は以下です。

- サインアップには仮登録と本登録があり、メールアドレス認証が成功してから本登録する
- メールアドレスとパスワードで、ログインができる
- ログインが成功したらトークンが発行され、認証で保護されたエンドポイントのリクエストヘッダーAuthorizationに付与し、トークンが有効であればアクセスが可能
- トークンには有効期限があり、最終アクセスから１時間
- サインアウトを実行すると、トークンは無効になる

オーソドックスな認証って感じですね！
もう少しだけ詳しく見ていきます。

**サインアップ**

1. メールアドレス、名前、パスワードで仮登録実行を行うと、確認コード入力画面が表示される
2. 指定したメール📧に確認コードが送信される
3. 期限内(１時間以内)に確認コードを入力すると本登録が完了
4. 本登録が成功したらトークン返却

**サインイン**
1. メールアドレスとパスワードでログイン実行
2. 成功したらトークン返却

**保護されたルートへのアクセス**
1. リクエストヘッダーAuthnticationにトークンを付与しリクエスト
2. 有効であれば、成功レスポンスを返却


**サインアウト**
1. Authnticationよりユーザを判定し、トークンを無効化

具体的なエンドポイントは以下のSwaggerのURLの「ユーザ登録・認証」になります。
https://hack-31.github.io/point-app-backend/swagger/index.html

![](https://storage.googleapis.com/zenn-user-upload/dabc5adbf6f4-20230216.png)

また、右端に鍵マークがついたエンドポイントが、保護されたエンドポイントとなります。
例えば、GET`/users`、GET`/account`などがあります。

![](https://storage.googleapis.com/zenn-user-upload/1190bffea9ed-20230216.png)

# どうやって実現する？
作りたいものはなんとなくわかりました。
さて、どの様に作るかというと、冒頭でも軽く述べた**JWT**と**Redis**によって実現していきます。

## JWT

JWT (JSON Web Token) は、Web アプリケーションや API において認証や認可に利用されるトークン形式の認証方式です。

https://jwt.io/


コンパクトな形式の JWT は、ドット ( .) で区切られた 3 つの部分で構成されています。

xxxxx.yyyyy.zzzzz

それぞれ以下3つのセクションとなっています。

1. Header(xxxxxの部分）
2. Payload（yyyyyの部分)
3. Signature(zzzzzの部分）

詳しく見ていきます。

### 1. Header（xxxxxの部分）
JWT のタイプ、使用する署名アルゴリズムなどのメタデータを定義します。ヘッダーは Base64 エンコードされます。

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

### 2. Payload（yyyyyの部分）
JWT に含めるエンティティ (通常はユーザー) と追加データに関するステートメント（クレーム）を定義します。追加データには、認証されるユーザーの情報や、トークンの有効期限、トークンを発行した認証サーバーの情報などが含まれます。ペイロードは Base64 エンコードされます。暗号化されないため、機密情報（パスワードなど）を含めることはできません。

```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "admin": true
}
```


### 3. Signature（zzzzzの部分）
ペイロードを署名するために使用される秘密鍵に基づく署名です。署名により、トークンが改ざんされていないことを確認できます。署名は、ヘッダー、ペイロード、秘密鍵を使用して生成されます。

署名を生成するアルゴリズムはいろいろあるのですが、今回はRS256を採用しました。
RS256 (SHA-256 を使用した RSA 署名) は、公開鍵と秘密鍵のペアを使用する非対称アルゴリズムです。ID プロバイダーには、署名を生成するための秘密鍵があります。JWT の受信者は、公開鍵を使用して JWT 署名を検証します。検証に使用される公開鍵と、トークンの署名に使用される秘密鍵はペアとして生成されるため、リンクされています。

署名の生成に秘密鍵、検証に公開鍵にすることで改ざん・偽装のリスクを減らすことができます。特に検証用のサーバーが複数ある時に有効になるかなって思います。
検証用サーバーが複数あるということは、漏洩のリスクが大きくなりますが、仮に公開鍵が漏洩しても、JWTは偽装できないので、なりすましアクセスを防ぐことができます。

**ただ、秘密鍵は漏洩すると第三者にJWTを偽装される可能性があるので、漏洩しないように取り扱いは注意する必要があります。**

具体的にできることは、以下が挙げられるかなって思います。
* 秘密鍵でJWTを署名するのは一箇所にまとめるべき（サーバーなど）
* 秘密鍵、公開鍵のペアは定期的にローテーションすべき
* 直接コード中には秘密鍵を書かない
* 秘密鍵をアクセスし閲覧できる人を制限する

今回は、GitHub のシークレットを利用して、秘密鍵、公開鍵を扱うようにして、制限をかけるようにして、セキュリティを向上させました。

https://docs.github.com/ja/actions/security-guides/encrypted-secrets

RS256の詳しい説明などは、以下のサイトをご覧ください。

https://auth0.com/blog/rs256-vs-hs256-whats-the-difference/

### JSON Web Tokenはどのように機能しますか?


認証では、ユーザーが資格情報(メールアドレスとパスワードなど)を使用して正常にログインすると、JSON Web Tokenが返されます。

ユーザーが保護されたルートまたはリソースにアクセスしたいときはいつでも、ユーザー エージェントは通常、 Bearerスキーマを使用してAuthorizationヘッダーで JWT を送信する必要があります。ヘッダーの内容は次のようになります。

```:リクエストヘッダー
Authorization: Bearer xxxxx.yyyyy.zzzzz
```


保護されたルートは、Authorizationヘッダーに有効なJWTがあるかどうかをチェックし、有効であれば、ユーザーは保護されたリソースへのアクセスを許可されます。JWTに必要なデータ(ユーザIDやメールアドレスなど）が含まれていれば、特定の操作のためにデータベースに問い合わせる必要性が減るかもしれませんが、常にそうなるとは限りません。

JWTは、場合によっては、ステートレス認可メカニズムになりえます。つまり、サーバー側にユーザのログイン情報を保存してなくとも、リクエストが送られたユーザ情報とログイン状況を判別できます。

JWTには、トークン有効期間を設定することができるので、JWTだけで、認証ができるじゃんと思われるかもしれませんが、実はそうではありません。

それは、サインアウトにより、手動でJWTを無効にすることができないということです。
サインアウトした後でも、有効期限が来るまではトークンは有効なので、保護されたルートにアクセスすることはできます。

サインアウトしたのにアクセスできるとなると少し怖いですよね。。
その問題を解決するのが**Redis**です。

## Redis

Redis は、リモートディクショナリサーバー(Remote Directionary Server)の略で、ミリ秒未満の応答時間を実現する高速なオープンソースメモリ内 key-value データストアです。

https://redis.io/

Redisにサインインユーザの情報を登録し、リクエストのたびにRedisにサインイン済みかどうかを確認する処理を追加します。サインアウトしたら、Redisからサインインユーザーの情報を削除し、再度同じトークンからリクエストが来ても、Redisにユーザ情報が登録されていないので、アクセスを制限する流れになります。

これによって、先ほどの問題となっていたサインアウト後でもアクセスが可能になる問題は解決できます。
ただ、これによってステートレス認可メカニズムではなくなってしまいますが、セキュリティを考えた場合は仕方ないのかなって思います。

RedisではなくてMySQLなどのRDBMSではダメなの？って思われた方もいるかもしれません。MySQLでも実現は可能ですが、パフォーマンス面で劣ります。ログインユーザ数が増えるとその分登録データが増え、ログイン済みかどうかを検索かけても、レスポンスまでの時間が大きくなる予想されます。保護されたルートは、毎度ログイン済みかをDBに問い合わせる必要があるため、APIのレスポンスが遅くなり、結果UXが下がることにつながりかねません。なので、より速度が速く、よりシンプルにデータにアクセスができるRedisが必要になると言うことです。

それでは、次から各APIの実際のフローとコードを紹介します。

# サインアップ（仮登録）
仮登録では、メールアドレスが実際に自身のものか検証するため、ユーザー情報（メール、名前、パスワードなど）はRDBMSに登録せず、Redisに保存します。この際は、有効期限をつけます。理由は、メールアドレスを間違った際など、不要なデータを蓄積したくないためです。

この時、保存する方法として、キーを`確認コード：UUID`、バリューを各ユーザー情報を改行コードで区切った文字列で保存します。このとき、パスワードはハッシュ化するのを忘れないようにします。キーにUUIDを付与するのは、確認コードのみだと、他の仮登録とかぶる可能性があるため、UUIDを追記しました。


```sh:Redisに保存されたキーとバリュー例
$ redis-cli
127.0.0.1:6379> keys *
1) "2816:1d7216d7-e9aa-41fd-a956-d045807156be"
2) "3656:e6907d61-fbaa-463d-bf73-ca1e57dcdf9b"
3) "4888:23c7d91a-be6c-4508-8f68-04e248ab8ae1"
4) "7106:a7a3edbf-8ea3-4a0d-8753-64ff9706f757"
127.0.0.1:6379> get 7106:a7a3edbf-8ea3-4a0d-8753-64ff9706f757
"\xe5\xa4\xaa\xe9\x83\x8e\n\xe3\x82\xbf\xe3\x83\xad\xe3\x82\xa6\n\xe5\xb1\xb1\xe7\x94\xb0\n\xe3\x83\xa4\xe3\x83\x9e\xe3\x83\x80\nyamada@sample.com\n$2a$10$iHzbevtMnWWsTpq1iowcN.vL5KEHZcDdrulj2oZt5DfxItMzrp9p2"
```


▼アクティビティ図
![](https://storage.googleapis.com/zenn-user-upload/e23113db1b4c-20230218.png)

```go:service/register_temporary_users.go
package service

import (
	"context"
	"fmt"
	"time"

	"github.com/google/uuid"
	"github.com/hack-31/point-app-backend/constant"
	"github.com/hack-31/point-app-backend/domain"
	"github.com/hack-31/point-app-backend/domain/model"
	"github.com/hack-31/point-app-backend/domain/service"
	"github.com/hack-31/point-app-backend/repository"
	utils "github.com/hack-31/point-app-backend/utils/email"
)

type RegisterTemporaryUser struct {
	DB    repository.Queryer
	Cache domain.Cache
	Repo  domain.UserRepo
}

// ユーザ仮登録サービス
//
// @params
// ctx コンテキスト
// firstName 名前
// firstNameKana 名前カナ
// familyName 名字
// familyNameKana 名字カナ
// email メールアドレス
// password パスワード
//
// @returns
// temporaryUserId 一時保存したユーザを識別するID
func (r *RegisterTemporaryUser) RegisterTemporaryUser(ctx context.Context, firstName, firstNameKana, familyName, familyNameKana, email, password string) (string, error) {
	// ユーザドメインサービス
	userService := service.NewUserService(r.Repo)

	// 登録可能なメールか確認
	existMail, err := userService.ExistByEmail(ctx, &r.DB, email)
	if err != nil {
		return "", err
	}
	if existMail {
		return "", fmt.Errorf("failed to register: %w", repository.ErrAlreadyEntry)
	}

	// パスワードハッシュ化
	pass, err := model.NewPasswrod(password)
	if err != nil {
		return "", fmt.Errorf("cannot create passwrod object: %w", err)
	}
	hashPass, err := pass.CreateHash()
	if err != nil {
		return "", fmt.Errorf("cannot create hash passwrod: %w", err)
	}

	// ユーザ情報をキャッシュに保存
	tempUserInfo := model.NewTemporaryUserString("")
	// キャッシュサーバーに保存するkeyの作成
	uid := uuid.New().String()
	confirmCode := model.NewConfirmCode().String()
	key := fmt.Sprintf("%s:%s", confirmCode, uid)
	// キャッシュのサーバーに保存するvalueを作成
	userString := tempUserInfo.Join(firstName, firstNameKana, familyName, familyNameKana, email, hashPass)
	// 保存
	err = r.Cache.Save(ctx, key, userString, time.Duration(constant.ConfirmationCodeExpiration_m))
	if err != nil {
		return "", fmt.Errorf("failed to save in cache: %w", err)
	}

	// メール送信
	subject := "【ポイントアプリ】本登録を完了してください"
	body := fmt.Sprintf("%s %sさん\n\nポイントアプリをご利用いただきありがとうございます。\n\n確認コードは %s です。\n\nこの確認コードの有効期限は1時間です。", familyName, firstName, confirmCode)
	_, err = utils.SendMail(email, subject, body)
	if err != nil {
		return "", fmt.Errorf("failed to send email: %w", err)
	}

	return uid, nil
}
```

# サインアップ（本登録）
本登録をするには、指定メールに送信される確認コードを検証します。検証が成功したらDBにユーザ情報を登録します。

署名のために利用する秘密鍵と公開鍵は以下のコマンドで作成することができます。

```sh
$ openssl genrsa 4096 > ./auth/certificate/secret.pem
$ openssl rsa -pubout < ./auth/certificate/secret.pem > ./auth/certificate/public.pem
```

▼アクティビティ図
![](https://storage.googleapis.com/zenn-user-upload/cb83e3454a36-20230219.png)

```go:service/register_users.go
package service

import (
	"context"
	"fmt"

	"github.com/hack-31/point-app-backend/constant"
	"github.com/hack-31/point-app-backend/domain"
	"github.com/hack-31/point-app-backend/domain/model"
	"github.com/hack-31/point-app-backend/repository"
)

type RegisterUser struct {
	DB             repository.Execer
	Cache          domain.Cache
	Repo           domain.UserRepo
	TokenGenerator domain.TokenGenerator
}

// ユーザ登録サービス
//
// @params ctx コンテキスト, temporaryUserId 一時保存ユーザid
//
// @return ユーザ情報
func (r *RegisterUser) RegisterUser(ctx context.Context, temporaryUserId, confirmCode string) (*model.User, string, error) {
	// 一時ユーザ情報を復元
	key := fmt.Sprintf("%s:%s", confirmCode, temporaryUserId)
	u, err := r.Cache.Load(ctx, key)
	if err != nil {
		return nil, "", fmt.Errorf("cannot load user in cache: %w", err)
	}

	// 復元が成功したら一時ユーザ情報削除
	if err := r.Cache.Delete(ctx, key); err != nil {
		return nil, "", fmt.Errorf("cannot delete in cache: %w", err)
	}

	// 復元したユーザ情報を解析
	temporyUser := model.NewTemporaryUserString(u)
	firstName, firstNameKana, familyName, familyNameKana, email, hashPass := temporyUser.Split()

	// DBに保存
	user := &model.User{
		FirstName:      firstName,
		FirstNameKana:  firstNameKana,
		FamilyName:     familyName,
		FamilyNameKana: familyNameKana,
		Email:          email,
		Password:       hashPass,
		SendingPoint:   constant.DefaultSendingPoint,
	}
	if err := r.Repo.RegisterUser(ctx, r.DB, user); err != nil {
		return nil, "", fmt.Errorf("failed to register: %w", err)
	}

	// JWTを作成
	jwt, err := r.TokenGenerator.GenerateToken(ctx, *user)
	if err != nil {
		return nil, "", fmt.Errorf("failed to generate JWT: %w", err)
	}

	return user, string(jwt), nil
}
```

```go:auth/jwt.go
package auth

import (
	"context"
	_ "embed"
	"fmt"
	"net/http"
	"strconv"
	"time"

	"github.com/gin-gonic/gin"
	"github.com/google/uuid"
	"github.com/hack-31/point-app-backend/constant"
	"github.com/hack-31/point-app-backend/domain/model"
	"github.com/hack-31/point-app-backend/utils/clock"
	"github.com/lestrrat-go/jwx/v2/jwa"
	"github.com/lestrrat-go/jwx/v2/jwk"
	"github.com/lestrrat-go/jwx/v2/jwt"
)

const (
	UserID = "user_id"
	Email  = "email"
)

//go:embed certificate/secret.pem
var rawPrivKey []byte

//go:embed certificate/public.pem
var rawPubKey []byte

type JWTer struct {
	PrivateKey, PublicKey jwk.Key
	Store                 Store
	Clocker               clock.Clocker
}

// jWTのインスタンス
func NewJWTer(s Store, c clock.Clocker) (*JWTer, error) {
	j := &JWTer{Store: s}
	privkey, err := parse(rawPrivKey)
	if err != nil {
		return nil, fmt.Errorf("failed in NewJWTer: private key: %w", err)
	}
	pubkey, err := parse(rawPubKey)
	if err != nil {
		return nil, fmt.Errorf("failed in NewJWTer: public key: %w", err)
	}
	j.PrivateKey = privkey
	j.PublicKey = pubkey
	j.Clocker = c
	return j, nil
}

func parse(rawKey []byte) (jwk.Key, error) {
	key, err := jwk.ParseKey(rawKey, jwk.WithPEM(true))
	if err != nil {
		return nil, err
	}
	return key, nil
}

// アクセストークンの作成
// @params
// ctx コンテキスト
// u ユーザエンティティ
//
// @returns
// token アクセストークン
func (j *JWTer) GenerateToken(ctx context.Context, u model.User) ([]byte, error) {
	tok, err := jwt.NewBuilder().
		JwtID(uuid.New().String()).
		Issuer(`github.com/hack-31/point-app-backend`).
		Subject("access_token").
		IssuedAt(j.Clocker.Now()).
		Expiration(j.Clocker.Now().Add(time.Duration(constant.MaxTokenExpiration_m)*time.Minute)).
		Claim(Email, u.Email).
		Claim(UserID, u.ID).
		Build()
	if err != nil {
		return nil, fmt.Errorf("GenerateToken: failed to build token: %w", err)
	}
	if err := j.Store.Save(ctx, fmt.Sprint(u.ID), tok.JwtID(), time.Duration(constant.TokenExpiration_m)); err != nil {
		return nil, err
	}

	// 署名
	signed, err := jwt.Sign(tok, jwt.WithKey(jwa.RS256, j.PrivateKey))
	if err != nil {
		return nil, err
	}
	return signed, nil
}
```

# サインイン

サインインでは、メールアドレスとパスワードを検証し、正しければ、JWTを返します。
ここでの注意点は、パスワードが間違っているからといってユーザに「パスワードが間違っています」と返してはいけません。これは、暗にメールアドレスは有効であるということをユーザに知らせています。つまり、悪気を持った第三者に悪用される可能性があります。有効なメールアドレスということで、パスワード総当たりでログインされたり、その情報で他のサービスにも不正アクセスされる可能性があったり、メールアドレスそのものを別目的で悪用されたりする可能性があります。なので、ユーザーレンポンスを返すときは、「メールアドレスまたはパスワードが異なります。」というふうにレスポンスします。

▼アクティビティ図
![](https://storage.googleapis.com/zenn-user-upload/f0dc2ba9b5f7-20230219.png)

```go:service/signin.go
package service

import (
	"context"
	"fmt"

	"github.com/hack-31/point-app-backend/domain"
	"github.com/hack-31/point-app-backend/domain/model"
	"github.com/hack-31/point-app-backend/repository"
)

type Signin struct {
	DB             repository.Queryer
	Cache          domain.Cache
	Repo           domain.UserRepo
	TokenGenerator domain.TokenGenerator
}

// サインインサービス
//
// @params
// ctx コンテキスト
// email メール
// password パスワード
//
// @return
// JWT
func (s *Signin) Signin(ctx context.Context, email, password string) (string, error) {
	// emailよりユーザ情報を取得
	u, err := s.Repo.FindUserByEmail(ctx, s.DB, &email)
	if err != nil {
		return "", fmt.Errorf("failed to find user : %w", repository.ErrNotMatchLogInfo)
	}

	// パスワードが一致するか確認
	pwd, err := model.NewPasswrod(password)
	if err != nil {
		return "", fmt.Errorf("cannot create password object: %w", err)
	}
	if isMatch, _ := pwd.IsMatch(u.Password); !isMatch {
		return "", fmt.Errorf("no match passwrod:  %w", repository.ErrNotMatchLogInfo)
	}

	// JWTを作成
	jwt, err := s.TokenGenerator.GenerateToken(ctx, u)
	if err != nil {
		return "", fmt.Errorf("failed to generate JWT: %w", err)
	}

	return string(jwt), nil
}
```

# 保護されたルートへのアクセス

アクセスされたらまず、トークンが有効かどうかを検証します。無効であれば、処理は中断し、エラーを返します。

ここでのポイントは２点です。
１点目は、アクセスの都度、有効期限を伸ばすようにしています。これは、連続ブラウザジングの最中にトークン無効エラーになるのを防ぐためです。
２点目は、JWTよりユーザ情報の解析が成功したら、コンテキストにユーザIDとメールアドレスをセットします。そうすることで、DBにアクセスしなくともサービス内でユーザIDを利用することができます。

▼アクティビティ図
![](https://storage.googleapis.com/zenn-user-upload/d28526299954-20230219.png)

```go:service/auth_router.go
package router

import (
	"context"

	"github.com/gin-gonic/gin"
	"github.com/hack-31/point-app-backend/auth"
	"github.com/hack-31/point-app-backend/config"
	"github.com/hack-31/point-app-backend/handler"
	"github.com/hack-31/point-app-backend/repository"
	"github.com/hack-31/point-app-backend/service"
	"github.com/hack-31/point-app-backend/utils/clock"
	"github.com/jmoiron/sqlx"
)

// アクセストークンを必要とする
// 認証が必要なルーティングの設定を行う
//
// @param
// ctx コンテキスト
// router ルーター
func SetAuthRouting(ctx context.Context, db *sqlx.DB, router *gin.Engine, cfg *config.Config) error {
	// レポジトリ
	clocker := clock.RealClocker{}
	rep := repository.Repository{Clocker: clocker}

	// トークン
	tokenCache, err := repository.NewKVS(ctx, cfg, repository.TemporaryUserRegister)
	if err != nil {
		return err
	}
	jwter, err := auth.NewJWTer(tokenCache, clocker)
	if err != nil {
		return err
	}

	// ルーティング設定
	// ミドルウェアの設定
	groupRoute := router.Group("/api/v1").Use(handler.AuthMiddleware(jwter))

	getUsersHandler := handler.NewGetUsers(&service.GetUsers{DB: db, Repo: &rep})
	groupRoute.GET("/users", getUsersHandler.ServeHTTP)

	getUserHandler := handler.NewGetAccount(&service.GetAccount{DB: db, Repo: &rep})
	groupRoute.GET("/account", getUserHandler.ServeHTTP)

	return nil
}

```

```go:handler/middleware.go
package handler

import (
	"log"
	"net/http"

	"github.com/gin-gonic/gin"
	"github.com/hack-31/point-app-backend/auth"
)

// 認証ミドルウェア
func AuthMiddleware(j *auth.JWTer) gin.HandlerFunc {
	return gin.HandlerFunc(func(ctx *gin.Context) {
		log.Print("AuthMiddleware")

		if err := j.FillContext(ctx); err != nil {
			log.Print(err.Error())
			ErrResponse(ctx, http.StatusUnauthorized, "認証エラー", "アクセストークンが無効です。再ログインしてください。")

			return
		}
		ctx.Next()
	})
}
```

```go:auth/jwt.go
package auth

import (
	"context"
	_ "embed"
	"fmt"
	"net/http"
	"strconv"
	"time"

	"github.com/gin-gonic/gin"
	"github.com/google/uuid"
	"github.com/hack-31/point-app-backend/constant"
	"github.com/hack-31/point-app-backend/domain/model"
	"github.com/hack-31/point-app-backend/utils/clock"
	"github.com/lestrrat-go/jwx/v2/jwa"
	"github.com/lestrrat-go/jwx/v2/jwk"
	"github.com/lestrrat-go/jwx/v2/jwt"
)

const (
	UserID = "user_id"
	Email  = "email"
)

//go:embed certificate/secret.pem
var rawPrivKey []byte

//go:embed certificate/public.pem
var rawPubKey []byte

type JWTer struct {
	PrivateKey, PublicKey jwk.Key
	Store                 Store
	Clocker               clock.Clocker
}

// jWTのインスタンス
func NewJWTer(s Store, c clock.Clocker) (*JWTer, error) {
	j := &JWTer{Store: s}
	privkey, err := parse(rawPrivKey)
	if err != nil {
		return nil, fmt.Errorf("failed in NewJWTer: private key: %w", err)
	}
	pubkey, err := parse(rawPubKey)
	if err != nil {
		return nil, fmt.Errorf("failed in NewJWTer: public key: %w", err)
	}
	j.PrivateKey = privkey
	j.PublicKey = pubkey
	j.Clocker = c
	return j, nil
}

func parse(rawKey []byte) (jwk.Key, error) {
	key, err := jwk.ParseKey(rawKey, jwk.WithPEM(true))
	if err != nil {
		return nil, err
	}
	return key, nil
}


// トークンを取得し解析
// @params
// ctx コンテキスト
// r リクエスト情報
//
// @returns
// トークン
func (j *JWTer) GetToken(ctx context.Context, r *http.Request) (jwt.Token, error) {
	token, err := jwt.ParseRequest(
		r,
		jwt.WithKey(jwa.RS256, j.PublicKey),
		jwt.WithValidate(false),
	)
	if err != nil {
		return nil, err
	}
	return token, nil
}

// トークンを解析し、contextにuserIDとEmailをセットする
// @params
// ctx コンテキスト
func (j *JWTer) FillContext(ctx *gin.Context) error {
	// トークンを解析
	token, err := j.GetToken(ctx.Request.Context(), ctx.Request)
	if err != nil {
		return err
	}

	// 有効期限が切れていないか確認
	if err := jwt.Validate(token, jwt.WithClock(j.Clocker)); err != nil {
		return fmt.Errorf("GetToken: failed to validate token: %w", err)
	}

	// キャッシュに対しても有効期限を確認
	id, ok := token.Get(UserID)
	if !ok {
		return fmt.Errorf("not found %s", UserID)
	}
	uid := fmt.Sprintf("%v", id)
	jwi, err := j.Store.Load(ctx, uid)
	if err != nil {
		return fmt.Errorf("GetToken: %v expired: %w", id, err)
	}

	// 他のログインを検査
	if jwi != token.JwtID() {
		return fmt.Errorf("expired token %s because login another", jwi)
	}

	// 有効期限延長
	j.Store.Expire(ctx, uid, time.Duration(constant.TokenExpiration_m))

	// コンテキストにユーザ情報追加
	intUid, _ := strconv.ParseInt(uid, 10, 64)
	ctx.Set(UserID, model.UserID(intUid))
	SetEmail(ctx, token)

	return nil
}

// メールをコンテキストに代入
//
// @paramss
// ctx コンテキスト
// tok トークン
func SetEmail(ctx *gin.Context, tok jwt.Token) {
	get, ok := tok.Get(Email)
	if !ok {
		ctx.Set(Email, "")
		return
	}
	ctx.Set(Email, get)
}
```

# サインアウト

▼アクティビティ図
![](https://storage.googleapis.com/zenn-user-upload/af66456ac45d-20230219.png)

```go:service/signout.go
package service

import (
	"fmt"

	"github.com/gin-gonic/gin"
	"github.com/hack-31/point-app-backend/auth"
	"github.com/hack-31/point-app-backend/domain"
	"github.com/hack-31/point-app-backend/domain/model"
)

type Signout struct {
	Cache domain.Cache
}

// サインアウトサービス
//
// @params
// ctx コンテキスト
// uid ユーザーID
//
// @return
// err
func (s *Signout) Signout(ctx *gin.Context) error {
	// ユーザIDの取得
	userId, _ := ctx.Get(auth.UserID)
	uid := userId.(model.UserID)

	// ユーザIDをキャッシュから削除
	if err := s.Cache.Delete(ctx, fmt.Sprint(uid)); err != nil {
		return fmt.Errorf("cannot delete in cache: %w", err)
	}

	return nil
}
```


# まとめ

以上で、JWTによる認証を実現しました。
実装されているGitHubのリポジトリは以下ですので、参考にしてみてください。
https://github.com/hack-31/point-app-backend

**訂正削除2022/2/20**
~~小さなアプリなどでは、全然利用できると思うので、よかったら参考にしてみてください！~~

**訂正追加2022/2/20**
今回のこの実装では、正直実務で使えるレベルではありません。

コメントをいただいたように、他にも、パスワード、確認コードの総当たりをされないような仕組みづくりなどをする必要があります。

他にも以下の点を考慮する必要があります。項目の[参照はこちら](https://gmo-cybersecurity.com/service/web-application/)
* ログインフォームおよび秘密情報の入力フォームに関する調査	
	* ログインフォームや他の秘密情報を入力するフォームについて、入力情報の取扱が適切であるか
* エラーメッセージによる情報推測	
	* 認証機能を利用するWebアプリケーション等で、認証失敗時のエラーメッセージ出力の問題により登録済の認証情報が推測できるといった脆弱性がないか
* 平文による秘密情報の送受信	
	* Webアプリケーションのパスワード等の秘密情報を、HTTPSで暗号化せずに平文で送受信していないか
* アカウントロックアウトの不備	
	* 認証機能について、試行回数の制限の有無
* ログアウト機能の不備	
	* 認証機能の存在するシステムで、ログアウト機能が提供されている事の確認及びログアウトの実行時にセッションが適切に破棄されているか診断
* パスワード変更または再発行機能の悪用	
	* 利用者や管理者がパスワードを変更または再発行する機能について、その欠陥により第三者によるパスワード変更や漏洩を招く脆弱性がないか
* 強制ブラウズ
	* アクセス制御の不備により、認証を要するページに認証なしに直接アクセスできる脆弱性がないか
* 認証の不備
	* 認証機能について、処理の欠陥により迂回を許す脆弱性がないか

これは一例ですが、他にも考慮する点はたくさんあります。
今回メールアドレスをクレームに含めていましたが、正直これも良くないと思っています。（常にヘッダーに送るので、漏洩にリスクが上がるため）

セキュリティ周りの不具合は、信用に失墜に繋がりかねない重要な機能の一つです。
専門の人がいない状況下では、自身で１から作成するより、外部サービス（Auth0、Firebase AuthなどIDaaS）を利用した方がいいと思います。
今回は車輪の再開発でしたが、勉強にはなりました：）

