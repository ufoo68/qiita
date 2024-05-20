---
title: GPT4oでLINEスタンプ返却botをつくってみる
tags:
  - 'LINE'
  - 'ChatGPT'
private: false
updated_at: ''
id: null
organization_url_name: linedc_jp
slide: false
ignorePublish: false
---
# はじめに

最近GPT4oが発表されました。

https://openai.com/index/hello-gpt-4o/

LINEDCでもGPT4oを使った記事が早速出始めています！

https://qiita.com/miu_crescent/items/a380473822adc8c98778

今回は、以前に試して失敗したLINEスタンプ返却bot作成をリベンジしてみようと思います。

https://qiita.com/ufoo68/items/57c54217457c16fabbdb

# GPT作成の流れ

前回の失敗の反省として、「ページのスクリーンショットの画像データを直接GPTに読み込ませることをしても効果はなさそう」ということがわかったので、以下のような流れで進めることにしました。

1. ChatGPT4oでLINEスタンプのページのスクリーンショットからテキストファイルを作成
2. テキストファイルをナレッジベースにAssistantsを作成

## テキストファイル作成

簡単のためにサリーのLINEスタンプの部分を切り取ったスクリーンショットを用意しました。

![developers.line.biz_ja_docs_messaging-api_sticker-list_.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/209689/d80044f3-1e8f-e16f-1755-eb11a557aeff.png)

最初はページ全体のスクリーンショットを用意したのですが、うまくいかなかったので一部のみを切り取る形にしました。次に、ChatGPT4oの画像アップロード機能を使って以下のプロンプトを投げました。

![スクリーンショット 2024-05-19 162253.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/209689/51d2ca02-5f51-cb50-07dc-34acd044f8b7.png)

そしたらこんな感じでリストを作ってくれたのでそれをそのまま`TEXT`ファイルに保存しました。

```txt
packageId: 789
stickerId: 10855
usage: 友達に「あいさつ」

packageId: 789
stickerId: 10856
usage: 喜びを表現する

packageId: 789
stickerId: 10857
usage: 食事中やおやつタイム

packageId: 789
stickerId: 10858
usage: 「OK!」と同意する

packageId: 789
stickerId: 10859
usage: 「YES!」と強調する

packageId: 789
stickerId: 10860
usage: 「NO!」と拒否する

packageId: 789
stickerId: 10861
usage: 困惑や混乱

packageId: 789
stickerId: 10862
usage: 医者のイメージ、健康について

packageId: 789
stickerId: 10863
usage: 喜びや感謝

packageId: 789
stickerId: 10864
usage: 他人のリアクションを観察する

packageId: 789
stickerId: 10865
usage: 料理や食べ物に関する会話

packageId: 789
stickerId: 10866
usage: 応援や励まし

packageId: 789
stickerId: 10867
usage: 驚きや困惑

packageId: 789
stickerId: 10868
usage: お風呂やリラックス

packageId: 789
stickerId: 10869
usage: お祝い、祝福

packageId: 789
stickerId: 10870
usage: 怪我や困難な状況

packageId: 789
stickerId: 10871
usage: リラックスや休息

packageId: 789
stickerId: 10872
usage: 成功や達成感

packageId: 789
stickerId: 10873
usage: スポーツやスケートボード

packageId: 789
stickerId: 10874
usage: 食事やおやつタイム

packageId: 789
stickerId: 10875
usage: 喜びや興奮

packageId: 789
stickerId: 10876
usage: リラックスや温泉

packageId: 789
stickerId: 10877
usage: 困惑や不思議

packageId: 789
stickerId: 10878
usage: 友情や愛情

packageId: 789
stickerId: 10879
usage: 日常の活動

packageId: 789
stickerId: 10880
usage: 運動やランニング

packageId: 789
stickerId: 10881
usage: リラックスや休息

packageId: 789
stickerId: 10882
usage: 応援やサポート

packageId: 789
stickerId: 10883
usage: 困惑や不思議

packageId: 789
stickerId: 10884
usage: 喜びやダンス

packageId: 789
stickerId: 10885
usage: スポーツや野球

packageId: 789
stickerId: 10886
usage: 喜びや感謝

packageId: 789
stickerId: 10887
usage: 寝る前のリラックス

packageId: 789
stickerId: 10888
usage: 運動やスポーツ

packageId: 789
stickerId: 10889
usage: 食事や料理

packageId: 789
stickerId: 10890
usage: お祝い、祝福

packageId: 789
stickerId: 10891
usage: 困惑や怒り

packageId: 789
stickerId: 10892
usage: 喜びや興奮

packageId: 789
stickerId: 10893
usage: スポーツやスケート

packageId: 789
stickerId: 10894
usage: 読書やリラックス
```

## Assistantsを作成

以下の設定でAssistantsを作成しました。

- モデル
  - gpt4o
- その他オプション
  - File search
  - Functions

### File search

このFile searchはAssistantsに与える辞書のようなものです。独自のDBに基づいて回答をしてほしいときに利用します。

https://platform.openai.com/docs/assistants/tools/file-search

扱えるファイル形式に制限はありますが、基本はファイルをアップロードするだけで利用できるので特に難しい知識を必要としないです。

### Functions

Function callingと呼ばれる機能です。例えばCPTから返ってくる値を関数入力に使いたいときに利用します。

https://platform.openai.com/docs/guides/function-calling

こんな感じで定義しました。

```json
{
  "name": "sendStamp",
  "description": "スタンプのIDを送信する",
  "parameters": {
    "type": "object",
    "properties": {
      "stickerId": {
        "type": "string",
        "description": "sticker ID"
      },
      "packageId": {
        "type": "string",
        "description": "package ID"
      }
    },
    "required": [
      "stickerId",
      "packageId"
    ]
  }
}
```

これでAssistantsからは`packageId`と`stickerId`という２つのIDが形式的に返ってくるようになります。

### Instructions

InstructionsはAssistantsに与える最初の命令のようなものです。今回はこんな感じでまとめました。

```
## 概要

入力されたテキスト内容に適した内容のLINEスタンプを返却してください。

## LINEスタンプの返却

sendStamp関数を使ってスタンプを返却します。そのためにpackageIdとstickerIdの２つの情報が必要となります。

## packageIdとstickerId

line-stamp-list.txtから参照してください。入力されたテキストに適した内容のusageを選択して、それに対応するpackageIdとstickerIdをsendStamp関数に入力してください。

## line-stamp-list
.txtについて

以下の３つの情報のリストとなってます。

- packageId
- stickerId
- usage
```

ナレッジベースとして与えるファイルの形式も事前に教えたほうが、File searchが起動させやすかったです。

# 動作検証

実際に作成したAssistantsをLINE botに組み込みます。今回はGASを使って作成しました（チャネル作成やAssistantsIdの取得部分は省略します）。

```js
const LINE_ACCESS_TOKEN = 'チャネルアクセストークン';
const OPENAI_APIKEY = 'API Key';

const URL_OPENAI_API = 'https://api.openai.com/v1/';
const ASSISTANT_ID = 'AssistantsID';

function doPost(e) {
  const event = parseEvent(e);
  const replyToken = event.replyToken;
  if (typeof replyToken === 'underfined') {
    return buildTextOutput('post ok');
  }
  const url = 'https://api.line.me/v2/bot/message/reply';

  try {
    const prompt = event.message.text;
    const stampInfo = convertStampInfo(prompt);
    postLineStamp(url, replyToken, stampInfo);
    return buildTextOutput('post ok');
  } catch (error) {
    return buildTextOutput('post ok');
  }
}

function buildTextOutput(content) {
  return ContentService.createTextOutput(JSON.stringify({ 'content': content }))
    .setMimeType(ContentService.MimeType.JSON);
}


function parseEvent(e) {
  return JSON.parse(e.postData.contents).events[0];
}

function postLineStamp(url, replyToken, stampInfo) {
  UrlFetchApp.fetch(url, {
    headers: {
      'Content-Type': 'application/json; charset=UTF-8',
      Authorization: 'Bearer ' + LINE_ACCESS_TOKEN,
    },
    method: 'post',
    payload: JSON.stringify({
      replyToken: replyToken,
      messages: [{
        type: 'sticker',
        ...stampInfo,
      }]
    })
  });
}

function convertStampInfo(prompt) {
  const thread = createThread();
  createMessage(thread.id, prompt);
  const resultRunThread = runThread(thread.id);
  waitRunCompletion(thread.id, resultRunThread.id);
  const fainalRun = retrieveRun(thread.id, resultRunThread.id)
  return JSON.parse(fainalRun.required_action.submit_tool_outputs.tool_calls[0].function.arguments)
}

function createThread() {
  const options = {
    method: 'post',
    headers: {
      'Content-Type': 'application/json',
      Authorization: 'Bearer ' + OPENAI_APIKEY,
      'OpenAI-Beta': 'assistants=v2'
    },
  };

  try {
    const response = UrlFetchApp.fetch(URL_OPENAI_API + '/threads', options);
    const result = JSON.parse(response.getContentText());
    Logger.log(result.id);
    return result;
  } catch (e) {
    Logger.log(e.toString());
  }
}

function createMessage(threadId, prompt) {
  const data = {
    role: 'user',
    content: [{
      type: 'text',
      text: prompt,
    }]
  };

  const options = {
    method: 'post',
    headers: {
      'Content-Type': 'application/json',
      Authorization: 'Bearer ' + OPENAI_APIKEY,
      'OpenAI-Beta': 'assistants=v2'
    },
    payload: JSON.stringify(data)
  };

  try {
    const response = UrlFetchApp.fetch(URL_OPENAI_API + '/threads/' + threadId + '/messages', options);
    const result = JSON.parse(response.getContentText());
    Logger.log(result.id);
    return result;
  } catch (e) {
    Logger.log(e.toString());
  }
}

function runThread(threadId) {
  const data = {
    'assistant_id': ASSISTANT_ID
  };

  const options = {
    method: 'post',
    headers: {
      'Content-Type': 'application/json',
      Authorization: 'Bearer ' + OPENAI_APIKEY,
      'OpenAI-Beta': 'assistants=v2'
    },
    payload: JSON.stringify(data)
  };

  try {
    const response = UrlFetchApp.fetch(URL_OPENAI_API + '/threads/' + threadId + '/runs', options);
    const result = JSON.parse(response.getContentText());
    Logger.log(result.id);
    return result;
  } catch (e) {
    Logger.log(e.toString());
  }
}
function retrieveRun(threadId, runId) {
  const options = {
    method: 'get',
    headers: {
      'Content-Type': 'application/json',
      Authorization: 'Bearer ' + OPENAI_APIKEY,
      'OpenAI-Beta': 'assistants=v2'
    },
  };

  try {
    const response = UrlFetchApp.fetch(URL_OPENAI_API + '/threads/' + threadId + '/runs/' + runId, options);
    const result = JSON.parse(response.getContentText());
    Logger.log(result.status);
    return result;
  } catch (e) {
    Logger.log(e.toString());
  }
}

function waitRunCompletion(threadId, runId) {
  let status = 'in_progress'
  const maxAttempts = 30;
  let attempts = 0;

  while (status === 'in_progress') {
    Utilities.sleep(500);
    const result = retrieveRun(threadId, runId);
    status = result.status;
    attempts++;
    if (attempts >= maxAttempts) {
      throw new Error('最大試行回数に達しました。');
    }
  };
}
```

実際にLINE上で動作確認をしたら今回はちゃんとスタンプが返却されることを確認できました！

![スクリーンショット 2024-05-19 192702.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/209689/4e7010d6-c0af-845c-5bbf-774e400b0224.png)

# さいごに

とりあえず中途半端で終わっていたLINEスタンプ返却bot作成のリベンジが出来て良かったです。毎回ちゃんと適した内容のスタンプが返ってくるかというと微妙なところではあるのですが、そのあたりはナレッジベースを充実させることで精度が向上できるのではと思います。
また今回GASで実装するにあたって、既にドンピシャな内容の先行記事があったのでそちらを大いに参考にさせていただきました（非常に助かりました、ありがとうございます）。

https://qiita.com/tregu148/items/5cc3146c531e483ea23f