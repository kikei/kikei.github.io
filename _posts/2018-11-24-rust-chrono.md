---
layout: post
title:  "Rustで日付、時刻を計算する"
categories: rust
---

Rust を使ってみた。
様子見と練習のため、日付、時刻を扱うプログラムを少し書いてみたのでここにメモする。

日付、時刻は何だかんだどんなアプリケーションでも使ってしまう重要な機能である。

### 1. Rust 言語

Rust は静的に型付けされたコンパイル言語である。

C言語の弟分でありつつ、Haskell とか OCaml のような ML っぽい概念を取り入れた言語だと個人的には思っている。ML 言語ファミリーは好きだが、ライブラリの品揃えや保守等、独りで書いていくのは時間的に辛いと思っており、Rust には期待している。

というわけで Rust 初心者だが、練習に日付を扱うプログラムを書いてみた。

今回使った Rustc と Cargo のバージョンは 1.30 である。

```shell
$ rustc --version
rustc 1.30.1 (1433507eb 2018-11-07)
$ cargo --version
cargo 1.30.0 (a1a4ad372 2018-11-02)
```

### 2. chrono

Rust で日付、時刻を扱うには [chrono](https://docs.rs/crate/chrono/) ライブラリを使うとよさそうである。名前がかっこいいので謎の 3rdParty ライブラリかと思った(偏見)が、[Wikipedia](https://ja.wikipedia.org/wiki/Rust_(%E3%83%97%E3%83%AD%E3%82%B0%E3%83%A9%E3%83%9F%E3%83%B3%E3%82%B0%E8%A8%80%E8%AA%9E)) でも紹介されており準公式みたいな位置付けのようだ。基本的な機能であっても外部ライブラリとするのが開発コミュニティの方針らしい。

現時点で chrono の最新バージョンは 0.4.6 である。

```toml
[dependencies]
chrono = "0.4.6"
```

なお本記事のコードでは、
以下のように事前にいくつかの型を読み込んであることを前提とする。

```rust
extern crate chrono;

use chrono::{TimeZone, Weekday, ParseResult};
use chrono::prelude::{DateTime, Utc, Local, Datelike, Timelike};
use chrono::offset::FixedOffset;
```

### 3. 現在の日付、時刻の取得

chrono ではありがたいことに、タイムゾーンの機能がちゃんと備えられており、
しかも簡単に使うことができる。それだけでなくタイムゾーンが型になっているので、
UTC とローカルタイムを区別しながらプログラムを書くことができる。

いま覇権な JavaScript、Python ではタイムゾーンは割と蔑ろにされている印象があり、
一方で Rust の開発はとても素晴らしい。

```rust
// UTC で現在の日付、時刻を取得する。
let utc: DateTime<Utc> = Utc::now();

// PC のタイムゾーンを使って取得。
let local: DateTime<Local> = Local::now();
```

こんな感じで UTC とローカルタイムの現在時刻がとれている。

```rust
println!("Now:\n    utc: {:?}\n    local: {:?}",
         utc.to_string(), local.to_string());
// Now:
//     utc: "2018-11-24 16:10:23.839292405 UTC"
//     local: "2018-11-25 01:10:23.839298519 +09:00"
```

エポック秒は同じ。

```rust
println!("Timestamp:\n    utc.timestamp(): {}\n    local.timestamp(): {}",
         utc.timestamp(), local.timestamp());
// Timestamp:
//     utc.timestamp(): 1543075823
//     local.timestamp(): 1543075823
```

私のタイムゾーンは +09:00 でした。

```rust
println!("TimeZone:\n    local.offset()={}", local.offset().to_string());
// TimeZone:
//     local.offset()=+09:00
```

こんなテストを書いてみた。
Rust はテストも気軽に書けてよい。

```rust
#[test]
fn test_now() {
    let utc = Utc::now();
    let local = Local::now();
    assert!(utc.timestamp() == local.timestamp());
}
```

### 4. 日付のフォーマット、パース

日付をテキスト表現にするには、 `DateTime::format` を使う。
フォーマットには [strftime]((https://docs.rs/chrono/*/chrono/format/strftime/index.html)) の文法を使える。

こんなテストを書いた。

```rust
#[test]
fn test_format_iso() {
    let datetime = Utc.ymd(2018, 1, 2).and_hms(3, 44, 55);
    assert!(datetime.format("%Y-%m-%dT%H:%M:%S%z").to_string() ==
            "2018-01-02T03:44:55+0000");
}
```

日付のパースは `DateTime::parse_from_str` を使う。
フォーマットには [strftime]((https://docs.rs/chrono/*/chrono/format/strftime/index.html)) の文法を使える。

こんなテストを書いた。

```rust
#[test]
fn test_parse_iso() {
    let datetime = "2018-01-02T03:44:55+0900";
    assert!(DateTime::parse_from_str(datetime, "%Y-%m-%dT%H:%M:%S%z") ==
            Ok(FixedOffset::east(60 * 60 * 9)
               .ymd(2018, 1, 2).and_hms(3, 44, 55)));
}
```

### 5. 年月日時分秒の取得

年、月、日とか時、分、秒(とかミリ秒とか)の情報を個別に取得したいこともあるだろう。

日付の情報は [`trait DateLike`](https://docs.rs/chrono/0.4.6/chrono/trait.Datelike.html) を、時刻の情報は [`trait TimeLike`](https://docs.rs/chrono/0.4.6/chrono/trait.Timelike.html) の型を、それぞれロードしておくと DateTime から取得できるようになる。

代表的なのはこんな感じ。

```rust
let year: u32 = datetime.year();     // 年 2018
let month: u32 = datetime.month();   // 月 1-12
let mday: u32 = datetime.day();      // 日 1-31
let wday: Weekday = datetime.weekday();
let hour: u32 = datetime.hour();     // 時 0-23
let minute: u32 = datetime.minute(); // 分 0-60
let second: u32 = datetime.second(); // 秒 0-60
```

曜日は `enum` 型。日本の曜日に変換するコードは以下のようにして書ける。

```rust
fn format_japan_weekday(weekday: &Weekday) -> &str {
    match weekday {
        Weekday::Mon => "月",
        Weekday::Tue => "火",
        Weekday::Wed => "水",
        Weekday::Thu => "木",
        Weekday::Fri => "金",
        Weekday::Sat => "土",
        Weekday::Sun => "日",
    }
}
```

`"2018年11月25日(日) 01:48:10"` みたいな和風(?)な時刻表記が欲しいかもしれない。
そんなときは以下のようなコードを使えばいい。

```rust
fn format_japan_date(datetime: &DateTime<Local>) -> String {
    format!("{}年{:02}月{:02}日({}) {:02}時{:02}分{:02}秒",
            datetime.year(),
            datetime.month(),
            datetime.day(),
            format_japan_weekday(&datetime.weekday()),
            datetime.hour(),
            datetime.minute(),
            datetime.second())
}
```

この時刻表記から DateTime を取り出したいときもあるかもしれない。

いや、ないかもしれない。でも書いてみた。

和風の時刻表記からローカルタイムを生成するコードはこう:

```rust
fn parse_japan_date(s: &String) -> ParseResult<DateTime<Local>>  {
    let chars = s.chars();
    let s1: String = chars.clone().take_while(|&c| c != '(').collect();
    let s2: String = chars.skip_while(|&c| c != ')').skip(1).collect();
    let text = s1 + &s2;
    Local.datetime_from_str(&text, "%Y年%m月%d日 %H時%M分%S秒")
}
```

基本的には `Local.datetime_from_str` を使った。
この関数は `DateTime::parse_from_str` と同じような機能を持つが、
出力がマシンのローカルタイムになる点が違う。

気をつけてほしいのは、この関数では、和風な時刻表記ならローカルタイムだよね、
みたいないい加減な仮定がなされている。
日本国外の人も使うアプリケーションでまじめに使いたいなら、
もうちょっと考えたほうがいいと思う。たぶん。

あと `(日)` みたいな曜日表記が strftime にはまらないのでこれはパース前に除去した。
上4行が該当。がんばって書いた。イテレータをいい感じに操作して短く書けた。
こういう書き方は関数型言語を使っているとよく出てくる。そこは今回は解説しない。

日本語みたいなマルチバイト文字の入った文字列を解析するときは
`String.chars` を使うとよいみたいだ。
これを使うと UTF-8 の文字単位で文字列から1文字ずつ取り出すイテレータが手に入る。

逆に言えば `String.chars` を使わないとバイト単位の処理が必要になる。
ご存知の通り、ASCII文字は1バイトである一方、日本語の文字は1文字で3バイトあることが多い。雑に書いて3バイトの途中で変な編集を入れると文字が壊れる。

`String.chars` を使って1文字ずつ取り出してみた例が以下:

```rust
fn main() {
    let t = String::from("2018年11月25日(日) 01:48:10");
    for c in t.chars() {
        print!("\"{}\", ", c);
    }
}
// "2", "0", "1", "8", "年", "1", "1", "月", "2", "5", "日", "(", "日", ")", " ", "0", "1", ":", "4", "8", ":", "1", "0", 
```

バイト単位の処理というのは以下。
`"11"` を文字列の中から探したときインデックスは `5` になって欲しいところだが、
`"年"` が3バイトなので結果は `7` になってしまった。それ以降も同じ。

```rust
let t = String::from("2018年11月25日(日) 01:48:10");
println!("2018 {}", t.find("2018").unwrap_or(0));
println!("年   {}", t.find("年").unwrap_or(0));
println!("11   {}", t.find("11").unwrap_or(0));
println!("月   {}", t.find("月").unwrap_or(0));
println!("25   {}", t.find("25").unwrap_or(0));
println!("日   {}", t.find("日").unwrap_or(0));
// 2018 0
// 年   4
// 11   7
// 月   9
// 25   12
// 日   14
```

何の話してたっけ。和風時刻のテストも書いた。

```rust
#[test]
fn test_format_japan_date() {
    let local = Local.ymd(2018, 1, 2).and_hms(3, 44, 55);
    assert!(format_japan_date(&local) == "2018年01月02日(火) 03時44分55秒");
}

#[test]
fn test_parse_japan_date() {
    let date = String::from("2018年01月02日(火) 03時44分55秒");
    assert!(parse_japan_date(&date).expect("error") ==
            Local.ymd(2018, 1, 2).and_hms(3, 44, 55));
}
```

### 6. エポック秒との相互変換

時刻情報をエポック秒に変換したり、逆にエポック秒から時刻へ変換したり、
ということは頻繁にある。

個人的には、時刻情報を DB に格納したり、HTTP 通信に載せたりするとき、エポック秒にしたい。

DateTime からエポック秒を取り出す。

```rust
fn epoch_from_datetime(&datetime: &DateTime<Local>) -> f64 {
    let secs = datetime.timestamp() as f64;
    let nanos = (datetime.timestamp_subsec_nanos() as f64) / 1e9;
    secs + nanos
}
```

エポック秒から DateTime を作る。

```rust
fn local_from_epoch(nanos: f64) -> DateTime<Local> {
    let secs = nanos as i64;
    let nsecs = (nanos - secs as f64) * 1e9;
    Local.timestamp(secs, nsecs as u32)
}
```

ナノ秒まで取れるからエポック秒に載せているけど、このコードでは精度はそこまで出ない。
`1e9` を掛け算するあたりで多分けっこう落ちている。
精度が落ちるのを嫌うならば、文字列経由でがんばってみればよいだろう。
でもここではやらない。そもそも取得できる値の精度がそんなレベルで信頼できるか疑問。

テストも書いた:

```rust
#[test]
fn test_epoch_from_datetime() {
    let local = Local.ymd(2018, 1, 1).and_hms(0, 0, 0);
    assert!(epoch_from_datetime(&local) == 1514732400.0);
}

#[test]
fn test_local_from_epoch() {
    let epoch: f64 = 1514732400.0;
    assert!(local_from_epoch(epoch) ==
            Local.ymd(2018, 1, 1).and_hms(0, 0, 0));
}
```

### 7. 時刻の加算、減算

一定時間前後の時刻を計算したり、2つの時刻の時間差を計算することができる。

Python の `datetime.timedelta` はとても便利だが、
Rust にも同等のものがあって [`Duration`](https://docs.rs/chrono/0.4.0/chrono/struct.Duration.html) を使う。

現在の翌日、前日の時刻を計算してみるテスト:

```rust
#[test]
fn test_tomorrow() {
    let local = Local::now();
    let offset = Duration::days(1);
    let tomorrow = local + offset;
    assert!(local.timestamp() + 24 * 60 * 60 == tomorrow.timestamp());
}

#[test]
fn test_yesterday() {
    let local = Local::now();
    let offset = Duration::days(-1);
    let tomorrow = local + offset;
    assert!(local.timestamp() - 24 * 60 * 60 == tomorrow.timestamp());
}
```

明日と今の時間差を計算してみるテスト:

```rust
#[test]
fn test_diff() {
    let local = Local::now();
    let tomorrow = local + Duration::days(1);
    assert!(tomorrow - local == Duration::days(1));
}
```

### 7. ソースコード

GitHub にソースコードをアップロードしてみた。

本記事に貼り付けられているコード一式は基本的に以下のレポジトリから入手できる。

https://github.com/kikei/rust-practice-date

テストを実行するときは以下:

```shell
cargo test
```

プログラムを実行するときは以下:
ちょっとしか処理無いけど。

```shell
cargo run
```

### 8. 参考

標準ライブラリ関係:

- [std::string::String \- Rust](https://doc.rust-lang.org/std/string/struct.String.html#method.find)
- [std::str::Chars \- Rust](https://doc.rust-lang.org/std/str/struct.Chars.html)
- [std::fmt \- Rust](https://doc.rust-lang.org/std/fmt/index.html)
- [std::option::Option \- Rust](https://doc.rust-lang.org/std/option/enum.Option.html)

chrono:

- [chrono::DateTime \- Rust](https://docs.rs/chrono/0.4.0/chrono/struct.DateTime.html)
- [chrono::offset::Utc \- Rust](https://docs.rs/chrono/0.4.0/chrono/offset/struct.Utc.html)
- [chrono::offset::Local \- Rust](https://docs.rs/chrono/0.4.0/chrono/offset/struct.Local.html)
- [chrono::format::strftime \- Rust](https://docs.rs/chrono/*/chrono/format/strftime/index.html)
- [chrono::Duration \- Rust](https://docs.rs/chrono/0.4.0/chrono/struct.Duration.html)
- [chrono::Datelike \- Rust](https://docs.rs/chrono/0.4.6/chrono/trait.Datelike.html)
- [chrono::Timelike \- Rust](https://docs.rs/chrono/0.4.6/chrono/trait.Timelike.html)

Rust には Repl が無いが、Web でプログラムを実行できるサービスがあって便利だった:

- [Rust Playground](https://play.rust-lang.org/)

### 9. おまけ

和風の時刻をパースするために書いたコードをせっかくだから残しておく。
バイト列を直接操作して文字を削り、それから UTF-8 文字列に直している。
バイト列から `String` に直す段階で失敗の可能性があるのがとても気に食わなかった。

このあと `String.chars` を使いうまく書けたので、本文ではうまいコードに差し替えた。

```rust
fn copy_out(left: usize, size: usize, v: &[u8]) -> Vec<u8> {
    let vlen = v.len();
    let end = vlen - (size + 1);
    let mut result = vec![0; end];
    result[..left].copyfrom_slice(&v[0..left]);
    result[left..end].copy_from_slice(&v[(left+size+1)..vlen]);
    result
}

fn parse_japan_date(s: &String) -> ParseResult<DateTime<Local>>  {
    let left = s.find('(').unwrap_or(0);
    let size = s[left..].find(')').unwrap_or(left);
    let text = copy_out(left, size, s.as_bytes());
    let text = String::from_utf8(text).expect("error");
    Local.datetime_from_str(&text, "%Y年%m月%d日 %H時%M分%S秒")
}
```
