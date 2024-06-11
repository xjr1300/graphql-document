# 実行

<https://graphql.org/learn/execution/>

- [実行](#実行)
  - [ルートフィールドとリゾルバー](#ルートフィールドとリゾルバー)
  - [非同期リゾルバー](#非同期リゾルバー)
  - [取るに足らないリゾルバー](#取るに足らないリゾルバー)
  - [スカラー強制](#スカラー強制)
  - [リストリゾルバー](#リストリゾルバー)
  - [結果の生成](#結果の生成)

検証された後、GraphQLクエリはGraphQLサーバーによって実行され、サーバーはリクエストされたクエリの形状を鏡で写した結果を、典型的にJSONとして返します。

GraphQLは型システム無しでクエリを実行できないため、クエリの実行を想像するために例となる型システムを使用しましょう。
これは、これらの記事の例を通じて使用されている同じ型システムの一部です。

```graphql
type Query {
  human(id: ID!): Human
}

type Human {
  name: String
  appearsIn: [Episode]
  starships: [Starship]
}

enum Episode {
  NEWHOPE
  EMPIRE
  JEDI
}

type Starship {
  name: String
}
```

クエリが実行されたときに何が発生するか説明するために、例を使用して順番に説明しましょう。

```graphql
{
  human(id: 1002) {
    name
    appearsIn
    starships {
      name
    }
  }
}
```

```json
{
  "data": {
    "human": {
      "name": "Han Solo",
      "appearsIn": [
        "NEWHOPE",
        "EMPIRE",
        "JEDI"
      ],
      "starships": [
        {
          "name": "Millennium Falcon"
        },
        {
          "name": "Imperial shuttle"
        }
      ]
    }
  }
}
```

GraphQLクエリ内のそれぞれのフィールドを、次の型を返す前の型の関数またはメソッドとして考えることができます。

> 前の型である`human`は、次の型である`name`、`appearsIn`、`starships`を返す。
> GraphQLの***フィールド***は、Pythonにおけるクラスの属性と考えることができる。
> Pythonのクラスの属性は、データ（属性）と、クラスやインスタンスを操作するメソッド（操作）が含まれる。
> Pythonの`str`型の属性は、`dir(str)`または`str.__dict__`で確認できる。
> GraphQLは、データ型を表現する`ObjectType`のフィールドと、`Query`や`Mutation`のような特殊な`ObjectType`を表現して、クエリ結果を返したり、データを変更するフィールドを持つ。

実際に、これがGraphQLが動作する正確な方法です。
それぞれの型のそれぞれのフィールドは、GraphQLサーバーの開発者によって提供された*リゾルバー*と呼ばれる関数によって裏付け（be backed: サポート）されています。
フィールドが実行されたとき、対応する*リゾルバー*は次の値を生成するために呼び出されます。

フィールドが文字列や数字のようなスカラー値を生成した場合、それは実行の完了です。
しかし、フィールドがオブジェクト型の値を生成した場合、クエリはそのオブジェクトに適用されたフィールドの他の選択が含まれます。

> オブジェクトのフィールドはネストされるため、次のネストのフィールドの値を生成することを説明している。

これはスカラー値にたどり着くまで続きます。
常に、GraphQLはスカラー値で終了します。

## ルートフィールドとリゾルバー

すべてのGraphQLサーバーの最上位は、GraphQL APIの可能性のあるすべてのエントリポイントを表現する型で、それはよく*ルートタイプ*または*クエリタイプ*と呼ばれます。

この例において、クエリタイプは`id`引数を受け付ける`human`と呼ばれるフィールドを提供します。
このフィールドのリゾルバー関数は、データベースにアクセスして、`Human`オブジェクトを構築して返します。

```javascript
Query: {
  human(obj, args, context, into) {
    return context.db.loadHumanByID(args.id).then(
      userData => new Human(userData)
    )
  }}
```

この例は、JavaScriptで記述されていますが、GraphQLサーバーは[多くの様々な言語](https://graphql.org/code/)で構築されています。
リゾルバー関数は、4つの引数を受け付けます。

- `obj`: ルートクエリタイプのフィールドを示す前のオブジェクトで、あまり使用されません(The previous object, which for a field on the root Query type is often not used)。
- `args`: GraphQLクエリでフィールドに提供された引数です。
- `context`: すべてのリゾルバーに提供され、現在ログインしているユーザー、またはデータへのアクセスのような重要で文脈に応じた情報を保持した値です。
- `info`: 現在のクエリに関連するフィールド特有の情報及びスキーマの詳細を保持する値で、詳細は[GraphQLResolveInfo型](https://graphql.org/graphql-js/type/#graphqlobjecttype)を参照してください。

## 非同期リゾルバー

このリゾルバー関数で何が発生するか詳細に確認しましょう。

```javascript
human(obj, args, context, info) {
  return context.db.loadHumanByID(args.id).then(
    userData => new Human(userData)
  )
}
```

`context`は、GraphQLクエリで引数として提供された`id`でユーザーデータをロードするために、データベースへのアクセスを提供するために使用されます。
データベースからのローディングは非同期動作であるため、これは[Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise)を返します。

> 上記JavaScriptの例では、`then`で`Promise`が成功裏に解決されるまで待機している。

JavaScriptに置いて、`Promise`は非同期な値と一緒に機能するために使用されますが、多くの言語で同じ概念が存在しており、よく*Future*、*Task*、または*Deferred*と呼ばれます。
データベースから戻ったとき、新しい`Human`オブジェクトを構築して返します。

リゾルバー関数が`Promise`を認識する必要がある一方で、GraphQLクエリはそうではないことに注意してください。
単純に、GraphQLクエリは、その`name`を質問できる何かを返すことを`human`フィールドを期待しています。
実行の間、GraphQLは`Promise`、`Future`そして`Task`が完了するまで待機してから続行することを、最適化された同時並行性で行います(During execution, GraphQL will wait for Promises, Futures, and Tasks to complete before continuing and will do so with optimal concurrency.)。

## 取るに足らないリゾルバー

現在、`Human`オブジェクトが利用できるようになったので、GraphQLの実行はそのオブジェクトでリクエストされたフィールドを続行できます。

> GraphQLにおいて、`field`は動詞のように扱える。

```javascript
Human: {
  name(obj, args, context, info) {
    return obj.name
  }
}
```

GraphQLサーバーは、次に何をするかを決定するために使用される型システムによって強化されています。
まだ`human`フィールドが何かを返す前に、型システムが`human`フィールドが`Human`を返すことをGraphQLに伝えるため、GraphQLは次のステップが`Human`型のフィールドを解決することであることを理解しています。

この場合、`name`を解決することはとても簡単です。
`name`リゾルバー関数が呼び出され、前のフィールドから返された`obj`引数は新しい`Human`オブジェクトです。
この場合、直接読み取られて返される`name`プロパティを持っていることを`Human`オブジェクトに期待しています。

> 前のフィールドである`human`フィールドのクエリは、`human`リゾルバーの呼び出しになり`Human`オブジェクトを返す。
> また、`human`フィールドは、`name`フィールドをクエリするため、それは`name`リゾルバーの呼び出しになり、その引数`obj`は`Human`オブジェクトになる。

実際、多くのGraphQLライブラリでは、この単純なリゾルバーを省略でき、フィールドにリゾルバーが提供されていない場合は、同じ名前のプロパティが読み取られて返されると想定します。

> フィールドにリゾルバー関数が提供されていない場合は、スカラー値であると想定する。

## スカラー強制

`name`フィールドが解決されている間、`appearsIn`と`starships`フィールドは同時並行で解決されます。
`appearsIn`フィールドは些細なリゾルバーも持っていますが、詳細に確認しましょう。

```javascript
Human: {
  appearsIn(obj) {
    return obj.appearsIn  // [4, 5, 6]を返します。
  }
}
```

型システムは、`appearsIn`が既知の値で列挙型の値を返すと主張しますが、この関数は複数の数値を返していることに注意してください。
確かに、もし結果を確認したとき、適切な列挙型の値が返されていることを確認できます。
何が起こっているのでしょうか？

これがスカラー強制の例です。
タイプシステムは、何を期待されているかを理解しており、リゾルバー関数から返された値をAPI契約を遵守した何かに変換します。
この場合、内部で`4`、`5`そして`6`のような数値を使用する列挙型が定義されている可能性があり、GraphQL型システムにおいてそれらを列挙型の値として表現します(there may be an Enum defined on our server which uses numbers like 4, 5, and 6 internally, but represents them as Enum values in the GraphQL type system.)。

## リストリゾルバー

上記で`appearsIn`フィールドを使用して、リストが返されたときに何が発生するかすでに少し確認しました。
列挙型の値のリストが返され、型システムがそれを予期していたため、リスト内のそれぞれの項目は適切な列挙型の値に強制されました。
`starships`フィールドが解決されたとき、何が発生するでしょうか？

```javascript
Human: {
  starships(obj, args, context, info) {
    return obj.starshipIDs.map(
      id => context.db.loadStarshipByID(id).then(
        shipData => new Starship(shipData)
      )
    )
  }
}
```

このフィールドのリゾルバーは、`Promise`を返さず、`Promise`のリストを返します。
`Human`オブジェクトは、彼らが操縦した`Starship`のIDのリストを持っていますが、実際の`Starship`オブジェクトを取得するためにそれらすべてのIDをロードする必要があります。

GraphQLは、続行する前に同時並行でこれらすべての`Promise`を待機して、オブジェクトのリストを残します。
GraphQLは、続行する前にこれらの`Promise`をすべて同時並行で待機し、オブジェクトのリストが残っている場合は、これらそれぞれアイテムの`name`フィールドのロードを再度同時並行で続行します。

## 結果の生成

それぞれのフィールドが解決されたため、その結果の値はキーバリュー型のマップに、キーとしてフィールド名またはエイリアス、そして値として解決された値を配置されます。

これは、クエリの1番下のリーフフィールドから、ルートクエリタイプのオリジナルなフィールドまで続きます。
これらをまとめて、元のクエリを鏡に写した構造を生成して、それを要求したクライアントに、通常はJSONとして送信できます。

最後にもう1度元のクエリを確認して、これらすべての解決関数がどのように結果を生成するかを確認しましょう。

```graphql
{
  human(id: 1002) {
    name
    appearsIn
    starships {
      name
    }
  }
}
```

```json
{
  "data": {
    "human": {
      "name": "Han Solo",
      "appearsIn": [
        "NEWHOPE",
        "EMPIRE",
        "JEDI"
      ],
      "starships": [
        {
          "name": "Millenium Falcon"
        },
        {
          "name": "Imperial shuttle"
        }
      ]
    }
  }
}
```
