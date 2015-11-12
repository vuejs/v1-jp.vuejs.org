---
title: リアクティブの探求
type: guide
order: 13
---

私達は基本のほとんどをカバーしてきました。これからは深いダイビングするための時間です！Vue.js の最も明確な特徴の1つは、控えめなリアクティブシステムです。モデルは単なるプレーンな JavaScript オブジェクトで、それを変更し View を更新します。それは状態管理が非常にシンプルで直感的になりますが、いくつかの一般的な落とし穴を避けるためにそれがどのように動作するか理解することも重要です。このセクションで、私達は Vue.js のリアクティブシステムの低レベルの詳細の一部について掘り下げていきます。

## 変更の追跡方法

プレーンな JavaScript オブジェクトを `data` オプションとして Vue インスタンスに渡すとき、Vue.js はその全てのプロパティを渡り歩いて、それらを [Object.defineProperty](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty) を使用して、getter/setter に変換します。これは ES5 だけのシム (shim) ができない機能で、Vue.js が IE8 以下をサポートしない理由です。

getter/setter はユーザーには見えませんが、内部ではそれらは Vue.js で依存関係の追跡を実行したり、プロパティがアクセスされたまたは変更されたときは、変更通知します。注意事項の1つは、データオブジェクトが記録されたとき、getter/setter のブラウザコンソールのフォーマットが異なるので、よりフレンドリな閲覧出力にするため、`vm.$log()` インスタンスメソッドを使用してください。

テンプレートでの全てのディレクティブ/データバインディングは、それが依存関係として評価されている間、全てのプロパティを**触れた (touched)** ものとして記録している**ウオッチャ (watcher)** オブジェクトが存在しています。その後、依存する setter が呼び出されるとき、それは再評価するウオッチャをトリガし、そして関連するディレクティブに DOM の更新する結果につながります。

![data](/images/data.png)

## 変更検出の注意事項

ES5 の制限のため、Vue.js は**プロパティの追加または削除を検出できません**。Vue.js はインスタンスの初期化中に、getter/setter 変換処理を実行するため、プロパティは、Vue.js がそれを変換しそしてそれをリアクティブにするために、`data` オブジェクトに存在しなければなりません。例えば:

``` js
var data = { a: 1 }
var vm = new Vue({
  data: data
})
// `vm.a` と `data.a` は今はリアクティブです

vm.b = 2
// `vm.b` はリアクティブでは"ありません"

data.b = 2
// `data.b` はリアクティブでは"ありません"
```

しかしながら、インスタンス作成後に、**プロパティを追加してリアクティブにする**方法があります。

Vue インスタンスに対して、`$set(path, value)` インスタンスメソッドを使用することができます:

``` js
vm.$set('b', 2)
// `vm.b` と `data.b` は今はリアクティブです
```

プレーンなデータオブジェクトに対しては、グロバール `Vue.set(object, key, value)` メソッドを使用することができます:

``` js
Vue.set(data, 'c', 3)
// `vm.c` と `data.c` は今はリアクティブです
```

[以前に リストレンダリング のセクションで議論した](/guide/list.html#Caveats) いくつかの配列に関連した注意事項があります。

## データの初期化

Vue.js は動的にその場でリアクティブなプロパティを追加するための API を提供していますが、`data` オプションで前もって全てのリアクティブなプロパティを宣言することを推奨します。

以下の代わりに:

``` js
var vm = new Vue({
  template: '<div>{{msg}}</div>'
})
// 後で、`msg` を追加する
vm.$set('msg', 'Hello!')
```

以下の方がよいです:

``` js
var vm = new Vue({
  data: {
    // 空の値として msg を宣言する
    msg: ''
  },
  template: '<div>{{msg}}</div>'
})
// 後で `msg` を設定する
vm.msg = 'Hello!'
```

このパターンに背後に以下の2つの理由があります:

1. `data` オブジェクトはあなたのコンポーネントの状態に対するスキーマのようなものです。

2. Vue インスタンスでトップレベルのリアクティブなプロパティを追加すると、それは、以前に存在しないそしてウオッチャが依存関係として追跡することができないスコープを再評価するために全てのウオッチャに強制します。

## 非同期更新キュー

デフォルトで、Vue.js は **非同期** に DOM 更新を実行します。データ変更が監視されている限り、Vue はキューをオープンし、同じイベントループで起こる全てのデータ変更をバッファリングします。同じウオッチャが複数回トリガされる場合、一度だけキューに押し込まれます。そして、次のイベントループの "tick" で、Vue はキューをフラッシュし、必要な DOM 更新だけ実行します。内部的には、Vue はもし非同期キューイング向けに `MutationObserver` が利用可能ならそれを使い、そうでなければ `setTimeout(fn, 0)` にフォールバックします。

例として、`vm.someData = 'new value'` をセットした時、DOM はすぐには更新しません。 キューがフラッシュされた時、次の "tick" で更新します。ほとんどの場合、私達はこれについて気にする必要はありませんが、更新した DOM の状態に依存する何かをしたい時、注意が必要です。Vue.js は一般的に"データ駆動"的な流儀で考えることを開発者に奨励していますが、時どき、それはあなたの手を汚し得る必要があるかもしれません。Vue.js でデータの変更後に、DOM の更新が完了するまでに待つためには、データが変更された直後に `Vue.nextTick(callback)` を使用することができます。コールバックが呼ばれた時、DOM は更新されているでしょう。例えば:

``` html
<div id="example">{{msg}}</div>
```

``` js
var vm = new Vue({
  el: '#example',
  data: {
    msg: '123'
  }
})
vm.msg = 'new message' // データを変更する
vm.$el.textContent === 'new message' // false
Vue.nextTick(function () {
  vm.$el.textContent === 'new message' // true
})
```

特に便利な内部コンポーネントのインスタンスメソッド `vm.$nextTick()` もあります。なぜなら、それはグローバルな `Vue` とそのコールバックの `this` コンテキストは自動的に現在の Vue インスタンスにバウンドされるからです:

``` js
Vue.component('example', {
  template: '<span>{{msg}}</span>',
  data: function () {
    return {
      msg: 'not updated'
    }
  },
  methods: {
    updateMessage: function () {
      this.msg = 'updated'
      console.log(this.$el.textContent) // => 'not updated'
      this.$nextTick(function () {
        console.log(this.$el.textContent) // => 'updated'
      })
    }
  }
})
```

## 算出プロパティの内部

Vue.js の算出プロパティ (computed property) は getter は単純では**ない**ことに注意すべきです。各算出プロパティは独自のリアクティブな依存関係の追跡します。算出プロパティが評価されるとき、Vue.js は依存関係リストを更新し、結果の値をキャッシュします。キャッシュされた値は追跡された依存関係の1つが変更されたときだけ、無効化されます。したがって、依存関係が変更されなかった間ずっと、算出プロパティのアクセスは getter を呼びだす代わりに、直接キャッシュされた値を返します。

なぜ、私達はキャッシングする必要があるのでしょうか？私達が、巨大な配列をループして計算をたくさんする必要がある、高価な算出プロパティ **A** を持っていると想像してください。その後、私達は **A** に依存する同様の他の算出プロパティを持っているかもしれません。キャッシングがなければ、私達は必要以上 **A** の getter を呼びだすことになるでしょう！

算出プロパティのキャッシングのために、getter 関数は、算出プロパティにアクセスするとき、常に呼び出されません。次の例を考えてみます:

``` js
var vm = new Vue({
  data: {
    msg: 'hi'
  },
  computed: {
    example: function () {
      return Date.now() + this.msg
    }
  }
})
```

算出プロパティ `example` は、`vm.msg` という1つだけの依存関係を持っています。`Date.now()` というタイムスタンプは、Vue のデータ監視システムとは何も関係ないため、リアクティブな依存関係では**ありません**。したがって、プログラムで `vm.example` にアクセスするとき、`vm.msg` が再評価をトリガしない限り、同じタイムスタンプを見つけるでしょう。

いくつかのユースケースでは、単純に再度 getter を呼びだす `vm.example` にアクセスする度に、簡単な getter のような振舞いを保存したいかもしれません。特定の算出プロパティに対してキャッシュをオフにすることによって、それを行うことができます:

``` js
computed: {
  example: {
    cache: false,
    get: function () {
      return Date.now() + this.msg
    }
  }
}
```

これにより、`vm.example` にアクセスする度に、タイムスタンプは最新になるでしょう。**しかしながら、これはプログラム的に JavaScript 内でのアクセスだけ影響することに注意してください。データバインディングはまだ依存関係駆動です。**テンプレートで `{% raw %}{{example}}{% endraw %}` として算出プロパティにバインドするとき、DOM はリアクティブな依存関係が変更されたときにだけ更新されます。