---
title: next-s3-uploadとMinIOでローカルのファイル投稿環境を構築する
tags:
  - S3
  - minio
  - Next.js
private: false
updated_at: '2024-09-25T10:53:32+09:00'
id: 9440e23a3cc9f09f4bc5
organization_url_name: null
slide: false
ignorePublish: false
---
# はじめに

最近Next.jsのウェブアプリでファイルの投稿機能を追加するときは[next-s3-upload](https://next-s3-upload.codingvalue.com/)を使っているのですが、ファイルストレージサービスを実環境のS3を使っていたので、ローカル用のS3環境を立ててみたいと思って色々と調べてやってみました。

# MinIOを使う

`MinIO`を使うことでS3互換性のあるサービスをDockerで構築することができます。

https://min.io/

以下は`docker-compose.yml`の例です。

```yml:docker-compose.yml
services:
  minio:
    image: quay.io/minio/minio:latest
    container_name: s3-minio
    environment:
      MINIO_ROOT_USER: admin123
      MINIO_ROOT_PASSWORD: admin123
    command: server --console-address ":9090" /data
    volumes:
      - ./docker/minio/data:/data
    ports:
      - 9000:9000
      - 9090:9090
```

ここではユーザー名とパスワードをadmin123で設定しています。まずは以下のコマンドでDockerを起動します。

```bash
docker compose up -d
```

次に`http://localhost:9090`にアクセスをしてコンソール画面にアクセスして、先ほど設定したユーザー名とパスワードでログインします。

![スクリーンショット 2024-09-25 10.30.37.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/209689/27e74d81-e2e1-2b65-13d4-13d5518aa2ab.png)

左メニューから`Buckets`を選択して`Create Bucket`でストレージを新規作成します。

![スクリーンショット 2024-09-25 10.28.36.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/209689/aa2e2575-2970-4abc-6798-649108a64fe1.png)

左メニューから`Access Keys`を選択して`Create Access Key`でアクセスキーを新規作成します。

![スクリーンショット 2024-09-25 10.32.11.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/209689/7a3ce098-0cde-81a2-94de-fb05f781951f.png)

これでローカルのS3の準備ができたので、作成した以下の情報を記録しておきます。

- バケット名
- アクセスキー
- シークレットキー

# next-s3-uploadでMinIOと繋ぎこみ

前提として`Next.js`と`next-s3-upload`は既にインストールされているものとします。今回はNext.jsの`page router`で実装するとして、`src/pages/s3-upload.ts`で以下を実装します。

```ts:src/pages/s3-upload.ts
import { APIRoute } from 'next-s3-upload'

export default APIRoute.configure({
  accessKeyId: process.env.S3_UPLOAD_KEY,
  secretAccessKey: process.env.S3_UPLOAD_SECRET,
  bucket: process.env.S3_UPLOAD_BUCKET,
  region: process.env.S3_UPLOAD_REGION,
  endpoint: process.env.S3_UPLOAD_ENDPOINT,
  forcePathStyle: true,
})
```

`forcePathStyle: true`という設定は、AWS S3以外を用いる場合は重要になるので必ずtrueで設定しておきます。`.env`は以下のように設定します。

```.env
S3_UPLOAD_KEY=アクセスキー
S3_UPLOAD_SECRET=シークレットキー
S3_UPLOAD_BUCKET=バケット名
S3_UPLOAD_REGION=ap-northeast-1
S3_UPLOAD_ENDPOINT=http://localhost:9000
```

`S3_UPLOAD_REGION`と`S3_UPLOAD_ENDPOINT`については内容そのままコピペで大丈夫です。AWS以外の環境で実装する場合の詳細は[公式ドキュメントのOther providersの項](https://next-s3-upload.codingvalue.com/other-providers)でまとめてあります。
あとはファイル投稿画面を実装するだけですが、アップロード機能のhooksは`usePresignedUpload`を使います。以下はコード例です。

```tsx
import { FC, useState } from 'react';
import { usePresignedUpload } from 'next-s3-upload'

export const Component: FC = ({ contents, handleSave }) => {
  const { uploadToS3, FileInput, openFileDialog } = usePresignedUpload()
  const [imageUrl, setImageUrl] = useState<string>('')
  const handleFileChange = async (file: File) => {
    const { url } = await uploadToS3(file)
    setImageUrl(url)
  }
  return (
    <>
      {imageUrl ? (
        <img src={imageUrl} />
      ) : (
        <div>サムネイルをアップロードしてください</div>
      )}
        <FileInput
          type="file"
          onChange={handleFileChange}
        />
        <button type="button" onClick={openFileDialog}>
          ファイル選択
        </button>
    </>
  )
}
```

# さいごに

MinIOとnext-s3-uploadの連携方法について、書いている記事が見つからなかったので今回はそれについてまとめてみました。最初の導入は若干面倒ではありますが、一度構築してしまえば料金の心配なしに開発ができるので、結構便利だと思います。
