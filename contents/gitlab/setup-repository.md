# リポジトリ構築手順

- 経緯 : BitBucket で pipeline しようとしたら 50分制限だった
- 目的 : プライベートリポジトリの CI ができる環境を整える
- 状況 : ユーザー数が少ない場合、GitLab の無料プランが使える
- 決定 : GitLab でホストして CI を整える

###### Table of Contents

- [手順](#user-content-手順)


### 手順

- GitLab リポジトリ作成
- 個人用 fork 作成
- BitBucket (GitHub) にリポジトリ作成
- release ブランチ作成、protect
- GITLAB_ACCESS_TOKEN、BITBUCKET_ACCESS_TOKEN 登録
- gitlab-ci テンプレート追加

[TOP](#user-content-リポジトリ構築手順)