---
title: CDKでGraphQLに挑戦する(Schema編)
tags:
  - GraphQL
  - AppSync
  - aws-cdk
private: false
updated_at: '2020-04-29T16:43:10+09:00'
id: cff4e8c2ead1f62ed2a9
organization_url_name: null
slide: false
ignorePublish: false
---
# はじめに

最近[この本](https://www.oreilly.co.jp/books/9784873118932/)を読んで**GraphQL**を完全理解した気になったので何かしら実践してみようと思いこの記事を書くことにしました。
AWSに[AppSync](https://aws.amazon.com/jp/appsync/)というGraphQLを用いるためのサービスがあるようなので今回はこれをCDKで触ってみようと思います。この記事は[GWアドベントカレンダー](https://gw-advent.9wick.com/calendars/2020/89)の１日目として作成しました。今回はSchema導入編で、CDKの導入からSchemaの読み込みまでを実装してみます。

# AppSyncとは

公式ページより、**GraphQLを使用してアプリケーションが必要なデータを正確に取得できるようにするマネージド型サービス**とのことです。
今までは、あるデータベースを操作するためにはいくつものエンドポイントを立ててAPIごとの処理を実装する必要があったのですが、このサービスを用いることで一つのAPIとスキーマを設計するだけでCRUDのような処理を実装することができるみたいですね。

# CDKで導入してみる

今回は簡単のために`addItem`でデータの追加ができるスキーマと、`allItem`でそれを取得できるスキーマを実装してみようと思います。まずはCDKでいつもどおり以下のコマンドを打って、プロジェクトを立ち上げます。

```bash
cdk init app --language typescript
```

とりあえず今日はSchemaを書いてその内容をAppSyncにデプロイさせるところまでを進めたいと思います。まずはCDKをこんな感じに実装してみます。

```typescript
import * as cdk from '@aws-cdk/core'
import { GraphQLApi, CfnApiKey } from '@aws-cdk/aws-appsync'

export class CdkAppsyncTestStack extends cdk.Stack {
  constructor(scope: cdk.Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props)

    const api = new GraphQLApi(this, 'graphQlApi', {
      name: 'test-api',
      schemaDefinitionFile: './schema.graphql',
    })
    
    new CfnApiKey(this, 'apiKey', {
      apiId: api.apiId,
    })
  }
}
```

まず`GraphQLApi()`でAPIを作成します。ここでは単純に**名前**と**スキーマのファイルのディレクトリ**だけを指定します。また、クエリの実行のために**API KEY**が必要みたいなので`CfnApiKey()`でAPI KEYを設定します。

# スキーマをかく

今回は簡単に以下のようなスキーマを定義しました。

```graphql
type Item {
    id: String!
    data: String!
}

type Query {
    allItem: [Item]
}

input ItemInput {
    id: String!
    data: String!
}

type Mutation {
    addItem(item: ItemInput!): Item
}
```

`id`と`data`を含む`Item`という型を定義してあとはそれの追加と取り出しを行う`Query`と`Mutation`を定義します。このスキーマがAppSyncの設計書になります。

# 実行する

この内容でもデプロイは可能なのでデプロイしてみます。

```bash
cdk deploy
```

デプロイできたら、クエリの実行が可能になるので今回はAWSコンソールからクエリの実行を行ってみます。

## Queryの実行

以下のクエリを実行します。

```graphql
query {
  allItem {
    id
    data
  }
}
```

当然`null`が返ってきますが、実行そのものには成功しました。

![query.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/209689/7d556169-a666-9db2-51d4-a0b3aee9425f.jpeg)

## Mutationの実行

以下のクエリを実行します。

```graphql
mutation($input:ItemInput!) {
  addItem(item:$input) {
    id
    data
  }
}
```

クエリ変数も定義します。

```json
{
  "input": {
    "id": "test",
    "data": "test"
  }
}
```

当然なにも追加されずに`null`が返ってくるだけですが、クエリの実行には成功しました。
![mutation.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/209689/0de50606-2f3c-27ba-4ce8-f698189117fb.jpeg)

# さいごに

とりあえず今日はここまでで、明日から中身の実装に入っていきたいと思います。ソースコードに関しては[ここ](https://github.com/ufoo68/cdk-appsync-test)で共有して、明日からアップデートしていこうと思います。
