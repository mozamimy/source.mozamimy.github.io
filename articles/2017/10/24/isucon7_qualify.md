---
title: ISUCON 7 の予選を突破した (†空中庭園†《ガーデンプレイス》)
date: 2017-10-24
tags: diary
---

今年も大盛り上がりな [ISUCON 7](http://isucon.net/archives/50949022.html) でしたが、わたしも†空中庭園†《ガーデンプレイス》というチームで、同僚の [@ryot\_a\_rai](https://twitter.com/ryot_a_rai) さんと [@eagletmt](https://twitter.com/eagletmt) さんと一緒に予選 (2 日目) に出場しました。

[ISUCON7 本選出場者決定のお知らせ : ISUCON公式Blog](http://isucon.net/archives/50956331.html) にある通り、最終スコアは **588,107** でなんと両日合わせてのトップでした。身に余る光栄..! 自分用の記録も兼ねて、チームでどのように考えて動いたのかをメモしておきたいと思います。

## 📃 リポジトリ

リポジトリは @ryot\_a\_rai さんが公開してくれていますので、以下の GitHub リポジトリを覗いてみてください。

[ryotarai/isucon7q](https://github.com/ryotarai/isucon7q)

参考実装は Golang を選択したので、主な変更は Golang の参考実装に対して行っています。また、MySQL の設定などもいじりましたが、リポジトリに載せるのを忘れていたのでここにはありません..。

## 💪 何をしたか?

何をやったのかという詳細なログを残していなかったので、コミットログを眺めながら当日の様子を思い出してみます。与えられた 3 台のサーバを、それぞれ srv1、srv2、srv3 と呼称することとし、srv1 と srv2 が初期状態でアプリケーションが動作していたサーバ、srv3 が MySQL が動作していたサーバとします。

### 🌱 下準備

地味ですが、重要。予選当日までに集まって素振りしていたので、手分けして滞りなく以下のような下準備を行いました。

- レギュレーションの確認
- SSH の設定
- 参考実装をまるっと GitHub のプライベートリポジトリにのせて push
- 動作している参考実装を Golang に切り替え
- master ブランチを pull し、バイナリをビルドして `$ sudo systemctl restart isu-go` するような deploy.sh をガッと作って srv1 と srv2 にまく
- [アクセスログ解析スクリプト (Ruby による秘伝のタレ)](https://github.com/ryotarai/isucon7q/blob/master/access-log.rb) を設置
  - [Add access-log.rb · ryotarai/isucon7q@5356cf8](https://github.com/ryotarai/isucon7q/commit/5356cf8149a5683396e30d6b5501cf52b081b193)
- 動作確認を兼ねてベンチマークをかけつつ、構成をざっと眺めて今回の問題についてあれこれ討論する

ここまでやって 1 時間くらい、初期スコアは 3000 ~ 4000 くらいだったと思います。

レギュレーションやサーバの仕様を確認していく中で、インターネットに面しているネットワークの帯域が 100Mbps ということが強調されていたのと、304 を返す場合の得点について書かれていたことから、「いかに帯域を絞るか」が効いてくる問題なのだろうなあきっと、まあ計測してみないとわかんないけど、みたいな話をしていた気がする。

### 👀 pprof を仕込む

[Install pprof. · ryotarai/isucon7q@76ec7ec](https://github.com/ryotarai/isucon7q/commit/76ec7ecdfc23bf5dd96984eb9eeb39dc6f312316)

Golang に標準で用意されている net/http/pprof を仕込みます。

Golang をあまり触ったことがなかったので素振りのときまで pprof を知らなかったのですが、実際に練習で使ってみてびっくり。ここまで便利なプロファイラがシュッと使えるのは非常に便利だなあと思いました。

ベンチマークをちょくちょく走らせてプロファイリング結果を眺めつつ、/incons や /message あたりがヤバイ、みたいなところまで突き止める。

### 🐢 スロークエリログを仕込む

pprof の導入と平行して、MySQL のスロークエリログを仕込みます。ここでも、pprof でわかったように、とにかく icons テーブルへのクエリがヤバイことがわかる。

### 🎨 画像ファイルを Redis に入れる

[Store icons to Redis · ryotarai/isucon7q@3e7e523](https://github.com/ryotarai/isucon7q/commit/3e7e523e5d693c3ee5aa7013c83ff92750981a25)

コードを見てみると、MySQL に画像ファイルのバイナリをそのまま突っ込んでいたので、これを Redis に逃がすようにしました。

ファイルの大きさが懸念事項だったけれど、大きくてもせいぜい 1MB に満たないとのことで Redis に入れる戦略をとることにしました。`/initialize_redis` という GET エンドポイントを生やし、初期化処理として MySQL から画像を根こそぎ引いてきて Redis に格納しています。

ちなみに、この時点では srv2 に Redis を置いていました。srv3 に置くという選択肢もありましたが、MySQL で CPU が飽和していたので srv2 に置くことにしました。

### ⤴️ フロントの Nginx の設定でアイコンをキャッシュする

- [Import nginx config · ryotarai/isucon7q@8ca269e](https://github.com/ryotarai/isucon7q/commit/8ca269ee0f34ea10f101f897687823f623efc349)
- [Update nginx conf · ryotarai/isucon7q@4314d59](https://github.com/ryotarai/isucon7q/commit/4314d595a41120486eebef292fbfcffeb8b5626a)
- [cache icons · ryotarai/isucon7q@c193919](https://github.com/ryotarai/isucon7q/commit/c19391901a8ca202dcb05e60b93ebdf7c0290603)

書いてあるとおりです。が、効果がなかったので戻した。

[Quit icons cache · ryotarai/isucon7q@95ffa32](https://github.com/ryotarai/isucon7q/commit/95ffa3203b937f98480de3620e6ff20f8172a822)

### 💻 srv3 にも Nginx を置いてリクエストを受けるようにする

ここで、srv1 と srv2 の CPU を全く使い切れていないことに気付く。

ここまでの作業で、帯域が絡んでくる課題だということには気づいてたため、ベンチマークを動かしつつネットワークトラフィックの状況を監視する。その結果、やはり帯域でサチっていていそうだという結論に至りました。

ここまで srv1 と srv2 の Nginx のみでリクエストを受けていたのですが、ここまでのチューニングで MySQL の負荷が少し軽くなったこともあり、srv1 と srv2 にリクエストをそのままパスする Nginx を srv3 に置き、全てのサーバでベンチマークを動かすようにしました。

[add proxy.conf · ryotarai/isucon7q@826f045](https://github.com/ryotarai/isucon7q/commit/826f045417961991f2eb7c8d4ff87ac7b492687e)

具体的な数字は思い出せませんが、ここまでやってジワジワスコアが上がりはじめて 3 人ともテンション上がってきた記憶があります。

### 📈 キャッシュコントロールを仕込んでブレイクスルーが起きる

- [Try 304 · ryotarai/isucon7q@6a5291d](https://github.com/ryotarai/isucon7q/commit/6a5291d91592180e837a3e21e21efb8811faeeb2)
- [Return Cache-Control · ryotarai/isucon7q@8730db5](https://github.com/ryotarai/isucon7q/commit/8730db54be56ad3bf6295915f7a29311b5ff6bef)

ここまでの作業で、愚直に 200 でアイコンを返すよりも、304 を返した方が点が稼げそうなことは明らかででした。

そこで、`ETag`・`Last-Modified`・`Cache-Control` ヘッダをアプリで付与して返すようにしたところ、ブレイクスルーが起こりました。この時点でスコアが 100,000 を超えて、ドキドキしてアドレナリンがモリモリ出てくる感覚があったことを覚えています。

ちなみに、ベンチマーカの挙動を調べるためにこの変更を入れるまでにパケットキャプチャしてヘッダを覗いたりしていました。

### 🔪 Kill N+1、徐々に MySQL から Redis にデータを逃す

/icons にまつわるスロークエリは既に潰していたので、次は他のスロークエリを潰しにかかります。

[一部のメッセージ数を Redis に格納するようにした](https://github.com/ryotarai/isucon7q/commit/109fae512adc7316d132a54daa87d3a011c318e7)り、[N+1 を潰したり](https://github.com/ryotarai/isucon7q/commit/9f68fa2d2efde33ee808758fd03ac579ee1692d1)、InnoDB のパラメータをチューニングしたりしました。

また、ここまでの作業で MySQL の負荷が下がったため、Redis を srv2 から srv3 にこのタイミングで移していたと思います。

このぐらいまでやったところで、スコアは 400,000 くらい。だんだんとやれることがなくなってきて 3 人とも手が止まってきます。

### 💡 Redis のシャーディングを仕込む

この時点で 18:30 ぐらい。そろそろ終盤にさしかかってきたところで、Redis を全てのサーバに置いてシャーディングのようなことをしてみてはどうか、という案が出てきます。

自分のサーバで動いている Redis に問い合わせる場合は帯域を節約できますし、CPU をさらに使い切るための戦略として @eagletmt さんが大工事してくれました。Redis クライアントを 3 つ保持しておき、キーの 3 の剰余で振り分けるという実装です。

[Use 3 Redis · ryotarai/isucon7q@f932e38](https://github.com/ryotarai/isucon7q/commit/f932e380aacc92bb6b053fce8917637e75246ac6)

この辺でまた少しスコアが上がり、500,000 くらいまで到達していたと思います。

### 🔨 不要なものを切って再起動チェックをする

さて、このあたりですでに 20:00 近くになっており、試合終了の 21:00 まで残り 1 時間強となりました。この辺からはリスクの高いチューニングが不可能となります。ましてやトップを狙えるスコアを出していたこともあり、日和ってこのあたりからは実装や構成に全く手を入れていません。

とはいえ少しでもスコアを上げたいため、最後の仕上げとしてプロファイラを外して各種ログを切ってみたところスコアが 590,000 ほどまで伸びました。特に効いたのが Nginx のログのようで、アクセスログを書き出すのも結構 CPU を使っているのだな〜という気づきがありました。

再起動チェックも滞りなく終了し、20:20 ぐらいの時点で出した 588,107 でスコアを凍結、あとは祈りながら待つ :pray: という感じでした。

## 🐰 というわけで

最終スコアは **588,107** でフィニッシュ、なんと両日通して 1 位というスコアで予選を通過することができました。

ISUCON 6 のときは初出場ということもあり、ほとんどなにもできずに悔しかった思いがありました。しかし、今回は厚めに個人練習をしていたことと、SRE というポジションに移ってから実務で 1 年強やってきた経験が生きて、去年と比較して圧倒的に動けていたと思います。

とはいえやはり @ryot\_a\_rai さんと @eagletmt さんが非常に頼りになるということもあり、終始おんぶにだっこだったと思います。本戦まで 1 ヶ月ほどあるので、個人練習を重ねてもっと動けるようになれるように準備しようと思っています 💪 この場をお借りして、チームメンバーのふたりにはお礼申し上げます。

そして ISUCON 7 のスタッフの皆様、本当に素晴らしい問題を提供してくださり、ありがとうございました。その苦労を考えると、ほんとうに頭の下がる思いです 🙇‍♀️

それではみなさん、ミライナタワーで会いましょう 👋
