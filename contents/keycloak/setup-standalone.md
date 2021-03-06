# standalone で動かすまで

- 経緯 : monban-core の認証部分を自分でやるのはまずいのでは？と思った
- 目的 : 認証部分を置き換えられる方法を見つける
- 状況 : インフラ勉強会で keycloak を使っているのを見た
- 決定 : keycloak を使って置き換えてみよう


###### Table of Contents

- [Getting Started](#user-content-Getting Started)
- [javascript adapter で接続](#user-content-javascript adapter で接続)
- [jwt を ruby で検証](#user-content-jwt を ruby で検証)


### Getting Started

###### 環境

- openjdk:8-jdk-slim-stretch コンテナで実行 (最初 12 を使ってしまって動かなかった)
- version: keycloak-4.6.0.Final

#### サーバー起動

まず standalone で起動する

```
$ ./bin/standalone.sh -b 0.0.0.0 -Djava.security.egd=file:/dev/urandom
```

`-b 0.0.0.0` は `0.0.0.0` に bind する設定。
コンテナ内で動かすため、外からアクセスできるようにする必要がある。

`-Djava.security.egd=file:/dev/urandom` はランダム生成機に `/dev/urandom` を使うように指示するもの。

> On Linux, it is recommended to use /dev/urandom as a source of random data to prevent Keycloak hanging due to lack of available entropy, unless /dev/random usage is mandated by your security policy.
> To achieve that on Oracle JDK 8 and OpenJDK 8, set the java.security.egd system property on startup to file:/dev/urandom.


#### ユーザーの作成

コンテナで動かしているのでブラウザから localhost でアクセスできない。
スクリプトで最初の admin を作成する。

```
$ ./bin/add-user-keycloak.sh -u <USER_NAME>
```

パスワードを聞かれるので入力する。


#### Realm とユーザーを作成する

管理画面の左上の「Master」がドロップダウンになっているのでそこから `Add Realm` を選択する。
Realm 名に「demo」を入力して作成する。
管理画面の左上が「Demo」に変わったことを確認。

メニューから「Users」を選択して「Add user」ボタンをクリックしてユーザーを作成。
ユーザー名のみ必須項目になっている。
ユーザーを作成したら一覧からユーザーの詳細を表示する。
全ユーザーの表示は、「View all users」ボタンをクリックする必要がある。

ユーザーの「Credential」タブでパスワードを設定する。
「Temprary」を「ON」にすると、ユーザーのログイン時にパスワードの変更が求められる。


### javascript adapter で接続

#### Client 作成

管理画面から「Clients」を選択し、「Create」で新しく作成する。
名前は「demoapp」にする。
`Access Type` を `public` にして、URL を front の URL に設定する。


#### front app

index.html と page.js を設置する。

- index.html

```html
<!doctype html>
<html lang="ja">
    <head>
        <meta charset="utf-8">
        <title>Hello, world!!</title>
    </head>
    <body>
        <h1>Hello, world!!</h1>

        <p id="message">messages display here</p>

        <script src="http://keycloak.example.com/auth/js/keycloak.js" defer></script>
        <script src="/js/page.js" defer></script>
    </body> 
</html>
```

- page.js

```js
"use strict";

const keycloak = Keycloak({
    url: "http://keycloak.example.com/auth",
    realm: "demo",
    clientId: "demoapp"
});                                                                                                    

keycloak.init({ onLoad: "login-required" })
    .success(function(authenticated) {
        document.getElementById("message").innerText = authenticated;
    })
    .error(function(){
        document.getElementById("message").innerText = "auth error!";
    });
```

index.html にアクセスすると keycloak のログイン画面にリダイレクトする。
ログインすると index.html が表示されるはず。


### jwt を ruby で検証

#### public key の用意

ログインに成功した後、developper console で `keycloak.token` をみてみると、keycloak が発行した jwt が取得できる。
この jwt を ruby で検証してみる。

管理画面の「Realm Settings」の「Keys」タブから RSA public key をコピーできる。
これを `rsa.pem` として保存。

- rsa.pem

```
-----BEGIN CERTIFICATE-----
(ここに Certificate ボタンで表示された内容をコピー)
-----END CERTIFICATE-----
```

#### ruby で検証

`jwt` gem を使って以下のように検証できる。

```ruby
require "jwt"

token = ARGV[0]
puts token

cert = OpenSSL::X509::Certificate.new File.read("rsa.pem")

result = JWT.decode token, cert.public_key, true, { algorithm: "RS256" }
puts result

result["resource_access"]["demoapp"]["roles"] # => ここに Client roles
```
