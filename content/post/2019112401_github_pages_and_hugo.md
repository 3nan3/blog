---
title: "GitHub Pages と Hugo でブログを書く"
date: 2019-11-24T19:28:29+09:00  
tags: ["Github Pages", "Hugo"]
draft: false
---

# はじめに

今までブログとか持ってなかったので何か欲しかったんですよね。  
ブログサービスとか色々あるんですけど、GitHubに静的なページおいて公開できる機能あったことを思い出して使ってみることにしました。

今回は、そのビルドログということで記録を残しておきます。  
ブログなので文量多めです。

### GitHub Pages とは

簡単に言うと、GitHubに `<username>.github.io` という名前のレポジトリを作ると、Webページとして公開してくれるサービスです。
構成とかデプロイとか何にも考えなくて良いので楽ちん。  
https://help.github.com/ja/github/working-with-github-pages

商用利用禁止であったり、アクセス数の制限があったり、その辺りの細かいことは公式の利用規約などをご確認ください。  

### Hugo とは

markdownから静的ページを生成するツールです。   
https://gohugo.io/
 
Hugoを選んだ理由は、markdownで書けるというのが一番の理由で、他はテーマが種類が豊富にありそうだったとか、Go製なので軽量＆速そうとか、そういう理由です。  

# 構成

要は公開したいファイルを `<username>.github.io` レポジトリに入れていけばいいんですが、せっかく作るので管理しやすい形にしたいです。  
Hugo自体のファイルや生成元のmarkdownを管理するレポジトリと、公開用のレポジトリ `<username>.github.io` と、２つ用意するのはどうかな、と考えていたところ同じような構成を紹介しているサイトを発見。  
https://www.ted027.com/post/hugo/#hugo

（本当にこのサイトのまま作ったので、紹介したら終わりになってしまうのですが）  
以下のような構成で構築を進めます。

```
─ blog (git repogitory)
　　├── public (git submodule <username.github.io>)
　　├── content # ブログ記事
　　│　　 └── post
　　│　　　　　├── 2019112401_github_pages_and_hugo.md
　　│　　　　　├── 2019112501_other_post.md
　　│　　　　　:
　　├── themas      # 選択したテーマ
　　├── archetypes  # 新しく記事を作ったときのテンプレート置き場
　　└── config.toml # 設定ファイル
```

# 導入

動作環境

```
Ubuntu 18.04 LTS
```

### Hugo のセットアップ

aptとかでインストールしてもいいんですが、最新版をいつでも使えるようにここではGitHubから取得する方法にします。

まず事前準備として、Go言語のインストールをしておきましょう。  
バージョンは、導入時に最新のstableをインストールしています。  
https://golang.org/doc/install#install
```
# Goのインストール
wget https://dl.google.com/go/go1.13.4.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.13.4.linux-amd64.tar.gz
export PATH=$PATH:/usr/local/go/bin:~/go/bin # あとは各自bash.profileなどを設定
```

インストールが成功したか確認します
```
> go version
go version go1.13.4 linux/amd64
```

次に、Hugoをインストールします。  
GitHubから取得するので `git clone` 使ってもいいのですが、ここは `go get` を使って配置とビルドまでやってもらいます。  
https://github.com/gohugoio/hugo#fetch-from-github
```
# Hugoのインストール
go get github.com/gohugoio/hugo.git
```

こちらも、インストールが成功したか確認しておきます
```
> hugo version
Hugo Static Site Generator v0.60.0-DEV linux/amd64 BuildDate: unknown
```

### プロジェクトの作成

まずは、Hugoで新しくサイトを作ります。
```
hugo new site blog
cd blog
```

次に、テーマを設定します。  
テーマはこちらに一覧があるので、好きなものを選択します。  
https://themes.gohugo.io/

今回は、 `minimal` というテーマを選択しました。

![screenshot-minimal](https://user-images.githubusercontent.com/11911806/69903213-16878580-13da-11ea-99a9-820c7c4bcbab.png)

```
cd themes
git clone https://github.com/minimal.git
cd ../
```

あとは、必要に応じて`config.toml`を更新します。
テーマごとに設定項目が違ったりするので、選択したテーマの公式ページを参考に編集してください。
```
> cat config.toml
baseURL = "http://3nan3.github.io/"
languageCode = "ja"
title = "ブログタイトル（仮）"
theme = "minimal"
disqusShortname = ""
googleAnalytics = ""
...
```

### 記事の作成

記事を追加するときは、
```
hugo new post/2019112401_title.md
```

これで新規の記事が作成されます。  
```
> cat content/post/2019112401_title.md
---
title: "2019112401_title"
date: 2019-11-24
tags: []
draft: true
---
```

本当はファイル名とタイトルは違うものを設定したいとか、整理してディレクトリ構造とかきれいにしたいとかやりたいですが、一旦ここは初期設定のまま行きます。  
おそらく、`archetypes/default.md`でフォーマットを変更できるので、いずれチャレンジします。

そして、このファイルにmarkdown形式で内容を書き込んで記事を作っていきます。

あとは、ローカルサーバで動作を確認します。
```
hugo server --buildDrafts --watch # hugo server -wD でもOK
```

`--buildDrafts` は、draft: trueとなっている記事もビルドして一覧に表示されるようになるオプションです。書きかけの記事をローカルで確認するための機能でしょうか。  
`--watch`は、特定のディレクトリを監視して変更があれば即ビルドするためのオプションです。

記事が完成したら、`draft: false`に変更してビルドします。
```
hugo
```

サブコマンド無しでビルドなんですね。あんまり馴染みがないので少し驚きました。  
これで`public`フォルダが出来上がったと思います。

### GitHub Pages のセットアップ

まずは、レポジトリを作成しましょう

- `<username>.github.io`
- `blog`

`<username>`は、ご自身のGitHubアカウント名を入れてください。  
`blog`は名前なんでもいいですが、ここではわかりやすく`blog`としています。

まずはgitレポジトリの設定をしましょう。
```
git init
git remote add origin git@github.com:<username>/blog.git
```

そして、publicフォルダを`<username>.github.io`というレポジトリのサブモジュールとして登録します。
```
cd public
git init
git remote add origin git@github.com:<username>/<username>.github.io.git
git push origin master
cd ../
git submodule add git@github.com:<usernane>/<username>.github.io.git public
```

残りをaddして、GitHubにpushします。
```
git add -A
git commit -m "Initial commit"
git push origin master 
```

あとは、<username>.github.io で見られるようになっているはずなので確認してみましょう。  
pushして直後に反映されるわけではないので、表示されない場合は少し待ちます。


### 記事の追加

１つ記事が出来てしまえばあとは簡単で、

- `hugo new post/2019012301_title.md` で記事を新規作成
- 記事を編集してローカルで確認
- 完成したら`hugo`でビルドしたあとに GitHub へ push

```
hugo
cd public
git checkout master
git add -A
git commit -m "Add post 2019123401_title"
git push origin master
cd ../
git add -p # 必要なものだけコミットしてください
git push origin master
```

これで更新が出来ます。

# 感想

コンソール上から簡単にブログが書けるようになりそうで個人的には大変満足です。ただ、記事の更新で思ったよりコマンド叩かなければいけなさそうなのでその辺はもっと改良したいところ（GitHub Pagesはmasterブランチを公開するんですが、git submoduleはHash管理なので同期が難しいですね）。  
多分、更新頻度は数カ月に1回とかになりそうですが、何かまたネタが思いついたら書いてみようと思います。

