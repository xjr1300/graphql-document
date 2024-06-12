# HTTPを介して提供する

<https://graphql.org/learn/serving-over-http/>

- [HTTPを介して提供する](#httpを介して提供する)
  - [Webリクエストパイプライン](#webリクエストパイプライン)
  - [URIとルート](#uriとルート)
  - [HTTPメソッド、ヘッダそしてボディ](#httpメソッドヘッダそしてボディ)
    - [GETリクエスト](#getリクエスト)
    - [POSTリクエスト](#postリクエスト)
  - [レスポンス](#レスポンス)
  - [ノード](#ノード)
  - [転送仕様のドラフト](#転送仕様のドラフト)

HTTPは至る所に存在するため、GraphQLを使用するとき、HTTPはクライアント／サーバープロトコルで最も一般的な選択です。
ここに、HTTPを介して操作するためにGraphQLサーバーのセットアップに関するいくつかのガイドラインを示します。

## Webリクエストパイプライン

最も現代的なWebフレームワークは、フィルター／プラグインとして知られている、ミドルウェアのスタックを経由してリクエストが渡されるパイプラインモデルを使用しています。
パイプラインを経由したリクエストが流れるため、リクエストは検証され、変換され、変更されまたはレスポンスを伴って終了します。
GraphQLはすべての認証ミドルウェアの後に配置されるべきであるため、HTTPエンドポイントで行った、同じセッションとユーザー情報でアクセスする必要があります(GraphQL should be placed after all authentication middleware, so that you have access to the same session and user information you would in your HTTP endpoint handlers.)。

## URIとルート

HTTPは、一般的に、核となる概念として"リソース"を使用するRESTと関連があります。
対照的に、GraphQLの概念的なモデルは完全にグラフです。
その結果、GraphQLのエンティティはURLによって識別されません。
代わりに、GraphQLサーバーは、通常`/graphql`となる単一のURLエンドポイントを操作して、サービスに与えられたすべてのGraphQLリクエストは、このエンドポイントに向かわされるべきです。

## HTTPメソッド、ヘッダそしてボディ

GraphQL HTTPサーバーは、HTTPのGETとPOSTメソッドを処理するべきです。

### GETリクエスト

HTTPのGETリクエストを受診したとき、GraphQLクエリは"query"のクエリ文字列に記述されているべきです。
例えば、もし次のGraphQLクエリを実行うしたい場合・・・

```graphql
{
  me {
    name
  }
}
```

このリクエストは、次のようなHTTPのGETを介して送信されます。

```text
http://myapi/graphql?query={me{name}}
```

クエリ変数は`変数`と呼ばれる追加のクエリパラメーター内にJSONでエンコードされた文字列として送信されます。
もし、クエリがいくつかの名前の付いた操作を含んでいる場合、`operationName`クエリパラメーターはどれを実行すべきかを制御するために使用されます。

### POSTリクエスト

標準的なGraphQLのPOSTリクエストは、`application/json`コンテントタイプを使用するべきで、次の形式のJSONでエンコードされたボディを含んでいます。

```json
{
  "query": "{ me { name } }",
  "operationName": "...",
  "variables": { "myVariable": "someValue", ... }
}
```

`operationName`と`variables`はオプションのフィールドです。
`operationName`は、クエリ内に複数の操作がある場合にのみ要求されます。

## レスポンス

どのメソッドでクエリと変数が送信されたにかかわらず、レスポンスはJSONフォーマットで、リクエストのボディで返される必要があります(When receiving an HTTP GET request, the GraphQL query should be specified in the “query” query string.)。
仕様で言及されたように、クエリによって一部のデータと一部のエラーになる可能性があり、それらは次の形式のJSONオブジェクトで返されるべきです。

```json
{
  "data": { ... },
  "errors": [ ... ]
}
```

もし、返されたエラーがない場合、`"errors"`フィールドはレスポンスに存在しないべきです。
もし、データが返されない場合、[GraphQL仕様に従って](https://spec.graphql.org/October2021/#sec-Data)、`"data"`フィールドは実行中にエラーが発生しなかった場合にのみ含まれるべきです。

## ノード

Node.jsを使用している場合、[サーバー実装のリスト](https://graphql.org/code/#javascript-server)を確認することを推奨します。

## 転送仕様のドラフト

詳細な[HTTP転送仕様](https://github.com/graphql/graphql-over-http)は開発中です。
それはまだ最終版ではありませんが、これらドラフト仕様はGraphQLクライアントとライブラリのメンテナーから、唯一の信頼できる情報源として機能しており、HTTP転送を使用してGraphQLのAPIを公開そして利用する方法を詳しく説明しています。
言語仕様と異なり、遵守は義務ではありませんが、ほとんどの実装は、相互運用性を最大化するために、これら標準に向かって移動しています。
