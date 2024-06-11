# スキーマと型

<https://graphql.org/learn/schema/>

- [スキーマと型](#スキーマと型)
  - [型システム](#型システム)
  - [型言語](#型言語)
  - [ObjectTypeとフィールド](#objecttypeとフィールド)
  - [引数](#引数)
  - [クエリとミューテーション型](#クエリとミューテーション型)
  - [スカラー型](#スカラー型)
  - [列挙型](#列挙型)
  - [リストと非null](#リストと非null)
  - [インターフェイス](#インターフェイス)
  - [ユニオン型](#ユニオン型)
  - [入力型](#入力型)

このページでは、GraphQLの型システムについて知る必要があるすべてのことと、GraphQLがどのデータがクエリされる可能性があるかを説明する方法を学びます。
GraphQLは、任意のバックエンドフレームワークまたはプログラミング言語といっしょに使用されるため、特定の実装の詳細から離れて、概念のみを説明します。

## 型システム

前にGraphQLのクエリを見たことがある場合、GraphQLのクエリ言語は基本的にオブジェクト上のフィールドを選択することを目的としています。
よって、例えば、次のクエリにおいて・・・

```graphql
{
  hero {
    name
    appearsIn
  }
}
```

```json
{
  "data": {
    "hero": {
      "name": "R2-D2",
      "appearsIn": [
        "NEWHOPE",
        "EMPIRE",
        "JEDI"
      ]
    }
  }
}
```

1. 特別な「root」オブジェクト(`{}`)で開始します。
2. その`hero`フィールドを選択します。
3. `hero`によって返されたオブジェクトの、`name`と`appearsIn`

GraphQLのクエリの形状は、結果とかなり一致するため、サーバーを理解することなしで、何をクエリが返したか予測できます。
しかし、どのフィールドを選択できるのか、どの種類のオブジェクトが返されるのかなど、質問できるデータの正確な詳細を知ることは役に立ちます。
そこでスキーマが登場します。

すべてのGraphQLサービスは、サービスでクエリ可能なデータの集合を完全に説明する型の集合を定義しています。
そして、クエリが登場したとき、それらはスキーマに対して検証されて実行されます。

## 型言語

GraphQLサービスは任意の言語で記述できます。
JavaScriptのような特定のプログラミング言語の構文に依存できないため、GraphQLについて話すために独自の単純な言語を定義しました。
「GraphQLスキーマ言語」を使用します。
GraphQLスキーマ言語はクエリ言語ととても似ていて、言語に依存しない方法でGraphQLスキーマにつて語ることをできるようにします。

## ObjectTypeとフィールド

GraphQLスキーマの最も基本的な構成物は`ObjectType`で、ちょうどそれはサービスから取得できるオブジェクトの種類と、どのフィールドを持っているかを表現します。
GraphQLスキーマ言語で、それを次のように表現します。

```graphql
type Character {
  name: String!
  appearsIn: [Episode!]!
}
```

言語はかなり読みやすいですが、共通の語彙を持つために言語を調べてみましょう。

- `Character`はGraphQLの`ObjectType`で、それはいくつかのフィールドを持つ型です。スキーマ内のほとんどの型は`ObjectType`です。
- `name`と`appearsIn`は`Character`型の*フィールド*です。それは、`Character`型を操作するGraphQLクエリの任意の部分に現れるフィールドが`name`と`appearsIn`だけであることを意味します。
- `String`はビルトインされたスカラー型の1つで、スカラー型は単独のスカラーオブジェクトに解決される型で、クエリ内にサブセクション(`sub-sections`)を持てません。後で詳細にスカラー型を調査します。
- `String!`はそのフィールドが非nullであることを意味しており、GraphQLサービスは、このフィールドをクエリしたとき、常に値を与えることを約束します。型言語において、それらをエクスクラメーションマーク(`!`)で表現します。
- `[Episode!]!`は、`Episode`オブジェクトの配列を表現します。またそれは非nullで、`appearsIn`フィールドをクエリしたとき、0個またはそれ以上のアイテムを持つ配列であることを予期できます。そして、`Episode!`もまた非nullであるため、いつでも配列内のアイテムを`Episode`オブジェクトにできることを予期できます。

現在、GraphQLの`ObjectType`がどのようなものか、基本的なGraphQLの型言語を読む方法を理解しました。

> サブセクションを持てないということは、スカラー型のフィールドに対してクエリを実行すると、そのフィールドが単一のスカラー値に解決されることを意味する。
> つまり、オブジェクトのように複合型に解決されることはない。

## 引数

GraphQLの`ObjectType`にあるすべてのフィールドは、0個またはそれ以上の引数を持つことができ、例えば`length`フィールドを次のとおりです。

```graphql
type Starship {
  id: ID!
  name: String!
  length(unit: LengthUnit = METER): Float
}
```

すべての引数は名前が付けられています。
関数が順番付けられた引数のリストを受け取るJavaScriptやPythonと異なり、GraphQLのすべての引数は、名前によって具体的に渡されます。
この場合、`length`フィールドは`unit`という1つの定義された引数を持ちます。

引数は必須またはオプションどちらにもなりえます。
引数がオプションのとき、デフォルト値を定義できます。
もし`unit`引数が渡されない場合、デフォルトでそれは`METER`に設定されます。

## クエリとミューテーション型

スキーマ内のほとんどの型は、普通の`ObjectType`ですが、スキーマ内に特別な2つの型があります。

```graphql
schema {
  query: Query
  mutation: Mutation
}
```

すべてのGraphQLサービスは`query`型と、もしかしたら`mutation`型を持っています。
これらの型は通常の`ObjectType`と同じですが、すべてのGraphQLクエリのエントリポイントを定義する為特別です。
よって、次のようなクエリを確認した場合・・・

```graphql
query {
  hero {
    name
  }
  droid(id: "2000") {
    name
  }
}
```

```json
{
  "data": {
    "hero": {
      "name": "R2-D2"
    },
    "droid": {
      "name": "C-3PO"
    }
  }
}
```

それは、GraphQLサービスが、`hero`と`droid`フィールドを持つ`Query`型を保つ必要があることを意味します。

```graphql
type Query {
  hero(episode: Episode): Character
  droid(id: ID!): Droid
}
```

> `hero`フィールドの`episode`引数がオプションであることに注意すること。
> また、`droid`フィールドの`id`引数が必須であることに注意すること。

ミューテーションも同様に機能します。
`Mutation`型のフィールドを定義すると、それらフィールドは、クエリで呼び出しできる最上位のミューテーションフィールド(`root mutation fields`)です。

> 最上位のミューテーションフィールドとは、上記`Query`型の`hero`または`droid`フィールドのように、ミューテーションの直下にあるフィールドを示す。
> 次の場合、`createReview`は最上位のミューテーションフィールドである。
>
> ```graphql
> type Mutation {
>   createReview(episode: Episode, review: ReviewInput!): Review
> }
> ```

他よりも覚えておくべき重要なことは、スキーマへの「エントリポイント」である特別な状態を除けば、`Query`と`Mutation`型は、他の任意のGraphQLの`ObjectType`と同じで、それらのフィールドは正確に同じ方法で機能することです。

## スカラー型

GraphQLの`ObjectType`は名前とフィールドを持ちますが、任意の時点で、それらのフィールドは任意の具体的なデータに解決される必要があります。
そこでスカラー型が登場します。
スカラー型はクエリのリーフを表現します。

> リーフ: 木構造のノードで、子ノードを持たないノードであるため、具体的な値を持つ。

次のクエリにおいて、`name`と`appearsIn`フィールドはスカラー型に解決されます。

```graphql
{
  hero {
    name
    appearsIn
  }
}
```

```json
{
  "data": {
    "hero": {
      "name": "R2-D2",
      "appearsIn": [
        "NEWHOPE",
        "EMPIRE",
        "JEDI"
      ]
    }
  }
}
```

`name`と`appearsIn`フィールドは任意のサブフィールドを持たないことを知っているため、それらはクエリのリーフです。

GraphQLには、標準でデフォルトのスカラー型の集合がついています。

- `Int`: 符号付き32ビット整数値
- `Float`: 符号付き倍精度浮動小数点値
- `String`: UTF-8文字シーケンス
- `Boolean`: `true`または`false`
- `ID`: `ID`スカラー型は一意な識別子を表現して、オブジェクトを取得するため、またはキャッシュのキーとして、よく使用されます。
  しかし、`ID`としてそれを定義することは、人が読みやすいように意図されていないことを意味します。

また、ほとんどのGraphQLサービスの実装において、独自なスカラー型を記述する方法があります。
例えば、`Date`型を定義できます。

```graphql
scalar Date
```

そして、その型がシリアライズ、デシリアライズそして検証される方法を定義することは実装次第です(Then it’s up to our implementation to define how that type should be serialized, deserialized, and validated.)。
例えば、常に`Date`型は整数のタイムスタンプにシリアライズされるように記述して、クライアントが任意の`Date`型フィールドについてそのフォーマットを予期することを知る必要があります。

## 列挙型

また、`Enum`と呼ばれる列挙型は許可された値の特定な集合に制限される特別な種類のスカラーです。
列挙型は次をできるようにします。

1. この型の引数が許可された値の1つであるか検証します。
2. 型システムを経由して、常にフィールドが値の有限集合の1つであることを伝えます。

ここに、GraphQLスキーマ言語で列挙型の定義は次のようになります。

```graphql
enum Episode {
  NEWHOPE
  EMPIRE
  JEDI
}
```

これは、スキーマで`Episode`型を使用するときはいつでも、それが正確に`NEWHOPE`、`EMPIRE`または`JEDI`の1つであることを予期することを意味します。

さまざまな言語によるGraphQLサービスの実装は、列挙型を扱う言語独自の方法を持っていることに注意してください。
第1級市民として列挙型をサポートする言語では、その列挙型を利用するかもしれません。
JavaScriptのような列挙型をサポートしていない言語では、これらの値は内部的に整数値の集合にマップされるかもしれません。
しかし、これらの詳細はクライアントに漏れないため、クライアントは完全に列挙型の値の文字列の名前に関して操作できます。

> ```graphql
> enum Episode {
>   NEWHOPE
>   EMPIRE
>   JEDI
> }
>
> type Query {
>   hero(episode: Episode!): String
> }
> ```
>
> ```javascript
> client.query({
>   query: GET_HERO,
>   variables: { episode: 'EMPIRE' }  // 列挙型の値を文字列で扱う
> }).then(response => {
>   console.log(response.data.hero);
> });
> ```

## リストと非null

GraphQLで定義できる型の種類は、`ObjectType`、スカラーそして列挙型のみです。
しかし、スキーマの別の部分またはクエリ変数の宣言で型を使用するとき、それらの値の検証に影響を与える追加的な型修飾子を適用できます。
例を確認しましょう。

```graphql
type Character {
  name: String!
  appearsIn: [Episode]!
}
```

ここで、`String`型を使用して、エクスクラメーションマーク(`!`)を型名の後に追加することで非nullとしてそれをマークしています。
これは、常にサーバーがこのフィールドを非null値を返すことを予期しており、もし、実際にGraphQLの実行エラーをトリガーするnull値を最終的に得た場合、クライアントに何かが悪くなったことを知らせます。

また、非null型修飾子は、フィールドの引数を定義するときにも仕様され、もし、GraphQL文字列または変数に、null値が引数として渡された場合、サーバーが検証エラを返します。

```graphql
# クエリ
query DroidById($id: ID!) {
  droid(id: $id) {
    name
  }
}
```

```json
// 引数
{
  "id": null
}
```

```json
// 結果（検証エラー）
{
  "errors": [
    {
      "message": "Variable \"$id\" of non-null type \"ID!\" must not be null.",
      "locations": [
        {
          "line": 1,
          "column": 17
        }
      ]
    }
  ]
}
```

リストも同様に機能します。
`List`として型をマークするための型修飾子を試用でき、それはこのフィールドがその型の配列を返すことを示します。
スキーマ言語において、これは、`[`と`]`の角括弧で型をラップすることで示されます。
それは、引数でも同じように機能して、検証ステップでその値が配列であることを予期します。

非nullとリスト修飾子は組み合わせられます。
例えば、非null文字列のリストを持てます。

```graphql
myField: [String!]
```

これは、リスト自身がnullになることができますが、nullメンバーを持つことができません。例えば、JSONでは・・・

```json
myField: null   // 妥当
myField: []     // 妥当
myField: ["a", "b"]  // 妥当
myField: ["a", null, "b"]  // エラー、不適合
```

ここで、文字列の非nullリストを定義しましょう。

> 文字列と`null`を要素として許容するリストのこと。

```graphql
myField: [String]!
```

これは、リスト自身はnullになることができませんが、null値を含むことができます。

```json
myField: null   // エラー、不適合
myField: []     // 妥当
myField: ["a", "b"]  // 妥当
myField: ["a", null, "b"]  // 妥当
```

必要に従って、任意の数の非nullとリスト修飾子を任意にネストできます。

## インターフェイス

多くの型システムと同様に、GraphQLはインターフェイスをサポートします。
*インターフェイス*は、インターフェイスを実装するために、に含める必要がある特定のフィールドの集合を含む抽象型です。

例えば、スターウォーズ三部作の任意のキャラクターを表現する`Character`インターフェイスを持つ可能性があります。

```graphql
interface Character {
  id: ID!
  name: String!
  friends: [Character]
  appearsIn: [Episode]!
}
```

これは、`Character`を実装する任意の形は、これらの引数と戻り値の型を備えたフィールドを正確に持つ必要があることを意味します。

例えば、ここに`Character`を実装する型をいくつか示します。

```graphql
type Human implements Character {
  id: ID!
  name: String!
  friends: [Character]
  appearsIn: [Episode]!
  starship: [Starship]
  totalCredit: Int
}

type Droid implements Character {
  id: ID!
  name: String!
  friends: [Character]
  appearsIn: [Episode]
  primaryFunction: String
}
```

これらの型両方が`Character`インターフェイスのフィールドをすべて持っていることを確認できますが、`totalCredits`、`startships`そして`primaryFunction`の追加フィールドを導入して、それらはキャラクターに固有なものです。

インターフェイスは、異なる型になるオブジェクトまたはオブジェクトの集合を返したいときに便利です。

例えば、次のクエリはエラーを生成することに注意してください。

```graphql
query HeroForEpisode($ep: Episode!) {
  hero(episode: $ep) {
    name
    primaryFunction
  }
}
```

```json
{
  "ep": "JEDI"
}
```

```json
{
  "errors": [
    {
      "message": "Cannot query field \"primaryFunction\" on type \"Character\". Did you mean to use an inline fragment on \"Droid\"?",
      "locations": [
        {
          "line": 4,
          "column": 5
        }
      ]
    }
  ]
}
```

`hero`フィールドは`Character`型を返しますが、それは`episode`引数に依存して`Human`または`Droid`のどちらかになり得ることを意味します。
上記クエリは、`Character`インターフェイスに存在するフィールドのみを質問できますが、それは`primaryFunction`を含んでいません。

特定の`ObjectType`のフィールドを質問するために、インラインフラグメントを使用する必要があります。

```graphql
query HeroForEpisode($ep: Episode!) {
  hero(episode: $ep) {
    name
    ... on Droid {
      primaryFunction
    }
  }
}
```

```json
{
  "id": "JEDI"
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

これについては、クエリガイドの[インラインフラグメント](https://graphql.org/learn/queries/#inline-fragments)のセクションで学んでください。

## ユニオン型

ユニオン型はインターフェイスと類似点を共有します。
しかし、構成要素となる型の間で共有されるフィールドを定義する能力が欠けています。

```graphql
union SearchResult = Human | Droid | Starship
```

スキーマ内の`SearchResult`を返すときはいつでも、`Human`、`Droid`または`Starship`を得る可能性があります。
ユニオン型のメンバーは具体的な`ObjectType`である必要があることに注意してください。
インターフェイスまたは他のユニオンからユニオン方を作成できません。

この場合では、もし`SearchResult`ユニオン型を返すフィールドをクエリする場合、任意のフィールドをクエリするためにインラインフラグメントを使用する必要があります。

```graphql
{
  search(text: "an") {
    __typename
    ... on Human {
      name
      height
    }
    ... on Droid {
      name
      primaryFunction
    }
    ... on Starship {
      name
      length
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
        "name": "Han Solo",
        "height": 1.8
      },
      {
        "__typename": "Human",
        "name": "Leia Organa",
        "height": 1.5
      },
      {
        "__typename": "Starship",
        "name": "TIE Advanced x1",
        "length": 9.2
      }
    ]
  }
}
```

`__typename`フィールドは、クライアントで異なるデータ型を互いに区別できるように`String`に解決されます。

また、この場合、`Human`と`Droid`は共通の`Character`インターフェイスを共有しているため、複数の型にわたって同じフィールドを繰り返すよりも、1つの場所で共通のフィールドをクエリできます。

```graphql
{
  search(text: "an") {
    __typename
    ... on Character {
      name
    }
    ... on Humann {
      height
    }
    ... on Droid {
      primaryFunction
    }
    ... on Starship {
      mame
      length
    }
  }
}
```

`name`は、まだ`Starship`で指定されていることに注意してください。
そうでない場合、`Starship`は`Character`でないため、結果に現れません。

## 入力型

これまで、フィールドへの引数として、列挙型や文字列のようなスカラー値を渡すことについてのみ議論してきました。
しかし、複雑なオブジェクトも渡せます。
これは、作成するオブジェクト全体を渡したいミューテーションの場合で特に価値があります。
GraphQLスキーマ言語において、入力型は通常の`ObjectType`と正確に同じですが、`type`の代わりに`input`キーワードを使用します。

```graphql
input ReviewInput{
  stars: Int!
  commentary: String
}
```

ここに、ミューテーションで入力オブジェクト型を使用する方法を示します。

```graphql
mutation CreateReviewForEpisode($ep: Episode!, $review: ReviewInput!) {
  createReview(episode: $ep, review: $review){
    stars
    commentary
  }
}
```

```json
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
      "commentary": "This is a great movie!"
    }
  }
}
```

入力オブジェクト型のフィールド自体が入力オブジェクト型を参照できますが、スキーマ内で入力型と出力型を混ぜることはできません。
また、入力オブジェクト型のフィールドに引数を持つこともできません。
