---
title: "FooPooという\U0001F4A9予測アプリを作った"
tags:
  - クソアプリ
  - v0
  - fal
private: true
updated_at: '2025-12-14T00:02:26+09:00'
id: ad7f595221ceb4755498
organization_url_name: null
slide: false
ignorePublish: false
---
# はじめに

初めてのクソアプリへの参加ということで、文字通り💩アプリを作ってみようと思いました。

# できたもの

FooPooという食べ物の画像から💩を予測するアプリです。

![スクリーンショット 2025-12-13 230722.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/209689/bba06818-688b-435d-a183-e902defe5fda.png)

![スクリーンショット 2025-12-13 230732.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/209689/07e79454-fff1-44e9-b74e-28b540f98bf9.png)



# 開発環境

v0というNext.js特化型の生成AIを使って開発しました。

https://v0.app/

最初に投げたプロンプトはこんな感じです。

```
FooPooという食べ物の写真からうんちを予測してイラスト出力するアプリを作りたい。AI機能部分は@openai/agentsを使って実装する。
```

このプロンプトだけで先ほどのUIを自動で生成してくれました。ただ、v0はUI以外にはあまり強くなくエラーが頻出したのでこちらで多少API部分の修正をしました。それでもかかった時間は２時間ほどだったのでv0は恐ろしいほど強力だと感じました。

# 分析と画像生成

今回は２つのAIを使いました。

- OpenAI
- Fal

OpenAiについては今更説明は不要だと思いますが、Falは画像などのメディア生成に特化したAIサービスです。

https://fal.ai/

これらを使って↓のように役割分担をしました。

1. OpenAIで食べ物の画像を分析
2. 分析結果から💩のイラスト生成用のプロンプトを作成
3. Falで💩のイラストを生成

# @openai/agentsでアウトプットの信頼性を高める

AIエージェントを実装するときに苦労するのが、アウトプットの型を固定することだと思います。例えば最初に与えるプロンプトが以下のような感じだとします。

```
あなたは食べ物の写真を分析して、消化後のうんちの特徴を予測する専門家です。
    
以下の形式でJSON応答を返してください：
- shape: "バナナ型", "コロコロ", "ペースト状", "やわらかめ", "硬め" のいずれか
- color: "茶色", "黄色がかった茶色", "濃い茶色", "緑がかった茶色", "黒っぽい" のいずれか
- hardness: "とても硬い", "硬め", "普通", "やわらかめ", "とてもやわらかい" のいずれか
- healthScore: 1から10の数値（10が最も健康的）
- description: 50文字以上300文字以内の日本語での詳しい説明
- illustrationPrompt: 英語でのかわいいうんちキャラクターのイラストプロンプト

食物繊維、水分、脂質などの成分を考慮して、科学的に妥当な予測を行ってください。
```

これでJSONの形でアウトプットしてくれたら、`JSON.parse`をすれば良いのですが。↓のような余計な文字で囲まれたりしてパースエラーを起こしてしまったりします。

````
```json
{
  "shape": "やわらかめ",
  "color": "茶色",
  "hardness": "やわらかめ",
  "healthScore": 7,
  "description": "この料理は麻婆豆腐で、豆腐とひき肉が主成分です。豆腐には水分が多く含まれており、繊維質は少なめのため、消化後の便はやわらかめになる可能性が高いです。また、調味料や油分が使用されているため、健康スコアは適度な範囲になります。トウガラシなどの辛味成分が含まれていることも考えられますが、量によっては消化に影響を与えることがあります。",
  "illustrationPrompt": "A cute cartoon character resembling a soft brown poop, happily smiling with a spicy texture and a tofu hat."
}
```
````

今回OpenAIで使ったのは、@openai/agentsというAIエージェント作成向けのパッケージです。

https://openai.github.io/openai-agents-js/

これとzodというバリエーションのパッケージを組み合わせて実装をしました。

https://zod.dev/

実装の雰囲気はこんな感じです。

```typescript
new Agent({
  name: "FooPoo Analyzer",
  instructions: `あなたは食べ物の写真を分析して、消化後のうんちの特徴を予測する専門家です。
  
以下の形式でJSON応答を返してください：
- shape: "バナナ型", "コロコロ", "ペースト状", "やわらかめ", "硬め" のいずれか
- color: "茶色", "黄色がかった茶色", "濃い茶色", "緑がかった茶色", "黒っぽい" のいずれか
- hardness: "とても硬い", "硬め", "普通", "やわらかめ", "とてもやわらかい" のいずれか
- healthScore: 1から10の数値（10が最も健康的）
- description: 50文字以上300文字以内の日本語での詳しい説明
- illustrationPrompt: 英語でのかわいいうんちキャラクターのイラストプロンプト

食物繊維、水分、脂質などの成分を考慮して、科学的に妥当な予測を行ってください。`,
  model: "gpt-4o",
  outputType: z.object({
    shape: z.string(),
    color: z.string(),
    hardness: z.string(),
    healthScore: z.number(),
    description: z.string(),
    illustrationPrompt: z.string(),
  }),
})
```

この`outputType`の部分で出力される型を固定化できるので、パースの部分で不安定になることがなくなります。

# Falを使う

Falを使った画像生成のリクエストもかなりシンプルです。↓のようにOpenAIから生成したプロンプトを渡します。今回は`fal-ai/flux/schnell`というモデルを使用しました。

```typescript
const fluxInput: FluxSchnellInput = {
  prompt: 'OpenAIからの分析結果で生成したプロンプト',
  image_size: "square",
  num_inference_steps: 4,
  num_images: 1,
}
await fal.subscribe<FluxSchnellInput, FluxSchnellOutput>("fal-ai/flux/schnell", {
  input: fluxInput,
})
```

# さいごに

コードは↓で公開しています。

https://github.com/ufoo68/foo-poo

あとクレジットがなくなれば終了しますが、アプリのほうも公開しています。

https://v0-foo-poo-poop-prediction.vercel.app/

今回はほぼほぼ生成AIに丸投げをしたのであまり書くことはありませんが、@openai/agentsはこういったAIエージェントを作るのにかなり便利なのでぜひ活用してみてください。
