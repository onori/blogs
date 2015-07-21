+++
author = "onori"
comments = false
date = "2015-07-20T20:05:40+09:00"
draft = false
image = ""
menu = ""
share = true
slug = "first-hugo"
tags = ["hugo"]
title = "ブログをhugoに移行してみた"

+++

記事の移行は終わってないけど、ブログをhugoへ移行してみた。  
hugoのインストールは[ココ](http://qiita.com/syui/items/869538099551f24acbbf)の記事なんかを見ながらやるとサクサクと入る。

スタティックサイトジェネレーターは過去に使ってたけどhugoはGoで出来てるし、速くていい感じ。

## 使用しているテーマ

 * https://github.com/vjeantet/hugo-theme-casper

主にfont-familyなどをちょいちょい編集して利用中。
もうちょっとテーマが増えればいろいろ使い道がありそう。

## テーマのCSSファイル変更

既存のthemeディレクトリにある`static/css`をルートディレクトリにあるstatic/cssにコピーして利用するのが正っぽい。

## github pagesにデプロイ

http://hori-ryota.com/blog/create-blog-with-hugo-and-circleci/

hori blogさんの記事を参考に設定。
circle CIやwerckerでもデプロイ試してみた。  
初めて使ったけど便利だ！  
社内ではjenkins使ってるけど職人化がすすむ。  

CIツールというよりgithub pagesの仕組みがイマイチ理解しておらずそこに若干苦労。  
まあ終わってしまえばなんてことはない。
