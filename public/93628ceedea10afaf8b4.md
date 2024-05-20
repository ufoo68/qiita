---
title: '「Bonfire Frontend #6」参加まとめ '
tags:
  - 勉強会メモ
private: false
updated_at: '2020-08-11T20:21:44+09:00'
id: 93628ceedea10afaf8b4
organization_url_name: null
slide: false
ignorePublish: false
---
# はじめに

[Bonfire Frontend #6](https://yj-meetup.connpass.com/event/181363)の内容を私的にまとめます。

# Reactを使ったYahoo!地図の技術刷新への挑戦 by 一円 真治さん

[Yahoo!地図のPC版](https://map.yahoo.co.jp/blog/archives/20200330_map_pcverup.html)を[React](https://ja.reactjs.org/)に刷新する話でした。歴史が長いことからレガシー技術が蓄積され、追加の実装が困難になってきたことなどの背景があるようです。地図に関しては新たに[Mapbox](https://www.mapbox.com/)を用いたものに移行したとのこと。
ちなみに、サーバーサイドは[Express](https://expressjs.com/ja/)を用いており全体的に[Node.js](https://nodejs.org/ja/)を用いた実装になっているようです。

課題として残っていることとして、SEOの観点からSSRを今後は検討したり、[cypress](https://www.cypress.io/)を使ったE2Eテストなどがを行っていくとのこと。

# Yahoo! BEAUTY パフォーマンス改善とCore Web Vitals対応について by 松本 華歩さん

Yahoo!ビューティーのパフォーマンス改善のために[Lighthouse CLI](https://www.npmjs.com/package/lighthouse)を用いた計測を定期的に行っているといった話や、[Core Web Vitals](https://ferret-plus.com/15951)というUX向上のための新たな指標への取り組みについての話について行われました。
Core Web Vitalsの計測については[Next.js](https://nextjs.org/)のメソッドを用いた計測を行っているとのこと。

例として、CLSへの対応として画像読み込み中のUIを作成したり、LCP対応としてCSSのセレクタ削減などを行ったようです。

パフォーマンス改善のために*Lighthouse**などを使った数値的な計測を行うと定量的な評価ができて良さそうです。

# 広告管理ツールのプロダクト刷新・改善 by 小川 健史さん

広告管理ツールのUIの刷新についての話でした。ここではPJに**スクラム開発**を取り入れて行ったとのこと。
スプリント（ここでは２週間）ごとに期間を区切っての開発を行うことで、そのときわかっている範囲で最小のものを作っていくという進め方を行った。結果的には大きな手戻りが発生することがなかったとのことです。
また、PRのレビューコスト削減のためにレビュアーを２人体制としたり、lintの導入を行うなどの工夫をしたようです。

細かくリリースを続けることによって、都度改善を行えるのがスクラム開発の利点かと思いました。

# PayPayモール 速度改善に挑んだ日々 by 清土 伸康さん

ここでは主に速度改善の話がされました。開発には**ドメイン駆動設計**を取り入れていて、機能の拡張などに対応しやすくなっているとのこと。

速度改善のためには以下のような取り組みを行ったとのこと。

* 表示画像をプリロードさせる
* 初期表示（FV）をSSRする
* AMP Optimizerの使用
* IaaSからCaaSへの移行

最終的には１秒以上の削減に成功したようです。
