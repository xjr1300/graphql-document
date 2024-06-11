# イントロスペクション（スキーマの問い合わせ）

<https://graphql.org/learn/introspection/>

なんのクエリをサポートしているかについて情報をGraphQLスキーマに質問することは、よく役に立ちます。
GraphQLは、イントロスペクションシステムを使用して、それをできるようにします。

スターウォーズの例で、[starWarsIntrospection-test.ts](https://github.com/graphql/graphql-js/blob/main/src/__tests__/starWarsIntrospection-test.ts)ファイルは、イントロスペクションシステムを実演するいくつかのクエリが含まれており、イントロスペクションシステムのリファレンス実装を訓練するために実行できるテストファイルです。

どのような型が利用可能化を知るために型システムを設計しましたが、もし設計しなかった場合、常に`Query`のルートタイプに存在する`__schema`フィールドをクエリすることでGraphQLに質問できます。
ここで、それを行って、どの型が存在するか質問しましょう。

```graphql
{
  __schema {
    types {
      name
    }
  }
}
```

```json
{
  "data": {
    "__schema": {
      "types": [
        {
          "name": "Query"
        },
        {
          "name": "String"
        },
        {
          "name": "ID"
        },
        {
          "name": "Mutation"
        },
        {
          "name": "Episode"
        },
        {
          "name": "Character"
        },
        {
          "name": "Int"
        },
        {
          "name": "LengthUnit"
        },
        {
          "name": "Human"
        },
        {
          "name": "Float"
        },
        {
          "name": "Droid"
        },
        {
          "name": "FriendsConnection"
        },
        {
          "name": "FriendsEdge"
        },
        {
          "name": "PageInfo"
        },
        {
          "name": "Boolean"
        },
        {
          "name": "Review"
        },
        {
          "name": "ReviewInput"
        },
        {
          "name": "Starship"
        },
        {
          "name": "SearchResult"
        },
        {
          "name": "__Schema"
        },
        {
          "name": "__Type"
        },
        {
          "name": "__TypeKind"
        },
        {
          "name": "__Field"
        },
        {
          "name": "__InputValue"
        },
        {
          "name": "__EnumValue"
        },
        {
          "name": "__Directive"
        },
        {
          "name": "__DirectiveLocation"
        }
      ]
    }
  }
}
```

多くの型があります。
それらは何でしょうか？
それらをグループ分けしましょう。

- `Query`, `Character`, `Human`, `Episode`, `Droid`: これらは型システムで定義したものです。
- `String`, `Boolean`: これらはタイプシステムが提供するビルトインのスカラー型です。
- `__Schema`, `__Type`, `__TypeKind`, `__Field`, `__InputValue`, `__EnumValue`, `__Directive`: これらはすべて二重アンダースコアで始まっており、それらがイントロスペクションシステムの一部であることを示しています。

ここで、どのようなクエリが利用できるか探求を開始する良い場所を見つけましょう。
型システムを設計したとき、すべてのクエリが開始する型を記述しました。
それについてイントロスペクションシステムに質問しましょう。

```graphql
{
  __schema {
    queryType {
      name
    }
  }
}
```

```json
{
  "data": {
    "__schema": {
      "queryType": {
        "name": "Query"
      }
    }
  }
}
```

上記の結果は型システムのセクションで述べた、`Query`型が開始する場所であるということと一致します。
単に、ここの名前付けは慣例によってであることに注意してください。
`Query`型には他の名前を付けることができますが、クエリの開始する型であると指定した場合でも、ここで返されるはずです。
ただし、それに`Query`と名前をつけることは役に立つ慣例です。

1つ特定の型を試すことはよく役立ちます。
`Droid`型を確認しましょう。

```graphql
{
  __type(name: "Droid") {
    name
  }
}
```

```json
{
  "data": {
    "__type": {
      "name": "Droid"
    }
  }
}
```

より`Droid`について知りたい場合は動詞たら良いでしょうか？
例えば、それはインターフェイスまたはオブジェクトでしょうか？

```graphql
{
  __type(name: "Droid") {
    name
    kind
  }
}
```

```json
{
  "data": {
    "__type": {
      "name": "Droid",
      "kind": "OBJECT"
    }
  }
}
```

`kind`は`__TypeKind`列挙型を返し、それらの値の1つは`OBJECT`です。
もし、代わりに`Character`について質問した場合、それがインターフェイスであることがわかります。

```graphql
{
  __type(name: "Character") {
    name
    kind
  }
}
```

```json
{
  "data": {
    "__type": {
      "name": "Character",
      "kind": "INTERFACE"
    }
  }
}
```

利用できるフィールドがなにか知ることはオブジェクトにとって役に立ちます。
よって、`Droid`についてイントロスペクションシステムに質問しましょう。

```graphql
{
  __type(name: "Droid") {
    name
    fields {
      name
      type {
        name
        kind
      }
    }
  }
}
```

```json
{
  "data": {
    "__type": {
      "name": "Droid",
      "fields": [
        {
          "name": "id",
          "type": {
            "name": null,
            "kind": "NON_NULL"
          }
        },
        {
          "name": "name",
          "type": {
            "name": null,
            "kind": "NON_NULL"
          }
        },
        {
          "name": "friends",
          "type": {
            "name": null,
            "kind": "LIST"
          }
        },
        {
          "name": "friendsConnection",
          "type": {
            "name": null,
            "kind": "NON_NULL"
          }
        },
        {
          "name": "appearsIn",
          "type": {
            "name": null,
            "kind": "NON_NULL"
          }
        },
        {
          "name": "primaryFunction",
          "type": {
            "name": "String",
            "kind": "SCALAR"
          }
        }
      ]
    }
  }
}
```

それらは`Droid`に定義したフィールドです。

`id`は少し奇妙に見え、それは方の名前を持っていません。
それは、種類が`NON_NULL`のラッパー型だからです。
もし、そのフィールドの肩を`ofType`でクエリした場合、そこに`ID`型を見つけ、これが非nullな`ID`であることを伝えます。

同様に、`friends`と`appearsIn`は、それらが`LIST`のラッパー型であるため、名前を持ちません。
それらの型に`ofType`でクエリでき、それはこれらがリストであることを伝えます。

```graphql
{
  __type(name: "Droid") {
    name
    fields {
      name
      type {
        name
        kind
        ofType {
          name
          kind
        }
      }
    }
  }
}
```

```json
{
  "data": {
    "__type": {
      "name": "Droid",
      "fields": [
        {
          "name": "id",
          "type": {
            "name": null,
            "kind": "NON_NULL",
            "ofType": {
              "name": "ID",
              "kind": "SCALAR"
            }
          }
        },
        {
          "name": "name",
          "type": {
            "name": null,
            "kind": "NON_NULL",
            "ofType": {
              "name": "String",
              "kind": "SCALAR"
            }
          }
        },
        {
          "name": "friends",
          "type": {
            "name": null,
            "kind": "LIST",
            "ofType": {
              "name": "Character",
              "kind": "INTERFACE"
            }
          }
        },
        {
          "name": "friendsConnection",
          "type": {
            "name": null,
            "kind": "NON_NULL",
            "ofType": {
              "name": "FriendsConnection",
              "kind": "OBJECT"
            }
          }
        },
        {
          "name": "appearsIn",
          "type": {
            "name": null,
            "kind": "NON_NULL",
            "ofType": {
              "name": null,
              "kind": "LIST"
            }
          }
        },
        {
          "name": "primaryFunction",
          "type": {
            "name": "String",
            "kind": "SCALAR",
            "ofType": null
          }
        }
      ]
    }
  }
}
```

道具として特に役に立つイントロスペクションシステムの機能で終わりましょう。
システムのドキュメントを質問しましょう。

```graphql
{
  __type(name: "Droid") {
    name
    description
  }
}
```

```json
{
  "data": {
    "__type": {
      "name": "Droid",
      "description": null
    }
  }
}
```

イントロスペクションを使用してタイプシステムに関するドキュメントにアクセスできるため、ドキュメントブラウザや機能が多いIDE体験を作成できます(So we can access the documentation about the type system using introspection, and create documentation browsers, or rich IDE experiences.)。

これは、イントロスペクションシステムの表面にフラタだけです。
列挙型の値、方が実装するインターフェイスなどをクエリできます。
イントロスペクションシステム自体をイントロスペクションすることもできます。
この仕様については、「イントロスペクション」セクションで詳しく説明されており、GraphQL.jsの[イントロスペクション](https://github.com/graphql/graphql-js/blob/main/src/type/introspection.ts)ファイルには、仕様に準拠したGraphQLクエリイントロスペクションシステムを実装するコードが含まれています(The specification goes into more detail about this topic in the “Introspection” section, and the introspection file in GraphQL.js contains code implementing a specification-compliant GraphQL query introspection system.)。
