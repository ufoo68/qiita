---
title: TurboWarpでmqtt-min.jsというMQTTカスタムブロックを実装した話
tags:
  - JavaScript
  - mqtt
  - Scratch
  - IoT
  - turbowarp
private: false
updated_at: '2025-12-11T07:03:35+09:00'
id: e0cc2e4a87b1976de8d7
organization_url_name: iotlt
slide: false
ignorePublish: false
---
# はじめに

TurboWarp/Scratchの作品をIoTとつなぎたい時、ブラウザからMQTTを直接扱える拡張ブロックが欲しくなるのですが、既存の拡張は常時稼働している仲介サーバーが必要だったり、ブラウザのWebSocket制限に引っかかったりと気軽に使えるものが少ないと感じていました。そこで、Codex（OpenAIのコーディングアシスタント）にペアプロしてもらいながら `MiniMQTT` という超ミニ実装を仕上げ、`mqtt-min.js` として公開しました。私は仕様整理や検証に集中し、実際のコード生成や最適化はCodexに頼っています。

リポジトリ: https://github.com/ufoo68/turbowarp-custom-block  
CDN URL: https://ufoo68.github.io/turbowarp-custom-block/mqtt-min.js

# どんなブロックか

![拡張ブロック](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/209689/06eb553b-def3-443b-bab1-6465673ac1d3.png)

TurboWarpの「拡張機能を追加」から上記URLを貼ると、以下のブロックが出現します。

- `MQTTに接続 scheme://host:port/path clientId ...`
- `MQTT切断` / `接続中？`
- `購読 topic ...`
- `publish topic ... msg ...`
- `最後に受信したメッセージ`
- `MQTTで受信したら`（ハットブロック）

# Codexと進めたプレーンJavaScript実装のざっくりメモ

プレーンJSでMQTTをしゃべると聞くと難しく感じますが、Codexと会話しながら次の3要素だけ押さえれば十分でした。

- **パケット生成**: CONNECT/SUB/PUBなどの先頭バイトとRemaining Length、文字列の長さプリフィックスを素直に `Uint8Array` で組み立てるだけ。`TextEncoder/Decoder` を使えばUTF-8処理も完結。
- **ブラウザ接続**: `new WebSocket(url, ['mqtt'])` でバイナリ接続し、`onopen` でCONNECT、`setInterval` でPINGを送る。`binaryType='arraybuffer'` にしておけば受信ハンドラはそのままバイト列として扱える。
- **Scratch連携**: メッセージを受けたら `runtime.startHats('minimqtt_onMessage')` を呼び出すだけでハットブロックが反応。複数メッセージは `_pendingMessages` でキューにしておけば取りこぼしにくい。

ざっくり流れが掴めるように、Codexが生成したコードを最小限にしたスニペットを貼っておきます。

```js
const ws = new WebSocket('wss://broker.hivemq.com:8884/mqtt', ['mqtt']);
ws.binaryType = 'arraybuffer';

ws.onopen = () => {
  ws.send(pktConnect('scratch-' + Math.random().toString(16).slice(2)));
  setInterval(() => ws.send(pktPing()), 24000);
};

ws.onmessage = (ev) => {
  const { type, payload } = parseMqtt(new Uint8Array(ev.data));
  if (type === 'publish') {
    lastMsg = new TextDecoder().decode(payload);
    runtime.startHats('minimqtt_onMessage');
  }
};
```

`pktConnect` や `parseMqtt` は本文で紹介した `Uint8Array` ベースの関数をそのまま呼んでいるだけで、ライブラリに頼らなくてもMQTTの最小セットが動くことがわかると思います。

Codexがバイナリ処理の骨組みを一気に書いてくれるので、私は仕様の確認とテストに集中できました。MQTTが分からなくても、この3点さえ把握していれば最低限のクライアントを作れるはずです。

# 使い方

TurboWarpエディタを開き、「拡張機能を追加」から `https://ufoo68.github.io/turbowarp-custom-block/mqtt-min.js` を貼る。

![スクリーンショット 2025-11-10 234532.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/209689/3a1c0eb9-84b0-4016-bacc-f5a57f693277.png)

例として HiveMQ Public Broker を使う場合は `SCHEME: wss / HOST: broker.hivemq.com / PORT: 8884 / PATH: mqtt` を指定して接続。

# 動作検証

以下のような感じでブロックを組んでみました。

![スクリーンショット 2025-11-10 235633.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/209689/f7a50fe6-02c0-4826-a37f-142fb4af6f9e.png)

1. `SCHEME: wss / HOST: broker.hivemq.com / PORT: 8884 / PATH: mqtt`に接続
2. `topic: scratch/mqtt/in`でsubscribe
3. メッセージを受信したら`topic: scratch/mqtt/out`でメッセージを返す

簡単のためもう一方の通信クライアントは https://www.hivemq.com/demos/websocket-client を使いました。結果としてそれぞれにメッセージのやり取りができたことを確認できました。

![スクリーンショット 2025-11-11 000223.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/209689/baf82d9b-f2a8-43c9-a490-998f3c0e9a0c.png)

![スクリーンショット 2025-11-11 000243.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/209689/fd5679f8-6cac-43a1-b9c6-1fc2584bf9bd.png)

# おわりに

MQTTは仕様がコンパクトなので、Scratchの拡張としてもまだ遊べる余地があります。`mqtt-min.js` をぜひ環境に合わせてforkしたり、より高度な拡張を作る際のベースにしてみてください。
