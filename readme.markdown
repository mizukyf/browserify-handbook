# はじめに

このドキュメントはモジュール化されたアプリケーションをビルドするために[browserify](http://browserify.org) を利用する方法を説明したものです。

[![cc-by-3.0](http://i.creativecommons.org/l/by/3.0/80x15.png)](http://creativecommons.org/licenses/by/3.0/)

browserifyは[Node.jsにより拡張された](http://nodejs.org/docs/latest/api/modules.html) CommonJSモジュールをWebブラウザ向けにコンパイルするためのツールです。

もし仮にバンドル作成とnpmコマンドによるパッケージ・インストール以外では [Node.js](http://nodejs.org) それ自体を利用していないとしても、あなたはあなたの製造したコードとサードパーティ製のライブラリ群を組み合わせるためにbrowserifyを利用することができます。

browserifyが利用するモジュール・システムはNode.jsが利用するそれと同じです。[npm](https://npmjs.org) 向けに公開されたパッケージは、それが元来ブラウザのランタイムではなくNode.jsランタイムにおいて利用されることを想定して作成されたものであっても、browserifyによってブラウザ上でも同じように機能します。

多くの人びとがNode.jsランタイム上だけでなくbrowserifyを利用することでWebブラウザ上でも動作するよう設計されたモジュールをnpm向けに公開しつつあります。そしてnpmで公開されている多くのパッケージはまさにWebブラウザ上での利用を想定して設計されるようになってきています。
[npm はすべてのJavaScriptランタイムのために](http://maxogden.com/node-packaged-modules.html)利用できるものです。フロントエンドとバックエンドに大きなちがいはないのです。

# 目次

- [はじめに](#はじめに)
- [目次](#目次)
- [Node.jsパッケージ化された原稿](#nodejsパッケージ化された原稿)
- [Node.jsパッケージ化されたモジュール](#nodejsパッケージ化されたモジュール)
  - [require関数](#require関数)
  - [exportsプロパティ](#exportsプロパティ)
  - [ブラウザのためのバンドル化](#ブラウザのためのバンドル化)
  - [browserifyのはたらき方](#browserifyのはたらき方)
  - [node_modulesのはたらき方](#node_modulesのはたらき方)
  - [なぜ連結するのか](#なぜ連結するのか)
- [開発](#開発)
  - [ソースマップ](#ソースマップ)
    - [exorcist](#exorcist)
  - [自動再コンパイル](#自動再コンパイル)
    - [watchify](#watchify)
    - [beefy](#beefy)
    - [wzrd](#wzrd)
    - [browserify-middlewareとenchilada](#browserify-middlewareとenchilada)
    - [livereactload](#livereactload)
    - [browserify-hmr](#browserify-hmr)
    - [budo](#budo)
  - [APIを直接利用する](#apiを直接利用する)
  - [grunt](#grunt)
  - [gulp](#gulp)
- [組み込みオブジェクト](#組み込みオブジェクト)
  - [Buffer](#Buffer)
  - [process](#process)
  - [global](#global)
  - [__filename](#__filename)
  - [__dirname](#__dirname)
- [トランスフォーム](#トランスフォーム)
  - [トランスフォームを実装する](#トランスフォームを実装する)
- [package.json](#package.json)
  - [browserフィールド](#browserフィールド)
  - [browserify.transformフィールド](#browserify.transformフィールド)
- [finding good modules](#finding-good-modules)
  - [module philosophy](#module-philosophy)
- [organizing modules](#organizing-modules)
  - [avoiding ../../../../../../..](#avoiding-)
  - [non-javascript assets](#non-javascript-assets)
  - [reusable components](#reusable-components)
- [testing in node and the browser](#testing-in-node-and-the-browser)
  - [testing libraries](#testing-libraries)
  - [code coverage](#code-coverage)
  - [testling-ci](#testling-ci)
- [bundling](#bundling)
  - [saving bytes](#saving-bytes)
  - [standalone](#standalone)
  - [external bundles](#external-bundles)
  - [ignoring and excluding](#ignoring-and-excluding)
  - [browserify cdn](#browserify-cdn)
- [shimming](#shimming)
  - [browserify-shim](#browserify-shim)
- [partitioning](#partitioning)
  - [factor-bundle](#factor-bundle)
  - [partition-bundle](#partition-bundle)
- [compiler pipeline](#compiler-pipeline)
  - [build your own browserify](#build-your-own-browserify)
  - [labeled phases](#labeled-phases)
    - [deps](#deps)
      - [insert-module-globals](#insert-module-globals)
    - [json](#json)
    - [unbom](#unbom)
    - [syntax](#syntax)
    - [sort](#sort)
    - [dedupe](#dedupe)
    - [label](#label)
    - [emit-deps](#emit-deps)
    - [debug](#debug)
    - [pack](#pack)
    - [wrap](#wrap)
  - [browser-unpack](#browser-unpack)
- [plugins](#plugins)
  - [using plugins](#using-plugins)
  - [authoring plugins](#authoring-plugins)

# Node.jsパッケージ化された原稿

このハンドブックはnpmを通じてインストールが可能です。それには次のようにするだけです：

```
npm install -g browserify-handbook
```

さあこれで、あなたは`browserify-handbook`コマンドを使うことができます。コマンドは`$PAGER`で指定されたペイジャーを利用してこのREADMEファイルを開きます。もちろんコマンドを利用せず現在そうしているようにこのドキュメントを読み続けていただいても構いません。

# Node.jsパッケージ化されたモジュール

browserifyをどのように使用するか、そしてそれがどのように機能するかについて立ち入った話をする前に、まずは[Node.jsにより拡張された](http://nodejs.org/docs/latest/api/modules.html)
CommonJSモジュール・システムがどのように機能するかについてまず理解することが重要です。

## require関数

Node.jsランタイムでは、他のファイルからコードを読み込むために`require()` 関数が利用されます。

例えば[npm](https://npmjs.org)コマンドでモジュールをインストールすると:

```
npm install uniq
```

`nums.js` ファイルの中で `require('uniq')` と関数を呼び出すことができます:

```
var uniq = require('uniq');
var nums = [ 5, 2, 1, 3, 2, 5, 4, 2, 0, 1 ];
console.log(uniq(nums));
```

Node.jsランタイムでこのプログラムを実行すると次のような出力がなされます:

```
$ node nums.js
[ 0, 1, 2, 3, 4, 5 ]
```

`.`で始まる文字列を使用して相対パスでファイルを指定することもできます。例えば`foo.js` from `main.js`ファイルから`foo.js`ファイルを読み込む場合、`main.js`の中で次のようにすることができます:

``` js
var foo = require('./foo.js');
console.log(foo(4));
```

もし`foo.js`が親ディレクトリの配下に存在するのであれば、`../foo.js`と指定します:

``` js
var foo = require('../foo.js');
console.log(foo(4));
```

同様にしてその他いかなる種類の相対パスも利用可能です。相対パスの解決は常に実行中のファイルのパスを基準にして行われます。

注意点として、この例では`require()`は関数を返しており私たちはその値を`uniq`という名前の変数に代入していますが、この名前は他のいかなるものであってもかわらず関数は機能します。 `require()` はあなたが指定した名前でモジュールが公開するAPIを利用できるようにしてくれます。

`require()`の機能の仕方は他の多くのモジュール・システムと異なっています。インポートは、あなたの制御の及ばないモジュール自体の中で宣言されたグローバル変数やファイル・ローカルなレキシカル・スコープ変数の宣言それ自体にそっくりです。`require()`によるNode.jsスタイルのインポートは、それぞれのモジュールがどこからやって来たものなのか、プログラムを読む人にとってわかりやすくできています。このアプローチの利点はアプリケーションを構成するモジュールの数が多くなるほど増していきます。

## exportsプロパティ

他のファイルからインポートできるよう単一のオブジェクトをエクスポートする場合、次のように`module.exports`プロパティの値として代入を行います:

``` js
module.exports = function (n) {
    return n * 111
};
```

ここでとあるモジュール`main.js`があなたが最前作成したモジュール`foo.js`を読み込むものとします。`require('./foo.js')`の戻り値はモジュールがエクスポートしている関数になります:

``` js
var foo = require('./foo.js');
console.log(foo(5));
```

このプログラムは次のような出力を行うでしょう:

```
555
```

`module.exports`を使えば関数に限らずいかなる値でもエクスポートできます。

例えば、次のコードは完璧に機能します:

``` js
module.exports = 555
```

それから次のコードも:

``` js
var numbers = [];
for (var i = 0; i < 100; i++) numbers.push(i);

module.exports = numbers;
```

オブジェクトのプロパティとして複数の項目をエクスポートするのに特化した別形式もあります。この場合は`module.exports`ではなく`exports`を使用します:

``` js
exports.beep = function (n) { return n * 1000 }
exports.boop = 555
```

このプログラムは次のものと同じ意味になります:

``` js
module.exports.beep = function (n) { return n * 1000 }
module.exports.boop = 555
```

`module.exports`は`exports`と同義で、空のオブジェクトで初期化されています。

注意点として、次のようにすることはできません:

``` js
// これは機能しない
exports = function (n) { return n * 1000 }
```

畢竟エクスポートされるのは`module.exports`が参照するオブジェクトであり、
`exports`にはその初期値の空のオブジェクトへの参照がコピーされているだけなので、
`exports`に新しいオブジェクトを代入してしまうと当初参照していた空のオブジェクトへの参照が覆い隠されてしまいます。

したがって単一の項目をエクスポートするときにはいつでも次のようにします:

``` js
// instead
module.exports = function (n) { return n * 1000 }
```

もしまだ得心できないようであれば、バックグラウンドでモジュールがどのようにはたらいているかを考えてみましょう:

``` js
var module = {
  exports: {}
};

// モジュール定義部は関数にくるまれる
(function(module, exports) {
  // module.exportsが参照するのと同じ{}への参照を、別の値への参照で上書きしてしまう
  exports = function (n) { return n * 1000 };
}(module, module.exports))

// module.exportsは引続き{}を参照している
console.log(module.exports);
```

ふつう1つのモジュールは1つのことをもっぱらとするのがベストなので、大抵の場合`module.exports`でエクスポートするのは1つの関数かコンストラクタになるでしょう。

実はエクスポートの手段としては`exports`がもともとのものであり、`module.exports`は後発なのですが、
`module.exports`の方がより直接的で、明確で、重複を避けられる点で便利なものだということがわかりました。

当初は次のような書き方の方がより一般的なものでした:

foo.js:

``` js
exports.foo = function (n) { return n * 111 }
```

main.js:

``` js
var foo = require('./foo.js');
console.log(foo.foo(5));
```

しかし`foo.foo`という記述はちょっと冗長なものに見えないでしょうか。`module.exports`を使用したほうがより明確です:

foo.js:

``` js
module.exports = function (n) { return n * 111 }
```

main.js:

``` js
var foo = require('./foo.js');
console.log(foo(5));
```

## ブラウザのためのバンドル化

Node.jsランタイムでモジュールのコードを動かすには、どこかから起動をする必要があります。

Node.jsランタイムでモジュールを動かすには`node`コマンドにファイルを渡します:

```
$ node robot.js
beep boop
```

In browserifyを使う場合、モジュールを実行するには、ファイルを指定して実行するのではなく、JavaScriptファイル群を連結したストリームを標準出力に出力し、これを`>`演算子でファイルに書き出します:

```
$ browserify robot.js > bundle.js
```

この時点で`bundle.js`には`robot.js`を実行するのに必要なすべてのJavaScriptコードが含まれている状態になります。
そしてこのファイルへの参照をHTMLファイルのscriptタグに設定します:

``` html
<html>
  <body>
    <script src="bundle.js"></script>
  </body>
</html>
```

付言しておくと、`</body>`の直前にscriptタグを配置することで、あなたのコードはDOMのonreadyイベントを待つことなくすべてのDOM要素にアクセスすることができます。

バンドル化により行えることはもっとたくさんあります。バンドル化のセクションを確認してみてください。

## browserifyのはたらき方

browserifyは指定されたエントリーポイントとなるファイルから処理をはじめ、
ソースコードの[抽象構文木](https://en.wikipedia.org/wiki/Abstract_syntax_tree)を
[静的解析](http://npmjs.org/package/detective)することにより見つかった
`require()`呼び出しを順番に辿っていきます。

すべての`require()`呼び出しについて、browserifyは指定されたモジュール名からファイルパスを導出し、
当該のモジュールのファイルを検索します。
`require()`の再帰的な呼び出しが形づくる依存性グラフのすべてを辿り尽くすまでこの手続きは終わりません。

静的解決されたモジュール名と内部的なIDを紐付けるマップである`require()`に関する最小限の定義情報とともに、
各ファイルは単一のJavaScriptファイルに連結されます。

バンドルは完全に自己完結しており、あなたのアプリケーションが必要とするすべてのコードを内包しています。

browserifyのはたらき方についてより詳しくは、コンパイラ・パイプラインのセクションを参照してください。

## node_modulesのはたらき方

Node.jsは賢いモジュール解決アルゴリズムを使っています。これは競合するプラットフォームの中でもユニークなものです。

コマンドラインにおいて`$PATH`がその役割を果たしているようなシステムで規定された検索パスの配列からパッケージを解決するかわりに、Node.jsが採用するメカニズムはデフォルトでローカルなものといえます。

`/beep/boop/bar.js`の中で`require('./foo.js')`を呼び出したとき、
Node.jsは`/beep/boop/foo.js`を検索します。
`./`や`../`で始まるパスは必ずローカルのファイルを指すものとして処理されます。

一方`/beep/boop/foo.js`の中で`require('xyz')`のように非・相対パスでモジュールを指定した場合、
Node.jsはそれらのパスを次の順序で検索し、マッチするものが見つかればそこで終わり、
もし最後まで見つからなければエラーを発生させます:

```
/beep/boop/node_modules/xyz
/beep/node_modules/xyz
/node_modules/xyz
```

上記のすべての`xyz`ディレクトリが存在したとしても、
Node.jsは最初に見つかった`xyz/package.json`の中に`"main"`フィールドが存在するか確認します。
`"main"`フィールドは`require()`で当該モジュールが指定されたとき
読み込まれるべきファイルを規定する項目です。

例えば、`/beep/node_modules/xyz`が一番はじめにマッチしたディレクトリで、
その`/beep/node_modules/xyz/package.json`の内容が次のようなコードであった場合:

```
{
  "name": "xyz",
  "version": "1.2.3",
  "main": "lib/abc.js"
}
```

`/beep/node_modules/xyz/lib/abc.js`がエクスポートするオブジェクトが
、`require('xyz')`が呼び出し元に返すオブジェクトとなります。

ディレクトリ内に`package.json`が見つからなかったり、
JSONファイル内に`"main"`フィールドが存在しなかった場合、
デフォルト値として`index.js`が採用されます:

```
/beep/node_modules/xyz/index.js
```

もし必要なら、ディレクトリ内の特定のファイルを取り出すこともできます。
例えば `dat`パッケージから`lib/clone.js`を読み込むには、次のようにします:

```
var clone = require('dat/lib/clone.js')
```

再帰的なnode_modules解決メカニズムによりディレクトリ階層構造の中から`dat`が発見されます。
そして今度はそのディレクトリの中で`lib/clone.js`というファイルが検索されます。
この`require('dat/lib/clone.js')`式のアプローチは`require('dat')`が可能な場所であればどこでも機能します。

Node.jsはパスの配列を使用してモジュール解決を行うメカニズムも持っています。
しかしこのメカニズムは廃止予定となっており、
よほどの理由がない限り`node_modules/`を用いたデフォルトのメカニズムを使用すべきです。

Node.jsのアルゴリズムとnpmによるパッケージ・インストールの仕組みの素晴らしい点は、
他のほとんどのプラットフォームに存在するバージョン競合の問題を回避できることです。

すべてのライブラリは自身のローカルな`node_modules/`ディレクトリの中に
依存するライブラリを格納しておくことができ、
それらのライブラリのそれぞれはまた自身のローカルな`node_modules/`ディレクトリの中に
依存するライブラリを格納しておくことができ、・・・という具合に再帰的に続きます。

つまり同じアプリケーションの中で同じライブラリの異なるバージョンを利用することができます。
これによりAPI群を協調=一致させるためのオーバーヘッドが大幅に低減されます。
npmのような非・中央集権的なエコシステムにおいて、
公開され互いの依存関係によって組織されているパッケージを管理する上で、この機能はとても重要なものです。
開発者の各々は彼らが適切を考える方法でパッケージを公開するだけでよく、
それらのパッケージの依存性のバージョン選択が同一のアプリケーション内における
他の依存性に悪影響を及ぼすことを心配しなくてよいのです。

開発者は自身のローカルのアプリケーション・モジュールの組織のされ方について
`node_modules/`のはたらき方を変更することが可能です。
この点について詳しくは `avoiding ../../../../../../..`セクションを参照してください。

## なぜ連結するのか

browserifyの処理はビルド・ステップで行われます。
アプリケーションの動作に必要なコードは連結され、生成されたバンドル・ファイルにはすべてが内包された状態となります。

しかしブラウザのために開発されたモジュール・システムには異なるアプローチをとるものもあります。
それらの強みと弱みについて見ていきましょう:

### windowグローバル変数

アプリケーションを構成する各ファイルが、特定のモジュール・システムを利用せず、
windowグローバルなオブジェクトを定義したり、独自の内部的な名前空間を作り上げたりするアプローチです。

このアプローチでは、アプリケーションが表示する各ページのHTMLすべてについて
必要となる新しいJavaScriptファイルのための`<script>`を追加するという、
恐るべき労苦なしにアプリケーションの拡張ができません。
加えてまた、それらのファイルの順序関係が問題となります。
あるファイル群は別のファイル群が読み込まれる前に読み込まれていないといけない、
なぜなら後者のファイル群では前者のファイル群により定義される
グローバル変数が環境上にすでに存在していることを前提に動作するから、・・・と言った具合にです。

この方法でビルドされたアプリケーションのリファクタリングやメンテナンスは困難なものとなるでしょう。
この方法の強みは、すべてのブラウザがネイティブにサポートしており、
ビルドのために追加のツールが一切不要であるということです。

各`<script>`タグは新たなHTTPリクエストのやり取りを発生させるので、
このアプローチはアプリケーションのスタートが遅くなるという問題も抱えています。

### ファイル連結

サーバ側で事前にすべてのスクリプトを連結するアプローチです。
このコードは依然として順序関係の問題と、メンテナンスが困難である問題を抱えています。
しかし`<script>`タグによるHTTPリクエストは1回だけになるので、
アプリケーションの読み込み時間は改善されます。

ただしソースマップなしには、例外がスローされたコードの位置と
連結前のオリジナルのファイルにおけるコードの位置とを対応させることができません。

### AMD

`define()`関数とその引数として渡すコールバック関数により各モジュールをくるむアプローチです。
[これをAMDといいます](http://requirejs.org/docs/whyamd.html)。

第1引数はロードするモジュール名の配列で、それらのモジュールはロードされたあと
第2引数のコールバックの引数として、配列内で指定された順序で渡されます。

``` js
define(['jquery'] , function ($) {
    return function () {};
});
```

`define()`関数の第1引数により、あなたは自身のモジュールに名前を付与することができます。
こうすることであなたのモジュールを他のモジュールが読み込むことも可能になります。

AMDではCommonJSの糖衣構文も用意されてます。
コールバックのコードは文字列としてAMDにより解析され、
[正規表現によって](https://github.com/jrburke/requirejs/blob/master/require.js#L17)`require()`の呼び出しが検出されます。

この方法で記述されたコードでは、モジュールの依存関係が明示的に示されていることで、
読み込み順序を適切に解決できるため、ファイル連結やwindowグローバル変数のアプローチに比べて
順序関係の問題が低減されています。

ふつうAMDを利用する場合、パフォーマンス上の観点からモジュールはサーバ側で単一ファイルにバンドル化されます。
そして開発中だけAMDの非同期読み込みの機能が利用されます。

### CommonJSモジュールをサーバ側でバンドル化

どうせパフォーマンス上の理由でビルド・ステップを踏む必要があり、`require()`の糖衣構文を利用することになるのであれば、
AMDの作業をすべて廃しCommonJSモジュールを直接バンドル化するのが理にかなっているでしょう（これがbrowserifyのアプローチです）。
バンドル化のツールにより順序問題は解消され、開発環境も本番環境もより似通った状態となり安定性を増します。
CommonJSの構文はよくできており、そのエコシステムはNode.jsとnpmのおかげで急拡大を続けています。

あなたはNode.jsランタイムとブラウザのランタイムとの間でシームレスにコードを共有できます。
必要なのはビルド・ステップと、ソースマップを生成するツール、そして自動リビルドのためのツールだけです。

加えるに、私たちはNode.jsのモジュール検索アルゴリズムを使用することで、
バージョン・ミスマッチから来る神経衰弱から自身を守ることもできます。
同一アプリケーションの中で読み込んでいるモジュールのバージョン競合を気にする必要はありません。
読み込みバイト数の削減のため重複排除も行われます。これについてはこの資料の別の場所で説明しています。

# 開発

コード連結にいくらか負の側面があることは事実ですが、
開発ツールを利用することでそれらの問題に対して適切に対処することができます。

## ソースマップ

browserifyはソースマップを有効化するための`--debug`/`-d`フラグと`opts.debug`パラメータをサポートしています。
ソースマップはWebブラウザに対してバンドル化されたJavaScriptコードの行と列の位置情報を
オリジナルのファイルの行と列の位置情報に変換する方法を提示します。

ソースマップはすべてのオリジナル・ファイルの内容をインラインで保持しています。
このため、すべてのオリジナルのファイルがWebサーバ上の適切なパスに存在するかどうか確認する必要はなく、
バンドル化されたJavaScriptファイルをWebサーバ上に配備するだけでよいのです。

### exorcist

コード連結の負の側面の1つは、すべてのオリジナルのファイルの内容がインライン化されたソースマップ内に保持されているためにバンドルのサイズが倍化してしまうことです。
ローカルでデバッグを行う分にはこれでもよいのですが、このままソースマップを本番環境に投入するのは現実的ではありません。
この問題を解決するために[exorcist](https://npmjs.org/package/exorcist)を利用できます。
このツールはインライン化されているソースマップを独立したファイルに分離してくれます。次の例では`bundle.map.js`という名前のファイルにソースマップを分離しています:

``` sh
browserify main.js --debug | exorcist bundle.js.map > bundle.js
```

## 自動再コンパイル

バンドルの再コンパイルのために都度コマンドを実行していては開発スピードは低下しますし、だいいち面倒です。
幸いにも、この問題を解決してくれるツールはたくさんあります。
これらのうちいくつかは、様々なレベルのライブ・リロード（LiveReload）機能を提供しています。
それ以外のツールは再コンパイル後に伝統的な手動再読込みを必要とします。

実際にあなたが使うことになるツールはわずかでしょうが、npmにはたくさんのツールが存在します。
多くの異なるトレードオフと開発スタイルを持つツール群です。
あなた自身の個人的な期待や経験に照らして最適なツールを見つけるのにちょっとばかり苦労するかもしれませんが、
しかしこの多様性は開発者の生産性を高め、創造と試行錯誤のための場を提供してくれるものでもあります。
思うに、ツールの多様性と小ぶりのbrowserifyコアAPIが独立して並存している状況は、
わずかな「勝者たち」がbrowserifyコアAPIを取り込まれて特別扱いされるような状況
（それはAPIのバージョンが意味するところをめぐるあらゆる種類の混乱と、
APIの内部におけるコードの経年劣化を生じさせるでしょう）よりも、
中長期的にみて健全なものでしょう。

ここで紹介するのはbrowserifyを利用した開発ワークフローをセットアップするために
あなたが必要とするであろうモジュール群です。
けれどもこのリストに（まだ）含まれていないその他のツールにも目を光らせておいてください。

### [watchify](https://npmjs.org/package/watchify)

`watchify`は`browserify`と同じように使用できるツールですが、
バンドル化されたファイルの出力を行うだけでなく
依存性グラフに含まれるすべてのファイル群の変更を監視します。
ファイルの変更を検知すると再度バンドル化が行われファイルが出力されます。
この処理はキャッシュの仕組みにより初回のバンドル化よりも高速に行われます。

`-v`フラグを使用することでバンドル化が行われるたびにメッセージを出力させることができます:

```
$ watchify browser.js -d -o static/bundle.js -v
610598 bytes written to static/bundle.js  0.23s
610606 bytes written to static/bundle.js  0.10s
610597 bytes written to static/bundle.js  0.14s
610606 bytes written to static/bundle.js  0.08s
610597 bytes written to static/bundle.js  0.08s
610597 bytes written to static/bundle.js  0.19s
```

package.jsonの"scripts"フィールドに次のように記述すると、watchifyとbrowserifyを使用するのに便利です:

``` json
{
  "build": "browserify browser.js -o static/bundle.js",
  "watch": "watchify browser.js -o static/bundle.js --debug --verbose",
}
```

本番稼動フェーズのためにバンドル化するときは`npm run build`を実行し、
開発フェーズの間ファイル監視をするときは`npm run watch`を実行します。

[`npm run`についてより詳しくはこちらを参照してください](http://substack.net/task_automation_with_npm_run)。

### [beefy](https://www.npmjs.org/package/beefy)

コードが変更されたときそれを自動で再コンパイルしてくれるWebサーバを求めているのであれば、
[beefy](http://didact.us/beefy/)について調べてみてください。

この開発サーバを起動するにはエントリーポイントとなるファイルを指定します:

```
beefy main.js
```

これだけでデフォルトのポート番号で開発サーバが起動します。

### [wzrd](https://github.com/maxogden/wzrd)

beefyと似ていますがより小ぶりの開発サーバとして[wzrd](https://github.com/maxogden/wzrd)があります。

`npm install -g wzrd`コマンドでインストールしたら、次のように起動します:

```
wzrd app.js
```

そして http://localhost:9966 をWebブラウザで開きます。

### browserify-middlewareとenchilada

expressを利用している場合は
[browserify-middleware](https://www.npmjs.org/package/browserify-middleware)
もしくは[enchilada](https://www.npmjs.org/package/enchilada)
について調べてみてください。

いずれのモジュールもexpressアプリケーションにbrowserifyバンドルを配信させるための
ミドルウェアを提供するものです。

### [livereactload](https://github.com/milankinen/livereactload)

livereactloadは[react](https://github.com/facebook/react)アプリケーションのためのツールです。
コードが変更されるとツールによりあなたのWebページの状態も更新されます。

livereactloadは何の変哲もないbrowserifyのトランスフォームの1つとして実装されており、
`-t livereactload`オプションを指定することでこのトランスフォームを読み込むことができます。
より詳しい情報を得るため
[プロジェクトのREADME](https://github.com/milankinen/livereactload#livereactload)
を参照することを強くおすすめします。

### [browserify-hmr](https://github.com/AgentME/browserify-hmr)

browserify-hmrは動的モジュール置換（Hot Module Replacement）を行うためのプラグインです。

個々のファイルは自身で動的更新を受け容れるかどうかを指定します。
更新を受け容れると指定しているファイルやその依存性が更新されると、
当該ファイルの新しいコードが再実行されます。

例えば、ここに`main.js`というファイルがあります:

``` js
document.body.textContent = require('./msg.js')

if (module.hot) module.hot.accept()
```

そして`msg.js`というファイルも:

``` js
module.exports = 'hey'
```

`browserify-hmr`プラグインを使ってこの`main.js`の変更を監視し再読み込みさせることができます:

```
$ watchify main.js -p browserify-hmr -o public/bundle.js -dv
```

`public/`配下の静的コンテンツを静的ファイル・サーバを使ってクライアント側に配信します:

```
$ ecstatic public -p 8000
```

この時点でブラウザを使って`http://localhost:8000`を閲覧すると、Webページには`hey`と表示されます。

もし`msg.js`を次のように変更すると:

``` js
module.exports = 'wow'
```

すぐさまWebページのコンテンツが自動的に更新され`wow`と表示されるでしょう。

browserify-hmrと[react-hot-transform](https://github.com/AgentME/react-hot-transform)
を併用することで、`module.hot`APIを使用するコードに加えて
Reactのコンポーネントについても自動更新させることができるようになります。
変更されるたびバンドル化が行われる[livereactload](https://github.com/milankinen/livereactload)とちがい、
変更されたファイルが再実行されるだけです。

### [budo](https://github.com/mattdesl/budo)

budoはインクリメンタル・ビルドとライブ・リロード（LiveReload）機能の統合（CSSインジェクションを含む）に
強く焦点を絞ったbrowserify開発サーバの1つです。

次のようにインストールし:

```sh
npm install budo -g
```

エントリーファイルを指定して実行します:

```
budo app.js
```

こうすることでWebサーバが起動し、ブラウザで[http://localhost:9966](http://localhost:9966)
にアクセスするとデフォルトの`index.html`ファイルが配信されます。
ファイルを変更し保存するとインクリメンタル・ビルドが行われます。
リクエストへの応答はバンドル化が完了するまで遅らせられるので、
ファイルの更新中に古いバンドルや空っぽのバンドルが配信される心配はありません。

ライブ・リロード（LiveReload）を有効にしてJS/HTML/CSSファイルが更新されるたびブラウザに再読み込みをさせるには、
次のようにします:

```
budo app.js --live
```

## APIを直接利用する

開発のため`http.createServer()`から直接browserifyが公開するAPIを利用することもできます:

``` js
var browserify = require('browserify');
var http = require('http');

http.createServer(function (req, res) {
    if (req.url === '/bundle.js') {
        res.setHeader('content-type', 'application/javascript');
        var b = browserify(__dirname + '/main.js').bundle();
        b.on('error', console.error);
        b.pipe(res);
    }
    else res.writeHead(404, 'not found')
});
```

## grunt

もしgruntを使用されているのであれば、たぶん
[grunt-browserify](https://www.npmjs.org/package/grunt-browserify)
を利用したくなることでしょう。

## gulp

もしgulpを使用されているのであれば、browserifyの公開するAPIを直接利用することになるでしょう。

こちらにgulpとbrowserifyを使用した開発フローの
[導入のためのガイド](http://viget.com/extend/gulp-browserify-starter-faq)
があります。

こちらには
[gulpを使用した開発フローにおいてwatchifyによりbrowserifyのビルドを高速化する](https://github.com/gulpjs/gulp/blob/master/docs/recipes/fast-browserify-builds-with-watchify.md)
方法についてのガイドがあります。

# 組み込みオブジェクト

もともとNode.jsランタイム向けに開発されたモジュールがブラウザのランタイムでもなるべくそのまま動作するよう、
browserifyはブラウザ環境向けに実装し直したNode.jsコア・ライブラリを提供しています:

* [assert](https://npmjs.org/package/assert)
* [buffer](https://npmjs.org/package/buffer)
* [console](https://npmjs.org/package/console-browserify)
* [constants](https://npmjs.org/package/constants-browserify)
* [crypto](https://npmjs.org/package/crypto-browserify)
* [domain](https://npmjs.org/package/domain-browser)
* [events](https://npmjs.org/package/events)
* [http](https://npmjs.org/package/http-browserify)
* [https](https://npmjs.org/package/https-browserify)
* [os](https://npmjs.org/package/os-browserify)
* [path](https://npmjs.org/package/path-browserify)
* [punycode](https://npmjs.org/package/punycode)
* [querystring](https://npmjs.org/package/querystring)
* [stream](https://npmjs.org/package/stream-browserify)
* [string_decoder](https://npmjs.org/package/string_decoder)
* [timers](https://npmjs.org/package/timers-browserify)
* [tty](https://npmjs.org/package/tty-browserify)
* [url](https://npmjs.org/package/url)
* [util](https://npmjs.org/package/util)
* [vm](https://npmjs.org/package/vm-browserify)
* [zlib](https://npmjs.org/package/browserify-zlib)

events、stream、url、pathそしてquerystringはブラウザ環境でもとくに役立つものでしょう。

加うるに、browserifyはコードの中で`Buffer`や`process`、`global`、`__filename`、`__dirname`
が利用されていることを検知すると、ブラウザ向けに適切に定義しなおされたそれを読み込みます。

モジュールがbufferやstreamを利用したコードを含んでいる場合でも、
サーバー上のIOを行ったりしない限り、だいたいはブラウザ上で正しく動作します。

Node.jsランタイムでの開発経験がない場合、それらのグローバル・オブジェクトで何ができるのか、
次に示す例でいくばくか理解できることでしょう。
注意すべき点として、これらのグローバル・オブジェクトはあなたのコード
もしくはあなたのコードが依存するモジュールがそれを利用している場合に限って
バンドルに組み込まれます。

## [Buffer](http://nodejs.org/docs/latest/api/buffer.html)

Node.jsランタイムにおいてはファイルとネットワークに関するAPIはすべてBufferチャンクを処理するように作られています。
browserifyにおいて、Buffer APIは[buffer](https://www.npmjs.org/package/buffer)モジュールにより提供されます。
このBufferは拡張されたTypedArrayを用いて非常に効率よく処理されます。
TypedArrayを利用できない古いブラウザではフォールバックが利用されます。

次に示すのは`Buffer`を利用してbase64文字列を16進数文字列に変換する例です:

```
var buf = Buffer('YmVlcCBib29w', 'base64');
var hex = buf.toString('hex');
console.log(hex);
```

プログラムを実行すると次のように出力がなされるでしょう:

```
6265657020626f6f70
```

## [process](http://nodejs.org/docs/latest/api/process.html#process_process)

Node.jsにおいて、`process`は環境変数やシグナル、標準入出力ストリームなど実行中のプロセスに関する
情報とその制御を処理するための特別なオブジェクトです。

中でもイベント・ループとともに用いられる`process.nextTick()`の実装はことに重要です。

In browserify the process implementation is handled by the
[process module](https://www.npmjs.org/package/process) which just provides
`process.nextTick()` and little else.
browserifyにおいてprocessの実装は
[process module](https://www.npmjs.org/package/process)モジュールにより処理されています。
このモジュールは`process.nextTick()`とその他ほんのわずかのAPIだけを提供します。

次の例を見れば`process.nextTick()`が何をするものか理解できるでしょう:

```
setTimeout(function () {
    console.log('third');
}, 0);

process.nextTick(function () {
    console.log('second');
});

console.log('first');
```

このスクリプトは次のような出力をします:

```
first
second
third
```

ご覧のように`process.nextTick(fn)`は`setTimeout(fn, 0)`に似ていますが、それよりも速く動いています。
それというのも`setTimeout`は互換性の理由から意図的に遅く動くようにJavaScriptエンジンにより制御されているからです。

## [global](http://nodejs.org/docs/latest/api/all.html#all_global)

Node.jsにおいて、`global`はトップレベル・スコープであり、
ブラウザにおける`window`のようにグローバル変数が関連付けられるオブジェクトです。
browserifyにおいては、`global`はまさしく`window`のエイリアスに他なりません。

## [__filename](http://nodejs.org/docs/latest/api/all.html#all_filename)

`__filename`は現在のファイルのパスです。各ファイルでその内容は変わってきます。

システムのパス情報が公開されてしまうのを防ぐため、このパスのルートは
`browserify()`に渡されたオプション情報の`opts.basedir`の項目で規定されたものになります。
そしてこの項目の既定値は[カレントディレクトリ](https://en.wikipedia.org/wiki/Current_working_directory)です。

仮に`main.js`というファイルがあったとします:

``` js
var bar = require('./foo/bar.js');

console.log('here in main.js, __filename is:', __filename);
bar();
```

そして`foo/bar.js`というファイルも:

``` js
module.exports = function () {
    console.log('here in foo/bar.js, __filename is:', __filename);
};
```

`main.js`と同じディレクトリでbrowserifyを実行すると次のような出力が得られます:

```
$ browserify main.js | node
here in main.js, __filename is: /main.js
here in foo/bar.js, __filename is: /foo/bar.js
```

## [__dirname](http://nodejs.org/docs/latest/api/all.html#all_dirname)

`__dirname`は現在のファイルのディレクトリのパスです。`__filename`と同じで、
`__dirname`のルートもまた`opts.basedir`により規定されます。

次に`__dirname`がどのようにはたらくかを示します:

main.js:

``` js
require('./x/y/z/abc.js');
console.log('in main.js __dirname=' + __dirname);
```

x/y/z/abc.js:

``` js
console.log('in abc.js, __dirname=' + __dirname);
```

出力は次のようになります:

```
$ browserify main.js | node
in abc.js, __dirname=/x/y/z
in main.js __dirname=/
```

# トランスフォーム

browserifyは何でもかんでもをサポートするのではなく、
柔軟なデータ変換（トランスフォーム）システムを採用してソースファイルを変換する道を選びました。

結果として、CoffeeScriptやテンプレート、その他何であれJavaScriptにコンパイルができるようになりました。

例えば[CoffeeScript](http://coffeescript.org/)をコンパイルする場合、
[coffeeify](https://www.npmjs.org/package/coffeeify)モジュールが提供するトランスフォームを利用します。
`npm install coffeeify`によりモジュールをインストールした上で、次のようにコマンドを実行します:

```
$ browserify -t coffeeify main.coffee > bundle.js
```

もしくはbrowserifyのAPIを利用して次のようにすることもできます:

```
var b = browserify('main.coffee');
b.transform('coffeeify');
```

トランスフォームのすばらしいところは、
上述の例で`--debug`もしくは`opts.debug`でソースマップ生成を有効化している場合、
bundle.jsはCoffeeScriptで記述されたオリジナルのソースファイルにマッピングされるということです。
FirebugやChromeのインスペクタを利用してアプリケーションをデバッグする際、これはとても便利でしょう。

## トランスフォームを実装する

トランスフォームはシンプルなストリーム操作を実装するだけで作成できます。
次に示すのは`$CWD`を`process.cwd()`の返す値で置き換えるトランスフォームです:

``` js
var through = require('through2');

module.exports = function (file) {
    return through(function (buf, enc, next) {
        this.push(buf.toString('utf8').replace(/\$CWD/g, process.cwd()));
        next();
    });
};
```

トランスフォーム関数は現在のパッケージのファイルの1つ1つについて実行され、
都度トランスフォーム・ストリームを生成して返します。
browserifyはストリームに対してオリジナルのファイルの内容を書き込み、
続いて同じストリームから変換後のファイルの内容を読み取るのです。

実装したトランスフォームは単にファイルとして保存するかnpmパッケージ化した上で、
コマンドラインで`-t ./your_transform.js`のように指定して使用します。

Node.jsのStreamがどのように機能するのか、より詳しい情報については
[stream handbook](https://github.com/substack/stream-handbook)（[邦訳](https://github.com/meso/stream-handbook)）を参照してください。

# package.json

## browserフィールド

package.jsonの`"browser"`フィールドを定義することで、browserifyに対して
`"main"`フィールドの内容を上書きするようパッケージ個別に設定ができます。

例えば、パッケージのNode.jsランタイム向けのエントリーポイントは`main.js`で提供されており、
一方ブラウザ向けのそれが `browser.js`で提供されている場合、次のようにすることができます:

``` json
{
  "name": "mypkg",
  "version": "1.2.3",
  "main": "main.js",
  "browser": "browser.js"
}
```

このようにしたとき、
Node.jsランタイムで`require('mypkg')`を実行すると`main.js`がエクスポートするオブジェクトが得られますが、
ブラウザのランタイムで`require('mypkg')`を実行すると`browser.js`がエクスポートするオブジェクトが得られます。

Node.jsで実行するときとブラウザで実行するときとで読み込みたいモジュールは異なってくるのですから、
`"browser"`フィールドを指定することでバンドルする対象コードを切り替えるこの手法は、
実行時にロジカルに判断する方法よりもよほど好ましいものです。
Node.jsとブラウザ双方のランタイムのための`require()`呼び出しが同じファイルにある場合、
browserifyは静的解析の結果に従いそれが必要なら両方の依存性を、
さもなくばいずれか必要な方の依存性だけをバンドル化の対象に含めます。

ところで、"browser"フィールドには文字列の代わりにオブジェクトを指定することもできます。

例えば、エントリーポイントではなくパッケージを構成するファイルのうち`lib/`配下の1ファイルだけを
ブラウザ向けのバージョンに差し替えたいという場合、次のようにすることができます:

``` json
{
  "name": "mypkg",
  "version": "1.2.3",
  "main": "main.js",
  "browser": {
    "lib/foo.js": "lib/browser-foo.js"
  }
}
```

あるいはまた、パッケージ内で使用している依存性モジュールを別のものに置き換えたい場合、次のようにすることができます:

``` json
{
  "name": "mypkg",
  "version": "1.2.3",
  "main": "main.js",
  "browser": {
    "fs": "level-fs-browser"
  }
}
```

"browser"フィールドのオブジェクトのプロパティに`false`を指定することで、
当該モジュールを無視する（本来のモジュールがエクスポートするオブジェクトの代わりに空のオブジェクトが使用される）
ようにさせることもできます:

``` json
{
  "name": "mypkg",
  "version": "1.2.3",
  "main": "main.js",
  "browser": {
    "winston": false
  }
}
```

"browser"フィールドは現在のパッケージに *のみ* 適用されます。
例えばあなたがいかなるマッピング情報を指定したとしても、当該パッケージが依存する先の他のパッケージや、
当該パッケージに依存する他のパッケージに対して、推移的に影響を及ぼすことはありません。
この分離があるおかげであなたのモジュールを他のモジュール群における設定から守られます。
そのためあなたが`require()`を使用して依存性モジュールを読み込んだからといって、
当該の依存性モジュールからシステム全般に渡る影響が生じるようなことは心配せずに済みます。
同様に、あなたのパッケージのローカルな構成変更が、
モジュールの依存性グラフの果の果てまで波及してしまうようなことも心配せずに住みます。

## browserify.transformフィールド

`browserify.transform`フィールドを定義することで、
パッケージ内のモジュールが読み込まれるとき、トランスフォームが自動的に適用されるよう指定することができます。
例えば、[brfs](https://npmjs.org/package/brfs)トランスフォームを自動適用したい場合は、
package.jsonに次のように記述します:

``` json
{
  "name": "mypkg",
  "version": "1.2.3",
  "main": "main.js",
  "browserify": {
    "transform": [ "brfs" ]
  }
}
```

さて、`main.js`は次のようにします:

``` js
var fs = require('fs');
var src = fs.readFileSync(__dirname + '/foo.txt', 'utf8');

module.exports = function (x) { return src.replace(x, 'zzz') };
```

すると、`fs.readFileSync()`呼び出しはコンパイル時にbrfsによりインライン化されます。
このモジュールを使用する側のコードはこの変化を知る必要がありません。
必要ならトランスフォームはいくらでも指定できます。それらは配列で指定された順番で実行されてゆきます。

`"browser"`フィールド同様、package.jsonにおけるトランスフォームの設定は
当該のパッケージのローカルのコードにしか適用されません。理由についても同様です。

### トランスフォームの構成変更

トランスフォームによってはコマンドラインから構成変更のためのオプションを指定できることがあります。
またそれらのオプションをpackage.jsonから指定することもできます。

**コマンドラインで指定する**
```
browserify -t coffeeify \
           -t [ browserify-ngannotate --ext .coffee --bar ] \
           index.coffee > index.js
```

**package.jsonで指定する**
``` json
"browserify": {
  "transform": [
    "coffeeify",
    ["browserify-ngannotate", {"ext": ".coffee", "bar": true}]
  ]
}
```


# finding good modules

Here are [some useful heuristics](http://substack.net/finding_modules)
for finding good modules on npm that work in the browser:

* I can install it with npm

* code snippet on the readme using require() - from a quick glance I should see
how to integrate the library into what I'm presently working on

* has a very clear, narrow idea about scope and purpose

* knows when to delegate to other libraries - doesn't try to do too many things itself

* written or maintained by authors whose opinions about software scope,
modularity, and interfaces I generally agree with (often a faster shortcut
than reading the code/docs very closely)

* inspecting which modules depend on the library I'm evaluating - this is baked
into the package page for modules published to npm

Other metrics like number of stars on github, project activity, or a slick
landing page, are not as reliable.

## module philosophy

People used to think that exporting a bunch of handy utility-style things would
be the main way that programmers would consume code because that is the primary
way of exporting and importing code on most other platforms and indeed still
persists even on npm.

However, this
[kitchen-sink mentality](https://github.com/substack/node-mkdirp/issues/17)
toward including a bunch of thematically-related but separable functionality
into a single package appears to be an artifact for the difficulty of
publishing and discovery in a pre-github, pre-npm era.

There are two other big problems with modules that try to export a bunch of
functionality all in one place under the auspices of convenience: demarcation
turf wars and finding which modules do what.

Packages that are grab-bags of features
[waste a ton of time policing boundaries](https://github.com/jashkenas/underscore/search?q=%22special-case%22&ref=cmdform&type=Issues)
about which new features belong and don't belong.
There is no clear natural boundary of the problem domain in this kind of package
about what the scope is, it's all
[somebody's smug opinion](http://david.heinemeierhansson.com/2012/rails-is-omakase.html).

Node, npm, and browserify are not that. They are avowedly ala-carte,
participatory, and would rather celebrate disagreement and the dizzying
proliferation of new ideas and approaches than try to clamp down in the name of
conformity, standards, or "best practices".

Nobody who needs to do gaussian blur ever thinks "hmm I guess I'll start checking
generic mathematics, statistics, image processing, and utility libraries to see
which one has gaussian blur in it. Was it stats2 or image-pack-utils or
maths-extra or maybe underscore has that one?"
No. None of this. Stop it. They `npm search gaussian` and they immediately see
[ndarray-gaussian-filter](https://npmjs.org/package/ndarray-gaussian-filter) and
it does exactly what they want and then they continue on with their actual
problem instead of getting lost in the weeds of somebody's neglected grand
utility fiefdom.

# organizing modules

## avoiding ../../../../../../..

Not everything in an application properly belongs on the public npm and the
overhead of setting up a private npm or git repo is still rather large in many
cases. Here are some approaches for avoiding the `../../../../../../../`
relative paths problem.

### symlink

The simplest thing you can do is to symlink your app root directory into your
node_modules/ directory.

Did you know that [symlinks work on windows
too](http://www.howtogeek.com/howto/windows-vista/using-symlinks-in-windows-vista/)?

To link a `lib/` directory in your project root into `node_modules`, do:

```
ln -s ../lib node_modules/app
```

and now from anywhere in your project you'll be able to require files in `lib/`
by doing `require('app/foo.js')` to get `lib/foo.js`.

### node_modules

People sometimes object to putting application-specific modules into
node_modules because it is not obvious how to check in your internal modules
without also checking in third-party modules from npm.

The answer is quite simple! If you have a `.gitignore` file that ignores
`node_modules`:

```
node_modules
```

You can just add an exception with `!` for each of your internal application
modules:

```
node_modules/*
!node_modules/foo
!node_modules/bar
```

Please note that you can't *unignore* a subdirectory,
if the parent is already ignored. So instead of ignoring `node_modules`,
you have to ignore every directory *inside* `node_modules` with the
`node_modules/*` trick, and then you can add your exceptions.

Now anywhere in your application you will be able to `require('foo')` or
`require('bar')` without having a very large and fragile relative path.

If you have a lot of modules and want to keep them more separate from the
third-party modules installed by npm, you can just put them all under a
directory in `node_modules` such as `node_modules/app`:

```
node_modules/app/foo
node_modules/app/bar
```

Now you will be able to `require('app/foo')` or `require('app/bar')` from
anywhere in your application.

In your `.gitignore`, just add an exception for `node_modules/app`:

```
node_modules/*
!node_modules/app
```

If your application had transforms configured in package.json, you'll need to
create a separate package.json with its own transform field in your
`node_modules/foo` or `node_modules/app/foo` component directory because
transforms don't apply across module boundaries. This will make your modules
more robust against configuration changes in your application and it will be
easier to independently reuse the packages outside of your application.

### custom paths

You might see some places talk about using the `$NODE_PATH` environment variable
or `opts.paths` to add directories for node and browserify to look in to find
modules.

Unlike most other platforms, using a shell-style array of path directories with
`$NODE_PATH` is not as favorable in node compared to making effective use of the
`node_modules` directory.

This is because your application is more tightly coupled to a runtime
environment configuration so there are more moving parts and your application
will only work when your environment is setup correctly.

node and browserify both support but discourage the use of `$NODE_PATH`.

## non-javascript assets

There are many
[browserify transforms](https://github.com/substack/node-browserify/wiki/list-of-transforms)
you can use to do many things. Commonly, transforms are used to include
non-javascript assets into bundle files.

### brfs

One way of including any kind of asset that works in both node and the browser
is brfs.

brfs uses static analysis to compile the results of `fs.readFile()` and
`fs.readFileSync()` calls down to source contents at compile time.

For example, this `main.js`:

``` js
var fs = require('fs');
var html = fs.readFileSync(__dirname + '/robot.html', 'utf8');
console.log(html);
```

applied through brfs would become something like:

``` js
var fs = require('fs');
var html = "<b>beep boop</b>";
console.log(html);
```

when run through brfs.

This is handy because you can reuse the exact same code in node and the browser,
which makes sharing modules and testing much simpler.

`fs.readFile()` and `fs.readFileSync()` accept the same arguments as in node,
which makes including inline image assets as base64-encoded strings very easy:

``` js
var fs = require('fs');
var imdata = fs.readFileSync(__dirname + '/image.png', 'base64');
var img = document.createElement('img');
img.setAttribute('src', 'data:image/png;base64,' + imdata);
document.body.appendChild(img);
```

If you have some css you want to inline into your bundle, you can do that too
with the assistence of a module such as
[insert-css](https://npmjs.org/package/insert-css):

``` js
var fs = require('fs');
var insertStyle = require('insert-css');

var css = fs.readFileSync(__dirname + '/style.css', 'utf8');
insertStyle(css);
```

Inserting css this way works fine for small reusable modules that you distribute
with npm because they are fully-contained, but if you want a more wholistic
approach to asset management using browserify, check out
[atomify](https://www.npmjs.org/package/atomify) and
[parcelify](https://www.npmjs.org/package/parcelify).

### hbsify

### jadeify

### reactify

## reusable components

Putting these ideas about code organization together, we can build a reusable UI
component that we can reuse across our application or in other applications.

Here is a bare-bones example of an empty widget module:

``` js
module.exports = Widget;

function Widget (opts) {
    if (!(this instanceof Widget)) return new Widget(opts);
    this.element = document.createElement('div');
}

Widget.prototype.appendTo = function (target) {
    if (typeof target === 'string') target = document.querySelector(target);
    target.appendChild(this.element);
};
```

Handy javascript constructor tip: you can include a `this instanceof Widget`
check like above to let people consume your module with `new Widget` or
`Widget()`. It's nice because it hides an implementation detail from your API
and you still get the performance benefits and indentation wins of using
prototypes.

To use this widget, just use `require()` to load the widget file, instantiate
it, and then call `.appendTo()` with a css selector string or a dom element.

Like this:

``` js
var Widget = require('./widget.js');
var w = Widget();
w.appendTo('#container');
```

and now your widget will be appended to the DOM.

Creating HTML elements procedurally is fine for very simple content but gets
very verbose and unclear for anything bigger. Luckily there are many transforms
available to ease importing HTML into your javascript modules.

Let's extend our widget example using [brfs](https://npmjs.org/package/brfs). We
can also use [domify](https://npmjs.org/package/domify) to turn the string that
`fs.readFileSync()` returns into an html dom element:

``` js
var fs = require('fs');
var domify = require('domify');

var html = fs.readFileSync(__dirname + '/widget.html', 'utf8');

module.exports = Widget;

function Widget (opts) {
    if (!(this instanceof Widget)) return new Widget(opts);
    this.element = domify(html);
}

Widget.prototype.appendTo = function (target) {
    if (typeof target === 'string') target = document.querySelector(target);
    target.appendChild(this.element);
};
```

and now our widget will load a `widget.html`, so let's make one:

``` html
<div class="widget">
  <h1 class="name"></h1>
  <div class="msg"></div>
</div>
```

It's often useful to emit events. Here's how we can emit events using the
built-in `events` module and the [inherits](https://npmjs.org/package/inherits)
module:

``` js
var fs = require('fs');
var domify = require('domify');
var inherits = require('inherits');
var EventEmitter = require('events').EventEmitter;

var html = fs.readFileSync(__dirname + '/widget.html', 'utf8');

inherits(Widget, EventEmitter);
module.exports = Widget;

function Widget (opts) {
    if (!(this instanceof Widget)) return new Widget(opts);
    this.element = domify(html);
}

Widget.prototype.appendTo = function (target) {
    if (typeof target === 'string') target = document.querySelector(target);
    target.appendChild(this.element);
    this.emit('append', target);
};
```

Now we can listen for `'append'` events on our widget instance:

``` js
var Widget = require('./widget.js');
var w = Widget();
w.on('append', function (target) {
    console.log('appended to: ' + target.outerHTML);
});
w.appendTo('#container');
```

We can add more methods to our widget to set elements on the html:

``` js
var fs = require('fs');
var domify = require('domify');
var inherits = require('inherits');
var EventEmitter = require('events').EventEmitter;

var html = fs.readFileSync(__dirname + '/widget.html', 'utf8');

inherits(Widget, EventEmitter);
module.exports = Widget;

function Widget (opts) {
    if (!(this instanceof Widget)) return new Widget(opts);
    this.element = domify(html);
}

Widget.prototype.appendTo = function (target) {
    if (typeof target === 'string') target = document.querySelector(target);
    target.appendChild(this.element);
};

Widget.prototype.setName = function (name) {
    this.element.querySelector('.name').textContent = name;
}

Widget.prototype.setMessage = function (msg) {
    this.element.querySelector('.msg').textContent = msg;
}
```

If setting element attributes and content gets too verbose, check out
[hyperglue](https://npmjs.org/package/hyperglue).

Now finally, we can toss our `widget.js` and `widget.html` into
`node_modules/app-widget`. Since our widget uses the
[brfs](https://npmjs.org/package/brfs) transform, we can create a `package.json`
with:

``` json
{
  "name": "app-widget",
  "version": "1.0.0",
  "private": true,
  "main": "widget.js",
  "browserify": {
    "transform": [ "brfs" ]
  },
  "dependencies": {
    "brfs": "^1.1.1",
    "inherits": "^2.0.1"
  }
}
```

And now whenever we `require('app-widget')` from anywhere in our application,
brfs will be applied to our `widget.js` automatically!
Our widget can even maintain its own dependencies. This way we can update
dependencies in one widgets without worrying about breaking changes cascading
over into other widgets.

Make sure to add an exclusion in your `.gitignore` for
`node_modules/app-widget`:

```
node_modules/*
!node_modules/app-widget
```

You can read more about [shared rendering in node and the
browser](http://substack.net/shared_rendering_in_node_and_the_browser) if you
want to learn about sharing rendering logic between node and the browser using
browserify and some streaming html libraries.

# testing in node and the browser

Testing modular code is very easy! One of the biggest benefits of modularity is
that your interfaces become much easier to instantiate in isolation and so it's
easy to make automated tests.

Unfortunately, few testing libraries play nicely out of the box with modules and
tend to roll their own idiosyncratic interfaces with implicit globals and obtuse
flow control that get in the way of a clean design with good separation.

People also make a huge fuss about "mocking" but it's usually not necessary if
you design your modules with testing in mind. Keeping IO separate from your
algorithms, carefully restricting the scope of your module, and accepting
callback parameters for different interfaces can all make your code much easier
to test.

For example, if you have a library that does both IO and speaks a protocol,
[consider separating the IO layer from the
protocol](https://www.youtube.com/watch?v=g5ewQEuXjsQ#t=12m30)
using an interface like [streams](https://github.com/substack/stream-handbook).

Your code will be easier to test and reusable in different contexts that you
didn't initially envision. This is a recurring theme of testing: if your code is
hard to test, it is probably not modular enough or contains the wrong balance of
abstractions. Testing should not be an afterthought, it should inform your
whole design and it will help you to write better interfaces.

## testing libraries

### [tape](https://npmjs.org/package/tape)

Tape was specifically designed from the start to work well in both node and
browserify. Suppose we have an `index.js` with an async interface:

``` js
module.exports = function (x, cb) {
    setTimeout(function () {
        cb(x * 100);
    }, 1000);
};
```

Here's how we can test this module using [tape](https://npmjs.org/package/tape).
Let's put this file in `test/beep.js`:

``` js
var test = require('tape');
var hundreder = require('../');

test('beep', function (t) {
    t.plan(1);

    hundreder(5, function (n) {
        t.equal(n, 500, '5*100 === 500');
    });
});
```

Because the test file lives in `test/`, we can require the `index.js` in the
parent directory by doing `require('../')`. `index.js` is the default place that
node and browserify look for a module if there is no package.json in that
directory with a `main` field.

We can `require()` tape like any other library after it has been installed with
`npm install tape`.

The string `'beep'` is an optional name for the test.
The 3rd argument to `t.equal()` is a completely optional description.

The `t.plan(1)` says that we expect 1 assertion. If there are not enough
assertions or too many, the test will fail. An assertion is a comparison
like `t.equal()`. tape has assertion primitives for:

* t.equal(a, b) - compare a and b strictly with `===`
* t.deepEqual(a, b) - compare a and b recursively
* t.ok(x) - fail if `x` is not truthy

and more! You can always add an additional description argument.

Running our module is very simple! To run the module in node, just run
`node test/beep.js`:

```
$ node test/beep.js
TAP version 13
# beep
ok 1 5*100 === 500

1..1
# tests 1
# pass  1

# ok
```

The output is printed to stdout and the exit code is 0.

To run our code in the browser, just do:

```
$ browserify test/beep.js > bundle.js
```

then plop `bundle.js` into a `<script>` tag:

```
<script src="bundle.js"></script>
```

and load that html in a browser. The output will be in the debug console which
you can open with F12, ctrl-shift-j, or ctrl-shift-k depending on the browser.

This is a bit cumbersome to run our tests in a browser, but you can install the
`testling` command to help. First do:

```
npm install -g testling
```

And now just do `browserify test/beep.js | testling`:

```
$ browserify test/beep.js | testling

TAP version 13
# beep
ok 1 5*100 === 500

1..1
# tests 1
# pass  1

# ok
```

`testling` will launch a real browser headlessly on your system to run the tests.

Now suppose we want to add another file, `test/boop.js`:

``` js
var test = require('tape');
var hundreder = require('../');

test('fraction', function (t) {
    t.plan(1);

    hundreder(1/20, function (n) {
        t.equal(n, 5, '1/20th of 100');
    });
});

test('negative', function (t) {
    t.plan(1);

    hundreder(-3, function (n) {
        t.equal(n, -300, 'negative number');
    });
});
```

Here our test has 2 `test()` blocks. The second test block won't start to
execute until the first is completely finished, even though it is asynchronous.
You can even nest test blocks by using `t.test()`.

We can run `test/boop.js` with node directly as with `test/beep.js`, but if we
want to run both tests, there is a minimal command-runner we can use that comes
with tape. To get the `tape` command do:

```
npm install -g tape
```

and now you can run:

```
$ tape test/*.js
TAP version 13
# beep
ok 1 5*100 === 500
# fraction
ok 2 1/20th of 100
# negative
ok 3 negative number

1..3
# tests 3
# pass  3

# ok
```

and you can just pass `test/*.js` to browserify to run your tests in the
browser:

```
$ browserify test/* | testling

TAP version 13
# beep
ok 1 5*100 === 500
# fraction
ok 2 1/20th of 100
# negative
ok 3 negative number

1..3
# tests 3
# pass  3

# ok
```

Putting together all these steps, we can configure `package.json` with a test
script:

``` json
{
  "name": "hundreder",
  "version": "1.0.0",
  "main": "index.js",
  "devDependencies": {
    "tape": "^2.13.1",
    "testling": "^1.6.1"
  },
  "scripts": {
    "test": "tape test/*.js",
    "test-browser": "browserify test/*.js | testlingify"
  }
}
```

Now you can do `npm test` to run the tests in node and `npm run test-browser` to
run the tests in the browser. You don't need to worry about installing commands
with `-g` when you use `npm run`: npm automatically sets up the `$PATH` for all
packages installed locally to the project.

If you have some tests that only run in node and some tests that only run in the
browser, you could have subdirectories in `test/` such as `test/server` and
`test/browser` with the tests that run both places just in `test/`. Then you
could just add the relevant directory to the globs:

``` json
{
  "name": "hundreder",
  "version": "1.0.0",
  "main": "index.js",
  "devDependencies": {
    "tape": "^2.13.1",
    "testling": "^1.6.1"
  },
  "scripts": {
    "test": "tape test/*.js test/server/*.js",
    "test-browser": "browserify test/*.js test/browser/*.js | testling"
  }
}
```

and now server-specific and browser-specific tests will be run in addition to
the common tests.

If you want something even slicker, check out
[prova](https://www.npmjs.org/package/prova) once you have gotten the basic
concepts.

### assert

The core assert module is a fine way to write simple tests too, although it can
sometimes be tricky to ensure that the correct number of callbacks have fired.

You can solve that problem with tools like
[macgyver](https://www.npmjs.org/package/macgyver) but it is appropriately DIY.

## code coverage

### coverify

A simple way to check code coverage in browserify is to use the
[coverify](https://npmjs.org/package/coverify) transform.

```
$ browserify -t coverify test/*.js | node | coverify
```

or to run your tests in a real browser:

```
$ browserify -t coverify test/*.js | testling | coverify
```

coverify works by transforming the source of each package so that each
expression is wrapped in a `__coverageWrap()` function.

Each expression in the program gets a unique ID and the `__coverageWrap()`
function will print `COVERED $FILE $ID` the first time the expression is
executed.

Before the expressions run, coverify prints a `COVERAGE $FILE $NODES` message to
log the expression nodes across the entire file as character ranges.

Here's what the output of a full run looks like:

```
$ browserify -t coverify test/whatever.js | node
COVERAGE "/home/substack/projects/defined/test/whatever.js" [[14,28],[14,28],[0,29],[41,56],[41,56],[30,57],[95,104],[95,105],[126,146],[126,146],[115,147],[160,194],[160,194],[152,195],[200,217],[200,218],[76,220],[59,221],[59,222]]
COVERED "/home/substack/projects/defined/test/whatever.js" 2
COVERED "/home/substack/projects/defined/test/whatever.js" 1
COVERED "/home/substack/projects/defined/test/whatever.js" 0
COVERAGE "/home/substack/projects/defined/index.js" [[48,49],[55,71],[51,71],[73,76],[92,104],[92,118],[127,139],[120,140],[172,195],[172,196],[0,204],[0,205]]
COVERED "/home/substack/projects/defined/index.js" 11
COVERED "/home/substack/projects/defined/index.js" 10
COVERED "/home/substack/projects/defined/test/whatever.js" 5
COVERED "/home/substack/projects/defined/test/whatever.js" 4
COVERED "/home/substack/projects/defined/test/whatever.js" 3
COVERED "/home/substack/projects/defined/test/whatever.js" 18
COVERED "/home/substack/projects/defined/test/whatever.js" 17
COVERED "/home/substack/projects/defined/test/whatever.js" 16
TAP version 13
# whatever
COVERED "/home/substack/projects/defined/test/whatever.js" 7
COVERED "/home/substack/projects/defined/test/whatever.js" 6
COVERED "/home/substack/projects/defined/test/whatever.js" 10
COVERED "/home/substack/projects/defined/test/whatever.js" 9
COVERED "/home/substack/projects/defined/test/whatever.js" 8
COVERED "/home/substack/projects/defined/test/whatever.js" 13
COVERED "/home/substack/projects/defined/test/whatever.js" 12
COVERED "/home/substack/projects/defined/test/whatever.js" 11
COVERED "/home/substack/projects/defined/index.js" 0
COVERED "/home/substack/projects/defined/index.js" 2
COVERED "/home/substack/projects/defined/index.js" 1
COVERED "/home/substack/projects/defined/index.js" 5
COVERED "/home/substack/projects/defined/index.js" 4
COVERED "/home/substack/projects/defined/index.js" 3
COVERED "/home/substack/projects/defined/index.js" 7
COVERED "/home/substack/projects/defined/index.js" 6
COVERED "/home/substack/projects/defined/test/whatever.js" 15
COVERED "/home/substack/projects/defined/test/whatever.js" 14
ok 1 should be equal

1..1
# tests 1
# pass  1

# ok
```

These COVERED and COVERAGE statements are just printed on stdout and they can be
fed into the `coverify` command to generate prettier output:

```
$ browserify -t coverify test/whatever.js | node | coverify
TAP version 13
# whatever
ok 1 should be equal

1..1
# tests 1
# pass  1

# ok

# /home/substack/projects/defined/index.js: line 6, column 9-32

          console.log('whatever');
          ^^^^^^^^^^^^^^^^^^^^^^^^

# coverage: 30/31 (96.77 %)
```

To include code coverage into your project, you can add an entry into the
`package.json` scripts field:

``` json
{
  "scripts": {
    "test": "tape test/*.js",
    "coverage": "browserify -t coverify test/*.js | node | coverify"
  }
}
```

There is also a [covert](https://npmjs.com/package/covert) package that
simplifies the browserify and coverify setup:

``` json
{
  "scripts": {
    "test": "tape test/*.js",
    "coverage": "covert test/*.js"
  }
}
```

To install coverify or covert as a devDependency, run
`npm install -D coverify` or `npm install -D covert`.

# bundling

This section covers bundling in more detail.

Bundling is the step where starting from the entry files, all the source files
in the dependency graph are walked and packed into a single output file.

## saving bytes

One of the first things you'll want to tweak is how the files that npm installs
are placed on disk to avoid duplicates.

When you do a clean install in a directory, npm will ordinarily factor out
similar versions into the topmost directory where 2 modules share a dependency.
However, as you install more packages, new packages will not be factored out
automatically. You can however use the `npm dedupe` command to factor out
packages for an already-installed set of packages in `node_modules/`. You could
also remove `node_modules/` and install from scratch again if problems with
duplicates persist.

browserify will not include the same exact file twice, but compatible versions
may differ slightly. browserify is also not version-aware, it will include the
versions of packages exactly as they are laid out in `node_modules/` according
to the `require()` algorithm that node uses.

You can use the `browserify --list` and `browserify --deps` commands to further
inspect which files are being included to scan for duplicates.

## standalone

You can generate UMD bundles with `--standalone` that will work in node, the
browser with globals, and AMD environments.

Just add `--standalone NAME` to your bundle command:

```
$ browserify foo.js --standalone xyz > bundle.js
```

This command will export the contents of `foo.js` under the external module name
`xyz`. If a module system is detected in the host environment, it will be used.
Otherwise a window global named `xyz` will be exported.

You can use dot-syntax to specify a namespace hierarchy:

```
$ browserify foo.js --standalone foo.bar.baz > bundle.js
```

If there is already a `foo` or a `foo.bar` in the host environment in window
global mode, browserify will attach its exports onto those objects. The AMD and
`module.exports` modules will behave the same.

Note however that standalone only works with a single entry or directly-required
file.

## external bundles

## ignoring and excluding

In browserify parlance, "ignore" means: replace the definition of a module with
an empty object. "exclude" means: remove a module completely from a dependency graph.

Another way to achieve many of the same goals as ignore and exclude is the
"browser" field in package.json, which is covered elsewhere in this document.

### ignoring

Ignoring is an optimistic strategy designed to stub in an empty definition for
node-specific modules that are only used in some codepaths. For example, if a
module requires a library that only works in node but for a specific chunk of
the code:

``` js
var fs = require('fs');
var path = require('path');
var mkdirp = require('mkdirp');

exports.convert = convert;
function convert (src) {
    return src.replace(/beep/g, 'boop');
}

exports.write = function (src, dst, cb) {
    fs.readFile(src, function (err, src) {
        if (err) return cb(err);
        mkdirp(path.dirname(dst), function (err) {
            if (err) return cb(err);
            var out = convert(src);
            fs.writeFile(dst, out, cb);
        });
    });
};
```

browserify already "ignores" the `'fs'` module by returning an empty object, but
the `.write()` function here won't work in the browser without an extra step like
a static analysis transform or a runtime storage fs abstraction.

However, if we really want the `convert()` function but don't want to see
`mkdirp` in the final bundle, we can ignore mkdirp with `b.ignore('mkdirp')` or
`browserify --ignore mkdirp`. The code will still work in the browser if we
don't call `write()` because `require('mkdirp')` won't throw an exception, just
return an empty object.

Generally speaking it's not a good idea for modules that are primarily
algorithmic (parsers, formatters) to do IO themselves but these tricks can let
you use those modules in the browser anyway.

To ignore `foo` on the command-line do:

```
browserify --ignore foo
```

To ignore `foo` from the api with some bundle instance `b` do:

``` js
b.ignore('foo')
```

### excluding

Another related thing we might want is to completely remove a module from the
output so that `require('modulename')` will fail at runtime. This is useful if
we want to split things up into multiple bundles that will defer in a cascade to
previously-defined `require()` definitions.

For example, if we have a vendored standalone bundle for jquery that we don't want to appear in
the primary bundle:

```
$ npm install jquery
$ browserify -r jquery --standalone jquery > jquery-bundle.js
```

then we want to just `require('jquery')` in a `main.js`:

``` js
var $ = require('jquery');
$(window).click(function () { document.body.bgColor = 'red' });
```

defering to the jquery dist bundle so that we can write:

``` html
<script src="jquery-bundle.js"></script>
<script src="bundle.js"></script>
```

and not have the jquery definition show up in `bundle.js`, then while compiling
the `main.js`, you can `--exclude jquery`:

```
browserify main.js --exclude jquery > bundle.js
```

To exclude `foo` on the command-line do:

```
browserify --exclude foo
```

To exclude `foo` from the api with some bundle instance `b` do:

``` js
b.exclude('foo')
```

## browserify cdn

# shimming

Unfortunately, some packages are not written with node-style commonjs exports.
For modules that export their functionality with globals or AMD, there are
packages that can help automatically convert these troublesome packages into
something that browserify can understand.

## browserify-shim

One way to automatically convert non-commonjs packages is with
[browserify-shim](https://npmjs.org/package/browserify-shim).

[browserify-shim](https://npmjs.org/package/browserify-shim) is loaded as a
transform and also reads a `"browserify-shim"` field from `package.json`.

Suppose we need to use a troublesome third-party library we've placed in
`./vendor/foo.js` that exports its functionality as a window global called
`FOO`. We can set up our `package.json` with:

``` json
{
  "browserify": {
    "transform": "browserify-shim"
  },
  "browserify-shim": {
    "./vendor/foo.js": "FOO"
  }
}
```

and now when we `require('./vendor/foo.js')`, we get the `FOO` variable that
`./vendor/foo.js` tried to put into the global scope, but that attempt was
shimmed away into an isolated context to prevent global pollution.

We could even use the [browser field](#browser-field) to make `require('foo')`
work instead of always needing to use a relative path to load `./vendor/foo.js`:

``` json
{
  "browser": {
    "foo": "./vendor/foo.js"
  },
  "browserify": {
    "transform": "browserify-shim"
  },
  "browserify-shim": {
    "foo": "FOO"
  }
}
```

Now `require('foo')` will return the `FOO` export that `./vendor/foo.js` tried
to place on the global scope.

# partitioning

Most of the time, the default method of bundling where one or more entry files
map to a single bundled output file is perfectly adequate, particularly
considering that bundling minimizes latency down to a single http request to
fetch all the javascript assets.

However, sometimes this initial penalty is too high for parts of a website that
are rarely or never used by most visitors such as an admin panel.
This partitioning can be accomplished with the technique covered in the
[ignoring and excluding](#ignoring-and-excluding) section, but factoring out
shared dependencies manually can be tedious for a large and fluid dependency
graph.

Luckily, there are plugins that can automatically factor browserify output into
separate bundle payloads.

## factor-bundle

[factor-bundle](https://www.npmjs.org/package/factor-bundle) splits browserify
output into multiple bundle targets based on entry-point. For each entry-point,
an entry-specific output file is built. Files that are needed by two or more of
the entry files get factored out into a common bundle.

For example, suppose we have 2 pages: /x and /y. Each page has an entry point,
`x.js` for /x and `y.js` for /y.

We then generate page-specific bundles `bundle/x.js` and `bundle/y.js` with
`bundle/common.js` containing the dependencies shared by both `x.js` and `y.js`:

```
browserify x.js y.js -p [ factor-bundle -o bundle/x.js -o bundle/y.js ] \
  -o bundle/common.js
```

Now we can simply put 2 script tags on each page. On /x we would put:

``` html
<script src="/bundle/common.js"></script>
<script src="/bundle/x.js"></script>
```

and on page /y we would put:

``` html
<script src="/bundle/common.js"></script>
<script src="/bundle/y.js"></script>
```

You could also load the bundles asynchronously with ajax or by inserting a
script tag into the page dynamically but factor-bundle only concerns itself with
generating the bundles, not with loading them.

## partition-bundle

[partition-bundle](https://www.npmjs.org/package/partition-bundle) handles
splitting output into multiple bundles like factor-bundle, but includes a
built-in loader using a special `loadjs()` function.

partition-bundle takes a json file that maps source files to bundle files:

```
{
  "entry.js": ["./a"],
  "common.js": ["./b"],
  "common/extra.js": ["./e", "./d"]
}
```

Then partition-bundle is loaded as a plugin and the mapping file, output
directory, and destination url path (required for dynamic loading) are passed
in:

```
browserify -p [ partition-bundle --map mapping.json \
  --output output/directory --url directory ]
```

Now you can add:

``` html
<script src="entry.js"></script>
```

to your page to load the entry file. From inside the entry file, you can
dynamically load other bundles with a `loadjs()` function:

``` js
a.addEventListener('click', function() {
  loadjs(['./e', './d'], function(e, d) {
    console.log(e, d);
  });
});
```

# compiler pipeline

Since version 5, browserify exposes its compiler pipeline as a
[labeled-stream-splicer](https://www.npmjs.org/package/labeled-stream-splicer).

This means that transformations can be added or removed directly into the
internal pipeline. This pipeline provides a clean interface for advanced
customizations such as watching files or factoring bundles from multiple entry
points.

For example, we could replace the built-in integer-based labeling mechanism with
hashed IDs by first injecting a pass-through transform after the "deps" have
been calculated to hash source files. Then we can use the hashes we captured to
create our own custom labeler, replacing the built-in "label" transform:

``` js
var browserify = require('browserify');
var through = require('through2');
var shasum = require('shasum');

var b = browserify('./main.js');

var hashes = {};
var hasher = through.obj(function (row, enc, next) {
    hashes[row.id] = shasum(row.source);
    this.push(row);
    next();
});
b.pipeline.get('deps').push(hasher);

var labeler = through.obj(function (row, enc, next) {
    row.id = hashes[row.id];

    Object.keys(row.deps).forEach(function (key) {
        row.deps[key] = hashes[row.deps[key]];
    });

    this.push(row);
    next();
});
b.pipeline.get('label').splice(0, 1, labeler);

b.bundle().pipe(process.stdout);
```

Now instead of getting integers for the IDs in the output format, we get file
hashes:

```
$ node bundle.js
(function e(t,n,r){function s(o,u){if(!n[o]){if(!t[o]){var a=typeof require=="function"&&require;if(!u&&a)return a(o,!0);if(i)return i(o,!0);var f=new Error("Cannot find module '"+o+"'");throw f.code="MODULE_NOT_FOUND",f}var l=n[o]={exports:{}};t[o][0].call(l.exports,function(e){var n=t[o][1][e];return s(n?n:e)},l,l.exports,e,t,n,r)}return n[o].exports}var i=typeof require=="function"&&require;for(var o=0;o<r.length;o++)s(r[o]);return s})({"5f0a0e3a143f2356582f58a70f385f4bde44f04b":[function(require,module,exports){
var foo = require('./foo.js');
var bar = require('./bar.js');

console.log(foo(3) + bar(4));

},{"./bar.js":"cba5983117ae1d6699d85fc4d54eb589d758f12b","./foo.js":"736100869ec2e44f7cfcf0dc6554b055e117c53c"}],"cba5983117ae1d6699d85fc4d54eb589d758f12b":[function(require,module,exports){
module.exports = function (n) { return n * 100 };

},{}],"736100869ec2e44f7cfcf0dc6554b055e117c53c":[function(require,module,exports){
module.exports = function (n) { return n + 1 };

},{}]},{},["5f0a0e3a143f2356582f58a70f385f4bde44f04b"]);
```

Note that the built-in labeler does other things like checking for the external,
excluded configurations so replacing it will be difficult if you depend on those
features. This example just serves as an example for the kinds of things you can
do by hacking into the compiler pipeline.

## build your own browserify

## labeled phases

Each phase in the browserify pipeline has a label that you can hook onto. Fetch
a label with `.get(name)` to return a
[labeled-stream-splicer](https://npmjs.org/package/labeled-stream-splicer)
handle at the appropriate label. Once you have a handle, you can `.push()`,
`.pop()`, `.shift()`, `.unshift()`, and `.splice()` your own transform streams
into the pipeline or remove existing transform streams.

### recorder

The recorder is used to capture the inputs sent to the `deps` phase so that they
can be replayed on subsequent calls to `.bundle()`. Unlike in previous releases,
v5 can generate bundle output multiple times. This is very handy for tools like
watchify that re-bundle when a file has changed.

### deps

The `deps` phase expects entry and `require()` files or objects as input and
calls [module-deps](https://npmjs.org/package/module-deps) to generate a stream
of json output for all of the files in the dependency graph.

module-deps is invoked with some customizations here such as:

* setting up the browserify transform key for package.json
* filtering out external, excluded, and ignored files
* setting the default extensions for `.js` and `.json` plus options configured
in the `opts.extensions` parameter in the browserify constructor
* configuring a global [insert-module-globals](#insert-module-globals)
transform to detect and implement `process`, `Buffer`, `global`, `__dirname`,
and `__filename`
* setting up the list of node builtins which are shimmed by browserify

### json

This transform adds `module.exports=` in front of files with a `.json`
extension.

### unbom

This transform removes byte order markers, which are sometimes used by windows
text editors to indicate the endianness of files. These markers are ignored by
node, so browserify ignores them for compatibility.

### syntax

This transform checks for syntax errors using the
[syntax-error](https://npmjs.org/package/syntax-error) package to give
informative syntax errors with line and column numbers.

### sort

This phase uses [deps-sort](https://www.npmjs.org/package/deps-sort) to sort
the rows written to it in order to make the bundles deterministic.

### dedupe

The transform at this phase uses dedupe information provided by
[deps-sort](https://www.npmjs.org/package/deps-sort) in the `sort` phase to
remove files that have duplicate contents.

### label

This phase converts file-based IDs which might expose system path information
and inflate the bundle size into integer-based IDs.

The `label` phase will also normalize path names based on the `opts.basedir` or
`process.cwd()` to avoid exposing system path information.

### emit-deps

This phase emits a `'dep'` event for each row after the `label` phase.

### debug

If `opts.debug` was given to the `browserify()` constructor, this phase will
transform input to add `sourceRoot` and `sourceFile` properties which are used
by [browser-pack](https://npmjs.org/package/browser-pack) in the `pack` phase.

### pack

This phase converts rows with `'id'` and `'source'` parameters as input (among
others) and generates the concatenated javascript bundle as output
using [browser-pack](https://npmjs.org/package/browser-pack).

### wrap

This is an empty phase at the end where you can easily tack on custom post
transformations without interfering with existing mechanics.

## browser-unpack

[browser-unpack](https://npmjs.org/package/browser-unpack) converts a compiled
bundle file back into a format very similar to the output of
[module-deps](https://npmjs.org/package/module-deps).

This is very handy if you need to inspect or transform a bundle that has already
been compiled.

For example:

``` js
$ browserify src/main.js | browser-unpack
[
{"id":1,"source":"module.exports = function (n) { return n * 100 };","deps":{}}
,
{"id":2,"source":"module.exports = function (n) { return n + 1 };","deps":{}}
,
{"id":3,"source":"var foo = require('./foo.js');\nvar bar = require('./bar.js');\n\nconsole.log(foo(3) + bar(4));","deps":{"./bar.js":1,"./foo.js":2},"entry":true}
]
```

This decomposition is needed by tools such as
[factor-bundle](https://www.npmjs.org/package/factor-bundle)
and [bundle-collapser](https://www.npmjs.org/package/bundle-collapser).

# plugins

When loaded, plugins have access to the browserify instance itself.

## using plugins

Plugins should be used sparingly and only in cases where a transform or global
transform is not powerful enough to perform the desired functionality.

You can load a plugin with `-p` on the command-line:

```
$ browserify main.js -p foo > bundle.js
```

would load a plugin called `foo`. `foo` is resolved with `require()`, so to load
a local file as a plugin, preface the path with a `./` and to load a plugin from
`node_modules/foo`, just do `-p foo`.

You can pass options to plugins with square brackets around the entire plugin
expression, including the plugin name as the first argument:

```
$ browserify one.js two.js \
  -p [ factor-bundle -o bundle/one.js -o bundle/two.js ] \
  > common.js
```

This command-line syntax is parsed by the
[subarg](https://npmjs.org/package/subarg) package.

To see a list of browserify plugins, browse npm for packages with the keyword
"browserify-plugin": http://npmjs.org/browse/keyword/browserify-plugin

## authoring plugins

To author a plugin, write a package that exports a single function that will
receive a bundle instance and options object as arguments:

``` js
// example plugin

module.exports = function (b, opts) {
  // ...
}
```

Plugins operate on the bundle instance `b` directly by listening for events or
splicing transforms into the pipeline. Plugins should not overwrite bundle
methods unless they have a very good reason.
