# ページネーション

<https://graphql.org/learn/pagination/>

- [ページネーション](#ページネーション)
  - [複数形](#複数形)
  - [スライシング](#スライシング)
  - [ページネーションとエッジ](#ページネーションとエッジ)
  - [リストの最後、カウントとコネクション](#リストの最後カウントとコネクション)
  - [完全なコネクションモデル](#完全なコネクションモデル)
  - [コネクションの仕様](#コネクションの仕様)

> さまざまなページネーションモデルは、さまざまなクライアントの能力を有効にします。

GraphQLの一般的なユースケースは、オブジェクトの集合間を関連で横断することです。
GraphQLでこれらの関連を公開する方法は多くあり、クライアントの開発者に変化する能力の集合を与えます。

## 複数形

オブジェクト間の接続を公開する最も単純な方法は、複数の型を返すフィールドを持つことです。
例えば、もしR2-D2の友人のリストを取得したい場合、それらすべてを単に質問できます。

```graphql
{
  hero {
    name
    friends {
      name
    }
  }
}
```

```json
{
  "data": {
    "hero": {
      "name": "R2-D2",
      "friends": [
        {
          "name": "Luke Skywalker"
        },
        {
          "name": "Han Solo"
        },
        {
          "name": "Leia Organa"
        }
      ]
    }
  }
}
```

## スライシング

しかし、すぐに、クライアントが欲しがるかもしれない追加的な振る舞いがあることを認識します。
クライアントは、クラアントが取得したい友人の数を記述することができるように望むかもしれません。
多分、クライアントは最初の二人のみが欲しいのではないでしょうか。
よって、次のように公開したいと考えます。

```graphql
{
  hero {
    name
    friend(first: 2) {
      nam
    }
  }
}
```

しかし、単に最初の二人を取得した場合、リストをページ分けしたいと考えるかもしれません。
一度、クライアントが最初の二人の友人を取得すると、クライアントは次の二人の友人を質問する2番目のリクエストを送信したいと考えるかもしれません。
どのようにその振る舞いを有効にするのでしょうか？

## ページネーションとエッジ

ページネーションする方法はいくつかあります。

- リスト内の次の2つを質問するために、`friends(first:2 offset: 2)`のようにできます。
- 最後に取得した友人の後の次を質問するために、`friends(first:2 after:$friendId)`のようにできます。
- 最後のアイテムのカーソルを取得して、ページ分けするために使用するために、`friends(first:2 after:$friendCursor)`のようにできます。

一般的に、**カーソルに基づくページネーション**が、それらの設計で最も強力であることを発見しました。
特に、カーソルが不透明な場合、オフセットまたはIDに基づいたページネーションは、カーソルをオフセットまたはIDにすることによって、カーソルに基づくページネーションを使用して実装できます。
そして、カーソルの使用は、将来にページネーションモデルを変更する場合、追加的な柔軟性を与えます。
カーソルは不透明で、それらのフォーマットは依存するべきでないため、それらをBASE64でエンコードすることを覚えておいてください。

> カーソルは、オブジェクトのIDを表現するバイト列をBASE64でエンコードした文字列とする。

しかし、それは問題を導きます。
オブジェクトからどのようにカーソルを得るのでしょうか？
`User`型にカーソルを生きさせることは支度ありません。
それはコネクションの特徴で、オブジェクトではありません。
よって、間接的な新しいレイヤを導入します。
`friends`フィールドはリストのエッジを渡すべきで、そしてエッジはカーソルと根底にあるノードの両方を持ちます。

```graphql
{
  hero {
    name
    friends(first: 2) {
      edges {
        node {
          name
        }
        cursor
      }
    }
  }
}
```

エッジの概念は、1つのオブジェクトではなく、エッジ特有の情報がある場合に、役に立つことも証明します。
たとえば、APIで"フレンドシップタイム"を公開したい場合、それをエッジ上に配置するのが自然な場所です。

## リストの最後、カウントとコネクション

現在、カーソルを使用したコネクションを介してページ分けする能力を持ちましたが、コネクションの最後に到達したときをどのように知るのでしょうか？
空のリストを返すまでクエリを続ける必要がありますが、最後に到達したときに、追加のリクエストが必要ないことをコネクションに伝えてほしいと考えています。
同様に、コネクション自身の追加的な情報を知りたいと考えています。
例えば、R2-D2はどれだけ多くの友人がいますか？

これら両方の問題を解決するために、`friends`フィールドはコネクションオブジェクトを返します。
コネクションオブジェクトは、合計数と次のページが存在するかどうかの情報のような他の情報と同様に、エッジのためのフィールドを持ちます。
よって、最終的なクエリは次のようになります。

```graphql
{
  hero {
    name
    friends(first: 2) {
      totalCount
      edges {
        node {
          name
        }
        cursor
      }
      pageInfo {
        endCursor
        hasNextPage
      }
    }
  }
}
```

`pageInfo`オブジェクト内に`endCursor`と`startCursor`を含めるかもしれないことに注意してください。
これにより、ページネーションに必要なカーソルを`pageInfo`から得たため、エッジが含む追加的な情報を必要としなくなり、全くエッジについてクエリする必要がなくなりました。
これは潜在的にコネクションの使用性を改善することを導きます。
`edges`リストを単に公開する代わりに、間接的なレイヤを避けるために、ノードの献身的なリストを公開することもできます。

## 完全なコネクションモデル

明確にするために、これは、単に複数持つオリジナルの設計よりも複雑です。
しかし、この設計を採用することで、クライアントに多くの能力を開放します。

- リストをページ分けする能力
- `totalCount`または`pageInfo`のような、コネクション自身の情報を質問する能力
- `cursor`または`friendshipTime`のような、エッジ自身の情報を質問する能力
- ユーザーが単に不透明なカーソルを使用することによる、バックエンドがするページネーションを変更する能力

これを実際に確認するために、`friendshipConnection`と呼ばれる追加的なフィールドをスキーマ例にあります。
それをクエリ例から確認できます。
`after`パラメーターを`friendsConnection`から削除して、ページネーションにどのような影響を与えるか確認してください。
また、コネクションの`friends`ヘルパフィールドで`edges`フィールドを上書きしてください。
クライアントにとって、それが適切な場合、それは間接的な追加的なエッジレイヤなしで直接友人のリストを得るようにします。

```json
{
  "data": {
    "hero": {
      "name": "R2-D2",
      "friendsConnection": {
        "totalCount": 3,
        "edges": [
          {
            "node": {
              "name": "Han Solo"
            },
            "cursor": "Y3Vyc29yMg=="
          },
          {
            "node": {
              "name": "Leia Organa"
            },
            "cursor": "Y3Vyc29yMw=="
          }
        ],
        "pageInfo": {
          "endCursor": "Y3Vyc29yMw==",
          "hasNextPage": false
        }
      }
    }
  }
}
```

- `after`パラメーターを削除した結果

```json
{
  "data": {
    "hero": {
      "name": "R2-D2",
      "friendsConnection": {
        "totalCount": 3,
        "edges": [
          {
            "node": {
              "name": "Luke Skywalker"
            },
            "cursor": "Y3Vyc29yMQ=="
          },
          {
            "node": {
              "name": "Han Solo"
            },
            "cursor": "Y3Vyc29yMg=="
          }
        ],
        "pageInfo": {
          "endCursor": "Y3Vyc29yMg==",
          "hasNextPage": true
        }
      }
    }
  }
}
```

- `edges`を`friends`で置き換えたクエリ

```graphql
{
  hero {
    name
    friendsConnection(first: 2, after: "Y3Vyc29yMQ==") {
      totalCount
      friends {
        name
      }
      pageInfo {
        endCursor
        hasNextPage
      }
    }
  }
}
```

- `edges`を`friends`で置き換えたクエリの結果

```json
{
  "data": {
    "hero": {
      "name": "R2-D2",
      "friendsConnection": {
        "totalCount": 3,
        "friends": [
          {
            "name": "Han Solo"
          },
          {
            "name": "Leia Organa"
          }
        ],
        "pageInfo": {
          "endCursor": "Y3Vyc29yMw==",
          "hasNextPage": false
        }
      }
    }
  }
}
```

## コネクションの仕様

このパターンの一貫性のある実装を保証するために、カーソルに基づくコネクションパターンを使用して、GraphQLのAPIを構築する際に従う、[Relayプロジェクト](https://relay.dev/)は公式の[仕様書](https://facebook.github.io/relay/graphql/connections.htm)があります。
