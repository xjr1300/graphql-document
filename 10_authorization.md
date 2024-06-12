# 認可

> ビジネスロジックレイヤに認可ロジックを移譲してください。

認可は、与えられた`user/session/context`がアクションを実行する、またはデータの一部を参照する権限があるかを記述するビジネスロジックの一種です。
例えば・・・

"著者だけが彼らのドラフトを参照できます"

この種の振る舞いを強制することは、[ビジネスロジックレイヤ](https://graphql.org/learn/thinking-in-graphs/#business-logic-layer)で行うべきです。
次のように認可ロジックをGraphQLレイヤに配置したくなります。

```javascript
const postType = new GraphQLObjectType({
  name: 'Post',
  fields: {
    body: {
      type: GraphQLString,
      resolve(post, args, context, { rootValue }) {
        // ユーザーが投稿した著者のときのみ、投稿の本文を返します。
        if (context.user && (context.user.id === post.authorId)) {
          return post.body
        }
        return null
      }
    }
  }
})
```

投稿の`authorId`フィールドが現在のユーザーの`id`と等しいか確認することによって、"著者が所有する投稿"を定義していることに注意してください。
問題を見つけることができますか？
このコードをサービス内のそれぞれのエントリポイントに複製する必要があります。
そして、もし認可ロジックが完全に同期を維持しなかった場合、ユーザーはユーザーが使用したAPIによって異なるデータを確認する可能性があります。
認可のために[唯一信頼できる情報源](https://graphql.org/learn/thinking-in-graphs/#business-logic-layer)を持つことにより、これを回避できます。

GraphQLを学んでいるとき、またはプロトタイピングで、リゾルバー内に認可ロジックを定義することは大丈夫です。
しかし、プロダクションコードベースでは、認可ロジックをビジネスロジックレイヤに移譲してください。
ここに例を示します。

```javascript
// 認可ロジックはpostRepository内に存在します。
const postRepository = require('postRepository');

const postType = new GraphQLObjectType({
  name: 'Post',
  fields: {
    body: {
      type: GraphQLString,
      resolve(post, args, context, { rootValue }) {
        return postRepository.getBody(context.user, post)
      }
    }
  }
})
```

上記例において、ビジネスロジックレイヤは、ユーザーオブジェクトを提供する呼び出し者を要求します。
もし、GraphQL.jsを使用している場合、ユーザーオブジェクトは`context`引数、またはリソルバーの4つ目の引数の`rootValue`に存在するべきです。

不透明なトークンまたはAPIキーの代わりに十分に水分補給されたユーザーオブジェクトをビジネスロジックレイヤに渡すことを推奨します。
これにより、リクエスト処理パイプラインの異なるステージで、[認証](https://graphql.org/graphql-js/authentication-and-express-middleware/)と認可の概念を区別して処理できます。
