# Homeアプリからのアプリ起動

Personiumアプリには様々な形態がありますが、ここでは最も標準的なHomeアプリから起動されるアプリについて以下を説明します。

+ アプリ起動時に渡されるパラメタ
+ アプリ起動後に行うべき処理

## アプリ起動時に渡されるパラメタ

ホームアプリのアプリランチャは以下のURLを呼び出すことでアプリ起動を行います。

    {AppUrl}#cell={cellUrl}&refresh_token={refreshToken}

例

    myapp-custom-scheme://#cell=https://demo.personium.io/john.doe/&refresh_token=RA~egJWZ....FQ
    https://some.svr.example/my-app/index.html#cell=https://pds.personium.example/john.doe/&refresh_token=RA~egJ....FQ

## アプリ起動後に行うべき処理

アプリケーション開発の実際のプロセスはアプリケーションの実装手段(Androidアプリ、HTML5アプリ、iOSアプリ, etc.)や実装言語によって異なりますが、
アプリ起動後に行うべき流れは共通です。

1. 起動URLから必要パラメタを受け取る
1. アプリ認証トークンを取得する
1. アクセストークンを受け取る
1. BoxのURLを取得する
1. Box配下の各種リソースにアクセスする。

### 起動URLから必要パラメタを受け取る

Home Appのランチャから起動されるPersnium Appは、何らかの起動URLにより起動します。
起動URLはhttpsから始まるものかもしれませんし、何等かのカスタムスキームから始まるものかもしれません。
一般的にカスタムスキームURLで起動されたアプリは自分がどのようなURLで起動されたかを取得可能です。
またhttpsから始まるURLであればブラウザが起動するでしょうが、ブラウザアプリであっても自身がどのようなURLで起動されたかを取得可能です。
まず、それぞれの実装において起動URLを取得してください。

次に取得できた起動URLから#以下をパースして、cell, refresh_tokenパラメタを取得してください。

|項目|概要|
|:--|:--|
|cell|ターゲットとすべきユーザのCell URL|
|refresh_token|ターゲットとすべきユーザのCellで発行され、そこで有効である、現在アクセス中のユーザのアクセストークン取得のためのリフレッシュトークン|

これらのパラメタはこの先のプロセスで必要になります。

### アプリ認証トークンを取得する

あなたのアプリの正当性をユーザCellに対して証明するために、アプリ認証トークンを取得します。これはフィッシングアプリなど、
悪意のあるアプリケーションからの攻撃からあなたのアプリや、ユーザCellを守るためのセキュリティです。

アプリ認証トークンは、アプリCellのTokenエンドポイントに対してアプリのID/パスワードを送ることで取得できます。具体的には以下のような情報をPOSTします。

    grant_type=password&p_target={ユーザCell URL}&username={アプリID}&password={アプリパスワード} 

アプリID, アプリパスワードはあなたのアプリしか知らないものです。これをあなたのアプリからアプリCellのエンドポイントに投げるためには、
コード中に難読化して埋め込んでもよいですし、どこかのサーバにおいて、サーバから上記リクエストを出してもらってもよいでしょう。
アプリCell上にEngineスクリプトを置いて、これを実現するのもよいアイディアです。難読化して埋め込みを行った場合は、
攻撃者のリバースエンジニアリングによってアプリID, アプリパスワードが漏洩することは難読化による一定の抑止力はあるものの、完全には防ぐことができません。

いずれにせよ、このリクエストに対する認証成功応答JSONの"access_token"項目が「アプリ認証トークン」となります。JSONをパースして取得してください。

参考： http://personium.io/docs/ja/apiref/current/293_OAuth2_Token_Endpoint.html


### アクセストークンを受け取る

ユーザCellに対するあなたのAppとしてのアクセストークンを受け取ります。

1. アプリ認証トークン
1. リフレッシュトークン

今度はユーザCellのTokenエンドポイントに対して、以下の情報をPOSTします。

    grant_type=refresh&refresh_token={refreshToken}&client_id={アプリCellURL}&client_secret={アプリ認証トークン}

ここで返ってくるレスポンスJSONの中の"access_token"項目が、対象ユーザCellに対するあなたのアプリのためのアクセストークンです。

### BoxのURLを取得する

以下のAPIを使って、対象ユーザCell上のあなたのアプリのための領域であるBoxのURLを取得します。

参考: http://personium.io/docs/ja/apiref/current/304_Get_Box_URL.html

この際、Authorizationヘッダで取得したアクセストークンを指定するようにしてください。

    Authorization: Bearer {access_token}

これにより、どこのURLをルートとしてデータアクセスすればよいかがわかります。

### Box配下の各種リソースにアクセスする

アプリ開発者である貴方は、対象ユーザCell上のあなたのアプリのためのBox配下の構造については知っているはずです。
データアクセスのためのBoxレベルAPI (WebDAV, OData)を用いて、Box配下の各種リソースにアクセスしてください。

