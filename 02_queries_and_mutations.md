# クエリとミューテーション

このページでは、GraphQLサーバーにクエリする方法に付いて詳細を学びます。

## フィールド

最も単純に言えば、GraphQLはオブジェクトの特定のフィールドを要求するものです。
とても単純なクエリと、それを実行したときに得られる結果を確認することから始めましょう。

- クエリ

```graphql
{
  hero {
    name
  }
}
```

- 結果

```json
{
  "hero": {
    "name": "R2-D2"
  }
}
```

すぐにクエリは、結果と正確に同じ形状をしていることに気づけます。
予期するものが返ってきて、そしてサーバーは正確にクライアントが質問したフィールドを理解するため、これはGraphQLの本質です。

この場合、スターワーズのメインヒーロの名前は`R2-D2`で、`name`フィールドは`String`型を返します。

> もうひとつ、上記のクエリは相互作用します。
> それは、好きなようにクエリを変えて、新しい結果を確認できることを意味します。
> クエリの`hero`オブジェクトに`appearsIn`フィールドを追加して、新しい結果を確認してください。

前述の例において、単純に`String`を返すヒーロの名前を質問しましたが、フィールドはオブジェクトも参照できます。
その場合、そのオブジェクトようにフィールドの*サブセクション*を作成できます。
GraphQLクエリは、関連するオブジェクトとそれらのフィールドを横断することができ、従来のRESTアーキテクチャで必要となるいくつかのラウンドトリップの代わりに、クライアントが1つのリクエストで関連する多くのデータを取得できるようにします。

```graphql
{
  hero {
    name
    # クエリはコメントをサポートします！
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

この例において、`friends`フィールドはアイテムの配列を返すことに注意してください。
GraphQLクエリは、1つのアイテムまたはアイテムのリストの両方を同じように扱います。
しかし、スキーマに示されている内容に基づいて、どちらを予期しているかを知ることができます。

## 引数

オブジェクトとそれらのフィールドを横断することのみができる場合、すでにGraphQLはデータ取得用のとても便利な言語です。
しかし、フィールドに引数を渡す能力を追加したとき、とても興味深くなります。

```graphql
{
  human(id: "1000") {
    name
    height
  }
}
```

```json
{
  "data": {
    "human": {
      "name": "Luke Skywalker",
      "height": 1.72
    }
  }
}
```

RESTのようなシステムにおいて、リクエストのクエリパラメーターやURLセグメントで、引数の集合を1つのみ渡すことができます。
しかし、GraphQLでは、すべてのフィールドとネストされたオブジェクトは、それ独自の引数の集合を得ることができ、GraphQLは複数のAPIで取得することを完全に置き換えできます。
また、すべてのクライアントがそれぞれデータ変換する代わりに、サーバー側で一度データ変換を実装するために、スカラーフィールドにも引数を渡すことができます。

```graphql
{
  human(id: "1000") {
    name
    height(unit: FOOT)
  }
}
```

```json
{
  "data": {
    "human": {
      "name": "Luke Skywalker",
      "height": 5.6430448
    }
  }
}
```

引数は多くの異なる型にすることができます。
上記例では、有限の選択の集合の1つを表現する列挙型を使用しています。
この場合、長さの単位は`METER`または`FOOT`のいずれかです。
GraphQLは、デフォルトの型の集合が付属していますが、GraphQLサーバーは型を変換フォーマットにシリアライズすることができる限り、それ独自の型を宣言できます。

詳細は、[GraphQL型システム](https://graphql.org/learn/schema/)を参照してください。

## エイリアス

もし、観察眼のある人であれば、結果オブジェクトのフィールドがクエリのフィールドの名前に一致するが、引数が含まれていないため、異なる引数を使用して同じフィールドを直接クエリすることができないことに気付いたかもしれません。
これがエイリアスが必要な理由です。
エイリアスは、結果のフィールドの名前を任意に名前を変更できるようにします。

```graphql
{
  empireHero: hero(episode: EMPIRE) {
    name
  }
  jediHero: hero(episode: JEDI) {
    name
  }
}
```

```json
{
  "data": {
    "empireHero":{
      "name": "Luke Skywalker"
    },
    "jediHero": {
      "name": "R2-D2"
    }
  }
}
```

上記例において、2つの`hero`フィールドは衝突していますが、それらを異なる名前にエイリアスできるため、1つのリクエストで両方の結果を得られます。

## フラグメント

例えば、アプリに関連した複雑なページがあり、二人のヒーローとその友達を並べて見ることができたとします。
このようなクエリは、フィールドを少なくとも1回（比較の両側に1つ）繰り返す必要があるため、すぐに複雑になることを想像できます。

それが理由で、GraphQLは*フラグメント*と呼ばれる再利用可能な構成単位を含んでいます。
フラグメントは、フィールドの集合を構築して、クエリ内の必要な場所にそれらを含めることができます。
ここに、フラグメントを使用して上記の状況を解決する方法を示した例を示します。

```graphql
{
  leftComparison: hero(episode: EMPIRE) {
    ...comparisonFields
  }
  rightComparison: hero(episode: JEDI) {
    ...comparisonFields
  }
}

fragment comparisonFields on Character {
  name
  appearsIn
  friends {
    name
  }
}
```

> 同じクエリをしているため、エイリアスを使用している。

```json
{
  "data": {
    "leftComparison": {
      "name": "Luke Skywalker",
      "appearsIn": [
        "NEWHOPE",
        "EMPIRE",
        "JEDI"
      ],
      "friends": [
        {
          "name": "Han Solo"
        },
        {
          "name": "Leia Organa"
        },
        {
          "name": "C-3PO"
        },
        {
          "name": "R2-D2"
        }
      ]
    },
    "rightComparison": {
      "name": "R2-D2",
      "appearsIn": [
        "NEWHOPE",
        "EMPIRE",
        "JEDI"
      ],
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

上記のクエリが
もし、フィールドが繰り返されていた場合、どのように上記のクエリがかなり繰り返されることがわかります。
フラグメントの概念は、特に、異なるフラグメントを含む多くのUIコンポーネントを、1回の初期データ取得に結合する必要がある場合、複雑なアプリケーションデータを小さな塊に分割するためによく使用されます。

### フラグメント内で変数を使用する

フラグメントは、クエリまたはミューテーションで宣言された変数にアクセスできます。
[変数](https://graphql.org/learn/queries/#variables)を参照してください。

```graphql
query HeroComparison($first: Int = 3) {
  leftComparison: hero(episode: EMPIRE) {
    ...comparisonFields
  }
  rightComparison: hero(episode: JEDI) {
    ...comparisonFields
  }
}

fragment comparisonFields on Character {
  name
  friendsConnection(first: $first) {
    totalCount
    edges {
      node {
        name
      }
    }
  }
}
```

```json
{
  "data": {
    "leftComparison": {
      "name": "Luke Skywalker",
      "friendsConnection": {
        "totalCount": 4,
        "edges": [
          {
            "node": {
              "name": "Han Solo"
            }
          },
          {
            "node": {
              "name": "Leia Organa"
            }
          },
          {
            "node": {
              "name": "C-3PO"
            }
          }
        ]
      }
    },
    "rightComparison": {
      "name": "R2-D2",
      "friendsConnection": {
        "totalCount": 3,
        "edges": [
          {
            "node": {
              "name": "Luke Skywalker"
            }
          },
          {
            "node": {
              "name": "Han Solo"
            }
          },
          {
            "node": {
              "name": "Leia Organa"
            }
          }
        ]
      }
    }
  }
}
```

## 操作名

上記例のいくつかは、短縮構文を使用しており、`query`キーワードとクエリ名を省略していますが、プロダクションアプリでは、コードの曖昧さを減らすために、これらを使用することは便利です。

*操作型*として`query`キーワード、*操作名*として`HeroNameAndFriends`を含む例をここに示します。

```graphql
query HeroNameAndFriends {
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
        }
      ]
    }
  }
}
```

*操作型*は*query*、*mutation*、または*subscription*のいずれかで、操作名はすることを意図する操作の型を説明します。
クエリ短縮構文を使用している場合を除き、操作タイプは必須です。
クエリ短縮構文を使用している場合、操作名や変数定義を指定することはできません。

*操作名*は、操作にとって意味があり明示的な名前です。
操作名はドキュメントの複数操作で要求されますが、その使用はデバッグとサーバーサイドのログの記録にとても役立つため、その使用を推奨します。
ネットワークログまたはGraphQLサーバーのログで何かが悪くなったとことを確認したとき、コンテンツを解読することを試みる代わりに、名前でコードベース内のクエリを識別することを簡単にします。
単にこれを好きなプログラミング言語の関数名と考えてください。
例えば、JavaScriptにおいて、無名関数を簡単に機能させることができますが、関数に名前を与えたとき、それを追跡する、コードをデバッグする、そしてそれが呼び出されたときをログに記録することが容易になります。
それと同じ方法で、GraphQLクエリとミューテーション名、それに付随したフラグメント名は、サーバーサイドで様々なGraphQLリクエストを識別するための便利なデバッグツールになります。

## 変数

これまで、クエリ文字列の中にすべての引数を記述してきました。
しかし、ほとんどのアプリケーションでは、フィールドへの引数は動的です。
例えば、興味のあるスターウォーズのエピソードを選択するドロップダウン、または検索フィールド、またはフィルターの集合があるかもしれません。

クライアントサイドはランタイムでクエリ文字列を動的に操作して、それをGraphQL特別な書式にシリアライズする必要があるため、クエリ文字列の中に直接これら動的な引数を渡すことは良い考えではありません。
代わりに、GraphQLには、動的な値の要素をクエリの外側に置き、分離した辞書としてそれらを渡す最高級な方法があります(Instead, GraphQL has a first-class way to factor dynamic values out of the query, and pass them as a separate dictionary)。
これらの値は、*変数*と呼ばれます。

変数を使用するとき、次の3つのことを行う必要があります。

1. クエリ内の静的な値を`$variableName`で置き換えます。
2. クエリによって受け付けられる変数の1つとして、`$variableName`を宣言します。
3. 通常、JSONなど、分離して輸送に特化した変数の辞書で`variableName: value`を渡します。

ここに、すべてがどのようになるか示します。

```graphql
# クエリ
query HeroNameAndFriends($episode: Episode) {
  hero(episode: $episode) {
    name
    friends {
      name
    }
  }
}
```

```graphql
# 変数
{
  "episode": "JEDI"
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

現在、クライアントコードにおいて、完全に新しいクエリを構築する必要なく、単に異なる変数を渡せます。
また、通常、これはクエリのどの変数が動的であるかを予期されていることを示す、良い実践です。
決して、ユーザーから提供された値からクエリを構築するために文字列を補間するべきではありません。

### 変数の定義

変数の定義は、上記クエリの`($episode: Episode)`の部分です。
単にそれは、型付けされた言語における関数の変数宣言のように機能します。
それは、`$`でプレフィックスされ、その後に変数の型が続くすべての変数をリスとします（この場合は`Episode`）。

すべての宣言された変数は、スカラー、列挙型または入力オブジェクトのいずれかでなければなりません。
よって、もしフィールドに複雑なオブジェクトを渡す場合、サーバーにマッチする入力型が何か知る必要があります。
入力オブジェクトの型については、[Schema](https://graphql.org/learn/schema/)のページを参照してください。

変数定義はオプションまたは必須です。
上記の場合、`Episode`の次に`!`がないため、それはオプションです。
しかし、もし変数を渡すフィールドが非null引数を要求する場合、変数も必須にする必要があります。

これら変数定義構文に関する詳細を学ぶために、[GraphQLスキーマ言語](https://graphql.org/learn/schema/)を学ぶことは役に立ちます。
スキーマ言語は、[スキーマと型](https://graphql.org/learn/schema/)で詳細に説明されています。

### デフォルト値

デフォルト値は、型宣言の後にデフォルト値を追加することで、クエリの変数を割り当てれます。

```graphql
query HeroNameAndFriends($episode: Episode = JEDI) {
  hero(episode: $episode) {
    name
    friends {
      name
    }
  }
}
```

すべての変数にデフォルト値が提供されたとき、任意の変数を渡すことなくクエリを呼び出しできます。
もし、変数の辞書の一部として変数が渡された場合、それらはデフォルトを上書きします。

### ディレクティブ

> ディレクティブ (directive)は、指示や命令を意味する。

動的なクエリを構築するために手動で文字列を補間することを避けるために変数をゆう呼応にする方法を議論しました。
引数に変数を渡すことは、これらの問題のとても大きな部分を解決しますが、変数を使用してクエリの構造と計上を動的に変更する必要があるかもしれません。
例えば、要約と詳細なビューを持つUIコンポーネントを考えることができ、一方は他方よりも多くのフィールドを含みます。

そのようなコンポーネントのクエリを構築しましょう。

```graphql
# クエリ
query Hero($episode: Episode, $withFriends: Boolean!) {
  hero(episode: $episode) {
    name
    friends @include(if: $withFriends) {
      name
    }
  }
}
```

```graphql
# 変数
{
  "episode": "JEDI",
  "withFriends", false
}
```

```json
{
  "data": {
    "hero": {
      "name": "R2-D2"
    }
  }
}
```

上記を代わりに`withFriends`を`true`で渡すために変数を編集して、どのように結果が変わるか確認してください。

*ディレクティブ*と呼ばれるGraphQLの新しい機能を使用する必要がありました。
ディレクティブはフィールドに付属またはフラグメントに含まれて、サーバーが望むあらゆる方法で、クエリの実行を影響を与えれます。
GraphQL仕様のコアは正確に2つのディレクトリを含んでおり、任意の互換性のあるGraphQLサーバーの実装によってサポートされる必要があります。

- `@include(if: Boolean)`: 引数が`true`の場合にのみ結果内にこのフィールドが含まれます。
- `@skip(if: Boolean)`: 引数が`true`の場合にこのフィールドがスキップされます。

ディレクティブは、クエリ内でフィールドを追加または削除するために文字列を操作することが必要な状況を回避するために役に立ちます。
サーバーの実装は、完全に新しいディレクティブを定義することで、実験的な機能を追加できるようになっているかもしれません。

## ミューテーション

GraphQLのほとんどの議論はデータ取得に焦点を当てますが、任意の完全なデータプラットフォームは、同様にサーバー側のデータを修正する方法が必要です。

RESTに置いて、任意のリクエストはサーバーで任意の副作用を発生して終了するかもしれませんが、慣例で、データを修正するために`GET`リクエストを使用しないように推奨されています。
GraphQLも同様で、技術的に任意のクエリはデータの書き込みを起こすように実装させることもできます。
しかし、書き込みを発生する任意の操作は明示的にミューテーションを介して送信されるべきであることを、慣例として確立することは役に立ちます。

ちょうどクエリのように、ミューテーションのフィールドが`ObjectType`を返す場合、ネストしたフィールドを質問できます。
これは、更新の後でオブジェクトの新しい状態を取得するために役に立ちます。

> 厳密にはコマンド／クエリ分離の原則(CQS)従っていないが、コマンドはミューテーション、クエリはクエリとして分離されている。
> ラウンドトリップを増やすよりも、絶対的に便利である。

ミューテーションの簡単な例を確認しましょう。

```graphql
# ミューテーション
mutation CreateReviewForEpisode($ep: Episode!, $review: ReviewInput!) {
  createReview(episode: $ep, review: $review) {
    stars
    commentary
  }
}
```

```graphql
# 変数
{
  "ep": "JEDI",
  "review": {
    "stars": 5,
    "commentary": "This is a great movie!"
  }
}
```

```json
{
  "data": {
    "createReview": {
      "stars": 5,
      "commentary": "This si a great movie!"
    }
  }
}
```

`createView`フィールドは、新しく作成されたレビューの`stars`と`commentary`フィールドをどのように返すか注意してください。
これは、例えばフィールドが増加するとき、1つのリクエストでフィールドの新しい値を変更してクエリできるため、既存のデータを変更するとき特に役に立ちます。

また、この例で気付いたかもしれませんが、渡した`review`変数はスカラーではありません。
それは*入力オブジェクト型*で、引数として渡すことができる`ObjectType`の特別な種類です。
入力型の詳細は、[スキーマ](https://graphql.org/learn/schema/)ページで学んでください。

### ミューテーション内の複数フィールド

ミューテーションは、ちょうどクエリのように、複数のフィールドを含めます。
クエリとミューテーションには、名前以外に重要な違いが1つあります。

**クエリのフィールドは並列に実行される一方で、ミューテーションのフィールドは次々順番に実行します。**

これは、もし1つのリクエストで2つの`incrementCredits`ミューテーションを送信した場合、最初は2つ目が開始する前に終了することを保証され、自分自身で競合状態に終わることがないことを確信できることを意味します。

### インラインフラグメント

多くの他の型システムのように、GraphQLスキーマはインターフェイスとユニオン型を定義するの応力を含んでいます。
それらについては、[スキーマガイド](https://graphql.org/learn/schema/#interfaces)で学んでください。

もし、インターフェイスまたはユニオン型を返すフィールドをクエリした場合、具体的な型に基づいたデータにアクセスするために*インラインフラグメント*を使用する必要があります。
例を見たほうが簡単です。

```graphql
# クエリ
query HeroForEpisode($ep: Episode!) {
  hero(episode: $ep) {
    name
    ... on Droid {
      primaryFunction
    }
    ... on Human {
      height
    }
  }
}
```

```graphql
# 変数
{
  "ep": "JEDI"
}
```

```json
{
  "data": {
    "hero": {
      "name": "R2-D2",
      "primaryFunction": "Astromech"
    }
  }
}
```

このクエリにおいて、`hero`フィールドは`Character`型を返し、それは`episode`引数に依存して`Human`または`Droid`のどちらかになります。
直接選択では、`name`のような`Character`インターフェイスに存在するフィールドのみ質問できます。

具体的な型のフィールドを質問するために、型条件を付けて*インラインフラグメント*を使用する必要があります。
最初のフラグメントは`... on Droid`としてレベルされているため、`primaryFunction`フィールドは、`Character`が`Droid`型の`hero`を返した場合にのみ実行されます。
どうゆおうに、`height`フィールドは`Human`型の場合のみです。

また、名付けられたフラグメントは、常に型に付属しているため、同様に使用されます。

### メタフィールド

GraphQLサービスから返される型がわからない状況がいくつかあることを考慮すると、クライアントでデータを処理する方法を決定する何らかの方法が必要です。
GraphQLは、任意の時点でメタフィールドである`__typename`をリクエストすることで、その時点の`ObjectType`の名前を得られます。

```graphql
# クエリ
{
  search(text: "an") {
    __typename
    ... on HUman {
      name
    }
    ... on Droid {
      name
    }
    ... on Starship {
      name
    }
  }
}
```

```json
{
  "data": {
    "search": [
      {
        "__typename": "Human",
        "name": "Han Solo"
      },
      {
        "__typename": "Human",
        "name": "Leia Organa"
      },
      {
        "__typename": "Starship",
        "name": "TIE Advanced x1"
      }
    ]
  }
}
```

上記クエリにおいて、`search`は3つのオプションの内の1つになりえるユニオン型を返します。
`__typename`フィールド無しでクライアントから異なる型を見分けることは不可能です。

GraphQLサービスはいくつかのメタフィールドを提供しており、それらの残りはイントロスペクションシステムを公開するために使用されます。

> イントロスペクション (introspection): 実行時にオブジェクトの型や性質を調査すること
