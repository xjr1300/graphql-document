# 検証

<https://graphql.org/learn/schema/>

型システムを利用することで、GraphQLクエリが妥当化そうでないか事前に決定できます。
これは、サーバーとクライアントが、ランタイムな確認に依存することなしで、不正なクエリが作成されたときに効果的に開発者に伝えるようになります。

スターウォーズの例で、[starWarsValidation-test.ts](https://github.com/graphql/graphql-js/blob/main/src/__tests__/starWarsValidation-test.ts)は、色々な無効なクエリをいくつか含んでおり、またバリデーターのリファレンス実装を訓練する(exercise)するために実行されるテストファイルです。

始めるために、複雑で妥当なクエリを取り上げましょう。
このネストしたクエリは、前のセクションから例と似ていますが、重複したフィールドはフラグメントに分解されます。

```graphql
{
  hero {
    ...NameAndAppearances
    friends {
      ...NameAndAppearances
      friends {
        ...NameAndAppearances
      }
    }
  }
}

fragment NameAndAppearances on Character {
  name
  appearsIn
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
      ],
      "friends": [
        {
          "name": "Luke Skywalker",
          "appearsIn": [
            "NEWHOPE",
            "EMPIRE",
            "JEDI"
          ],
          "friends": [
            {
              "name": "Han Solo",
              "appearsIn": [
                "NEWHOPE",
                "EMPIRE",
                "JEDI"
              ]
            },
            {
              "name": "Leia Organa",
              "appearsIn": [
                "NEWHOPE",
                "EMPIRE",
                "JEDI"
              ]
            },
            {
              "name": "C-3PO",
              "appearsIn": [
                "NEWHOPE",
                "EMPIRE",
                "JEDI"
              ]
            },
            {
              "name": "R2-D2",
              "appearsIn": [
                "NEWHOPE",
                "EMPIRE",
                "JEDI"
              ]
            }
          ]
        },
        {
          "name": "Han Solo",
          "appearsIn": [
            "NEWHOPE",
            "EMPIRE",
            "JEDI"
          ],
          "friends": [
            {
              "name": "Luke Skywalker",
              "appearsIn": [
                "NEWHOPE",
                "EMPIRE",
                "JEDI"
              ]
            },
            {
              "name": "Leia Organa",
              "appearsIn": [
                "NEWHOPE",
                "EMPIRE",
                "JEDI"
              ]
            },
            {
              "name": "R2-D2",
              "appearsIn": [
                "NEWHOPE",
                "EMPIRE",
                "JEDI"
              ]
            }
          ]
        },
        {
          "name": "Leia Organa",
          "appearsIn": [
            "NEWHOPE",
            "EMPIRE",
            "JEDI"
          ],
          "friends": [
            {
              "name": "Luke Skywalker",
              "appearsIn": [
                "NEWHOPE",
                "EMPIRE",
                "JEDI"
              ]
            },
            {
              "name": "Han Solo",
              "appearsIn": [
                "NEWHOPE",
                "EMPIRE",
                "JEDI"
              ]
            },
            {
              "name": "C-3PO",
              "appearsIn": [
                "NEWHOPE",
                "EMPIRE",
                "JEDI"
              ]
            },
            {
              "name": "R2-D2",
              "appearsIn": [
                "NEWHOPE",
                "EMPIRE",
                "JEDI"
              ]
            }
          ]
        }
      ]
    }
  }
}
```

そして、このクエリは妥当です。
いくつかの不正なクエリを確認しましょう。

これは無制限の結果になる可能性があるため、フラグメントはそれ自身を参照またはサイクルを作成することはできません。
ここに、明示的な3段階のネストをなくした上記と同じクエリを示します。

```graphql
# 不正: フラグメントはそれ自身を参照できません。
{
  hero {
    ...NameAndAppearances
  }
}

fragment NameAndAppearances on Character {
  name
  appearsIn
  friends {
    ...NameAndAppearances
  }
}
```

```json
{
  "errors": [
    {
      "message": "Cannot spread fragment \"NameAndAppearancesAndFriends\" within itself.",
      "locations": [
        {
          "line": 11,
          "column": 5
        }
      ]
    }
  ]
}
```

フィールドをクエリするとき、与えられた型に存在するフィールドをクエリする必要があります。
`hero`は`Character`を返すため、`Character`のフィールドをクエリする必要があります。
その型は、`favoriteSpaceship`フィールドを持たないため、このクエリは不正です。

```graphql
# 不正: favoriteSpaceshipはCharacterに存在しません。
{
  hero {
    favoriteSpaceship
  }
}
```

```json
{
  "errors": [
    {
      "message": "Cannot query field \"favoriteSpaceship\" on type \"Character\".",
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

フィールドをクエリして、スカラーまたは列挙型以外の他の何かを返したときはいつでも、フィールドから返して欲しいデータが何か記述する必要があります。
`hero`は`Character`を返して、その`name`と`appearsIn`のようなフィールドを要求しました。
もし、それを省略した場合、クエリは妥当になりません。

```graphql
# 不正: heroはスカラーでないため、fieldが必要です。
{
  hero
}
```

同様に、フィールドがスカラーの場合、それに追加的なフィールドをクエリすることは意味がないため、それをするとクエリを不正にします。

```graphql
# 不正: nameはスカラーであるため、フィールドは許可されません。
{
  hero {
    name {
      firstCharacterOfName
    }
  }
}
```

```json
{
  "errors": [
    {
      "message": "Field \"name\" must not have a selection since type \"String!\" has no subfields.",
      "locations": [
        {
          "line": 4,
          "column": 10
        }
      ]
    }
  ]
}
```

以前、クエリは質問内の型のフィールドのみをクエリできると注意しました。
`Character`を返す`hero`をクエリしたとき、`Character`に存在するフィールドのみをクエリできます。
しかし、もし、R2-D2の主要な機能をクエリしたい場合、何が発生しますか？

```graphql
# 不正: primaryFunctionはCharacterに存在しません。
{
  hero {
    name
    primaryFunction
  }
}
```

```json
{
  "errors": [
    {
      "message": "Cannot query field \"primaryFunction\" on type \"Character\". Did you mean to use an inline fragment on \"Droid\"?",
      "locations": [
        {
          "line": 5,
          "column": 5
        }
      ]
    }
  ]
}
```

`primaryFunction`は`Character`のフィールドではないため、上記のクエリは不正です。
もし、`Character`が`Droid`の場合に`primaryFunction`を取得して、そうでない場合はそのフィールドを無視したいことを示す、何らかの方法が必要です。
これをするために以前導入したフラグメントを使用できます。
`Droid`を定義したフラグメントを準備して、それを含めることで、`primaryFunction`が定義されているところでのみそれをクエリできるように確信します。

```graphql
{
  hero {
    name
    ...DroidFields
  }
}

fragment DroidFields on Droid {
  primaryFunction
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

上記クエリは妥当ですが、少し冗長です。
名前が付けられたフラグメントは、複数回それらを使用するときに価値がありますが、これを1回しか使用していません。
名前を付けたフラグメントを使用する代わりに、インラインフラグメントを使用できます。
インラインフラグメントは、名前を付けた分離したフラグメントなしで、クエリする型を示すことができようにします。

```graphql
{
  hero {
    name
    ... on Droid {
      primaryFunction
    }
  }
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

これが検証システムの表面をなぞっただけです。
GraphQLクエリが意味的に意味を持つことを確信するために検証ルールがいくつかあります。
このトピックについては仕様の「検証」セクションで詳しく説明されており、GraphQL.jsの[検証ディレクトリ](https://github.com/graphql/graphql-js/blob/main/src/validation)には仕様に準拠したGraphQLバリデータを実装するコードが含まれています。
