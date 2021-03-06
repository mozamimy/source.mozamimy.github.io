---
title: Arch Linux の ufw を 0.35-2 にアップデートするときの注意点
date: 2016-05-14
tags: infra, linux
---

## Linux でのファイアウォール

Linux でファイアウォールを実現するためにはいろいろな方法があり、ユーザの好みや実現したい目的によってツールを切り替えることができます。挙動を細かくいじりたくて iptables を直接操作するのが好きな人もいれば、firewalld や ufw といった iptables のフロントエンドを使って、楽をするのが好きな人もいます。ちなみにわたしは Arch Linux で ufw を利用しています。

週 1 くらいのペースでパッケージのアップデートをしているのですが、今日 (2016-05-12) ufw を `0.34-1` から `0.35-2` にアップデートしたところ、外部からのパケットを一切寄せ付けないコミュ障サーバになってしまったので、原因を調べました。

## 結論: .rules ファイルの置き場所が変わった

どうやら、このバージョンで .rules ファイルの置き場所が変わったようです。

### 0.34-1 以前

```
/usr/lib/ufw/user.rules
/usr/lib/ufw/user6.rules
```

### 0.35-2

```
/etc/ufw/user.rules
/etc/ufw/user6.rules
```

パッケージアップデートの際に、pacman によって .rules ファイルが `/usr/lib/ufw/user.rules.pacsave` と `/usr/lib/ufw/user6.rules.pacsave` のようにリネームされているので、これらを新しい置き場所に移しましょう。

`/etc/ufw` 以下に配置されるようになって、あるべき場所に移動したという感じがします。
