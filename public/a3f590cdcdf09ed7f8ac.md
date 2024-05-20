---
title: Clovaで「日本酒診断」のスキルを作ってみる（２）
tags:
  - Express
  - TypeScript
  - LINE
  - Clova
private: false
updated_at: '2019-06-10T23:19:58+09:00'
id: a3f590cdcdf09ed7f8ac
organization_url_name: null
slide: false
ignorePublish: false
---
# はじめに
[前回](https://qiita.com/ufoo68/items/0df3e1c3c9beb4d4faf8)の続きです。一応完成しました。本当に思いつきで作ったのであまりボリュームのないコンテンツになってしまいました。

# 今回やったこと
前回示した会話モデルの実装を行いました。一応再掲します。

![会話モデル.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/209689/85bed17e-3f90-2fab-95ae-dd375262cf63.jpeg)

このモデルを実装するためには前回セッションのスロットを記憶しておく必要があります。そのために`setSessionAttributes()`と`getSessionAttributes()`が必要になります。これはあるセッションで用いた値などを次のセッションに持ち越すことができるメソッドです。Attributesの受け渡しにはobjectを用います。詳しい説明は[こちらの記事](https://qiita.com/miso_develop/items/6b256c1e8757c0ace4a9)を参照するといいと思います。

# 会話のモデルの実装
実装の参考として[公式のチュートリアルのサンプルコード](https://github.com/line/clova-extension-sample-cafe/blob/master/tutorials/tutorial-3.js)を拝借しました。以下にコードを示します。

```ts:functions/index.ts
import * as functions from 'firebase-functions';
import Express from 'express';
import * as Clova from '@line/clova-cek-sdk-nodejs';

const extensionId : string = functions.config().clova.extension.id;

const clovaSkillHandler = Clova.Client
    .configureSkill()

    //起動時に喋る
    .onLaunchRequest((responseHelper: { setSimpleSpeech: (arg0: { lang: string; type: string; value: string; }) => Clova.Context; }) => {
        responseHelper.setSimpleSpeech(Clova.SpeechBuilder.createSpeechText(`日本酒診断をします。２つの質問に答えてください。飲み方はヒヤか、熱燗、どちらがいいですか？`))
        .setSessionAttributes({}); //Attributesの初期化
    })

    //ユーザーからの発話が来たら反応する箇所
    .onIntentRequest(async (responseHelper: { 
        getIntentName: () => string; 
        getSlot: (slotName: string) => string; 
        getSessionAttributes: () => {drinkType?: string; tasteType?: string};
        setSimpleSpeech: { (arg0: { lang: string; type: string; value: string; }): Clova.Context;
        (arg0: Clova.Clova.SpeechInfoText, arg1: boolean): void; }; }) => {
        const intent = responseHelper.getIntentName();
        const drinkType = responseHelper.getSlot('drinkType') ? responseHelper.getSlot('drinkType') : responseHelper.getSessionAttributes().drinkType;
        const tasteType = responseHelper.getSlot('tasteType') ? responseHelper.getSlot('tasteType') : responseHelper.getSessionAttributes().tasteType;

        switch (intent) {
            case 'howDrinkIntent':
                if (drinkType && tasteType) {
                    responseHelper.setSimpleSpeech(Clova.SpeechBuilder.createSpeechText(`${drinkType}で飲む${tasteType}のタニガワダケをオススメします。`))
                    .endSession();
                }
                else {
                    responseHelper.setSimpleSpeech(Clova.SpeechBuilder.createSpeechText(`甘口か、辛口、どちらがいいですか？`))
                    .setSessionAttributes({drinkType: drinkType});
                }
                break;

            case 'howTasteIntent':
                if (drinkType && tasteType) {
                    responseHelper.setSimpleSpeech(Clova.SpeechBuilder.createSpeechText(`${drinkType}で飲む${tasteType}のタニガワダケをオススメします。`))
                    .endSession();
                }
                else {
                    responseHelper.setSimpleSpeech(Clova.SpeechBuilder.createSpeechText(`飲み方は冷か、燗、どちらがいいですか？`))
                    .setSessionAttributes({tasteType: tasteType});
                }
                break;

            default:
                responseHelper.setSimpleSpeech(Clova.SpeechBuilder.createSpeechText(`もう一度お願いします`));
                break;
        }
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

AttributesもしくはIntentのどちらかのスロットを持ってきたい場合には

```ts
const drinkType = responseHelper.getSlot('drinkType') ? responseHelper.getSlot('drinkType') : responseHelper.getSessionAttributes().drinkType;
const tasteType = responseHelper.getSlot('tasteType') ? responseHelper.getSlot('tasteType') : responseHelper.getSessionAttributes().tasteType;
```

のようにするとスッキリと書けます。こうして全部のスロットを最初に受け取っておけば、後はif文などでスロットがすべてそろっているかの判断をすればいいわけです。今回はスロットがすべてそろったと判断したときに、

```ts
Clova.SpeechBuilder.createSpeechText(`${drinkType}で飲む${tasteType}のタニガワダケをオススメします。`)
```

のようにして回答します。揃ってない場合は`etSessionAttributes(object)`を呼び出して次のセッションへとつなげます。`drinkType`には「ヒヤ・熱燗」、`tasteType`には「甘口・辛口」を登録しています。特に技術的な苦労はありませんでしたが、「ヒヤ」は「冷」と表記すると「レイ」とClovaが読んでしまうのでカタカナ表記にしました。同じ理由で「谷川岳」も「タニガワダケ」と表記しています（谷川タケシと読まれた）。
またこの`Attributes`を場合の注意点として、一番最初のセッション、つまりは`onLaunchRequest`のときにobjectの値を初期化する必要があります。どうやらセッションを終了しても値が残り続けるみたいです。

# 動画
[![動画](http://img.youtube.com/vi/0-UQdaIulO4/0.jpg)](https://www.youtube.com/watch?v=0-UQdaIulO4)

動画は1パターンのみですが、まあ会話モデルを見てくれたら他の結果も特に聞く意味はないと分かってくれるかと思います。

# さいごに
今回のスキル開発は`Attributes`をつかうこととTypeScriptで実装することが目的だったので、一旦終わりにしてスキル公開の申請をしようかと思います。次はLINE botとの連携をやってみたいです。
