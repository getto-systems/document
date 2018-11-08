# リポジトリ構築手順

- 経緯 : BitBucket で pipeline しようとしたら 50分制限だった
- 目的 : プライベートリポジトリの CI ができる環境を整える
- 状況 : ユーザー数が少ない場合、GitLab の無料プランが使える
- 決定 : GitLab でホストして CI を整える

###### Table of Contents

- [前提](#user-content-前提)
- [構成](#user-content-構成)
- [手順](#user-content-手順)


### 前提

- direnv で環境変数を設定
- git-release-request でバージョンアップコミットを作成


### 構成

- GitLab → BitBucket (バックアップ)
- release ブランチにコミットしたらバックアップホストに push


### 手順

- GitLab リポジトリ作成
- 個人用 fork 作成
- BitBucket (GitHub) にリポジトリ作成
- release ブランチ作成、protect
- GITLAB_ACCESS_TOKEN、BITBUCKET_ACCESS_TOKEN 登録
- gitlab-ci テンプレート追加


#### .envrc

```bash
export APP_ROOT=$(pwd)

export GIT_RELEASE_REQUEST_TARGET=release
```

#### .gitlab-ci.yml

```yaml
image: buildpack-deps:stretch

release:
  only:
    - release@<project>
  script:
    - ./bin/push_tags.sh
```

#### bin/push_tags.sh

```bash
#!/bin/bash

git remote add super https://gett-systems:$GITLAB_ACCESS_TOKEN@gitlab.com/<project>.git
git remote add github https://getto-systems:$GITHUB_ACCESS_TOKEN@github.com/getto-systems/<path>.git
git tag $(cat .release-version)
git push super HEAD:master --tags
git push github HEAD:master --tags
```

[TOP](#user-content-リポジトリ構築手順)
