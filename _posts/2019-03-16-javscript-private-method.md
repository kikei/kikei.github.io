---
layout: post
title:  "JavaScriptでprivate methodを使う"
categories: javascript
---

### JavaScript の private 対応

JavaScript で private なクラスプロパティやクラスメソッドを定義する仕様の策定が進んでいる。

[tc39/proposal\-class\-fields: Orthogonally\-informed combination of public and private fields proposals](https://github.com/tc39/proposal-class-fields)

これまでの JavaScript では private の概念は無く、
クラス内部で (普通に) 定義したプロパティやメソッドは全て外部に公開されてしまっていた。
Web Inspector 等のコンソールで見ると、補完機能により全てが筒抜けになっていることがわかる。

今後は、新しい文法を JavaScript に追加することにより、 private なプロパティ、メソッドを定義できるようになる。
とても欲しかった機能だ。本記事でさっそく使ってみる。

### 1. 本記事の目的

private を駆使して格好良いコードを書いてみる。

また、そのままでは最先端の Web ブラウザでしか使えないため、Babel を使い従来の JavaScript コードに変換する。今やっているプロジェクトでも使いたいので Browserify から利用できるようにする。

### 2. コード例

#### すごく簡単な private プロパティ:

以下のコードでは `#count` が private プロパティになっている。
変数名の先頭に `#` を附加すると private 扱いになる。

この `#count` 変数は `Counter` クラス外からは直接参照することができない。
クラス内部からは、`this.#count` とすることで参照が可能である。

```js
class Counter {
    #count = 0;
    incr() {
        return ++this.#count;
    }
}
```

#### すごく簡単な private メソッド:

以下のコードでは `#fibonacci` メソッドは private であるから、クラス外部から実行することができない。クラス内部からは `this.#fibonacci(i)` とすることで実行可能である。

これもメソッド名の先頭に `#` を書くことで private 扱いになる。

```js
class Fibonacci {
    calc(i) {
        if (i < 0) throw new TypeError('i must be >= 0');
        return this.#fibonacci(i);
    }
    #fibonacci(i) {
        if (i == 0 || i == 1)
            return 1;
        else
            return this.#fibonacci(i - 1) + this.#fibonacci(i - 2);
    }
}
```

### 3. ビルド設定

Babel では `#` を使った private の文法にに対応したプラグインが既に実装されている。
このプラグインを使って従来の JavaScript にコンパイルすることができる。

[@babel/plugin\-proposal\-private\-methods](https://babeljs.io/docs/en/babel-plugin-proposal-private-methods)

コンパイルは次のコマンドで実行できるようにした。

```sh
$ npm run build
```

#### 3.1. package.json

Babel7 を利用し以下のように作成した。

src/main.js は Counter.js とか Fibonacci.js に書き換えて欲しい。

```json
{
  "name": "try-private-method",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "build": "browserify src/main.js -o index.js"
  },
  "browserify": {
    "transform": [
      [
        "babelify",
        {
          "plugins": [
            "@babel/plugin-proposal-class-properties",
            "@babel/plugin-proposal-private-methods"
          ],
          "presets": [
            "@babel/preset-env"
          ]
        }
      ]
    ]
  },
  "license": "MIT",
  "devDependencies": {
    "@babel/core": "^7.3.4",
    "@babel/plugin-proposal-class-properties": "^7.3.4",
    "@babel/plugin-proposal-private-methods": "^7.3.4",
    "@babel/preset-env": "^7.3.4",
    "babelify": "^10.0.0",
    "browserify": "^16.2.3"
  }
}
```

#### 3.2. プラグインのインストール

普通。

```sh
$ npm install
```

#### 3.3. コンパイル実行

事前の約束通りに実行する。

```sh
$ npm run build
```

### 4. コンパイル結果

#### 4.1. Counter.js

コンパイルの結果、以下のようになった。

```js
(function(){function r(e,n,t){function o(i,f){if(!n[i]){if(!e[i]){var c="function"==typeof require&&require;if(!f&&c)return c(i,!0);if(u)return u(i,!0);var a=new Error("Cannot find module '"+i+"'");throw a.code="MODULE_NOT_FOUND",a}var p=n[i]={exports:{}};e[i][0].call(p.exports,function(r){var n=e[i][1][r];return o(n||r)},p,p.exports,r,e,n,t)}return n[i].exports}for(var u="function"==typeof require&&require,i=0;i<t.length;i++)o(t[i]);return o}return r})()({1:[function(require,module,exports){"use strict"; Object.defineProperty(exports, "__esModule", {value: true});
exports.default = void 0;

function _classCallCheck(instance, Constructor) { if (!(instance instanceof Constructor)) { throw new TypeError("Cannot call a class as a function"); } }

function _defineProperties(target, props) { for (var i = 0; i < props.length; i++) { var descriptor = props[i]; descriptor.enumerable = descriptor.enumerable || false; descriptor.configurable = true; if ("value" in descriptor) descriptor.writable = true; Object.defineProperty(target, descriptor.key, descriptor); } }

function _createClass(Constructor, protoProps, staticProps) { if (protoProps) _defineProperties(Constructor.prototype, protoProps); if (staticProps) _defineProperties(Constructor, staticProps); return Constructor; }

function _classPrivateFieldSet(receiver, privateMap, value) { if (!privateMap.has(receiver)) { throw new TypeError("attempted to set private field on non-instance"); } var descriptor = privateMap.get(receiver); if (descriptor.set) { descriptor.set.call(receiver, value); } else { if (!descriptor.writable) { throw new TypeError("attempted to set read only private field"); } descriptor.value = value; } return value; }

function _classPrivateFieldGet(receiver, privateMap) { if (!privateMap.has(receiver)) { throw new TypeError("attempted to get private field on non-instance"); } var descriptor = privateMap.get(receiver); if (descriptor.get) { return descriptor.get.call(receiver); } return descriptor.value; }

var Counter =
/*#__PURE__*/
function () {
  function Counter() {
    _classCallCheck(this, Counter);

    _count.set(this, {
      writable: true,
      value: 0
    });
  }

  _createClass(Counter, [{
    key: "incr",
    value: function incr() {
      return _classPrivateFieldSet(this, _count, +_classPrivateFieldGet(this, _count) + 1);
    }
  }]);

  return Counter;
}();

exports.default = Counter;

var _count = new WeakMap();

},{}]},{},[1]);
```

private 変数は、Counter クラス実装の外側で定義される WeakMap として実現されたようだ。

#### 4.2. Fibonacci.js

Fibonacci.js はコンパイルの結果、以下のようになった。

```js
(function(){function r(e,n,t){function o(i,f){if(!n[i]){if(!e[i]){var c="function"==typeof require&&require;if(!f&&c)return c(i,!0);if(u)return u(i,!0);var a=new Error("Cannot find module '"+i+"'");throw a.code="MODULE_NOT_FOUND",a}var p=n[i]={exports:{}};e[i][0].call(p.exports,function(r){var n=e[i][1][r];return o(n||r)},p,p.exports,r,e,n,t)}return n[i].exports}for(var u="function"==typeof require&&require,i=0;i<t.length;i++)o(t[i]);return o}return r})()({1:[function(require,module,exports){
"use strict";

Object.defineProperty(exports, "__esModule", {
  value: true
});
exports.default = void 0;

function _classCallCheck(instance, Constructor) { if (!(instance instanceof Constructor)) { throw new TypeError("Cannot call a class as a function"); } }

function _defineProperties(target, props) { for (var i = 0; i < props.length; i++) { var descriptor = props[i]; descriptor.enumerable = descriptor.enumerable || false; descriptor.configurable = true; if ("value" in descriptor) descriptor.writable = true; Object.defineProperty(target, descriptor.key, descriptor); } }

function _createClass(Constructor, protoProps, staticProps) { if (protoProps) _defineProperties(Constructor.prototype, protoProps); if (staticProps) _defineProperties(Constructor, staticProps); return Constructor; }

function _classPrivateMethodGet(receiver, privateSet, fn) { if (!privateSet.has(receiver)) { throw new TypeError("attempted to get private field on non-instance"); } return fn; }

var Fibonacci =
/*#__PURE__*/
function () {
  function Fibonacci() {
    _classCallCheck(this, Fibonacci);

    _fibonacci.add(this);
  }

  _createClass(Fibonacci, [{
    key: "calc",
    value: function calc(i) {
      if (i < 0) throw new TypeError('i must be >= 0');
      return _classPrivateMethodGet(this, _fibonacci, _fibonacci2).call(this, i);
    }
  }]);

  return Fibonacci;
}();

exports.default = Fibonacci;

var _fibonacci = new WeakSet();

var _fibonacci2 = function _fibonacci2(i) {
  if (i == 0 || i == 1) return 1;else return _classPrivateMethodGet(this, _fibonacci, _fibonacci2).call(this, i - 1) + _classPrivateMethodGet(this, _fibonacci, _fibonacci2).call(this, i - 2);
};

},{}]},{},[1]);
```

今度は WeakSet を使って実現されたり、参照方法が少し複雑になったりしているが、
実質的には `_fibonacci2.call(this, i)` を実行しているだけである。
WeakSet を使っているのは、呼び出し元がちゃんとメソッドが定義されたクラスであることを
動的に確認するためだけのようだ。

### 5. ソースコード

GitHub にも同じような趣旨のコードを置いた。

[kikei/javascript\-try\-private\-method](https://github.com/kikei/javascript-try-private-method)

### 6. おまけ Emacs js2-mode の改造

2019/3/23 現在時点で Emacs の [js2\-mode](https://github.com/mooz/js2-mode) は
`#` を使った文法に対応していない。
そのため、メソッド名、プロパティ名の先頭に `#` を書くと文法エラーになってしまう。
こうなると、コードの一部を赤字にされたり、インデントがうまくいかなくなる等、デメリットが大きい。

しっかりとした対応にはもう少し時間がかかりそうだが、暫定対処としてエラーを抑制する方法を見つけたので紹介しておく。

js2-mode.el を以下のように1行書き換えると、とりあえずエラーにはならなくなる:

```lisp
  (defun js2-identifier-start-p (c)
    "Is C a valid start to an ES5 Identifier?
  See http://es5.github.io/#x7.6"
    (or
<    (memq c '(?$ ?_))
>    (memq c '(?# ?$ ?_))
     (memq (get-char-code-property c 'general-category)
           ;; Letters
           '(Lu Ll Lt Lm Lo Nl))))
```

js2-mode.el のファイルは、私の Spacemacs 環境下では以下のパスにあった。

> .emacs.d/elpa/26.1/develop/js2-mode-20190307.1649/js2-mode.el

この修正では全ての変数名について先頭に `#` を許容するようになってしまうため、
あくまで暫定的な対応となる。例えば関数の引数名の先頭に `#` を附加しても怒ってもらえない。

### 7. 参考資料

private 対応について:

- [tc39/proposal\-class\-fields: Orthogonally\-informed combination of public and private fields proposals](https://github.com/tc39/proposal-class-fields)
- [Private Class Field の導入に伴う JS の構文拡張 \| blog\.jxck\.io](https://blog.jxck.io/entries/2019-03-14/private-class-field.html)
- [WeakMap \- JavaScript \| MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/WeakMap)

Babel:

- [7\.2\.0 Released: Private Instance Methods · Babel](https://babeljs.io/blog/2018/12/03/7.2.0)
- [@babel/plugin\-proposal\-private\-methods](https://babeljs.io/docs/en/babel-plugin-proposal-private-methods)

Emacs js2-mode:

- [mooz/js2\-mode: Improved JavaScript editing mode for GNU Emacs](https://github.com/mooz/js2-mode)

