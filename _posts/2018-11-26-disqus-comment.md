---
layout: post
title:  "GitHub Pages+JekyllにDisqusコメントフォームを追加した"
categories: blog
---

本ブログは GitHub Pages+Jekyll で運用している。
今回、Disqus サービスを使ったコメントフォームを追加した。

**追加してみたあとで、期待していたものと全然違ったので削除しました。**

本ブログは技術的な話題の多いブログであるが、おそらく記述の間違いとかがあったりするはずだ。その指摘を受け付ける窓口が無いのはよくないと以前から思っていた。コメントフォームがあれば、親切な人がいてそこで正誤を教えてくれることもあるだろう。

Disqus を選んだのは、世間で GitHub Page+Jekyll との組み合わせで一般的に利用されている手法っぽいからだ。
コメントフォーム界隈の再大手みたいだ。サンフランシスコの会社が運営しているらしい。

[Disqus](https://disqus.com/)

この記事では以下ような方針でコメントフォームを表示することにした:

- 全部の投稿記事にコメントフォームを表示する。
- 投稿記事の末尾(本文と記事ナビゲーションの間)に表示する。
- 記事個別にコメントフォームを非表示にすることができる。

なお、これまでの Jekyll 関連の記事は以下:

- [GitHub Page\+Jekyllに前回/次回の記事リンクを追加した](https://kikei.github.io/blog/2018/05/27/jekyll-navi.html)
- [GitHub Page\+JekyllにMathJaxを導入した](https://kikei.github.io/blog/2017/07/28/jekyll-mathjax.html)
- [GitHub Page\+Jekyllによるブログの構築](https://kikei.github.io/blog/2016/05/21/githubpage.html)

では設定作業に入る。

### 1. Disqus への登録

まず Disqus へ登録する必要がある。

ユーザーアカウント作成、ブログ埋め込みの選択、ブログ設定あたりを実施した気がする。
特に迷う点は無かった。

E-mail アドレスとブログの URL があればいける。

### 2. Jekyll 設定の編集

まずコメントフォームを表示するテンプレートを作成し、
`_include/disqus_comments.html` として保存した。

中身は以下:

{% raw %}
```html
{% if page.comments != false %}

<div class="post-comments">
  <div id="disqus_thread"></div>
  <noscript>Please enable JavaScript to view the <a href="https://disqus.com/?ref_noscript">comments powered by Disqus.</a></noscript>
</div>

<script>
  var disqus_config = function () {
    this.page.url = '{{ page.url | absolute_url }}';
    this.page.identifier = '{{ page.url | absolute_url }}';
  };
  (function() {
    var d = document, s = d.createElement('script');
    s.src = 'https://fang-zhi-yan-suan-zi.disqus.com/embed.js';
    s.setAttribute('data-timestamp', +new Date());
    (d.head || d.body).appendChild(s);
  })();
</script>

{% endif %}
```
{% endraw %}

{% raw %}
一行目 `{% if page.comments != false %}` とした。
{% endraw %}

全ての記事でコメントフォームをデフォルト表示するため、
明示的に `false` にしない限り表示、という条件式にした。

非表示にしたい場合には、記事の先頭 (`YAML mutter`) に非表示設定を追記すればよい。
この記事であれば、以下のように記述すればコメントフォームが表示されなくなることだろう。

```yaml
---
layout: post
title:  "GitHub Pages+JekyllにDisqusコメントフォームを追加した。
categories: blog
comments: false
---
```

あと `s.src` に謎の文字列が入っている。
ここにはブログ名が FQDN に入った URL が挿入されるようだ。
今回、ブログ名を「放置演算子」にして漢字で設定したところ、
中国読みに変換されたものになってしまった。まあこれはこれでいいか。

なお、`class="post-comments"` は本ブログのスタイル指定に追加で入れただけであるので、無くてもよい。

次に、さっき作ったテンプレートを記事から取り込むようにする。

`_layouts/post.html` のいい感じのところに以下を追加した。

```
{% raw %}
{% include disqus_comments.html %}
{% endraw %}
```

設定はこれだけである。

あとはこれをコミットして GitHub にプッシュしたら、少ししてフォームが表示された。

### 3. メールアドレス認証

ただしコメントフォームの初回表示時には、メールアドレス認証のリンクと、
感情ボタンの有効/無効切り替えの設定が表示される。

メールアドレス認証のリンクを押すと Disqus 登録時に使った連絡先へメールが飛んでくる。
そのメール内で認証を実行すると設定完了となる。

感情ボタンは無効にしておいた。

### 4. まとめ

コメントフォームを表示できた。

SNS への共有ボタンが表示されているが、要らないので消せないかな。
スタイルシートで強引に非表示にするしかない?

と思ったらコメントフォームは IFrame だったので、
スタイルシートでの非表示設定はできなそう。


### 5. 参考

- [Install instructions for Jekyll \- Disqus Admin](https://fang-zhi-yan-suan-zi.disqus.com/admin/settings/jekyll/)
- [How to add comments to your Jekyll blog with Disqus](https://desiredpersona.com/disqus-comments-jekyll/)
