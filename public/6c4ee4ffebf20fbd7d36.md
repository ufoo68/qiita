---
title: AppSyncをクライアントで動かす（query編）
tags:
  - React
  - amplify
  - AppSync
private: false
updated_at: '2020-05-03T21:05:26+09:00'
id: 6c4ee4ffebf20fbd7d36
organization_url_name: null
slide: false
ignorePublish: false
---
# はじめに

今日は[GWアドベントカレンダー](https://gw-advent.9wick.com/calendars/2020/89)５日目です。[昨日](https://qiita.com/ufoo68/items/73abb40716ebd4999c27)は事前準備をやったので今日はGraphQLの実装を進めます。今回は`allItem`のクエリを実行してデータを取得します。
また、GraphQLのサーバーですがCognito認証があると実装が面倒だったので[２日目の内容](https://qiita.com/ufoo68/items/512b74156789a128bc95)のものに一旦戻しました（目的はGraphQLのクエリを叩くことだったので）

# 必要パッケージのインストール

Amplifyのためのパッケージをインストールします。

```bash
yarn add aws-amplify-react aws-amplify
```

また今回はFooks(`useState()`)でクエリを叩きたいので、[availity](https://availity.github.io/)というFooksで簡単に非同期処理が扱えるパッケージをインストールします。

```
yarn add @availity/api-axios @availity/api-core @availity/hooks
```

# 実装

見せるほどの内容ではありませんが、単純に`useState()`で`allItem`のクエリを叩く処理はこんな感じになりました。

```typescript
import React, { useState } from 'react'
import { useEffectAsync } from '@availity/hooks'
import { API, graphqlOperation } from 'aws-amplify'
import { allItem } from './graphql/queries'

const App = () => {
  const [item, setItem] = useState([{
    id: '',
    data: '',
  }])
  useEffectAsync(async () => {
    const result = await API.graphql(graphqlOperation(allItem))
    setItem(result.data.allItem)
  })
  return (
    <ul>
      {item.map(i =>
        <li>
          {i.data}
        </li>
      )}
    </ul>
  )
}

export default App
```

前回の`codegen`で`./graphql`というフォルダと、`query`・`mutation`それぞれのクエリ用のファイルが自動で生成されたのでそれを用います。
`API.graphql()`で返ってきた結果は`data`というフィールドにネストされた形でいつもどおりのクエリ実行結果が返ってくるみたいですね。

# さいごに

AmplifyとAppSyncを用いると１行のコードでデータのフェッチが実装できてしまうようですね。バックエンドについてはロジックの実装を一切行っていないので爆速でアプリがつくれてしまうのがこの組み合わせの利点ですね。今回はここまでで明日は`mutation`を実装していきます。ではまた！
