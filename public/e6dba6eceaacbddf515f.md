---
title: LINE Bot「コンパスパンダ(ver0)」
tags:
  - JavaScript
  - leaflet
  - Firebase
  - linebot
private: false
updated_at: '2019-07-10T13:25:02+09:00'
id: e6dba6eceaacbddf515f
organization_url_name: null
slide: true
ignorePublish: false
---
# はじめに
数ヶ月前から黙々と開発していたLINE Bot「コンパスパンダ」が完成したのでまとめます。ソースコードや概要は[こちら](https://github.com/ufoo68/compassPanda)で公開しています。
概要や使い方に関する説明はリンク先のドキュメントにて記述しています。この記事では技術的なお話のみに絞ります。後、LTとかに使いたいのでスライド形式です。

---

## システム図
![tweet.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/209689/5a319643-ebcc-941c-0105-7d51b65e2a45.jpeg)


---

## システムの流れ

1. 基本的にユーザーは赤矢印のようにLINE Botとのインタラクションのみでこのアプリを操作
2. 位置情報の取得には[URLスキーム](https://developers.line.biz/ja/docs/messaging-api/using-line-url-scheme/)機能を用いて位置情報取得
3. 呟きのテキストは専用フォームを[LIFF](https://developers.line.biz/ja/docs/liff/overview/)で表示してそこから取得
4. 取得した情報をLINE Botのサーバーを介してデータベースに保存
5. 地図表示もLIFF上にてディスプレイ 

---

## 利用技術(LINE Bot)

- [Messaging API](https://developers.line.biz/ja/services/messaging-api/)
- [Cloud Functions](https://firebase.google.com/docs/functions/?hl=ja)
- [Node.js](https://nodejs.org/ja/)
    - [line-bot-sdk-nodejs](https://github.com/line/line-bot-sdk-nodejs)
    - [express](https://expressjs.com/ja/)
    - [firebase-functions](https://www.npmjs.com/package/firebase-functions)

---

## 利用技術（Form）

- [Bootstrap](https://getbootstrap.com/)
- [LIFF](https://developers.line.biz/ja/reference/liff/)

---

## 利用技術（データベース）

- [Cloud Firestore](https://firebase.google.com/docs/firestore/?hl=ja)

---

## 利用技術（Map）

- [React](https://ja.reactjs.org/)
- [Leaflet](https://leafletjs.com/)
- [gh-pages](https://github.com/tschaub/gh-pages)

---

# 苦労したこと

---

## 位置情報と呟きの結びつけ

- Messaging APIにはステート管理の機能がない
- 無理やりステート管理をするには
    1. データベースから一々データを引っ張る
    2. Push通知を使う（[BotBuilder](https://qiita.com/kenakamu/items/6dc043cfc1f199032883)が良さげ？）
    3. 何かしらの管理番号をトーク画面上に発行
-  今回は取っ付きやすい「3」の方法を選択 

---

## タイムスタンプを発行(一意となる管理番号がわりに)

![tweet.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/209689/f529ff06-2287-9d0b-9b50-696b7deafa0d.jpeg)

---

## FirestoreのDocumentを１つにまとめた

- Firestoreには呼び出し数に制限あり
 - CollectionsではなくDocumentの数をカウント
- ネストを１つ深く
 - 取り出す時が少し面倒

```js
{
  timestamp: {
    tweet : text,
    latitude: latitude,
    longitude:  longitude
  }
}
```

---

# 最後に

---

## 課題

- UIが使いにくすぎる
    - 一度に位置と呟きを取得したい
- 会話の応答と呟きのDBへの保存の処理を同じLINE Botのサーバで処理している
    - LIFFに任せるなど役割分担を整理したい
    - DB用のサーバを立ててAPI化させたい
- 地図にテキストしか残せない
    - 写真とかにも対応させたい

---

## さらにやりたいこと

- 呟き内容のテキストを感情分析してポインターのアイコンに反映させる
- LINE Thingsと連携したサービス

とり会えず今ところ作れたのは最低限動くもの。ここから面白いサービスを作って行きたい。
