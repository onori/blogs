+++
author = ""
comments = true
date = "2015-07-28T19:47:10+09:00"
draft = false
image = ""
menu = ""
share = true
slug = "post-title"
tags = ["laravel", "heroku"]
title = "laravel5.1でschedule + メール送信 + Heroku"

+++

## Schedule

Laravel5.1ではスケジューラによるcron処理が可能。  
利用方法は以下。  

### Laravel側の設定

`Console/Kernel.php` 内

```
protected function schedule(Schedule $schedule)
{
    $schedule->command('inspire')
             ->hourly();

    $schedule->command('user:change-password 5365576')
            ->everyFiveMinutes();
}
```

こんな感じでコマンドを登録しておく。  
関数群は以下↓  
http://readouble.com/laravel/5/1/ja/scheduling.html

### crontabの設定

```
$ crontab -e
```

以下のコマンドをファイルに記述

```
* * * * * php /path/to/artisan schedule:run 1>> /dev/null 2>&1
```

要は「毎秒cron走らせて、phpの関数で実行時間を管理」するということらしい。  

## メール

Laravel内部では、[SwiftMailer](https://github.com/swiftmailer/swiftmailer) というPHPライブラリを利用している。  
デフォルトでサポートされているドライバーは、

 * smtp
 * mail
 * sendmail
 * mailgun
 * mandrill
 * ses(amazon simple email service)
 * log

というラインナップ。「log」は __開発環境で送信結果のみをログ・ファイルに出力__ など開発環境向き。  
とりわけメールのテストは面倒くさいのでこういった気遣いはありがたい。
が、せっかくなので、テストはLaravel推奨の[mailtrap.io](https://mailtrap.io/) を利用する。  
productionではherokuを利用することを想定し、無料枠のあるmailgun（400/day）を使う。  

### 開発環境

とりま `guzzle`をcomposerに。  

```
"require": {
    "guzzlehttp/guzzle": "~5.3|~6.0"
}
```

で、`$ composer update`  

https://mailtrap.io/ →「Sing up」から、アカウントを作成。  
OAuthでサクッとやっちまうのがラク。  

![InboxesMailtrap.png](/images/InboxesMailtrap.png)

ログインページに遷移したら、「Demo inbox」をクリック  

![Demo_inboxMailtrap.png](/images/Demo_inboxMailtrap.png)

モザイクかかってるUsernameとPasswordを`.env` にいれればおｋ．


```.env
MAIL_DRIVER=smtp
MAIL_HOST=mailtrap.io
MAIL_PORT=2525
MAIL_USERNAME=[mailtrap_smtp_username]
MAIL_PASSWORD=[mailtrap_smtp_password]
MAIL_ENCRYPTION=null
```

上記を設定した状態で、http://readouble.com/laravel/5/1/ja/mail.html の「メール送信」項目を参考にコードを書けばまあ動くだろう。

### mailgunの設定

さてお次はmailgunの設定。

```
$ heroku addons:create mailgun:starter -a APP_NAME
```

でmailgunの無料プランを追加。

次に、`config/services.php` 以下のmailgunにdomainとsecretを記入。  
若干わかりづらかったが、

・ドメイン
![Domains_Mailgun.png](/images/Domains_Mailgun.png)


・secret
![mailgun_apikey.png](/images/mailgun_apikey.png)

である。（モザイクだらけ）  

あとは、`mail.php` をちょちょいと修正（ユーザー名とかパスワードとか）を入れればおｋ.  
herokuの環境変数にセットアップされているので、productionからはそっちから呼ばせても良い。  


## herokuでの運用

さて、schedule + メールという処理が開発環境で動くところまでは説明したが、  
ここまでやっといて何だけど、herokuからはLaravelのscheduleは __使えない。__  
cronの設定が出来ないから。  
ただし、そんなときのために __heroku scheduler__ がある。

## heroku schedulerのインストール

```
$ heroku addons:create scheduler:standard -a APP_NAME
```

でheroku schedulerをインストール。

## schedulerの設定

![Heroku_Scheduler.png](/images/Heroku_Scheduler.png)

`$` マーク以後に、実行させたいコマンドを入力し、 FREQUENCYから実行タイミングを指定すれば良い。最短は __10分__ である。
