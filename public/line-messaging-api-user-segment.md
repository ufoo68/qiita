---
title: LINE Messaging APIを使ったユーザーセグメントについて
tags:
  - automation
  - LINE
  - Marketing
  - CRM
  - MessagingAPI
private: false
updated_at: '2025-12-09T07:03:48+09:00'
id: 5cb8c82f36f6e05d72fc
organization_url_name: null
slide: false
ignorePublish: false
---
# はじめに

LINEでユーザーセグメントをしたいと思って調べていると、公式ドキュメントにでてくる用語と自分のやりたいことの関連性がいまいちわかりにくかったので記事でまとめてみることにしました。ざっくり言うと「ユーザーセグメント＝ナローキャストのターゲット」という意味で、APIレベルでは `narrowcast` の `recipient` にセットする情報をどう準備するか、という話になります。本記事ではその腹落ちポイントと、実装・運用でつまづきやすかったところをまとめました。

# ユーザーセグメント = ナローキャストでできること

- クリックや購買といった行動に応じてナーチャリング配信を分岐できる
- リテンションが落ちた層だけにクーポンを配布し、配信コストを抑えられる
- A/Bテスト用に小さな母集団へナローキャストし、結果が良かった配信だけを全体展開できる
- 取得したオーディエンス ID をデータ基盤と突き合わせることで、LINE外の行動と掛け合わせた分析ができる

# User segmentとNarrowcastの頭の整理

公式ドキュメントでは「User segment」という言葉が単体で出てくるものの、実態はナローキャストの `recipient` にセットできる条件群のことです（[narrowcast API](https://developers.line.biz/ja/reference/messaging-api/#send-narrowcast-message)）。`recipient` には `audienceGroupId` だけでなく `friends`, `demographic` など複数の条件を持ったJSONを渡せます。このJSONをどう組むか＝どんなセグメントを作るか、という理解に置き換えると話がシンプルになります。

イメージとしては以下のような構図です。

```
ユーザー行動ログ → セグメント設計 → (Audience API or Demographic filter) → narrowcast.recipient
```

Demographicだけで済むならオーディエンスは不要、より細かいルールが欲しい場合はAudience APIで `audienceGroupId` を作る、という役割分担です。

# LINE Messaging APIで使えるセグメント手段

## デモグラフィックフィルタ

`/v2/bot/message/narrowcast` の `filter.demographic` に性別・年代・居住エリア・OS・友だち期間を指定すると、LINE社が推定した属性で自動的に絞り込みできます（[Demographic filterドキュメント](https://developers.line.biz/ja/reference/messaging-api/#narrowcast-demographic-filter)）。顧客の個人情報を保持しなくても使えるので、まずはここから組み合わせるのが定番です。

## オーディエンスグループ

Messaging APIが保持するユーザー ID（`userId`）をもとに任意の集合を作れる機能です（[Audience APIリファレンス](https://developers.line.biz/ja/reference/messaging-api/#manage-audience-group)）。代表的な種類は以下の通りです。

| 種類 | 作り方 | 代表的な用途 |
| --- | --- | --- |
| Upload | サーバー側で取得した `userId` をAPIでアップロード | 会員ランク・購買履歴など社内DBベースのセグメント |
| Click | 特定リッチメニュー・URLのクリック者を自動収集 | 反応した人だけに追加情報を配信 |
| Impression | ビデオメッセージ視聴などのインプレッションデータから自動生成 | 視聴完了者へのフォローアップ |
| Chat tag | LINE Official Account Managerで手打ちしたチャットタグを同期 | 相談チャネルで付与した属性を配信へ活用 |

オーディエンスは最大500個まで、1つあたり100万人まで保持できます。サイズは `GET /v2/bot/audienceGroup/list` で確認します。

# セグメント作成の基本フロー

実際に組んでみて「ここで迷うな」と思ったポイントを順番に並べるとこんな感じでした。

1. LINEで友だちになったタイミングの `userId` をWebhookで拾い、会員IDなどと突き合わせてDBに保存しておく
2. 「休眠30日」や「VIP会員」のような条件を社内データ基盤で評価し、userIdのリストを生成する
3. Audience APIでリストをアップロードして `audienceGroupId` を発行する（ここまでがいわゆるUser segment作り）
4. ナローキャスト送信では `recipient.audienceGroupId` に今作ったIDを入れつつ、必要な場合だけDemographicフィルタでさらに絞り込む
5. `GET /v2/bot/message/progress/narrowcast` で配信の通過率を監視し、BIツールで効果測定する

## 例: ユーザーIDのアップロードでオーディエンスを作る

```bash
curl -X POST https://api.line.me/v2/bot/audienceGroup/upload \
  -H "Authorization: Bearer ${CHANNEL_ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
        "description": "2024Q4_休眠会員",
        "isIfaAudience": false,
        "audiences": [
          { "id": "U4af4980629..." },
          { "id": "U91eeaf62d..." }
        ]
      }'
```

レスポンスに含まれる `audienceGroupId` を控えておきます。リストが長い場合は1万件ずつ分割し、`PUT /v2/bot/audienceGroup/upload` で追加入力します。

## 例: 作成したセグメントにナローキャスト送信

```bash
curl -X POST https://api.line.me/v2/bot/message/narrowcast \
  -H "Authorization: Bearer ${CHANNEL_ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
        "messages": [
          {
            "type": "text",
            "text": "今週末まで使える復活クーポンです"
          }
        ],
        "recipient": {
          "type": "audience",
          "audienceGroupId": 1234567890
        },
        "filter": {
          "demographic": {
            "area": ["tokyo", "kanagawa"],
            "subscriptionPeriod": ["day_7", "day_30"]
          }
        },
        "limit": {
          "max": 5000
        }
      }'
```

`limit.max` を設定しておくと、想定よりセグメントが膨らんだ場合でも送信母数を制御できます。配信後は `GET /v2/bot/message/progress/narrowcast?requestId=xxxx` で必ず成功・失敗件数を確認しましょう。

# まとめ

- User segmentという言葉は「ナローキャストのrecipientに渡す条件セット」と捉えると理解しやすい
- デモグラフィックフィルタとオーディエンスグループを組み合わせれば、配信母数をコントロールしながら成果を最大化できる
- セグメントのサイズや鮮度を定期的に棚卸しするオペレーションを組み、作って終わりにしない

# おわりに

LINEのセグメント周りはドキュメントを拾い読みするだけだと「User segmentってどこで設定するの？」となりがちですが、実装の視点で分解すると `narrowcast` 一本に集約されているだけでした。ここが腑に落ちると、あとは配信設計をどう自動化するかの話に集中できるので、同じところで迷った方の助けになればうれしいです。
