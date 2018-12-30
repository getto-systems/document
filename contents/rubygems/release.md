# rubygems リリース手順

- 経緯 : rubygem の開発を行うことになった
- 目的 : ソースコードのホストサービスへの push をトリガーにして rubygems に publish したい
- 状況 : GitHub + Travis CI で無料でセットアップが可能
- 決定 : GitHub に Travis CI のトリガーを仕込んで rubygems に publish する


###### Table of Contents

- [手順](#user-content-手順)


### 手順

1. GitHub リポジトリを作成
1. そのリポジトリを clone
1. clone したディレクトリで `travis login --github-token=<GITHUB TOKEN>` して `travis encrypt`
1. 表示された secret を travis.yml に追記

リポジトリ名と gem 名が異なる場合、 travis.yml に `gem:` を記述すること
