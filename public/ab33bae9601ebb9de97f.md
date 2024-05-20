---
title: AWS Amplify Studioを使ってローコードでLIFFアプリを作る２
tags:
  - amplify
  - LIFF
private: false
updated_at: '2021-12-25T07:02:58+09:00'
id: ab33bae9601ebb9de97f
organization_url_name: null
slide: false
ignorePublish: false
---
# はじめに

こちらの続きです。

https://qiita.com/ufoo68/items/bc76e2c84af2e387dc06

今回はデータモデルを使ってLIFFでは取得できなかったユーザーのプロフィールデータを拡張してみようと思います。前回LIFFで取得できなかったデータは以下の２つです。

- 背景画像
- 自己紹介文(本当はgetProfileで取得できる`statusMessage`を代用しても良かったのですが、今回はあえてデータモデルを使ってみます)

この２つをデータモデルを使って拡張したいと思います。

# データモデルを作る

`Content`を選択してデータモデルを作成します。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/209689/4ca719c5-f271-251a-c512-b38f6eb15866.png)

`Profile`という名前で以下の要素を付け足します。

- liff_id(liffのユーザーIDと紐付けるため)
- background_image(背景画像のURL)
- bio(自己紹介文)

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/209689/ef0f6e75-d7fe-680f-d633-8243bc280568.png)


後は`Save and deploy`を実行して、後は各項目にデータを記入します。
`liff_id`に関してはgetProfileから取得できる`userId`を入力することにします。本来であれば専用フォームを実装したいところですが、今回は割愛して直接データ入力をすることにします。
ちなみにLIFFのplaygroudがあるので、単純に自分の`userId`を知りたい！などちょっとした用途でLIFFライブラリを使いたいときは↓を使うといいと思います。

https://liff-playground.netlify.app

# UI Libraryを変更する

Profileというデータモデルが作成できたので、これを前回Figmaから取り込んだProfileCardのコンポーネントに適用します。
`ProfileCard`の`Type`を`Profile`に設定します。あとはモデルデータを適用したいプロパティを選択してフィールドを選択します。（ここでは`background`のプロパティに対して、`backgroud_image`を`src`に適用）

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/209689/0f365ddb-ddbb-8e63-c84e-94e3243781f3.png)

変更したら、`amplify pull`で変更分をマージしましょう。

# modelのデータを利用する

データモデルを作成することで、`./models`というフォルダが作成されているはずです。このmodelsで定義されているクラスは、DataStoreのライブラリを使ってデータの操作を行うことができます。

https://docs.amplify.aws/lib/datastore

今回は`query`を使ってデータの読み取りを行います。

```js
import { Profile } from './models';
import { DataStore } from 'aws-amplify';

const profiles = await DataStore.query(Profile);
```

# 実装

コード全体は以下に公開されています。

https://github.com/ufoo68/my-liff-profile

`DataStore.query`を使って、データモデルの中に入力したデータを取得します。

```js:src/App.js
import { useEffect, useState } from 'react';
import { ProfileCard } from './ui-components';
import liff from '@line/liff';
import './App.css';
import { Profile } from './models';
import { DataStore } from 'aws-amplify';

function App() {
  useEffect(() => {
    liff.init({ liffId: process.env.REACT_APP_LIFF_ID }).then(async () => {
      if (!liff.isLoggedIn()) {
        await liff.login();
      }
      const profile = await liff.getProfile();
      const profiles = await DataStore.query(Profile);
      const { bio, background_image } = profiles.find((value) => value.liff_id === profile.userId);
      setProfile({
        avatorImageSrc: profile.pictureUrl,
        userName: profile.displayName,
        backgroundImageSrc: background_image,
        bio,
      });
    }).catch((error) => {
      console.error(error);
    });
  }, [])
  const [profile, setProfile] = useState({
    avatorImageSrc: '',
    userName: 'user name',
    bio: 'user profile',
    backgroundImageSrc: ''
  });
  return (
    <div className="App">
      <ProfileCard ProfileCard={{ bio: profile.bio, background_image: profile.backgroundImageSrc }} overrides={{
        "View.Image[1]": {
          alt: 'アバター',
          src: profile.avatorImageSrc,
        },
        "View.Text[0]": {
          children: profile.userName,
        },
      }} />
    </div>
  );
}

export default App;
```

今回は簡単のためにデータを全件取得した後にLIFFの`userId`とデータモデルの`liff_id`が一致する`Profile`を取り出すような実装を行いました。

# さいごに

データモデルに↓こんな感じでデータを入力してみました。

![無題の図形描画.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/209689/4d5ffcad-f7f7-0132-d0a3-7e150916b6a2.png)

その入力したデータ内容が事項紹介文と背景画像に反映されました↓。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/209689/9c294395-8a0a-4b12-12d7-67c5ab5631da.png)

データモデルの使い方もなんとなくわかった気がするので、気が向いたらちゃんとしたアプリも作ってみたいなと思いました。
