+++
author = "onori"
comments = false
date = "2015-07-21T13:47:07+09:00"
draft = true
image = ""
menu = ""
share = true
slug = "laravel51-heroku"
tags = ["laravel", "heroku"]
title = "laravel5.1 on herokuセットアップ"

+++

HerokuでLaravel5.1を動かすまでのセットアップ手順。  
ゴールはherokuへのデプロイですが単純なデプロイだけでなく、今後も継続的に開発が可能な段階までの設定までを行います。所要時間は15分程度。

##### 大まかな流れ
 1. laravelのインストール
 1. ローカル開発環境構築
	 1. Homesteadのインストール
	 3. envファイルの設定
	 4. DBの設定
	 5. Redisの設定
 1. githubにpush
 1. herokuに公開


## Laravel 5.1
Laravel 5.1はLTSです。バグの修正は2年間、セキュリティーは3年間の対応があります。


## インストール

composer globalにlaravelインストーラを入れちゃいましょう。

```
$ composer global require "laravel/installer=~1.1"
```

これで`laravel`コマンドが利用できるようになります。

```
$ laravel new myapps
```

これで現在のディレクトリに`myapps`のプロジェクトが作成されます。
`myapps`はアプリケーション名なので、自分の付けたい名前に変更してください。

## 設定

### vagrantの設定

[homesteadでlaravelラクラク開発環境](https://gstaff.docbase.io/posts/29459)の記事を参考にセットアップしてください。

## env / データベースの設定

`.env`ファイルに環境に必要な設定値を記述していきます。
Laravelは __`.env`ファイルが有る場合はその設定値を優先的に見て、なければ環境変数を見るようにします。__
ここでは、

 * ローカル（開発環境、Vagrant）-> .env
 * プロダクション（本番環境、heroku） -> 環境変数

として動作させるようにします。

### .env.exampleファイルの編集

今回はherokuで公開することを目的としているので、指定形式を __heroku側に合わせる__ ように記述していきます。
.env.example内のDBの設定を以下のようにコメントアウトしましょう。

```.env.example

# ↓をコメントアウト

# DB_HOST=localhost
# DB_DATABASE=homestead
# DB_USERNAME=homestead
# DB_PASSWORD=secret
```

代わりに次のURLを記述します。

```.env.example

DATABASE_URL=postgres://homestead:secret@localhost:5432/homestead # 追加

# DB_HOST=localhost
# DB_DATABASE=homestead
# DB_USERNAME=homestead
# DB_PASSWORD=secret
```


ここまで出来たら、`.env.example`をコピーして、`.env`ファイルを作成しましょう。

```
$ cp .env.example .env
```

これで`.env`ファイルの作成は完了です。

#### 注意
上記手順で行った場合、既に.gitignore環境に`.env`が記述されていますが、それ以外の方法でプロジェクトを作成した場合は、手動にて.gitignoreに登録する必要があります。前述のとおり、プロダクション環境で.envファイルが存在すると、Laravelは優先的にそちらの設定値を見に行ってしまうためです。

### config/database.phpの編集

次にconfig/database.phpを編集します。本番環境（=heroku）側で見る設定ですね。
herokuではDBなどのアドオン設定値は全て環境変数に書き込まれ、その形式はすべて`URL`になっています。
先ほどの.envファイルのコメントアウトや追加の記述は、開発環境（=Homestead）側の設定をURL形式に変更しました。

database.phpでもURL Schemaに対応させるため、以下の行を追加しましょう。

```config/database.php
$pg_url = parse_url(env('DATABASE_URL'));
```

次にherokuではpostgresを利用するので、以下の設定値を書き換えましょう

```config/database.php
    'default' => env('DB_CONNECTION', 'pgsql'),
```

更に、database.phpのpgsqlの設定を書き換えましょう。

```config/database.php
        'pgsql' => [
            'driver'   => 'pgsql',
			'host'     => $pg_url['host'],
			'database' => str_replace('/', '', $pg_url['path']),
			'username' => $pg_url['user'],
			'password' => $pg_url['pass'],
			'port'     => $pg_url['port'],
			'charset'  => 'utf8',
			'prefix'   => '',
			'schema'   => 'public',
        ],
```

これでDBの設定は終了です。

### AppKeyの作成

下記コマンドを入力し、AppKeyの生成を行います。
`.env`ファイルが存在すれば、下記コマンドを利用することで自動的に.envファイルに記述されます。

```
$ php artisan key:generate
```


### DBコネクションのテスト

DBが正常につなげているかどうかを確認するために、`artisan tinker` を使ってDB接続の簡単なテストを行います。
まずは、`homestead ssh`を使ってゲストOSにログインしましょう。

```
$ homestead ssh
```

ゲストOSのディレクトリ構造やフレームワークファイルがどこに置かれるかなどはhomestead.yamlに依存しますが、ドキュメント通りに進めていれば、おそらくホームディレクトリ直下に`Code`ディレクトリがあり、その中にプロジェクトが存在するかと思います。プロジェクトルートに移動した後、

```
$ php artisan migrate
```
とすると、マイグレーションが実行されます、マイグレーションについては次の機会にでも説明するとしましょう。

では、以下のコマンドを入力してください。

```
$ php artisan tinker
Psy Shell v0.4.4 (PHP 5.6.10-1+deb.sury.org~trusty+1 — cli) by Justin Hileman
>>>
```

このような入力待ち状態が確認できたら、

```
App\User::all()->toArray();
```
と入力し`=> []`と帰ってきたら問題なしです。エラーが出た場合は再度ドキュメントを見なおしてみてください。


### Redisの設定

続いてsession / cacheを扱うためにRedisの設定をしましょう。デフォルトでは`file`を利用する設定になっていますが、homesteadを利用しているので、なるべく本番環境に近づける意味も込めて、開発側でもredisを採用することにします。
方法は上記のpgsqlとほぼ一緒ですが、laravelでredisを利用する場合は、composerで`predis`をインストールする必要があります。

#### predisのインストール

```composer.json
"require": {
    "predis/predis": "~1.0",
}
```

```
$ composer update
```


#### .envの設定

```.env
REDIS_URL=redis://homestead:secret@127.0.0.1:6379/0 #追記

# CACHE_DRIVER=file
# SESSION_DRIVER=file
CACHE_DRIVER=redis
SESSION_DRIVER=redis
```

※homestead側ではusername、passなしでもいけますがheroku側に合わせるので上記のように記述してます。

#### config/database.phpの設定
```config/database.php
$redis_url = parse_url(env('REDIS_URL')); #追記

    'redis' => [
        'cluster' => false,
        'default' => [
            'host'     => $redis_url['host'],
            'port'     => $redis_url['port'],
            'database' => str_replace('/', '', $redis_url['path']),
        ],
    ],

```

#### 本番環境でredisを利用する設定

以下のファイルのdriverをいずれもfileから`redis`に変更

```config/session.php
'driver' => env('SESSION_DRIVER', 'redis'),
```


```config/cache.php
'default' => env('CACHE_DRIVER', 'redis'),
```

redisの設定はこれで終了です。


## 立ち上げ〜ログイン機能実装まで

`routes.php` 以下を追加してみましょう。

```routes.php
Route::controller('/', 'Auth\AuthController');
```

## artisan コマンド

artisanコマンドはいわゆる`便利コマンド`です。
artisanコマンドをある程度覚えておくと更に開発効率を上げることが可能です。

```
php artisan make:controller MyController
```

みたいな感じです。
ファイルの作成やコントローラー・モデル、migration関係などはartisanコマンドを使って生成したほうがいいかもしれません。

###### 一覧

```
php artisan list
```

###### ヘルプ

```
php artisan help migrate
```

```
php artisan help make
```

のように記述することで、


## migrationとseederについて

 * migration -> データベースのテーブル構成を記述するファイル
 * seeder -> データベースのテストデータを入力するためのデータの種（シード）

のようにして覚えてください。

### migration

Laravelのプロジェクトルートから `database/migrations`以下にmigrationファイルが存在しています。
初期で記述されている内容からmigrationが実行可能です。

```
php artisan migrate
```

これで既存のDBに`user`テーブルが作成され、userテーブル内に各カラムも作成されました。


### Seeder

Laravelのプロジェクトルートから`database/seeds`以下にseedファイルが存在します。デフォルトであるのは`DatabaseSeeder.php`
ですが、

```
$this->call(UserTableSeeder::class);
```

と記述通り、`UserTableSeeder`が無いため、先ほどの`make`コマンドで新しく作ってしまいましょう。

```
php artisan make:seeder UserTableSeeder
```

これで`seeds`以下に`UserTableSeeder.php`が作成されました。


次に上記で作成した`UserTableSeeder.php`のrunメソッド内に以下を追記しましょう。

```UserTableSeeder.php
<?php

use DB;	// 追記
use Illuminate\Database\Seeder;
use Illuminate\Database\Eloquent\Model; //追記

class UserTableSeeder extends Seeder
{
    /**
     * Run the database seeds.
     *
     * @return void
     */
    public function run()
    {
    	// 追記
        DB::table('users')->insert([
            'name' => str_random(10),
            'email' => str_random(10).'@gmail.com',
            'password' => bcrypt('secret'),
        ]);
    }
}
```

上記で利用する名前空間として、DBとEloquent\Modelを記述し、run内に実際に書き込むテストデータを記述します。
この状態で

```
$ php artisan db:seed
```

を実行すると、DBに上記のテストデータ（シード）が書き込まれます。


### Factoryモデルを使う

ここからは番外編ですが、Factoryモデルを利用したシードの挿入も可能です。
Factory（工場）の名の通り、 __データを大量生産__ する場所とも言えるでしょう。

上記のようにテストデータを挿入する際、例えば50件など大量のデータを挿入したい場合、一つひとつを手で記述していくのは骨が折れます。
こういった場合は`Factory`モデルを利用することで解決可能です。

Factoryモデルは

`database/factories`内にあります。デフォルトでは`ModelFactory.php`ファイルがありますね。
実際にFacotryモデルを利用してみましょう。

先ほどの`UserTableSeeder.php`内のrunメソッド内を一旦コメントアウトか削除し、次の記述を加えてみてください。

```
public function run()
{
    factory('App\User', 50)->create();
}
```

こう記述し、再度 `$ php artisan db:seed` を実行してみましょう。おそらく50件のランダムデータが挿入されたはずです。

ランダムなnameなどは`faker`メソッドにより生成されます。`faker`で利用可能なオプションは[ココ](http://laravel4.winroad.jp/2014/05/29/faker%E3%81%A7%E7%B0%A1%E6%98%93%E3%82%B7%E3%83%BC%E3%83%87%E3%82%A3%E3%83%B3%E3%82%B0/)に記述されています。
first_nameやaddressなどにも対応しているのでとても便利です。
