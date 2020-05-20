---
title: "rubocop で ディレクトリごとにルールを変える"
date: 2020-05-21
tags: ["rubocop", "ruby"]
draft: false
---

rails と rspec でアプリケーション書いてる時に度々困ることがあるんですけど、

例えば、

```
# app/models/user.rb

def hoge
  fuga = cars.find_by(...)
           .method_a(...)
           .method_b(...)
end
```

```
# spec/models/user_spec.rb

it do
  ecpext { user.hoge }
    .to change { user.reload.name }.from("aaa").to("bbb")
    .and change { ... }
    .and change { ... }
end
```

みたいな感じで書くんですよね。
で、これを rubocop さんに見てもらうと怒られちゃうんですよ。

種類としては `Layout/MultilineMethodCallIndentation` になるんですけど、  
app は `indented_relative_to_receiver` に従っていて、  
spec は `indented` に従っていることになります。

こんな感じで、ディレクトリによって rubocop のルール変えたいことがあったので、対応したときのメモになります。

# 対応

こうする

```
sample-app
  ├ app
  ├ spec
  │  ├ .rubocop.yml
  │  └ ...
  ├ ...
  └ .rubocop.yml
```

```
# sample-app/.rubocop.yml

# アプリ全体での設定はここに書く
Layout/MultilineMethodCallIndentation:
  EnforcedStyle: indented_relative_to_receiver
```

```
# sample-app/spec/.rubocop.yml

# アプリ全体の設定を読み込んで
inherit_from: ../.rubocop.yml

# 個別の設定で上書きする
Layout/MultilineMethodCallIndentation:
  EnforcedStyle: indented
```

コアコミッターが言ってるんだから間違いない
https://twitter.com/9sako6/status/1260756420036681729
