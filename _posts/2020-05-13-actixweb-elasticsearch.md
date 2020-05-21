---
layout: post
title:  "ActixとElasticsearchによるWebアプリ実装を試した"
categories: rust
---

プログラミング言語 Rust を使い (趣味で) Web アプリケーションを作成するにあたり、データベースとして Elasticsearch を、Web フレームワークとして Actix で利用したいと考えた。しかしこの組合せで開発した前例はインターネット上で発見できなかったので、CRUD など基本的なデータベース操作を実装した簡易なサーバーアプリケーションを作って事前検証を行った。

本稿ではその調査結果とテストコードを掲載する。

なお私にとって Elasticsearch の導入は今回が初めてとなるため、Elasticsearch についての深い知見は期待できない。もともとは MongoDB を使うつもりだったが、MongoDB は日本語検索の機能が得意でないため、Elasticsearch を選択してみた。

### 1. Elasticsearch

Elasticsearch はテキスト全文検索機能を特徴とするデータベースエンジン。MongoDB のようにドキュメント指向のデータベースである。

開発はオランダに本社をおく Elastic 社が中心となりオープンソースで行なわれている。同社では他に Kibana、Logstash や Beats も開発し、リアルタイム分析エンジン Elastic Stack として売り出している。開発言語は Java のようだ。

名称について、Elastic Search とか Elastic search とか書きたくなるがこれらは誤りで、先頭だけ大文字、あとは小文字とし一単語で書き切る Elasticsearch が正式名称のようだ。

- [Elastic](https://www.elastic.co/jp/)
- [Elasticsearch \- Wikipedia](https://ja.wikipedia.org/wiki/Elasticsearch)
- [elastic/elasticsearch](https://github.com/elastic/elasticsearch)

Elasticsearch の検索エンジンのコア機能は Apache Lucene を利用する。これは Apache Solr でも利用されている。

というよりはむしろ、Apache Lucene は検索エンジンである Lucene と検索サーバー Solr の両方を含んだプロジェクトを表すこともあり、Lucene の検索ライブラリだけを限定的に指す場合には Lucene Core と呼ぶこともあるようだ。

開発言語はこれも Java である。

[Apache Lucene \- Welcome to Apache Lucene](https://lucene.apache.org/)

Elasticsearch は NoSQL のメインデータベースとして利用可能なのか？という疑問に対しては、公式の回答があり「PostgreSQL などを主要データベースとし追加で導入されることが多いが」「いくつかの制限について問題とならない目的では使える」である。Elasticsearch を追加のデータベースとして利用する場合には、メイン側から自動的にデータが同期されるようにして使う方法がある。

本稿は Elasticsearch を紹介する記事ではないので、これ以上の内容は省略する。以下を読むとだいたいわかった感じになれると思う。

- [Elasticsearch as a NoSQL Database \| Elastic Blog](https://www.elastic.co/jp/blog/found-elasticsearch-as-nosql)
- [Elasticsearch in Production \| Elastic Blog](https://www.elastic.co/jp/blog/found-elasticsearch-in-production)

#### 日本語検索

Elasticsearch は基本機能として始めから全文検索を備えているが、日本語はサポートされていない。日本語検索を可能にするためには形態素解析エンジンも一緒に用いる必要がある。

サポートされる言語は以下の通りで、英語圏の言葉が主 (のみ？) である:

> The following analyzers support setting custom stem_exclusion list: arabic, armenian, basque, bengali, bulgarian, catalan, czech, dutch, english, finnish, french, galician, german, hindi, hungarian, indonesian, irish, italian, latvian, lithuanian, norwegian, portuguese, romanian, russian, sorani, spanish, swedish, turkish. 
[Language Analyzers \| Elasticsearch Reference \[7\.7\] \| Elastic](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-lang-analyzer.html)

日本語の形態素解析エンジンには最も有名であろう Mecab をはじめとし、いくつかの種類が存在しているが、今回は Elasticsearch との連携の簡単さを重視して Kuromoji を選択した。

Kuromoji は Atilika 社の開発するオープンソースの日本語形態素解析ライブラリ。Kuromoji の開発者は Lucene や Solr にもコミットを行なっているようだ。これも Java で実装されている。

- [kuromoji \| Atilika](https://www.atilika.com/ja/kuromoji/)
- [atilika/kuromoji: Kuromoji is a self\-contained and very easy to use Japanese morphological analyzer designed for search](https://github.com/atilika/kuromoji)

#### Rust ライブラリ

Rust から Elasticsearch を使うためのライブラリが、本家により α 版ではあるものの GitHub で公開されている。現在の最新版は 7.7.0-alpha で、これは Elasticsearch 本体のバージョンに対応するとのこと。よって 7.7.0 の接続先には Elasticsearch 7.7.0 を用いるべきだが、今回は Docker に 7.6.2 までしか無かったためバージョン違いのまま検証した。

[elasticsearch 7\.7\.0\-alpha\.1 \- Docs\.rs](https://docs.rs/crate/elasticsearch/7.7.0-alpha.1)

##### 非同期処理

本ライブラリでは async/await 文法が使われている。これは2019年11月リリースの Rust 1.39.0 で導入された、非同期処理を便利に記述するための文法である。

[Async\-await on stable Rust\! \| Rust Blog](https://blog.rust-lang.org/2019/11/07/Async-await-stable.html)

非同期処理のライブラリには Tokio が使われている。

##### Connection Pooling

Web アプリケーションからデータベースを利用する場合に、Persistent connection の機能が重要である。Collection pooling と呼ぶ場合もあるが、文脈によって意味が異なるので注意。これはアプリケーションとデータベース間の接続セッションを事前に生成しておき再利用する仕組み。再利用せずに各通信ごとに再接続した場合、例えば TCP であればそのたびに 3-handshake が発生し処理上のボトルネックになる場合がある。

本ライブラリでは Rust の HTTP 通信ライブラリである reqwest が持つ Persistent connection に依存しているようだ。Elasticsearch は RESTful な API を提供し、アプリケーションは HTTP でデータベースの操作を行う。

Elasticsearch の Transport struct 内で reqwest クライアントを保持し使い回している。

```rs
pub struct Transport {
    client: reqwest::Client,
    credentials: Option<Credentials>,
    conn_pool: Box<dyn ConnectionPool>,
}
```
[transport\.rs\.html \-\- source](https://docs.rs/elasticsearch/7.7.0-alpha.1/src/elasticsearch/http/transport.rs.html#265)



なお、Elasticsearch のライブラリ上には ConnectionPool というモジュールがあるが、上記で説明したような一般的に用いる意味でのコネクションプールとは異なり、Elasticsearch クラスタ上のノードに対する接続を管理するものとのこと。つまりクラスタ上にあるノードの集合や、追加/削除を管理する。以下は .NET 向けライブラリにおける説明だが、Rust でも同様と考えられる。

> Despite the name, a connection pool in NEST is not like connection pooling that you may be familiar with from interacting with a database using ADO.Net; for example, a connection pool in NEST is not responsible for managing an underlying pool of TCP connections to Elasticsearch, this is handled by the ServicePointManager in Desktop CLR.
[Connection pools \| Elasticsearch\.Net and NEST: the \.NET clients \[7\.x\] \| Elastic](https://www.elastic.co/guide/en/elasticsearch/client/net-api/current/connection-pooling.html#static-connection-pool)

現時点では残念なことに、SingleNodeConnection と CloudConnection しか実装されていない。つまり、オンプレミス環境において複数の Elasticsearch ノードを利用できないということだ。
データベースを冗長化するならば、Elastic クラウドを利用して CloudConnection を使うか、StickyNodeConnection などを使えるように手を加える必要がある。

### 2. Actix

Actix はプログラミング言語 Rust で開発された Web フレームワークである。Rust の Web フレームワーク界隈では近年一番開発が盛んである。

[Rust's powerful actor system and most fun web framework](https://actix.rs/)

actix-web は Actix の actor フレームワークと Tokio 非同期ライブラリを利用して実装された高レベル Web フレームワーク。

[actix\_web \- Rust](https://docs.rs/actix-web/2.0.0/actix_web/index.html)

Actix-web では r2d2 を使ったコネクションプーリングの方法が紹介されている。しかし先述の通り、Elasticsearch ライブラリでは reqwest を使ってコネクションの維持を実現するため r2d2 を使う必要が無い。


### 3. アプリケーション概要

温泉施設のデータを取り扱うシンプルなアプリケーションを実装する。
温泉施設のデータは名前、温泉地名、住所だけを持つ。

アプリケーションは下記のようにデータベースの CRUD に相当する API を持っている。
また、検索機能がある。

| Method | URL         | 機能名       | 概要                                 |
|:-------|:------------|:-------------|:------------------------------------|
| GET    | /           | index        | Welcome メッセージを出力するだけ。 |
| GET    | /onsen/     | search_onsen | 温泉施設を検索する。検索条件をクエリで受信。 |
| PUT    | /onsen/     | create_onsen | 新規に温泉施設を登録する。データはJSONボディで受信。 |
| GET    | /onsen/{id} | get_onsen    | 指定されたIDの温泉施設を取得する。 |
| POST   | /onsen/{id} | update_onsen | 指定されたIDの温泉施設を変更する。データはJSONボディで受信。 |
| DELETE | /onsen/{id} | delete_onsen | 指定されたIDの温泉施設情報を削除する。 |

サーバーは Actix による Web サーバーと、Elasticsearch データベースがあり、それぞれ1台構成とした。

### 4. 開発環境

開発環境は Docker を使って構築した。

#### ファイル構成

下記の通り:

```
.
├── docker
│   ├── elasticsearch
│   │   └── Dockerfile
│   └── webapp
│       └── Dockerfile
├── docker-compose.yml
├── test.sh
└── webapp
    ├── Cargo.lock
    ├── Cargo.toml
    └── src
        └── main.rs
```

webapp/ 以下に Rust プロジェクトを置いた。

docker/ 以下には各 Docker イメージの Dockerfile を配置し、全てのイメージは Docker compose で操作できるようにした。

#### Docker compose

Docker compose の動きは設定ファイル docker-compose.yml によって制御できる。

docker-compose.yml は下記の通り:

```yaml
# docker-compose.yml
version: "3.7"
services:
  webapp:
    build:
      context: .
      dockerfile: docker/webapp/Dockerfile
    ports:
      - "8080:8080"
    entrypoint: /usr/local/cargo/bin/cargo
    command: "run app"
    tty: true
  elasticsearch:
    build: docker/elasticsearch
    expose:
      - 9200
    environment:
      discovery.type: single-node
      ES_JAVA_OPTS: "-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
```

Rust のプロジェクトが webapp の Dockerfile よりもディレクトリで上の階層にあるため、`context` 、`dockerfile` を使って工夫している。

webapp で起動した Web サーバーには、ホスト側では localhost:8080 を経由して接続できる。

Elasticsearch はとりあえず単一ノードで構築した。
本格的にクラスターを組むことはいずれ考えたい。

#### Elasticsearch image

Elasticsearch のイメージでは、素の Elasticsearch に加え、日本語全文検索を有効化するために Kuromoji をインストールした。
`elasticsearch-plugin` というツールがあり、これを実行するだけで公式プラグインをインストールできる。

```
# docker/webapp/Dockerfile
FROM docker.elastic.co/elasticsearch/elasticsearch:7.6.2

RUN elasticsearch-plugin install analysis-kuromoji
```

#### webapp image

ソースコードをコピーし、`cargo run` で必要なライブラリをインストールすると共に Web サーバーを起動する。


```
# docker/webapp/Dockerfile
FROM rust:1.43

COPY webapp /usr/src/app

WORKDIR /usr/src/app

# Start
ENTRYPOINT ["cargo", "run", "--release"]
```

### 5. Web アプリケーション実装

勉強も終わったし、環境構築が終わったので、いよいよここから Actix を使った Web アプリケーションのソースコードを紹介する。

以降のコード中では `use` 文は省略する。
全文は GitHub にアップロードしたので興味があれば参照できる。

[kikei/actix\-with\-elasticsearch](https://github.com/kikei/actix-with-elasticsearch)

#### Cargo.toml

何はともあれ Cargo.toml を書く。以下が最小構成とは限らない。

```toml
[package]
name = "actixweb-es"
version = "0.1.0"
authors = ["kikei <fujii@xaxxi.net>"]
edition = "2018"

[dependencies]
actix-web = "2"
actix-rt = "1"
elasticsearch = "7.7.0-alpha.1"
futures = "0.3"
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
tokio = "0.2"
url = "2"
```

#### Elasticsearch 接続クラアント

Elasticsearch に接続するためのクライアントは `struct Elasticsearch` で表現される。
この struct を返す関数を作っておけば便利そうだ。

ここでは Url モジュールを使って URL 文字列をパースしたが、文字列から直接 Elasticsearch を取得する関数も元から用意されていた。

Elasticsearch の接続先は事前に docker-compose.yml で設定した通りに合わせる必要がある。
先述の通りの設定であれば `elasticsearch:9200` になる。

```rs
fn create_elasticsearch_client(url: Url)
    -> Result<Elasticsearch, BuildError>
{
    let conn_pool = SingleNodeConnectionPool::new(url);
    let transport = TransportBuilder::new(conn_pool).disable_proxy().build()?;
    Ok(Elasticsearch::new(transport))
}

fn main() {
    let esclient =
        Url::parse("http://elasticsearch:9200")
        .map_err(|e| format!("Failed to parse url: {}", &e))
        .and_then(|url|
                  create_elasticsearch_client(url)
                  .map_err(|e| format!("Failed to create \
                                        elasticsearch client: {}", &e)));
    // ...
}
```

`esclient` は `Result<Elasticsearch, BuildError>` 型。
以下では Elasticsearch 接続クライアントを使い、`setup_index` でインデックスの設定を行い、`start` で Web サーバーを開始する。

```rs
fn main() {
    // ...
    match esclient {
        Err(e) => {
            println!("Failed to parse url: {}", &e);
        },
        Ok(conn) => {
            // インデックスのセットアップ
            Runtime::new().expect("").block_on(setup_index(conn.clone()));
            if let Err(e) = start(conn) {
                println!("Failed to start server: {}", &e);
            }
            println!("finished");
        }
    }
}
```

#### Web サーバー起動

Actix では `HttpServer` を使って Web サーバーの設定および起動を行う。
`HttpServer::new` に渡したクロージャは Worker の数だけ実行される。以下では適当に2個としたので、"setup worker" の文字も2回出力される。

`App` では各 API のルーティングを決める (`.`service`) 他、データベース接続等の共通に使いたいオブジェクトを設定 (`.data`) することができる。このオブジェクトは API 呼び出しごとにコピーされるため、Copy trait を実装している必要がある。

`type DBConnection` には特に意味が無いが、以前 MongoDB で作ったときのものと型名を合わせてみた。

```rs
type DBConnection = Elasticsearch;

#[actix_rt::main]
async fn start(conn: DBConnection) -> std::io::Result<()>
{
    HttpServer::new(move || {
        // For each worker
        println!("setup worker");
        App::new()
            .data(conn.clone())
            .service(index)
            .service(web::scope("/onsen")
                     .route("/", web::get().to(search_onsen))
                     .route("/", web::put().to(create_onsen))
                     .route("/{id}", web::get().to(get_onsen))
                     .route("/{id}", web::post().to(update_onsen))
                     .route("/{id}", web::delete().to(delete_onsen))
            )
    })
        .bind("0.0.0.0:8080")?
        .workers(2)
        .run()
        .await
}
```

`index` は Hello, world! 相当で、文字列を応答するだけ。

```rs
#[get("/")]
async fn index(_conn: web::Data<DBConnection>) -> impl Responder
{
    format!("Let's try actix-web + elasticsearch!")
}
```

#### Elasticsearch データベース操作

##### Setup Indices

Elasticsearch との接続に成功したらまずインデックスを作成するようにしてみた。
気軽にインデックスという単語を用いたが、実は Elasticsearch において Index とは、一般的なリレーショナルデータベースでいうテーブル相当のものを意味している。

では検索を効率化するためのキーを設定するための仕組みについては？というと、ライブラリ上では indices として表現される。
日本語では区別が難しい。孤独な私には関係無いがチームで開発するときには注意。

日本語検索のための形態素解析器の設定もこのときに行う。
以下では "name" と "address" について検索可能にしている。

```rs
async fn setup_index(esclient: Elasticsearch) {
    let result = esclient.indices()
        .create(IndicesCreateParts::Index("onsen_index"))
        .body(json!({
            "mappings": {
                "properties": {
                    "name": {
                        "type": "text",
                        "analyzer": "kuromoji"
                    },
                    "address": {
                        "type": "text",
                        "analyzer": "kuromoji"
                    }
                }
            }
        }))
        .send()
        .and_then(|r| async { r.text().await })
        .await;
    match result {
        Ok(res) => {
            println!("Successfully created an index: {}", &res)
        },
        Err(e) => {
            println!("Failed to setup index, error: {}", &e)
        }
    }
}
```

なおここでは実装していないが、このままのソースコードではサーバー起動のたびにインデックスの作成を試みる (2回目以降はエラーになる) ので、インデックスが無い場合にのみ作成するよう修正すると気持ち良いだろう。

[elasticsearch::indices::Indices \- Rust](https://docs.rs/elasticsearch/7.7.0-alpha.1/elasticsearch/indices/struct.Indices.html)

##### Create document

HTTPリクエストのボディでJSONを受信し、Elasticsearch に登録する API を実装する。

`conn` は App で data として渡した Elasticsearch クライアントが得られる。
`data` は JSON をパースして Onsen 構造体に変換済みのデータが得られる。

Actix ではこのように、登録したハンドラ関数の型に応じて適当に前処理してデータを渡してくれる不思議な機構が備えられている。

[actix\-web/route\.rs at master · actix/actix\-web](https://github.com/actix/actix-web/blob/master/src/route.rs#L226)

Elasticsearch への新規データの作成には4通りの方法があるが、ここでは ID を自動的に採番する方法を選んだ。
`create` でなく `index` を使う必要があるので注意。
下記は Elasticsearch Index API の `POST /<index>/_doc/` へのリクエストを実行することに相当する。
この URL は `IndexParts` で組み立てている。

[Index API \| Elasticsearch Reference \[7\.7\] \| Elastic](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-index_.html)

Elasticsearch との通信は非同期で実行されるため、関数全体を async で作ることが望ましいだろう。
Actix とも相性が良い。

```rs
const TABLE_ONSEN = "onsen";

async fn create_onsen(conn: web::Data<DBConnection>,
                      data: web::Json<Onsen>) -> impl Responder
{
    println!("create_onsen, data: {:?}", &data);
    let mut onsen = data.into_inner();
    if onsen.id.is_some() {
        return HttpResponse::BadRequest().finish();
    }
    let parts = IndexParts::Index(TABLE_ONSEN);
    println!("elasticsearch url: {}", &parts.clone().url());
    let result = conn.get_ref().index(parts).body(onsen.clone())
        .send()
        .and_then(|r| async { r.json::<Document>().await })
        .await;
    match result {
        Ok(result) => {
            println!("created onsen, result: {:?}", &result);
            onsen.id = Some(result._id);
            HttpResponse::Ok().json(&onsen)
        },
        Err(e) => {
            println!("failed to create onsen, error: {:?}", &e);
            HttpResponse::NotFound().finish()
        }
    }
}
```

ソースコード中で登場する `Onsen`、`Document` は下記のように定義した。

`Onsen` は `Deserialize` と `Serialize` により、JSON との相互変換が可能。

```rs
#[derive(Clone, Deserialize, Serialize, Debug)]
struct Onsen {
    #[serde(default)]
    pub id: Option<String>, // ID.
    pub area: String,       // 地域
    pub name: String,       // 施設名/旅館名
    pub address: String     // 住所
}
```

`Document` は Elasticsearch からのレスポンスを serde_json を使って直接 Deserialize できるように定義した。
しかし、この作り方は制限が多くだんだん辛くなってくるため、いったん `serde_json::Value` にしたあと、`Value` から `Onsen` に変換する関数を作る方が良さそうに感じた。

```rs
#[derive(Deserialize, Debug)]
struct Document {
    _id: String,
    _index: String,
    _type: String
}
```

curl で登録する場合には以下のようにする。

```console
$ curl -XPUT -H "Content-Type: application/json" --data \
'{ "name": "西多賀旅館", "address": "宮城県大崎市鳴子温泉新屋敷78-3", "area": "鳴子温泉" }' http://localhost:8080/onsen/
{"id":"umbINHIB3Vl9TKW-8SVx","area":"鳴子温泉","name":"西多賀旅館","address":"宮城県大崎市鳴子温泉新屋敷78-3"}⏎ 

```

私と西多賀旅館には何の関係も無いが、勝手にデータとして利用した。宮城県の鳴子温泉の国道沿いにある小さい温泉旅館で、湯は硫黄成分を多く含んだ火山性の塩化物泉で独特な緑色 (と苦味) がいい感じである。緑灰色の湯が湛えられた長方形の素朴な浴槽は素晴しいもの。

[鳴子温泉 西多賀旅館\(公式ホームページ\)\-\-東北宮城の湯治宿、ウォーキング・長期滞在・自炊ができる宿](http://www.nishitaga.com/0-top/top2017.html)

##### Read a document

HTTPリクエストの URL 中で指定された ID に基づき、Elasticsearch に保存されたドキュメントを取得する API を実装する。

下記は Elasticsearch Get API のうち `GET /<index>/_doc/<_id>` へのリクエストを実行することに相当する。
この URL は `GetParts` で組み立てている。

[Get API \| Elasticsearch Reference \[7\.7\] \| Elastic](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-get.html)

```rs
async fn get_onsen(conn: web::Data<DBConnection>,
                   path: web::Path<OnsenPath>) -> impl Responder {
    println!("get_onsen id: {}", &path.id);
    let result =
        conn.get_ref().get(GetParts::IndexId(TABLE_ONSEN, path.id.as_str()))
        .send()
        .and_then(|r| async {
            r.json::<DocumentWithSource<Onsen>>().await.map(|r| Onsen {
                id: Some(r._id),
                ..r._source
            })
        })
        .await;
    match result {
        Ok(onsen) => {
            HttpResponse::Ok().json(&onsen)
        },
        Err(e) => {
            println!("Error in get_onsen: {}", &e);
            HttpResponse::NotFound().finish()
        }
    }
}
```

`DocumerntWithSource` は次のように定義した。
Document に加え `_source` を持てるようになっている。
`Document` とほぼ二重定義になっており、この実装はあまりよくない。

```rs
#[derive(Deserialize, Debug)]
struct DocumentWithSource<S>
where S: Serialize
{
    _id: String,
    _index: String,
    _type: String,
    _source: S
}
```

定義した API を利用し、登録済みのドキュメントが次のように取得できる。

```
$ curl http://localhost:8080/onsen/umbINHIB3Vl9TKW-8SVx
{"id":"umbINHIB3Vl9TKW-8SVx","area":"鳴子温泉","name":"西多賀旅館","address":"宮城県大崎市鳴子温泉新屋敷78-3"}
```

なおこの実装では、"onsen" インデックスにドキュメントが存在しない場合にエラーになってしまう。
これも修正すべきであるが、今回は飽きたので見なかったことにする。

##### Update a document

HTTPリクエストの URL 中でドキュメントの ID を指定し、ボディで JSON に従い指定したドキュメントを変更する API を実装する。JSON にも ID を指定することができるが、URL 中の ID と JSON 中の ID が一致しない場合にはエラーとする。

[Update API](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-update.html) の `POST /<index>/_update/<_id>` を利用した。

この URL は `UpdateParts` で組み立てられる。

```rs
async fn update_onsen(conn: web::Data<DBConnection>,
                      path: web::Path<OnsenPath>,
                      data: web::Json<Onsen>) -> impl Responder {
    println!("update_onsen id: {}, data: {:?}", &path.id, &data);
    let mut onsen = data.into_inner();
    if (&onsen.id).as_ref().filter(|id| id.to_string() == path.id).is_none() {
        println!("Id must match between url and body data");
        return HttpResponse::NotFound().finish();
    }
    onsen.id = None;
    let parts = UpdateParts::IndexId(TABLE_ONSEN, path.id.as_str());
    println!("elasticsearch url: {}", &parts.clone().url());
    let result =
        conn.get_ref().update(parts)
        .body(json!({
            "doc": onsen
        }))
        .send()
        .and_then(|r| async {
            r.text().await
        })
        .await;
    match result {
        Ok(result) => {
            println!("updated onsen, result: {:?}", &result);
            HttpResponse::Ok().json(&result)
        },
        Err(e) => {
            println!("Error in update_onsen: {}", &e);
            return HttpResponse::NotFound().finish()
        }
    }
}
```

全体的にはこれまで見てきたコードと同じように書くことができた。

定義した API を利用し、ドキュメントを次のように更新できる。

```console
$ curl -XPOST -H "Content-Type: application/json" --data \
'{ "id": "umbINHIB3Vl9TKW-8SVx", "name": "西多賀旅館 東北宮城の湯治宿", "address": "宮城県大崎市鳴子温泉新屋敷78-3", "area": "鳴子温泉" }' http://localhost:8080/onsen/umbINHIB3Vl9TKW-8SVx
{"id":"umbINHIB3Vl9TKW-8SVx","area":"鳴子温泉","name":"西多賀旅館 東北宮城の湯治宿","address":"宮城県大崎 市鳴子温泉新屋敷78-3"}
```

#### Delete a document

HTTP リクエスト中の ID に従い該当のドキュメントを削除する API を実装する。

[Delete API](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-delete.html) にある `DELETE /<index>/_doc/<_id>` を利用する。

これが簡単なコードになった。

```rs
async fn delete_onsen(conn: web::Data<DBConnection>,
                      path: web::Path<OnsenPath>) -> impl Responder {
    println!("delete_onsen id: {}", &path.id);
    let result =
        conn.get_ref().delete(DeleteParts::IndexId(TABLE_ONSEN, path.id.as_str()))
        .send()
        .and_then(|r| async {
            r.json::<Document>().await
        })
        .await;
    match result {
        Ok(result) => {
            HttpResponse::Ok().json(json!({ "id": result._id }))
        },
        Err(e) => {
            println!("Error in delete_onsen: {}", &e);
            HttpResponse::NotFound().finish()
        }
    }
}
```

定義した API を利用し、ドキュメントを次のように更新できる。

```console
$ curl -XDELETE http://localhost:8080/onsen/umbINHIB3Vl9TKW-8SVx
{"id":"umbINHIB3Vl9TKW-8SVx"}
```

#### Search documents

最後に検索 API を実装する。
シンプルな検索のみをサポートし、HTTP リクエストの URL に付与した検索文字列を使って名前と住所について全文検索するようにした。実用的には温泉地名も検索対象としたいところだが、それだと全部が対象になってしまいテスト実装として検証しにくいため除外した。

ここまでは概ねアプリケーションの1APIと Elasticsearch の1APIが対応していたが、ここでは検索文字列の有無により分岐することにした。

- 検索文字列が無い場合には何も指定せずに Elasticsearch の [Search APIs](https://www.elastic.co/guide/en/elasticsearch/reference/current/search.html) を実行する。これによって全てのドキュメントを取得できる。
- 検索文字列が有る場合には、[Multi\-match query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-multi-match-query.html) を利用し、`name` と `address` を対象に全文検索する。

実際には一度に取得できる最大数を制限したり、ページングしたりしたくなるはずだが、ここでは省略した。Search API にその機能がある。

```rs
async fn search_onsen(conn: web::Data<DBConnection>,
                      query: web::Query<SearchQuery>) -> impl Responder {
    println!("search_onsen, query: {:?}", &query);
    let result = match query.query.as_ref().map(|s| s.as_str()) {
        None | Some("") =>
            conn.get_ref().search(SearchParts::Index(&[TABLE_ONSEN]))
            .send()
            .and_then(|r| async {
                r.json::<SearchResult<Onsen>>().await
            })
            .await,
        Some(qs) =>
            conn.get_ref().search(SearchParts::Index(&[TABLE_ONSEN]))
            .body(json!({
                "query": {
                    "multi_match": {
                        "query": qs,
                        "fields": ["name", "address"]
                    }
                }
            }))
            .send()
            .and_then(|r| async {
                r.json::<SearchResult<Onsen>>().await
            })
            .await
    };
    match result {
        Ok(result) => {
            println!("search result, took: {}, hits: {:?}",
                     &result.took, &result.hits);
            HttpResponse::Ok().json(OnsenList {
                took: result.took,
                onsens: result.hits.hits.iter().map(|d| Onsen {
                    id: Some(d._id.clone()),
                    ..d._source.clone()
                }).collect()
            })
        },
        Err(e) => {
            println!("Error in search_onsen: {}", &e);
            HttpResponse::NotFound().finish()
        }
    }
}
```

`SearchQuery` は以下のように定義した。
HTTP リクエストのクエリパラメーターとして `query` を受け取ることができる。

```rs
#[derive(Debug, Deserialize)]
struct SearchQuery {
    query: Option<String>
}
```

次に `OnsenList` の定義は以下。

```rs
#[derive(Clone, Deserialize, Serialize, Debug)]
struct OnsenList {
    pub took: i32,
    pub onsens: Vec<Onsen>
}
```

`took` は検索に要した時間 (ミリ秒) を表す。

また `SearchResultHits` は以下の通り定義した。
単にパースのために必要だったために用意した。

```rs 
#[derive(Deserialize, Debug)]
struct SearchResultHits<S>
where S: Serialize
{
    hits: Vec<DocumentWithSource<S>>
}
```

定義した API を使い検索を実行してみる。

といきたいところだが、事前にデータをいくつか登録しておく。

```sh
cat <<EOF >onsen.list
西多賀旅館,鳴子温泉,宮城県大崎市鳴子温泉新屋敷78-3
初音旅館,東鳴子温泉,宮城県大崎市鳴子温泉字鷲ノ巣90‐3
高五郎の湯 高東旅館,川渡温泉,宮城県大崎市鳴子温泉字築沢23-1
あすか旅館,中山平温泉,宮城県大崎市鳴子温泉字星沼68-1
吹上温泉 峯雲閣,鬼首温泉,宮城県大崎市鳴子温泉鬼首吹上16
EOF
cat onsen.list | while read line; do \
  name=${line%%,*}
  tail=${line#*,}
  area=${tail%%,*}
  addr=${tail#*,}
  curl -XPUT -H "Content-Type: application/json" --data \
    "{ \"name\": \"$name\", \"address\": \"$addr\", \"area\": \"$area\" }" \
    http://localhost:8080/onsen/
done
```

条件無しで検索してみよう。

```console
$ curl http://localhost:8080/onsen
{"took":692,"onsens":[{"id":"u2YjOHIB3Vl9TKW-iiXC","area":"鳴子温泉","name":"西多賀旅館","address":"宮城県大崎市鳴子温泉新屋敷78-3"},{"id":"vGYjOHIB3Vl9TKW-iiXT","area":"東鳴子温泉","name":"初音旅館","address":" 宮城県大崎市鳴子温泉字鷲ノ巣90‐3"},{"id":"vWYjOHIB3Vl9TKW-iiXh","area":"川渡温泉","name":"高五郎の湯 高東 旅館","address":"宮城県大崎市鳴子温泉字築沢23-1"},{"id":"vmYjOHIB3Vl9TKW-iiXv","area":"中山平温泉","name":"あすか旅館","address":"宮城県大崎市鳴子温泉字星沼68-1"},{"id":"v2YjOHIB3Vl9TKW-iiX_","area":"鬼首温泉","name":"吹上温泉 峯雲閣","address":"宮城県大崎市鳴子温泉鬼首吹上16"}]}
```

`query=温泉` とすると以下のレスポンスが得られた。

```console
$ curl "http://localhost:8080/onsen/?query=%E6%B8%A9%E6%B3%89"
{"took":4,"onsens":[{"id":"v2YjOHIB3Vl9TKW-iiX_","area":"鬼首温泉","name":"吹上温泉 峯雲閣","address":"宮城県大崎市鳴子温泉鬼首吹上16"},{"id":"u2YjOHIB3Vl9TKW-iiXC","area":"鳴子温泉","name":"西多賀旅館","address":"宮城県大崎市鳴子温泉新屋敷78-3"},{"id":"vWYjOHIB3Vl9TKW-iiXh","area":"川渡温泉","name":"高五郎の湯 高東旅館","address":"宮城県大崎市鳴子温泉字築沢23-1"},{"id":"vmYjOHIB3Vl9TKW-iiXv","area":"中山平温泉","name":"あすか旅館","address":"宮城県大崎市鳴子温泉字星沼68-1"},{"id":"vGYjOHIB3Vl9TKW-iiXT","area":"東鳴子温泉","name":"初音旅館","address":"宮城県大崎市鳴子温泉字鷲ノ巣90‐3"}]}
```

`query=旅館` とすると、鬼首温泉の吹上温泉峯雲閣が除外され、残りの4件が返された。

```console
$ curl "http://localhost:8080/onsen/?query=%E6%97%85%E9%A4%A8"
{"took":4,"onsens":[{"id":"vGYjOHIB3Vl9TKW-iiXT","area":"東鳴子温泉","name":"初音旅館","address":"宮城県大崎市鳴子温泉字鷲ノ巣90‐3"},{"id":"u2YjOHIB3Vl9TKW-iiXC","area":"鳴子温泉","name":"西多賀旅館","address":" 宮城県大崎市鳴子温泉新屋敷78-3"},{"id":"vmYjOHIB3Vl9TKW-iiXv","area":"中山平温泉","name":"あすか旅館","address":"宮城県大崎市鳴子温泉字星沼68-1"},{"id":"vWYjOHIB3Vl9TKW-iiXh","area":"川渡温泉","name":"高五郎の湯 高東旅館","address":"宮城県大崎市鳴子温泉字築沢23-1"}]}
```
`query=す` では「あすか温泉」だけがヒットした。

```console
$ curl "http://localhost:8080/onsen/?query=%E3%81%99"
{"took":1,"onsens":[{"id":"vmYjOHIB3Vl9TKW-iiXv","area":"中山平温泉","name":"あすか旅館","address":"宮城県大崎市鳴子温泉字星沼68-1"}]}
```

`query=の旅` では4件がヒットした。「の旅」という文字列の含まれたドキュメントは存在しないが、柔軟に「旅」で拾ってくれたように見える。

```console
$ curl "http://localhost:8080/onsen/?query=%E3%81%AE%E6%97%85"
{"took":6,"onsens":[{"id":"vWYjOHIB3Vl9TKW-iiXh","area":"川渡温泉","name":"高五郎の湯 高東旅館","address":"宮城県大崎市鳴子温泉字築沢23-1"},{"id":"vGYjOHIB3Vl9TKW-iiXT","area":"東鳴子温泉","name":"初音旅館","address":"宮城県大崎市鳴子温泉字鷲ノ巣90‐3"},{"id":"u2YjOHIB3Vl9TKW-iiXC","area":"鳴子温泉","name":"西多賀旅館","address":"宮城県大崎市鳴子温泉新屋敷78-3"},{"id":"vmYjOHIB3Vl9TKW-iiXv","area":"中山平温泉","name":"あすか旅館","address":"宮城県大崎市鳴子温泉字星沼68-1"}]}
```

### 6. まとめ

Elasticsearch と Actix を利用した RESTful Web アプリケーションが作成可能か、簡単なアプリを作って検証した。

機能的には十分使えると判断したので、次のアプリケーションではこの構成でやってみるつもりである。ただし、本格的な商用サービスを制作するつもりならば、Elasticsearch のクラスタリングを実現する方法を考えることが必須と思う。

本文中にも書いたが、作成したソースコードの全文は下記 GitHub から見ることができる。

[kikei/actix\-with\-elasticsearch](https://github.com/kikei/actix-with-elasticsearch)

### 7. 参考資料

参考にした資料は記事中で都度記載してきた。

記載していないものだけ、以下に列挙する。


- [Elasticsearchで全文検索サービスを作るときに最初に見るべきだったリンク集 \- Qiita](https://qiita.com/k-take/items/6e44f97bfd4ab9521f7e)
    Elasticsearch で日本語全文検索をするための設定方法が紹介されている。
- [はじめての Elasticsearch \- Qiita](https://qiita.com/nskydiving/items/1c2dc4e0b9c98d164329)
    Elasticsearch に対する HTTP レスポンス/リクエストの例が多く紹介されており便利。


