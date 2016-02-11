+++
author = ""
comments = true
date = "2015-10-22T16:00:18+09:00"
draft = false
image = ""
menu = ""
share = true
slug = "laravel-policy"
tags = ["laravel", "5.1.11"]
title = "laravel policyのハマりどころ"

+++

## authorization

Laravel5.11からauthorization（認可）が追加されたよ！

 * http://laravel.com/docs/5.1/authorization
 * http://readouble.com/laravel/5/1/ja/authorization.html

使い方は簡単なので上で見てね！！！  
使い方の説明じゃないよ！！クソなハマりポイントの解説だよ！！！


これでLaravelでもデフォルトでACLが出来るよ！  
やったねタエちゃん！！

## ControllerとPolicyとのひも付け

AuthServiceProviderにモデルとポリシーファイルとを紐付けるよ！

 * AuthServiceProvider.php
```
protected $policies = [
    'App\Measure' => 'App\Policies\MeasurePolicy',
]
```

ポリシーファイルはartisanコマンドから生成できるよ！

```
$ php artisan make:policy MeasurePolicy
```

ポリシーファイルに制限規則を記述するよ！  
`true` が返れば処理は正常に続行されるよ！

 * MeasurePolicy.php
```
public function show(User $user, Customer $customer)
{
    return $user->bureau_id === $customer->measure->bureau->id;
}
```

コントローラーとポリシーのメソッド名は自動で紐づくんだって！   
かしこいね！！  

>ポリシーのメソッドはコントローラーのメソッドと頻繁に対応します。たとえば前記のupdateメソッドのように、updateと言う名前がコントローラーメソッドとポリシーメソッドで共通です。そのため、Laravelはauthorizeメソッドの引数にインスタンスだけを渡すことも許しています。認可するアビリティは呼び出しているメソッドの名前を元に自動的に決められます。この例では、コントローラーのupdateメソッドからauthorizeが呼びだされていますから、PostPolicy上でもupdateメソッドが呼びだされます。

MeasureController

```
public function create($id)
{
    $customer = Customer::find(1);
    $this->authorize($customer);
}
```


その結果...  
↓  ↓  ↓  ↓  


＿人人人人人人＿  
＞　突然の死　＜  
￣Y^Y^Y^Y^Y￣  


### なぜなのか？

ポリシーファイルが暗黙的にコントローラメソッドと連携する場合、  
**第二引数のインスタンスに紐づくポリシー** が適用されるよ！  
つまり上の例だと、`CustomerPolicy` の`create`メソッドが呼ばれてることになるよ！


`ほんと死ねばいいのにね！！！！！！！！！！`

### 回避方法

第二引数のインスタンスから取得するので、第二引数に（たとえPolicy内で使わないとしても）その **コントローラーに紐付いてるモデルインスタンス** を投げるよ！！！

 * MeasureController.php
```
public function create($id)
{
    $customer = Customer::find(1);
    $this->authorize([new Exchange(), $customer]);
}
```

ポリシー側も変更するよ！

 * MeasurePolicy.php
```
public function show(User $user, Measure $measure, Customer $customer)
{
    return $user->bureau_id === $customer->measure->bureau->id;
}
```

`(*^◯^*) これで動くんだ！`

ちなみにコントローラー側から authorize関数の引数に2つ以上のインスタンスを渡す時は、**配列** にして渡すんだ！わかりづらいんだ！！死ねばいいのに！！
