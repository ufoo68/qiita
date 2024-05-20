---
title: Clovaで「日本酒診断」のスキルを作ってみる（１）
tags:
  - Node.js
  - Express
  - TypeScript
  - Clova
private: false
updated_at: '2019-06-09T23:49:24+09:00'
id: 0df3e1c3c9beb4d4faf8
organization_url_name: null
slide: false
ignorePublish: false
---
# はじめに
前回作った[抵抗値換算](https://qiita.com/ufoo68/items/f6b04e1ca89a931ca206)スキルが無事に公開できたということで、今回はまた別のスキルに挑戦してみました。最近私自身が日本酒にはまっているということで「日本酒診断」というスキルを開発していこうと思います。ちなみに私は[谷川岳](http://www.nagai-sake.co.jp/item_category/tanigawadake/)が好きです。

# スキルについて
Clovaがユーザにいくつかの質問をして、そのユーザの好みにあった日本酒を提案するスキルです。一応事前に伝えておきますが、**これは完全なるネタスキルです**。

# 日本酒の提案モデルについて
下の図のような会話のフローで日本酒を提案します。

![会話モデル.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/209689/b493ef45-8a34-930b-2cba-79a86c795e5b.jpeg)

こんな感じで会話を分岐させてユーザにあった日本酒を提案しようと思っています。まあ、ちょっと主観が入っちゃって谷川岳が多めになってしまいましたね(^^;

# 技術的に挑戦すること
今回は以下の３つの項目で挑戦しようと思っています。

- TypeScriptで書く
- 公式のSDKを使う
- 前回セッションのスロットを記憶する

TypeScriptで書く方法は[こちら](https://qiita.com/daisukeArk/items/78b82bb97d4b76d7ca08)を参考にしたりなどをしました。とりあえずスタートラインとして、TypeScriptのための環境構築、実装としては「インテントを受け取りそれを返事として返す」ところまでをやりました。また公式SDKを使う場合は[前回](https://qiita.com/ufoo68/items/f6b04e1ca89a931ca206)少し触れたように、少し厄介なことがあったのでそれの対処を行いました。

# TypeScriptの環境構築
今回もFirebaseを用いたので、そのまま`firebase init`を適当なディレクトリで実行した後に使用言語にTypeScriptを選んだだけです。ベーシックな依存関係とか云々はFirebaseが用意してくれました。しかし`functions/tsconfig.json`に関しては少しだけ変更を加えました。

```js:functions/tsconfig.json
{
  "compilerOptions": {
    "module": "commonjs",
    "noImplicitReturns": true,
    "noUnusedLocals": true,
    "outDir": "lib",
    "sourceMap": true,
    "strict": true,
    "esModuleInterop": true,
    "target": "es2017"
  },
  "compileOnSave": true,
  "include": [
    "src"
  ]
}
```

変更点としては`"esModuleInterop": true`を加えました。理由はこの後にinstallする`@types/express`のimport/exportに関する記述の部分でtscのerror発生を防ぐためです。
続けて以下の３つのパッケージを追加でインストールしました。

- @line/clova-cek-sdk-nodejs
- @types/express
- express

TypeScript（以下TS）でexpressを使う場合は`@types/express`も必要みたいです。

# TypeScriptでの実装
こんな感じで実装しました。TS初心者なので色々ツッコミどころあるかもなのでコメントくれると嬉しいです。

```ts:functions/index.ts
import * as functions from 'firebase-functions';
import Express from 'express';
import * as Clova from '@line/clova-cek-sdk-nodejs';

const extensionId : string = functions.config().clova.extension.id;

const clovaSkillHandler = Clova.Client
    .configureSkill()

    //起動時に喋る
    .onLaunchRequest((responseHelper: { setSimpleSpeech: (arg0: { lang: string; type: string; value: string; }) => void; }) => {
        responseHelper.setSimpleSpeech({
            lang: 'ja',
            type: 'PlainText',
            value: `日本酒診断をします。いくつかの質問に答えてください。`,
        });
    })

    //ユーザーからの発話が来たら反応する箇所
    .onIntentRequest(async (responseHelper: { 
        getIntentName: () => string; getSlots: () => string; setSimpleSpeech: { (arg0: { lang: string; type: string; value: string; }): void; (arg0: Clova.Clova.SpeechInfoText, arg1: boolean): void; }; }) => {
        const intent = responseHelper.getIntentName();
        //const slots = responseHelper.getSlots();

        console.log('Intent:' + intent);

        let speech = {
            lang: 'ja',
            type: 'PlainText',
            value:  `${intent}のインテントを受け取りました`
        }

        responseHelper.setSimpleSpeech(speech);
        responseHelper.setSimpleSpeech(
            Clova.SpeechBuilder.createSpeechText('開発中です'), true
        );
    })

    //終了時
    .onSessionEndedRequest((responseHelper: { getSessionId: () => void; }) => {
        //const sessionId = responseHelper.getSessionId();
    })
    .handle();

const app = Express();

const clovaMiddleware = Clova.Middleware({applicationId: extensionId});
app.use( function (req, res, next) {
    req.body = JSON.stringify(req.body)
    next()
})
app.post('/clova', clovaMiddleware, <Express.RequestHandler>clovaSkillHandler);

exports.clova = functions.https.onRequest(app);
```

実のところほとんどJavaScript（以下JS）の実装のコピペでなんとかなった感じです。`resonseHelper`のところはJSでは一々オブジェクトの中のフィールドや型を指定する必要がなかったのですがTSではそれが許されませんでした。でもこの辺りはvscodeの機能が勝手に対応してくれました（やっぱりvscodeは便利）。後は`app.post()`のところで`clovaSkillHandler`を指定するときに、clovaSkillHandlerのメソッドチェーンの最後に`handle()`があるわけですが、このままだとclovaSkillHandlerはただのFunctionなので`<Express.RequestHandler>`を付けてRequestHandlerとして指定する必要があります。

# 公式SDKとFirebaseの相性の悪さ
[この記事](https://blog.tanakamidnight.com/2018/09/firebase-clova-sdk-node8/)でも述べられている通りRequestがオブジェクトで自動で返ってくるせいで`Clova.Middleware`が実行したときに文字列が受け取れないことからerrorを起こしてしまいます。先ほどの記事ではSDKのソースをご自身で書き換えたみたいですが、できれば公式のSDK使いたいと思ったので（実はTSを選んだのも公式SDKがTSで実装されているから）、ミドルウェアがリクエストボディを受け取る前に

```ts
app.use( function (req, res, next) {
    req.body = JSON.stringify(req.body)
    next()
})
```

を加えて`req.body`をstringに変換させる処理を加えてやりました。

# いまのところまでの進捗

![2019-06-09 (3).png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/209689/ca254111-b13a-f6c3-ad67-661556b36ca2.png)

事前に「冷/熱燗」というスロットを含んだ`howDrinkIntent`を用意してそれを正しく受け取れたかのテストを行いました。まあこのくらいの内容はJSだったら一瞬だったと思いますが、TSで苦労して書いたことでまた別の感動がありますね(笑)

# さいごに
とりあえずベースができたので次は分岐する会話のモデルの実装をしていこうと思います。`session.sessionAttributes`という[CEK API](https://clova-developers.line.biz/guide/CEK/References/CEK_API.md#CustomExtRequestMessage)のMessage fieldを使うと前回のセッションの内容の受け渡しができるみたいなので、これでやっていこうと思います。今回までの分も[GitHub](https://github.com/ufoo68/nihonshuFortune)にコミットしておきました。
