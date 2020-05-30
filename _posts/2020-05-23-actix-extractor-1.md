---
layout: post
title:  "Actix-web type-safe extractionを理解する その1"
categories: rust
---

Actix-web はプログラミング言語 Rust で開発された Web フレームワークである。
近年の Rust の Web フレームワーク界隈では最も開発が盛んだと言われている。
Actix Team によってオープンソースソフトウェアとして開発されており、
GitHub を見ると現在は18人のメンバーが参加しているようだ。

Actix と Actix-web の用語の使い分けとしては、Actix-web の上位プロジェクトが Actix であり、Actix-web は Actix の Actor システムをベースにして実装されている。
この記事で扱うのは Actix-web だけである。

Actix-web は HTTP 1, HTTP 2, Websocket, SSL/TLS といった基本的な Web サーバーの機能を持つ他、データベース接続や非同期処理もサポートしている。

[Rust's powerful actor system and most fun web framework](https://actix.rs/)

Actix-web で Web アプリケーションを実装するとき、はじめに URL と関数のマッピングテーブルを作成する。これは大体どのプログラミング言語の Web フレームワークでも一般的な設計である。ルーティングテーブルとも呼ばれる。

その中で Actix-web には Type-safe information extraction という機能があり、
不思議なことに、URL とマッピングされる関数の型がバラバラでも記述でき、
実際、直感通りに動いてしまう。
さらに型どころか、引数の数が違ったり、順番が違っても問題無い。

Rust は静的型付き言語で、関数オーバーロードやデフォルト引数の機能も無いため、
本来であれば同じ関数に渡す引数の型は全て綺麗に揃っていなければならないはず。

[Extractors](https://actix.rs/docs/extractors/)

一体 Actix-web の内部では何が起きているのだろうか。
気になりすぎて冷静に Actix-web を使うことができないので今回はこの謎に迫った。

これからその仕組みについて解説していこうと思う。

なお、この記事は3部に分割して掲載する。

3部に分けることに特に意味は無いが、生活の合間の時間に書いていることもあり、
記事が長くなると管理しきれなくなるしモチベーションも下がってくるので手頃な長さで切り分けることにした。

- 第1部では、Actix-web の Extraction について紹介する。
- 第2部では、Actix-web のソースコードを追い、Extraction の実装を解明する。
- ~~第3部では、第2部で判明したことを参考に簡易版を自分で実装してみる。~~

本稿ははじめの第1部である。
Actix-web を使ったアプリケーションのごく簡単な実装方法と、
Type-safe information extraction の基本動作について説明する。
普通にドキュメントに書いてあることを述べるだけなので、
既にご存知の方は読む必要が無い。

なお元々第3部にて自分でユーティリティ化してみるつもりだったが、
第2部まで理解して飽きたので今回は見送りとする。

### Actix-web の基本

Actix-web を使った Web アプリケーションは次のように記述する。
このプログラムでは "http://127.0.0.1:8088/" に GET リクエストを行うと
"Hello world!" という文字列が応答される。

アプリケーションにおける URL とリソースの関係は、
ルーティングテーブルを作成して記述することになる。
Web アプリケーションが提供する URL と、
それがリクエストされたときに実行する関数の関係を定義する。

ルーティングテーブルは主に `web::route` 関数を使って構築できる。
この関数の第1引数には URL を、第2引数には HTTP メソッドと該当リソースを応答する関数を指定する。

いま次のようなルーティングテーブルを用意するとする。

| URL    | Method | Function   |
|:-------|:-------|:-----------|
| /      | GET    | index      |
| /en_US | GET    | index      |
| /ja_JP | GET    | index2     |
| /zh_CN | GET    | index3     |

各関数 index, index2, index3 は引数 `()` を受け取って `trait Responder` を実装する値を返却する非同期関数とする。

このとき、Actix-web を使って実装するソースコードは以下のようになる。

```rs
use actix_web::{web, App, Responder, HttpServer};

async fn index() -> impl Responder {
    "Hello world!"
}

async fn index2() -> impl Responder {
    "こんにちは世界"
}

async fn index3() -> impl Responder {
    "你好 世界"
}

#[actix_rt::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new().route("/", web::get().to(index)),
        App::new().route("/en_US", web::get().to(index)),
        App::new().route("/ja_JP", web::get().to(index2)),
        App::new().route("/zh_CN", web::get().to(index3))
    })
    .bind("127.0.0.1:8088")?
    .run()
    .await
}
```

プログラムをビルドするためには例えば Cargo.toml に以下のように記述し、

```toml
# Cargo.toml

[package]
name = "project-name"
version = "0.1.0"
authors = ["author <author@example.net>"]
edition = "2018"

[dependencies]
actix-web = "2.0"
actix-rt = "1.0"
```

起動には `cargo` コマンドを利用する。

```sh
cargo run
```

起動に成功すると、
127.0.0.1 で 8088 番ポートに LISTEN する Web サーバーがたちあがる。

### Type-safe information extraction

先のコードでは引数無しで実装したが、リソースを生成する関数は
HTTP リクエストや、アプリケーション上のデータを静的に型付けした状態で受け取ることができる。受け取れるデータには次に示すものなどがある。

- URL 中の一部の文字列
- クエリパラメーター
- リクエストボディ
- リクエストボディの JSON パース結果
- データベースのコネクションプール
- アプリケーション状態変数

例えば下記のプログラムでは関数 `get_article` において、
リクエストの URL に含まれた年月日の文字列を受け取る。
`ArticleDate` 構造体として自動的に型付けされた状態で渡されるのがポイントである。

```rs
#[derive(Deserialize)]
struct ArticleDate {
    year: u32,
    month: u32,
    mday: u32
}

async fn get_article(path: web::Path<ArticleDate>) -> impl Responder {
    format!("Show article on {}/{}/{}", &path.year, &path.month, &path.mday);
}

#[actix_rt::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .route("/articles/{year}/{month}/{mday}",
                   web::get().to(get_article))
    })
    .bind("127.0.0.1:8088")?
    .run()
    .await
}
```

このプログラムでは、特定日の記事情報が取得したいといった場合に
`GET /articles/2019/03/09` へ HTTP リクエストを行うと、
関数 `get_article` が呼び出され、引数の `path` の `year`、`month`、`mday` にはそれぞれ 2019、3、9 が整数としてセットされる。

また、リクエストボディを JSON としてパースした結果を受け取ることもできる。
以下のように書くと、Actix-web では JSON データとして自動的にパースし、
関数 `create_article` へ構造体として型を付けて渡してくれる。

```rs
#[derive(Deserialize)]
struct Article {
    author: String,
    content: String
}

async fn index() -> impl Responder {
    "Hello world!"
}

async fn get_article(path: web::Path<ArticleDate>) -> impl Responder {
    format!("Show article on {}/{}/{}", &path.year, &path.month, &path.mday);
}

async fn create_article(article: web::Json<Article>) -> impl Responder {
    format!("New article, author is {}, \n{}",
             &article.author,
             &article.content);
}

#[actix_rt::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .route("/", web::get().to(index))
            .route("/articles/{year}/{month}/{mday}",
                   web::get().to(get_article))
            .route("/articles/{year}/{month}/{mday}",
                   web::put().to(create_article))
            .route("/articles",
                   web::put().to(create_article2))
    })
    .bind("127.0.0.1:8088")?
    .run()
    .await
}
```

`create_onsen` では引数に `path` と `article` の2つを定義しているため、
JSON のパース結果の他、既に紹介した URL の一部も受け取ることができる。

URL の情報が必要無ければ、引数 `path` の定義を省略することもできる。
この場合には引数 `article` だけを渡し関数を実行する。

その他、引数指定の順番を逆にしたり、別の引数を増やしても問題無くビルドでき、期待通り動作する。

つまり Actix-web では、リソースを応答する関数の引数リストを任意に変更することができ、かつ Web サーバーは引数の種類に合わせて柔軟に動作するということだ。
Python など動的型付きの言語では何の変哲も無い当然のことだが、
静的型付き言語である Rust では実に不思議なことである。

いったい `web::get().to()` や `web::post():to` の型はどうなっているのだろうか？
何故、このように柔軟な型でコンパイルが通るのか？
そして Actix-web はこの関数をどうやって呼び出しているのか？

次回の記事、第2部で Actix-web のソースコードに潜って、これを調べてみよう。
