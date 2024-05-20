---
title: 今更ながらGraphQLについてまとめてみた
tags:
  - 勉強メモ
  - GraphQL
private: false
updated_at: '2020-07-19T12:36:46+09:00'
id: 875d0afc7d42dc0af1cd
organization_url_name: null
slide: true
ignorePublish: false
---
# はじめに

[GraphQLについてのオライリー本](https://www.oreilly.co.jp/books/9784873118932/)を読んだり、[AppSync](https://aws.amazon.com/jp/appsync/)を趣味で触ったりしたので今更ながらGraphQLについて社内勉強会用にまとめてみました。

---

## GraphQLとは

[GraphQL](https://graphql.org/)を一言で説明するとWEB APIのためのクエリ・スキーマ言語である。
クライアントアプリケーションのデータモデルの機能と要件を記述することを目的に**Facebook**の開発チームによって作成された。[^1]

[^1]: [GraphQLの2015年に公開された仕様](http://spec.graphql.org/July2015/)

---

## クエリ言語とスキーマ言語

ここで扱う**クエリ言語**と**スキーマ言語**については以下のようなイメージとなっている。

* クエリ言語
  * クライアントアプリがGraphQLサーバーにリクエストを送信するための構文
* スキーマ言語
  * GraphQLサーバーのデータ型の集合を定義するための構文

---

## GraphQLの立ち位置

アプリケーション開発に用いられるGraphQLのイメージ。

![GraphQLについて.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/209689/f3223de1-2d48-a0bd-300d-c77467351a23.jpeg)

---

## GraphQLサーバーが実際に行うこと

* **APIサーバーとの通信**を行う
  * もしくは**直接データベースの操作**を行う
* **クエリ構文**を解析する
  * 解釈した構文に基づいて処理（**リゾルバ**）が実行される

![GraphQLの動作.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/209689/6a395ee0-14fe-eeb6-3a47-ba1138ba21b5.jpeg)

---

## WEB APIとしてのクエリ

GraphQLサーバーへ送るクエリはHTTPリクエストの`POST`メソッドを用いて送信される。以下はcURLでの例。

```bash
curl --location --request POST 'http://snowtooth.moonhighway.com/' \
--header 'Content-Type: application/json' \
--data-raw '{"query":"query {\r\n  allLifts {\r\n    name\r\n  }\r\n}","variables":{}}
```

クエリのリクエストを試す場合は、[GraphiQL](https://github.com/graphql/graphiql)や[GraphQL Playground](https://github.com/prisma-labs/graphql-playground)などのツールもしくは[PostmanのGraphQL機能](https://learning.postman.com/docs/postman/sending-api-requests/graphql/)を使用することができる。

---

## クエリの種類

クライアントアプリケーションが用いるクエリの構文では、３つのオペレーションを用いることができる。

1. **query**
2. **mutation**
3. **subscription**

---

## query

データを取得するためのオペレーション。実際に取得したいデータを**field**に指定する。以下は[スノートゥース](https://snowtooth.moonhighway.com/)で、すべての`Lift`の`name`と`status`の情報をとってくる例。

```graphql
query getAllLifts {
  allLifts {
    name
    status
  }
}
```

---

## mutation

データの書き込み操作を担当する。基本的に構文はqueryと変わらない。以下は前ページで用いた**スノートゥース**でとある`Lift`の`status`を更新する例。

```graphql
mutation openLift {
  setLiftStatus(id:"astra-express" status: OPEN) {
    name
    status
  }
}
```

---

## subscription

サーバーからリアルタイムでデータを受信することができる。例えば**mutation**で更新した`status`を毎回受け取りに行くために、一々**query**を叩かずとも**subscription**を用いればWebSocketを通じてリアルタイムにデータを受け取ることができる。以下はその構文の例。

```graphql
subscription listenStatusChange {
  liftStatusChange {
    name
    status
  }
}
```

---

## ここで省略した話

* フラグメント
  * 冗長な構文を共通化させる仕組み
* ユニオン型
  * ２つの異なるオブジェクト型をまとめる仕組み
* インタフェース
  * 実装すべきフィールドのリストを指定する抽象型

---

## スキーマの設計

実際にGraphQLサーバーを実装するために、まずはデータ型の集合である**スキーマ**を設計する。実装例については[過去の記事の内容](https://qiita.com/ufoo68/items/cff4e8c2ead1f62ed2a9)を使いまわしてみる。

```graphql
enum Category {
    HOGEHOGE
    HUGAHUGA
}

type Item {
    id: String!
    data: String!
    category: Category!
}

type Query {
    allItem: [Item]
    allItemOnCategory(category: Category): [Item]
}

input ItemInput {
    data: String!
    category: Category!
}

type Mutation {
    addItem(item: ItemInput!): Item
}
```

---

## 型

**型**はスキーマの核になるもの。それぞれの型には対応する**フィールド**が存在する。**Query**や**Mutation**もそれぞれ**使用可能なクエリがフィールドとして定義されている**型となる。

```graphql
type Item {
    id: String!
    data: String!
    category: Category!
}

type Query {
    allItem: [Item]
    allItemOnCategory(category: Category): [Item]
}

type Mutation {
    addItem(item: ItemInput!): Item
}
```

---

## スカラー型とリスト

それぞれのフィールドが固有の型のデータを持っていて、その組み込みの型を**スカラー型**という。例えば`id`には`String`というスカラー型で定義されている。ちなみに`String!`と`!`を付与することでそのフィールドが**nullにならない**ことを示すことができる。
また、型を`[]`で囲むことでフィールドに**型のリスト**を指定することができる。例えば`[Item]`は`Item`という型のリストであることを示している。

---

## Enum

**Enum**は限られた**選択肢の文字列**が入力・出力されることをフィールドに定義したいときに用いる

```graphql
enum Category {
    HOGEHOGE
    HUGAHUGA
}

```

ここで作成した`Category`はフィールドの型として用いることができる。

---

## 入力型

引数に対して使用する型。多数の引数をまとめるために用いる。

```graphql
input ItemInput {
    data: String!
    category: Category!
}
```

ここで作成された`ItemInput`は引数にのみ用いることができる。

---

## ここで省略した話

* カスタムスカラー型
  * 独自で定義されたスカラー型。
* ユニオン型
  * 複数の型のうち一つを返す型。
* インタフェース
  * 型の中で必ず存在するフィールドを定義する抽象型。

---

## リゾルバ

**リゾルバ**は特定のフィールドを返す関数。クエリで指定されたフィールドに対して発行される。以下は[過去の記事](https://qiita.com/ufoo68/items/512b74156789a128bc95)で書いたAppSyncとCDKの実装例。

```typescript
...
    itemDS.createResolver({
      typeName: 'Query',
      fieldName: 'allItem',
      requestMappingTemplate: MappingTemplate.dynamoDbScanTable(),
      responseMappingTemplate: MappingTemplate.dynamoDbResultList(),
    })
    itemDS.createResolver({
      typeName: 'Mutation',
      fieldName: 'addItem',
      requestMappingTemplate: MappingTemplate.dynamoDbPutItem(
          PrimaryKey.partition('id').auto(),
          Values.projecting('item')),
      responseMappingTemplate: MappingTemplate.dynamoDbResultItem(),
    })
...
```

直感的な理解としては、リゾルバは**リクエストとレスポンスのボディに対するマッピング・そのための内部的な処理**を実装するものである。

---

## さいごに

* GraphQLを用いた開発では、スキーマとクエリの２つの言語を書くことになる
  * スキーマはAPIの仕様となるもの
  * クエリはクライアントアプリからのリクエスト
* 基本的にライブラリを用いた開発となる
  * 手で実装するのはおそらく難解
  * GraphQL用のライブラリについては[公式ページ](https://graphql.org/code)で紹介されている。

---

# GraphQLについて気になった問題などを取り上げた記事などの紹介

---

## N+1問題

**N+1問題**とは「1つのSQLでN件のレコードをフェッチしたあと、それぞれ対して関連するレコードを個別にフェッチするのにNつのSQLを発行している」状態を指す問題です。GraphQLのリゾルバでSQLを用いる場合このが引き起こされやすくなります。
この問題については[この記事](https://qiita.com/yuku_t/items/2c1735cbf45e75c0bfb8)が参考になりました。

---

## REST APIの代替技術となるのか

[こちらの記事](https://employment.en-japan.com/engineerhub/entry/2018/12/26/103000#GraphQL%E3%81%AE%E6%AC%A0%E7%82%B9)にある通り、GraphQLにも欠点があります。

* フルスクラッチでの実装が難しい（基本はライブラリ依存）
* 画像や動画などの大容量バイナリの扱いが難しい（JSONなので）

---

## Redux vs GraphQL

この話題について[とあるスライド](https://www.slideshare.net/JordonMcKoy/redux-vs-graphql)の28ページ目の表がわかりやすくまとまっていました。
[こういった記事](https://www.nerdwallet.com/blog/engineering/migrating-redux-graphql-nerdwallet-internship-experience/)にもあるように基本的には[apollo-client](https://github.com/apollographql/apollo-client)と[Redux](https://redux.js.org/)との対比でよく議論されているようです。
