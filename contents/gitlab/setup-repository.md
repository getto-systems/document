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

- release ブランチにコミットしたら tag を作成
- GitLab → BitBucket (バックアップ)


### 手順

1. GitLab リポジトリ作成
1. 個人用 fork 作成
1. BitBucket (GitHub) にリポジトリ作成、 push
1. release ブランチ作成、protect
1. `GITLAB_ACCESS_TOKEN`、`BITBUCKET_ACCESS_TOKEN` 登録
1. gitlab-ci テンプレート追加

```
git clone https://gist.github.com/b3db4b58731bed1820c3d88534bac62a.git
```


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
