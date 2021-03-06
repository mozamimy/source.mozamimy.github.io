---
title: Cerebro の実用的な Docker image を作った (Elasticsearch 管理のおともに)
date: 2017-03-12
tags: middleware
---

## Cerebro とは

Elasticsearch クラスタを運用するにあたって、REST API を使ってインデックスやクラスタそのものの操作をすることは避けられません。
その際、API の結果として返ってくる JSON だけでクラスタの状態を想像してオペレーションを行うのは、ウサギや人類には結構厳しいものがあります。

また、インフラエンジニアだけではなく、サービスエンジニアが自分たちのプロダクトで使っている Elasticsearch を責任を持って管理できるようにするためにも、使いやすい管理ツールは強力な武器となります。

Elasticsearch 5.0 以前は [Kopf](https://github.com/lmenezes/elasticsearch-kopf) というプラグインが主流でしたが、site plugin が廃止されたためスタンドアロンな管理ツールが必要になります。
site plugin の廃止については以下のブログ記事が詳しいです。

[Running site plugins with Elasticsearch 5.0](https://www.elastic.co/blog/running-site-plugins-with-elasticsearch-5-0)

Kopf もスタンドアロンで動くようですが、Kopf のオリジナルの作者である [@lmenezes](https://github.com/lmenezes) の新作である [Cerebro](https://github.com/lmenezes/cerebro) に置き換わる予定のようです。
業務で Docker 化して Amazon ECS に乗せて導入してみたところ、普通に実用段階でした。いますぐ Kopf を捨てて Cerebro に乗り換えましょう。

## すぐに業務で使える Docker image

Cerebro が v0.5.1 のときに初めて導入し、Dockerfile は会社の GHE のリポジトリに置いていました。
現在 [v0.6.1](https://github.com/lmenezes/cerebro/releases/tag/v0.6.1) が available なので、いい機会なのでアップデートがてら Dockerfile を OSS にしました。
ちょうどご家庭の Elasticsearch にも導入したかったので。

- GitHub: [https://github.com/mozamimy/docker-cerebro](https://github.com/mozamimy/docker-cerebro)
- Docker Hub: [https://hub.docker.com/r/mozamimy/cerebro](https://hub.docker.com/r/mozamimy/cerebro)

使い方は、GitHub リポジトリに置いてある docker-compose.yml を見てください。

<p>
{{ embed_code "/2017/03/12/docker-compose.yml" }}
</p>

docker-compose.yml では例として Elasticsearch コンテナを 2 台起動してクラスタを組んでいますが、これはあくまでローカルで試すためのサンプルです。
実際には `HOSTS` 環境変数に対象の Elasticsearch の FQDN をプリセットとして用意する感じになると思います。
`APPLICATION_SECRET` には適当なランダムな文字列を秘匿値としてセットしてください。

`docker-compose up` すると、こんな感じでシュッと動きます。

<a href='/images/2017/03/12/docker_cerebro/cerebro_top.png'><img alt='cerebro_top' src='/images/2017/03/12/docker_cerebro/cerebro_top.png' style='width: 500px;'></a>

オシャンティ〜🐰💕

<a href='/images/2017/03/12/docker_cerebro/cerebro_cluster.png'><img alt='cerebro_top' src='/images/2017/03/12/docker_cerebro/cerebro_cluster.png' style='width: 500px;'></a>

たぶん v0.6.1 からなのですが、各ノードのリソースの様子をリアルタイムで見れる画面が追加されていました。

<a href='/images/2017/03/12/docker_cerebro/cerebro_nodes.png'><img alt='cerebro_top' src='/images/2017/03/12/docker_cerebro/cerebro_nodes.png' style='width: 500px;'></a>


## 雑記

### Entrykit を使った設定ファイルのテンプレート化

設定ファイルである application.conf は [Entrykit](https://github.com/progrium/entrykit) を利用してテンプレート化してあり、環境変数経由でトップ画面のリストにノードを追加できるようにしてあります。
そのため、ノードを追加するためだけに docker build する必要がなくなり、単に環境変数を書き換えてコンテナを上げ直せば反映されるようになっています。

設定ファイルで Basic 認証をかけたり LDAP の設定も可能なのですが、認証などは必要なら Cerebro の手前に用意するリバースプロキシでやればいいし、あんまりモチベーションがないのでテンプレート化していません。
わたしのモチベーションがないだけで、環境変数経由で設定できれば便利そうなのでパッチはウェルカムです。

### SQLite.....

おそらく v0.6.1 からの機能で、発行した REST API リクエストの履歴を保持する機能が追加されており、履歴を保持するためのストレージとして SQLite が使われています。

これはあんまりだと思ったので、なんとか設定ファイルに MySQL のドライバを設定したりしてみたのですが、 [どうも SQLite きめうちのように見える](https://github.com/lmenezes/cerebro/blob/v0.6.1/app/dao/RestRequest.scala#L6) のであきらめました。

コンテナを落とせば当然履歴も消えてなくなるのですが、これはもうあきらめました。
ボリュームをマウントして Docker ホストに置くことも考えましたが、インスタンスが変われば当然意味をなさないので..

### ベースイメージに java:openjdk-8u111-jre-alpine を使いたかった

見出しのとおりです。

深く追ってないので正確にはわからないですが、Java の SQLite のドライバをロードするときに Apline Linux あるあるの glibc がない問題にぶち当たり、結局 java:openjdk-8u111-jre をベースイメージにしました。

ちなみに v0.5.1 までは SQLite を使っていなかったので、java:openjdk-8u111-jre-alpine をベースイメージにできました。
