# 開発環境構築手順

- 経緯 : 機械を新しくした時にセットアップが面倒
- 目的 : 新しい機械ですぐに開発できるようにする
- 状況 : mac を windows にすることも想定する
- 決定 : 開発は CoreOS を使用して Docker で行う


###### Table of Contents

- [構成](#user-content-構成)
- [手順](#user-content-手順)


### 構成

- mac → ssh → CoreOS
- CoreOS 上では shell 入りのコンテナで作業
- 実行環境は別なコンテナで動かす


### 手順

1. /home/core/volumes をバックアップ
1. バックアップを新しい機械にリストア
1. connect スクリプト設置 `git clone https://github.com/getto-systems/labo-connect.git`


```
$ ./labo-connect/bin/connect <USER>
```
