+++
author = ""
comments = true
date = "2015-08-18T22:09:24+09:00"
draft = false
image = ""
menu = ""
share = true
slug = "laravel-keenio-heroku"
tags = ["laravel", "Keen.io", "heroku"]
title = "laravel+Keen.io+herokuで簡易解析"

+++

とあるシステムで簡単なアクセス解析をする必要があったため、  
せっかくなので存在だけは知っていたKeen.ioを使ってみた。

## Keen.io

![keen_io_top.png](/images/keen_io_top.png)

[Keen.io](https://keen.io/) は簡単にいうと解析サービス。  
Googleアナリティクスとかmixpanelとかに近いっちゃー近いけど、  
Keen.ioは基本的に **自分で** 解析したい情報をJSONで投げる。

上述アクセスログに関しては、herokuのpostgreSQLなんかにぶっこんでるともったいない。
そんな時はログ基盤としてKeen.ioを使いましょ、って話。  
とりあえずJSONでぶん投げておけば、「後で◯◯のデータを解析したい！」なんていう時にも役に立つ。  
グラフの出力もしてくれるし。  

そんなわけでHeroku + LaravelでのKeen.ioの使い方。


## Heroku

HerokuのアドオンからKeen.ioをインストール。

![keen_io_heroku.png](/images/keen_io_heroku.png)

Developerプランは月50,000回のイベント送信まで無料。  
ホント助かりまっせ！送信回数が増えたらぜひ有料プランを！  

コンソールからインストール

```
$ heroku addons:create keen:developer
```

Herokuの管理画面から、Setting > Config Variables > Edit

![keen_heroku_env.png](/images/keen_heroku_env.png)

のような感じで環境設定値が入る。

## Laravel


### .envファイルの編集

上記の環境設定は、.envファイルに同じように記述する。

```.env
KEEN_API_URL=####
KEEN_MASTER_KEY=####
.
.
.
```

### composerパッケージに追加

ありがたいことにLaravelからKeen.ioを利用が可能なcomposerパッケージを作ってくれている人がいる。  
せっかくなので利用させていただきましょう。

 * https://github.com/garethtdavies/keen-io-laravel

```
$ composer require wensleydale/keen-io-laravel:1.*
$ composer update
```

更新が終了したら、パッケージ追加完了。

### サービスプロバイダーとエイリアスの追加

config/app.php を開き、

```config/app.php
'providers' => array(
    ...
    Wensleydale\KeenLaravel\KeenLaravelServiceProvider::class
)

'aliases' => array(
    ...
    'Keen'      => Wensleydale\KeenLaravel\KeenFacade::class,
)
```

を追加したのち

```
$ php artisan vendor:publish
```

を実行すると、同じくconfigディレクトリ内にkeen.phpというファイルがジェネレイトされる。


### keen.php

keen.phpには先ほど.envファイルに記述した設定値を記述。

```keen.php
<?php

return array(
...
    'projectId' => env('KEEN_PROJECT_ID'),

...
    'masterKey' => env('KEEN_MASTER_KEY'),
    'writeKey' => env('KEEN_WRITE_KEY'),
    'readKey' => env('KEEN_READ_KEY'),

);
```

このようにenv関数から設定値を読み込む。


### Keen.ioに値を投げる

さて、ここまで準備が出来たら、いよいよKeen.ioに値をぶん投げる時が来る。  
今回の要件はアクセス解析、というより「誰がいつログインしたか？」の情報だけで良かったので、ログイン部分にこのKeenの処理を挟む。

```app/Controller/AuthController.php
<?php
namespace App\Http\Controllers\Auth;

use Auth;
use Keen;
use App\User;
use Illuminate\Http\Request;
use App\Http\Controllers\Controller;
use App\Http\Controllers\Auth\Redirect;

class AuthController extends Controller
{
    public function postLogin (Request $request)
    {
        if (Auth::attempt(['email' => $request->email, 'password' => $request->password])) {

            // KEEN.IOにイベントを送信
            $user = Auth::user();
            $event = ['login' => ['name' => $user->name]];
            Keen::addEvent('login', $event);

            return redirect()->intended('dashboard');
        } else {
            return redirect()->intended('/');
        }
    }
}
```

`$event = ['login' => ['name' => $user->name]]` 部分でKeen.ioへ送信するデータを作成し、
`Keen::addEvent()`で送信、これだけ。とても簡単。投げられた値は、herokuからSSOでKeen.ioにアクセスし、

![Keenio_json.png](/images/Keenio_json.png)

で確認できる。Keenはデータを取得した際のtimestampもよしなにつけてくれるためありがたい。

### Keen.ioからデータを取得

Keen.ioは様々な解析が可能だが、全部紹介していたらとっても長くなりそうなので割愛。  
詳しくは https://keen.io/docs/ を確認しながら、いろいろ弄ってみてください。  

本アプリケーションでは1日間でのログイン集計が欲しかったので、こんな取得イベントを走らせてました。

```
// ログイン履歴を取得
$event = Keen::count('login',
['target_property' => 'purchase.item',
    'group_by' => 'login.name',
    'interval' => 'daily',
    'timezone' => 'UTC',
    'timeframe' => 'this_1_days'
]);
```

これを1日1回、heroku schedulerで走らせて、その日のログインを確認するフロー。  


## まとめ

今回のようにちょっとしたことから、自分自身で心ゆくまで解析したいときまで、Keen.ioの汎用性はとても高い。  
反面、googleアナリティクスのように、 **JS一行埋め込んで、はい終わり** という代物ではないため、
利用できるのは開発者に限るのかも。  

ただ、様々な要求が飛び交う中で、こういうことがパッとできるのはやっぱりHerokuの良いところだし、
豊富なアドオンのおかげでもある、と感じる次第です。
