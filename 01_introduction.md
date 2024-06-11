# GraphQLの紹介

<https://graphql.org/learn/>

- [GraphQLの紹介](#graphqlの紹介)

> GraphQLについて学び、どのように機能して、どのように使用するかを学びましょう。
> GraphQLサービスを構築する方法を記述したドキュメントを探していますか？
> [多くの異なる言語](https://graphql.org/code/)でGraphQLの実装を支援するライブラリがあります。
> 実践的なチュートリアルによる深い学習体験を得たい場合は、[利用可能なトレーニングコース](https://graphql.org/community/resources/training-courses/)を参照してください。

GraphQLは、APIのクエリ言語で、データを定義した型システムを使用してクエリを実行する、サーバーサイドのランタイムです。
GraphQLは、任意の特定のデータベースまたはストレージエンジンに固定されておらず、代わりに既存のコードとデータによって支えられます。

GraphQLサービスは、定義された型とそれらの型のフィールドを定義して、それぞれのフィールドのそれぞれの型に関数を提供することによって作成されます。
例えば、ログインしたユーザーが`自分(me)`とそのユーザー名を伝えるGraphQLサービスは、次のようになります。

```graphql
type Query {
  me: User
}

type User {
  id: ID
  name: String
}
```

それぞれの型のそれぞれのフィールドの関数は次のとおりです。

```js
const Query_me = (request) => request.auth.user;

const User_name = (user) => user.getName();
```

典型的にWebサービスのURL

通常、WebサービスのURLで、GraphQLサービスが起動した後、それは検証と実行するGraphQLクエリを受け付けることができます。
最初、サービスは、クエリが定義された型とフィールドのみを参照していることを保証しているか確認して、結果を生成するために提供された関数を実行します。

例えば、クエリは次のようになります。

```graphql
{
  me {
    name
  }
}
```

次のJSON結果を生成する可能性があります。

```json
{
  "me": {
    "name": "Luke Skywalker"
  }
}
```
