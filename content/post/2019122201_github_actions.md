---
title: "GitHub Actions で別のリポジトリに git push する"
date: 2019-12-22
tags: ["GitHub Actions", "GitHub", "CI"]
draft: false
---

# はじめに

GitHubに新しく `GitHub Actions` というものがリリースされました。いつもはCircle CIを使っていたのですが、試しに使ってみようということで軽く触ってみました。  
このブログは先日作ったばかりなのですが、その時に「ブログ管理用レポジトリにmargeされたら、静的ページをビルドして、ブログ公開用レポジトリにpushする」という機能をあとでやる([※](/post/2019112401_github_pages_and_hugo/#感想))って言っていたので、これをサンプルとして作ってみようと思います。

# tl;dr

- push先のレポジトリで、Settingsから `Deploy keys` （公開鍵）を登録
- GitHub Actionsを設定するレポジトリで、Settingsの `Secrets` にDeploy keysに対応するの秘密鍵を登録
- GitHub Actions用のymlをいい感じに書いてpushする

# やってみる

CI的なところはだいたい何ができるか想像つくんですけど、レポジトリ間の認証どうするのかと思って調べてみましたが、特段用意されてるわけではなさそうですね、知ってました。
自分自身のレポジトリに対する認証用に `GITHUB_TOKEN` というものが勝手に追加される程度でした。
なので、いつも通りの認証方式を取ります。

先に、レポジトリ構成を書いておきますと、 `blog` が `3nan3.github.io` をpublicというフォルダ名でsubmoduleとして持っています。
なので、構図としては blog レポジトリが 3nan3.github.io レポジトリにpushする形になります。
```
─ blog (git repogitory blog)
　　├── public (git submodule 3nan3.github.io)
　　└── ... # 他いろんなファイル
```

### Deploy keys

GitHub Actionsを設定する前に、先に認証周りの設定を済ませておきます。  
pushされる側のレポジトリで作業を進めます。

GitHubのSettingsにある、`Deploy keys`を使います。
これは、ユーザに紐付かない、設定したレポジトリにのみアクセスできる鍵を作成できる機能で、読み込み専用・読み書き可能などの権限設定をすることも可能です。
今回のように、人ではないサービスなどが読み込んだり書き込んだりする時に適切な権限設定ができるのでとても便利です。
サービスの規模などに合わせてどのように管理するべきか、みたいなものが公式のページに書かれているので、悩んだことがある方にはおすすめです。

https://developer.github.com/v3/guides/managing-deploy-keys/#deploy-keys

で、鍵の生成はコマンドで、

```
> ssh-keygen -t ed25519
Generating public/private ed25519 key pair.
Enter file in which to save the key (/home/username/.ssh/id_ed25519): ./id_ed25519
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in ./id_ed25519.
Your public key has been saved in ./id_ed25519.pub.
The key fingerprint is:
...
```

そして、出来上がった鍵のうち公開鍵（今回の場合はid_ed25519.pub）の中身をDeploy keysに登録します。

Settings -> Deploy keys -> Add deploy key の順に遷移して、管理できるようにいい感じのTitleと鍵の中身を入力します。
あとは、今回はpushしたいので`Allow write access`にチェックを入れて、Add keyを押せば完了。

![Screenshot from 2019-12-14 18-57-40-2](https://user-images.githubusercontent.com/11911806/71270587-4c92a680-2395-11ea-803b-16a3c2b8a023.png)

![Screenshot from 2019-12-14 18-56-43](https://user-images.githubusercontent.com/11911806/71270652-7946be00-2395-11ea-864c-e462d1741748.png)


これで、pushされる側の設定は終わりです。

### Secrets

次に、GitHub Actions側の設定です。

GitHub Actionsのリリースとともに、Settingsに `Secrets` という項目が増えました。
何かと言うとそのまんま、秘匿情報を安全に保持して、GitHub Actions内から環境変数として使えるという機能です。
ここに、先程Deploy keysに登録した鍵の秘密鍵の方を入れておけば、GitHub Actionsで認証を通すことが出来ます。
注意点として、値にJsonとかは登録してはいけないとのこと（ログに残ってしまう可能性が出てくるらしい）。
他、細かく注意点などが書かれているので、ドキュメントには目を通しておきましょう。

https://help.github.com/ja/actions/automating-your-workflow-with-github-actions/creating-and-using-encrypted-secrets

で、先程コマンドで生成した鍵を登録すれば良いので、秘密鍵（今回の場合はid_ed25519）の中身をコピペします。
Settings -> Secrets -> Add a secret の順に遷移して、環境変数名と鍵の中身を入力して Add secret を押します。

![Screenshot from 2019-12-14 19-15-48](https://user-images.githubusercontent.com/11911806/71270664-806dcc00-2395-11ea-95d7-f74bc3cc8580.png)

### GitHub Actions

最後に、GitHub Actionsを設定します。
公式に詳細のドキュメントがあるので、これ見ながら書いていけば良いと思います。

https://help.github.com/ja/actions/automating-your-workflow-with-github-actions

今回やりたいのは、

- 自リポジトリのmasterブランチがpushされたら発火
- 自リポジトリのmasterブランチで静的サイトをビルド
- ビルドされたものを別リポジトリのmasterブランチにcommit
- 別リポジトリのmasterブランチをpush

こんな流れ。
で、出来上がったものがこちら。

```yml
# .github/workflows/build_and_deploy.yml
name: Build and Deploy

on:
  push:
    branches: master

jobs:
  build-and-deploy:
    name: Build static pages and deploy
    runs-on: ubuntu-latest
    steps:
      - name: Setup Hugo
        run: |
          wget https://github.com/gohugoio/hugo/releases/download/v0.61.0/hugo_0.61.0_Linux-64bit.deb
          sudo dpkg -i hugo_0.61.0_Linux-64bit.deb
      
      - name: Checkout
        uses: actions/checkout@v2
      - name: Checkout submodules
        run: |
          auth_header="$(git config --local --get http.https://github.com/.extraheader)"
          git submodule sync --recursive
          git -c "http.extraheader=$auth_header" -c protocol.version=2 submodule update --init --force --recursive --depth=1

      - name: Setup git repogitories
        env:
          GITHUB_IO_REPO_DEPLOY_KEY: ${{ secrets.GITHUB_IO_REPO_DEPLOY_KEY }}
        run: |
          echo "$GITHUB_IO_REPO_DEPLOY_KEY" > ~/deploy_key.pem
          chmod 600 ~/deploy_key.pem
          git config --global user.email "3nan3@github.io"
          git config --global user.name "3nan3"
          cd public
          git config remote.origin.url "git@github.com:3nan3/3nan3.github.io.git"
          git checkout master

      - name: Build pages
        run: hugo

      - name: Deploy pages
        env:
          GIT_SSH_COMMAND: ssh -i ~/deploy_key.pem -o StrictHostKeyChecking=no -F /dev/null
        run: |
          cd public
          git add -A
          if ! git diff --cached --quiet; then
            git commit -m "Deploy $GITHUB_SHA by GitHub Actions"
            git pull origin master
            git push origin master
          fi

      - name: Update dependency of submodule
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          git add public
          if ! git diff --cached --quiet; then
            origin=https://${GITHUB_ACTOR}:${GITHUB_TOKEN}@github.com/3nan3/blog.git
            git commit -m "Update dependency of submodule"
            git pull $origin master
            git push $origin master
          fi
# refs
# - actions/checkout@v2 and submodule update
#   https://github.com/actions/checkout/tree/c85684db76ba6ef08713c10cf4befe3318887415#checkout-submodules
```

想定より記述量が多くなってしまいましたが、使い方はわかったので良しとします。
それと、sshとかgit周りで思ったより躓いてしまって、結構大変でした。。。
GitHub Actionsと全然関係ない話なので、やる気があったらまた後日別記事にでもまとめようかと思います。

実行結果はと、Actionsタブで確認することが出来ます

![Screenshot from 2019-12-21 01-47-20](https://user-images.githubusercontent.com/11911806/71270588-4d2b3d00-2395-11ea-96b7-531f03d66d47.png)

![Screenshot from 2019-12-21 01-49-41](https://user-images.githubusercontent.com/11911806/71270589-4d2b3d00-2395-11ea-8eb4-935d3132e8a1.png)

# さいごに

Circle CIと違って簡単に結果画面に遷移できるのはやっぱり楽でいいですね。
あと、無料でmacOS使えるのもすごい（無料だよね？あんまり調べてないけど）。
AWSとかと連携するためのactionsも用意されているみたいなので、またの機会に試してみようかと思います。

